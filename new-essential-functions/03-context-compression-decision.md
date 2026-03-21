# 03 - 上下文压缩/摘要/保留区策略 架构决策

**技术点**: 上下文压缩/摘要/保留区策略 (Context Compression, Summarization & Preserve Zones)

**分析日期**: 2026-03-21

---

## 1. 已有实现

### 1.1 Aider (Python) — 递归摘要

**位置**: `reference-source-code/coding-agents/aider/aider/history.py`

**核心设计**:

```python
class ChatSummary:
    def __init__(self, models, max_tokens=1024):
        self.models = models
        self.max_tokens = max_tokens

    def summarize_real(self, messages, depth=0):
        """递归摘要算法"""
        sized = self.tokenize(messages)
        total = sum(tokens for tokens, _msg in sized)

        if total <= self.max_tokens and depth == 0:
            return messages

        # 分割点：保留尾部 (最近对话)
        split_index = len(messages)
        tail_tokens = 0
        half_max_tokens = self.max_tokens // 2

        for i in range(len(sized) - 1, -1, -1):
            tokens, _msg = sized[i]
            if tail_tokens + tokens < half_max_tokens:
                tail_tokens += tokens
                split_index = i
            else:
                break

        # 确保头部以 assistant 消息结束
        while messages[split_index - 1]["role"] != "assistant" and split_index > 1:
            split_index -= 1

        # 分割头部和尾部
        head = messages[:split_index]
        tail = messages[split_index:]

        # 摘要头部
        summary = self.summarize_all(head)

        # 递归检查
        summary_tokens = self.token_count(summary)
        tail_tokens = sum(tokens for tokens, _ in sized[split_index:])
        if summary_tokens + tail_tokens < self.max_tokens:
            return summary + tail

        # 递归
        return self.summarize_real(summary + tail, depth + 1)

    def summarize_all(self, messages):
        """使用 LLM 生成摘要"""
        content = ""
        for msg in messages:
            role = msg["role"].upper()
            content += f"# {role}\n{msg['content']}\n"

        summarize_messages = [
            dict(role="system", content=prompts.summarize),
            dict(role="user", content=content),
        ]

        # 多模型回退
        for model in self.models:
            try:
                summary = model.simple_send_with_retries(summarize_messages)
                if summary is not None:
                    return [dict(role="user", content=prompts.summary_prefix + summary)]
            except Exception as e:
                print(f"Summarization failed for model {model.name}: {e}")

        raise ValueError("All models failed")
```

**特点**:
- 递归摘要 (深度限制 3)
- 尾部保留策略 (最近对话优先)
- 多模型回退 (容错)
- 分割点智能调整 (确保 assistant 消息结尾)

---

### 1.2 CoPaw (Python) — ReMe 压缩 + 工具结果压缩

**位置**: `reference-source-code/personal-general-ai/CoPaw/src/copaw/agents/hooks/memory_compaction.py`

**核心设计**:

```python
class MemoryCompactionHook:
    """内存压缩钩子"""

    async def __call__(self, agent, kwargs):
        # 1. 获取配置
        agent_config = load_agent_config(self.memory_manager.agent_id)
        token_counter = get_copaw_token_counter(agent_config)

        # 2. 计算剩余压缩阈值
        system_prompt = agent.sys_prompt
        compressed_summary = memory.get_compressed_summary()
        str_token_count = await token_counter.count(
            messages=[],
            text=(system_prompt or "") + (compressed_summary or ""),
        )
        left_compact_threshold = (
            running_config.memory_compact_threshold - str_token_count
        )

        # 3. 压缩工具结果
        await self.memory_manager.compact_tool_result(
            messages=messages,
            recent_n=running_config.tool_result_compact_recent_n,
            old_threshold=running_config.tool_result_compact_old_threshold,
            recent_threshold=running_config.tool_result_compact_recent_threshold,
            retention_days=running_config.tool_result_compact_retention_days,
        )

        # 4. 检查上下文
        messages_to_compact, _, is_valid = await self.memory_manager.check_context(
            messages=messages,
            memory_compact_threshold=left_compact_threshold,
            memory_compact_reserve=running_config.memory_compact_reserve,
            as_token_counter=token_counter,
        )

        if messages_to_compact:
            # 执行压缩
            await self._print_status_message(agent, "Compacting memory...")
            await self.memory_manager.compact_memory(
                messages=messages,
                token_counter=token_counter,
            )
```

**配置参数**:
```python
# 内存压缩阈值 (默认 128K)
memory_compact_threshold = 128 * 1024

# 保留最近 N 轮
memory_compact_keep_recent = 5

# 工具结果压缩
tool_result_compact_recent_n = 5
tool_result_compact_old_threshold = 0.5  # 旧结果保留 50%
tool_result_compact_recent_threshold = 0.8  # 新结果保留 80%
```

**特点**:
- 双层压缩 (内存 + 工具结果)
- 可配置保留比例
- ReMe 向量存储集成
- 热重载配置

---

### 1.3 OpenClaw (TypeScript) — 自动压缩 + 手动压缩

**位置**: `reference-source-code/personal-general-ai/openclaw/docs/concepts/compaction.md`

**核心设计**:

```typescript
// 配置
interface CompactionConfig {
  mode: "auto" | "manual";
  targetTokens: number;
  model?: string;  // 可选：使用不同模型压缩
  identifierPolicy: "strict" | "off" | "custom";
  identifierInstructions?: string;
}

// 自动压缩触发条件
// 当会话接近上下文窗口限制时触发
// 1. 检测 token 超限
// 2. 可选：静默内存 flush (存储到 MEMORY.md)
// 3. 执行压缩摘要
// 4. 重试原请求

// 手动压缩
/compact Focus on decisions and open questions
```

**压缩流程**:
```
原始消息 → 提取可压缩部分 → LLM 摘要 → 替换为摘要消息 → 保留最近消息
```

**特点**:
- 自动 + 手动双模式
- 可指定压缩模型 (如用轻量模型做摘要)
- 标识符保留策略 (strict/off/custom)
- 插件化上下文引擎支持

---

### 1.4 IronClaw (Rust) — 三策略压缩 (已在 02 分析)

**位置**: `reference-source-code/personal-general-ai/ironclaw/src/agent/compaction.rs`

**核心设计**:

```rust
pub enum CompactionStrategy {
    Summarize { keep_recent: usize },  // 摘要 + 保留最近
    Truncate { keep_recent: usize },   // 直接截断
    MoveToWorkspace,                   // 移动到工作区
}
```

**摘要 Prompt**:
```
Summarize the following conversation concisely. Focus on:
- Key decisions made
- Important information exchanged
- Actions taken
- Outcomes achieved

Be brief but capture all important details. Use bullet points.
```

---

### 1.5 Gemini-CLI (TypeScript) — 两阶段压缩 + 探测验证

**位置**: `reference-source-code/coding-agents/gemini-cli/packages/core/src/services/contextManager.ts`

**核心设计**:

```typescript
// 两阶段压缩
// 1. 第一阶段：粗粒度压缩 (快速)
// 2. 第二阶段：细粒度压缩 (精确)
// 3. 探测验证：确保压缩后上下文可用

class ContextManager {
  // 分层记忆 (全局/扩展/项目)
  private globalMemory: string = '';
  private extensionMemory: string = '';
  private projectMemory: string = '';

  // JIT 按需加载
  async discoverContext(accessedPath: string, trustedRoots: string[]): Promise<string> {
    // 从访问路径向上遍历到项目根
    // 加载沿途的 GEMINI.md 文件
  }
}
```

---

## 2. 值得直接复用

### 2.1 Aider 递归摘要算法 ⭐⭐⭐⭐⭐

**推荐指数**: 5/5

**理由**:
- 递归处理超长对话
- 尾部保留策略 (保留最近上下文)
- 智能分割点调整
- 多模型回退容错

**复用方式**:
```python
# 直接复用 summarize_real 算法
# 调整 max_tokens 和 depth 限制
```

---

### 2.2 CoPaw 工具结果压缩 ⭐⭐⭐⭐⭐

**推荐指数**: 5/5

**理由**:
- 工具结果往往占用大量 token
- 可配置保留比例 (旧 50% / 新 80%)
- 与主上下文压缩分离

**复用方式**:
```python
# 实现 compact_tool_result 方法
# 配置 recent_n, old_threshold, recent_threshold
```

---

### 2.3 OpenClaw 自动压缩触发 ⭐⭐⭐⭐

**推荐指数**: 4/5

**理由**:
- 自动检测上下文窗口
- 静默 flush 到 MEMORY.md
- 可指定压缩模型

**复用方式**:
```typescript
// 实现自动检测逻辑
// 配置 targetTokens 和触发阈值
```

---

### 2.4 IronClaw 三策略选择 ⭐⭐⭐⭐

**推荐指数**: 4/5

**理由**:
- 根据超限程度选择策略
- 危急时激进截断
- 正常时摘要保留

---

## 3. 需要垂直特化

### 3.1 保留区 (Preserve Zones)

**通用实现**: 保留最近 N 轮

**垂直特化需求**:

| 领域 | 必须保留内容 | 压缩策略 |
|------|-------------|----------|
| **医疗** | 诊断结论、处方、过敏史、检查报告 | 永久保留，不压缩 |
| **法律** | 案件编号、关键证据、法条引用、判决结果 | 摘要保留引用 |
| **金融** | 交易指令、账户信息、风险评估 | 结构化存储 |
| **教育** | 知识点、错题、学习进度 | 提取为知识卡片 |
| **客服** | 订单号、问题分类、解决方案 | 压缩为工单摘要 |

**特化方案**:
```rust
pub struct PreserveZone {
    pub name: String,
    pub trigger_keywords: Vec<String>,     // 触发保留的关键词
    pub trigger_patterns: Vec<Regex>,      // 正则匹配模式
    pub retention_policy: RetentionPolicy,
}

pub enum RetentionPolicy {
    Permanent,              // 永久保留
    SummarizeOnly,          // 仅摘要
    StructuredExtract,      // 结构化提取
    CompressToTicket,       // 压缩为工单
}

// 医疗示例
let diagnosis_zone = PreserveZone {
    name: "diagnosis".to_string(),
    trigger_keywords: vec!["诊断", "处方", "过敏", "检查报告"],
    trigger_patterns: vec![
        Regex::new(r"诊断[：:]\s*(.+)")?,
        Regex::new(r"处方[：:]\s*(.+)")?,
    ],
    retention_policy: RetentionPolicy::Permanent,
};
```

---

### 3.2 多模态压缩

**通用实现**: 仅文本

**垂直特化需求**:
- 医疗：医学影像 (X 光/CT)、检验报告
- 教育：公式、图表、代码片段
- 法律：合同扫描件、证据照片

**特化方案**:
```typescript
type CompressibleItem =
  | { type: 'text'; content: string; compression: 'summarize' | 'truncate' }
  | { type: 'image'; url: string; compression: 'caption' | 'ocr' | 'thumbnail' }
  | { type: 'document'; path: string; compression: 'summary' | 'key_points' }
  | { type: 'structured'; schema: string; data: any; compression: 'condense' };

class MultiModalCompressor {
  async compress(item: CompressibleItem): Promise<CompressedItem> {
    switch (item.type) {
      case 'text':
        return this.compressText(item);
      case 'image':
        return this.compressImage(item);  // 生成 caption 或 OCR
      case 'document':
        return this.compressDocument(item);  // 提取关键信息
      case 'structured':
        return this.compressStructured(item);  // 数据压缩
    }
  }
}
```

---

### 3.3 合规性审计

**通用实现**: 无审计

**垂直特化需求**:
- 医疗：HIPAA 合规 (访问日志)
- 金融：交易审计 (完整记录)
- 法律：证据链 (不可篡改)

**特化方案**:
```rust
pub struct CompressionAudit {
    pub timestamp: i64,
    pub original_tokens: usize,
    pub compressed_tokens: usize,
    pub strategy: CompactionStrategy,
    pub preserved_zones: Vec<String>,
    pub hash_before: String,  // 原始内容哈希
    pub hash_after: String,   // 压缩后哈希
}

impl CompressionAudit {
    pub fn verify_integrity(&self) -> bool {
        // 验证压缩前后数据完整性
        // 医疗/法律场景：确保证据链完整
    }
}
```

---

## 4. 推荐数据结构/接口/模块边界

### 4.1 核心数据结构

```rust
// 压缩策略
#[derive(Clone, Debug)]
pub enum CompressionStrategy {
    Truncate { keep_recent: usize },
    Summarize { keep_recent: usize, model: Option<String> },
    MoveToStorage { storage: StorageBackend },
}

// 压缩结果
#[derive(Clone, Debug)]
pub struct CompressionResult {
    pub original_tokens: usize,
    pub compressed_tokens: usize,
    pub removed_messages: Vec<Message>,
    pub summary: Option<Message>,
    pub preserved_zones: Vec<PreserveZone>,
    pub audit_log: CompressionAudit,
}

// 工具结果压缩配置
#[derive(Clone, Debug)]
pub struct ToolResultCompressionConfig {
    pub recent_n: usize,              // 保留最近 N 个
    pub old_threshold: f32,           // 旧结果保留比例 (0.0-1.0)
    pub recent_threshold: f32,        // 新结果保留比例
    pub retention_days: i32,          // 保留天数
}

// 压缩器配置
#[derive(Clone, Debug)]
pub struct CompressorConfig {
    pub max_tokens: usize,
    pub trigger_threshold: f32,       // 触发阈值 (0.8 = 80%)
    pub critical_threshold: f32,      // 危急阈值 (0.95 = 95%)
    pub default_strategy: CompressionStrategy,
    pub tool_result_config: ToolResultCompressionConfig,
    pub preserve_zones: Vec<PreserveZone>,
    pub enable_audit: bool,
}
```

### 4.2 核心接口

```rust
// 压缩器
#[async_trait]
pub trait Compressor: Send + Sync {
    // 判断是否需要压缩
    fn needs_compression(&self, tokens: usize) -> CompressionNeeded;

    // 推荐压缩策略
    fn suggest_strategy(&self, tokens: usize) -> CompressionStrategy;

    // 执行压缩
    async fn compress(&self, context: &mut Context) -> Result<CompressionResult>;

    // 压缩工具结果
    async fn compress_tool_results(
        &self,
        messages: &mut Vec<Message>,
        config: &ToolResultCompressionConfig,
    ) -> Result<usize>;  // 返回压缩的 token 数
}

// 摘要生成器
#[async_trait]
pub trait Summarizer: Send + Sync {
    // 生成摘要
    async fn summarize(&self, messages: &[Message]) -> Result<String>;

    // 批量摘要 (用于递归)
    async fn summarize_batch(&self, batches: Vec<&[Message]>) -> Result<Vec<String>>;
}

// 保留区管理器
pub trait PreserveZoneManager: Send + Sync {
    // 检测消息是否属于保留区
    fn detect_zones(&self, message: &Message) -> Vec<String>;

    // 获取保留区的消息
    fn get_preserved_messages(&self, zone_name: &str) -> Vec<Message>;

    // 配置保留区
    fn configure_zone(&mut self, zone: PreserveZone);
}
```

### 4.3 模块边界

```
compression/
├── mod.rs
├── core/                      # 核心压缩
│   ├── mod.rs
│   ├── compressor.rs          # 主压缩器
│   ├── strategies.rs          # 策略实现
│   └── config.rs              # 配置
├── summarization/             # 摘要生成
│   ├── mod.rs
│   ├── llm_summarizer.rs      # LLM 摘要
│   ├── recursive.rs           # 递归摘要 (Aider)
│   └── prompts.rs             # Prompt 模板
├── tool_results/              # 工具结果压缩
│   ├── mod.rs
│   ├── compactor.rs           # 工具结果压缩器
│   └── config.rs              # 配置
├── preserve_zones/            # 保留区
│   ├── mod.rs
│   ├── manager.rs             # 保留区管理器
│   ├── detector.rs            # 检测器
│   └── policies.rs            # 保留策略
├── multimodal/                # 多模态压缩
│   ├── mod.rs
│   ├── image.rs               # 图片压缩
│   ├── document.rs            # 文档压缩
│   └── structured.rs          # 结构化数据
└── audit/                     # 审计
    ├── mod.rs
    ├── logger.rs
    └── verification.rs
```

---

## 5. MVP 最小实现方案

### 5.1 第一版目标

**范围**: 简单截断 + 基础摘要

**核心组件**:
1. Token 估算
2. 简单截断 (保留最近 N 轮)
3. 基础 LLM 摘要

### 5.2 MVP 代码结构

```python
# 最小可运行版本 (~400 行)

import json
from typing import List, Optional
from dataclasses import dataclass

@dataclass
class Message:
    role: str
    content: str

class SimpleCompressor:
    """简单压缩器"""

    def __init__(self, max_tokens: int = 4000, keep_recent: int = 5):
        self.max_tokens = max_tokens
        self.keep_recent = keep_recent
        self.tokens_per_word = 1.3

    def estimate_tokens(self, text: str) -> int:
        """Token 估算"""
        words = len(text.split())
        return int(words * self.tokens_per_word) + 4

    def needs_compression(self, messages: List[Message]) -> bool:
        """判断是否需要压缩"""
        total = sum(self.estimate_tokens(m.content) for m in messages)
        return total > self.max_tokens * 0.8  # 80% 阈值

    def truncate(self, messages: List[Message]) -> List[Message]:
        """简单截断"""
        if len(messages) <= self.keep_recent:
            return messages
        return messages[-self.keep_recent:]

    async def summarize(self, messages: List[Message], llm) -> str:
        """生成摘要"""
        content = "\n\n".join([
            f"{m.role.upper()}: {m.content}" for m in messages
        ])

        prompt = """Summarize the following conversation concisely.
Focus on key decisions and actions.

{content}

Summary:"""

        response = await llm.complete(prompt.format(content=content))
        return response

    async def compress(self, messages: List[Message], llm) -> List[Message]:
        """压缩消息"""
        if not self.needs_compression(messages):
            return messages

        # 分割：保留尾部 + 压缩头部
        split_idx = max(0, len(messages) - self.keep_recent)
        head = messages[:split_idx]
        tail = messages[split_idx:]

        if not head:
            return tail

        # 摘要头部
        summary = await self.summarize(head, llm)
        summary_msg = Message(role="system", content=f"[Summary] {summary}")

        return [summary_msg] + tail

# 使用示例
compressor = SimpleCompressor(max_tokens=4000, keep_recent=5)
messages = [
    Message("user", "Hello"),
    Message("assistant", "Hi!"),
    # ... 更多消息
]

compressed = await compressor.compress(messages, llm)
```

### 5.3 MVP 技术栈选择

| 组件 | 选择 | 理由 |
|------|------|------|
| 语言 | Python | 开发最快 |
| Token 估算 | 词数×1.3 | 无需 tokenizer |
| 压缩策略 | 截断 + 简单摘要 | 最简实现 |
| LLM | OpenAI GPT-3.5 | 成本低，速度快 |

---

## 6. 后续扩展路线

### 阶段一：基础增强 (1-2 周)

- [ ] 递归摘要 (Aider 算法)
- [ ] 工具结果压缩
- [ ] 多模型回退

### 阶段二：保留区 (2-3 周)

- [ ] 保留区检测
- [ ] 领域特定策略
- [ ] 配置热重载

### 阶段三：高级功能 (3-4 周)

- [ ] 多模态压缩
- [ ] 自动触发
- [ ] 审计日志

### 阶段四：生产就绪 (4-6 周)

- [ ] 压缩模型选择
- [ ] 性能优化
- [ ] 可视化调试

---

## 附录：参考源码位置

| 项目 | 核心文件 | 行数 |
|------|----------|------|
| Aider | `aider/history.py` | ~200 |
| CoPaw | `agents/hooks/memory_compaction.py` | ~150 |
| CoPaw | `agents/memory/memory_manager.py` | ~400 |
| OpenClaw | `docs/concepts/compaction.md` | ~100 |
| IronClaw | `src/agent/compaction.rs` | ~700 |

---

*文档版本：1.0*
*最后更新：2026-03-21*
