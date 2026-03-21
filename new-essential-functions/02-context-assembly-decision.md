# 02 - Context Assembly Pipeline 架构决策

**技术点**: 上下文组装管线 (Context Assembly Pipeline)

**分析日期**: 2026-03-21

---

## 1. 已有实现

### 1.1 Kimi-CLI (Python) — D-Mail 检查点系统

**位置**: `reference-source-code/coding-agents/kimi-cli/src/kimi_cli/soul/context.py`

**核心设计**:

```python
class Context:
    def __init__(self, file_backend: Path):
        self._file_backend = file_backend      # JSONL 文件后端
        self._history: list[Message] = []      # 消息历史
        self._token_count: int = 0             # Token 计数
        self._next_checkpoint_id: int = 0      # 检查点 ID
        self._system_prompt: str | None = None

    # 检查点机制 (Steins;Gate 风格)
    async def checkpoint(self, add_user_message: bool):
        checkpoint_id = self._next_checkpoint_id
        self._next_checkpoint_id += 1
        # 写入检查点标记到 JSONL
        await f.write(json.dumps({"role": "_checkpoint", "id": checkpoint_id}) + "\n")

    # 回滚到指定检查点
    async def revert_to(self, checkpoint_id: int):
        # 1. 旋转当前文件 (备份)
        # 2. 从备份恢复至指定检查点
        # 3. 重建历史、token 计数、系统提示

    # 清空上下文
    async def clear(self):
        # 旋转文件 + 重置状态
```

**特点**:
- JSONL 流式存储 (逐行追加)
- 检查点元数据 (_checkpoint, _system_prompt, _usage)
- 文件旋转备份机制 (防止数据丢失)
- 支持回滚到任意检查点

---

### 1.2 IronClaw (Rust) — 三策略压缩器

**位置**: `reference-source-code/personal-general-ai/ironclaw/src/agent/compaction.rs`

**核心设计**:

```rust
pub enum CompactionStrategy {
    /// 摘要旧对话，保留最近 N 轮
    Summarize { keep_recent: usize },

    /// 简单截断 (无摘要)
    Truncate { keep_recent: usize },

    /// 移动到工作区存储 (不删除)
    MoveToWorkspace,
}

pub struct ContextCompactor {
    llm: Arc<dyn LlmProvider>,
}

impl ContextCompactor {
    pub async fn compact(
        &self,
        thread: &mut Thread,
        strategy: CompactionStrategy,
        workspace: Option<&Workspace>,
    ) -> Result<CompactionResult, Error> {
        match strategy {
            CompactionStrategy::Summarize { keep_recent } => {
                // 1. 提取旧对话
                // 2. 调用 LLM 生成摘要
                // 3. 写入工作区日志
                // 4. 截断对话
            }
            CompactionStrategy::Truncate { keep_recent } => {
                // 直接截断
            }
            CompactionStrategy::MoveToWorkspace => {
                // 1. 格式化旧对话
                // 2. 写入工作区文件
                // 3. 保留更多轮次 (有备份)
            }
        }
    }
}
```

**特点**:
- 三种压缩策略可选
- LLM 生成摘要 (高质量压缩)
- 工作区备份 (可追溯)
- 失败保护 (写入失败保留原对话)

---

### 1.3 IronClaw (Rust) — 上下文监控器

**位置**: `reference-source-code/personal-general-ai/ironclaw/src/agent/context_monitor.rs`

**核心设计**:

```rust
pub struct ContextMonitor {
    context_limit: usize,        // 上下文限制 (默认 100K)
    threshold_ratio: f64,        // 触发阈值 (默认 80%)
}

impl ContextMonitor {
    // 评估是否需要压缩
    pub fn needs_compaction(&self, messages: &[ChatMessage]) -> bool {
        let tokens = self.estimate_tokens(messages);
        let threshold = (self.context_limit as f64 * self.threshold_ratio) as usize;
        tokens >= threshold
    }

    // 智能推荐压缩策略
    pub fn suggest_compaction(&self, messages: &[ChatMessage]) -> Option<CompactionStrategy> {
        let tokens = self.estimate_tokens(messages);
        let overage = tokens as f64 / self.context_limit as f64;

        if overage > 0.95 {
            // 危急：激进截断 (保留 3 轮)
            Some(CompactionStrategy::Truncate { keep_recent: 3 })
        } else if overage > 0.85 {
            // 高：摘要 + 保留较少 (5 轮)
            Some(CompactionStrategy::Summarize { keep_recent: 5 })
        } else {
            // 中等：移动到工作区
            Some(CompactionStrategy::MoveToWorkspace)
        }
    }

    // Token 估算 (基于词数)
    fn estimate_message_tokens(message: &ChatMessage) -> usize {
        let word_count = message.content.split_whitespace().count();
        let overhead = 4;  // 角色和结构开销
        (word_count as f64 * TOKENS_PER_WORD) as usize + overhead
    }
}

const TOKENS_PER_WORD: f64 = 1.3;  // 1.3 tokens/word
```

**特点**:
- 三级预警机制 (中等/高/危急)
- 动态策略推荐
- 词数估算 (1.3 tokens/word)

---

### 1.4 OpenClaw (TypeScript) — 运行时上下文构建器

**位置**: `reference-source-code/personal-general-ai/openclaw/src/agents/pi-embedded-runner/compaction-runtime-context.ts`

**核心设计**:

```typescript
export type EmbeddedCompactionRuntimeContext = {
  // 会话标识
  sessionKey?: string;
  messageChannel?: string;
  messageProvider?: string;

  // 通道上下文
  currentChannelId?: string;
  currentThreadTs?: string;
  currentMessageId?: string | number;

  // 认证与权限
  authProfileId?: string;
  senderIsOwner?: boolean;
  senderId?: string;

  // 模型配置
  provider?: string;
  model?: string;
  thinkLevel?: ThinkLevel;
  reasoningLevel?: ReasoningLevel;

  // 工作区
  workspaceDir: string;
  agentDir: string;

  // 扩展
  config?: OpenClawConfig;
  skillsSnapshot?: SkillSnapshot;
  extraSystemPrompt?: string;
  ownerNumbers?: string[];
};
```

**特点**:
- 丰富的运行时元数据
- 多通道上下文追踪
- 权限感知 (senderIsOwner)
- Skills 快照集成

---

### 1.5 Gemini-CLI (TypeScript) — 分层记忆系统

**位置**: `reference-source-code/coding-agents/gemini-cli/packages/core/src/services/contextManager.ts`

**核心设计**:

```typescript
export class ContextManager {
  private loadedPaths: Set<string> = new Set();
  private globalMemory: string = '';
  private extensionMemory: string = '';
  private projectMemory: string = '';

  // 刷新记忆 (重新加载)
  async refresh(): Promise<void> {
    const paths = await this.discoverMemoryPaths();
    const contentsMap = await this.loadMemoryContents(paths);
    this.categorizeMemoryContents(paths, contentsMap);
    this.emitMemoryChanged();
  }

  // 发现记忆路径
  private async discoverMemoryPaths() {
    const [global, extension, project] = await Promise.all([
      getGlobalMemoryPaths(),           // 全局记忆 (~/.gemini/)
      getExtensionMemoryPaths(loader),  // 扩展记忆
      getEnvironmentMemoryPaths(dirs),  // 项目记忆 (GEMINI.md)
    ]);
    return { global, extension, project };
  }

  // JIT 上下文发现 (按需加载)
  async discoverContext(accessedPath: string, trustedRoots: string[]): Promise<string> {
    // 从访问路径向上遍历到项目根
    // 加载沿途的 GEMINI.md 文件
    // 返回拼接的指令
  }
}
```

**特点**:
- 三层记忆 (全局/扩展/项目)
- JIT 按需加载 (减少初始开销)
- 文件去重 (处理大小写不敏感文件系统)
- 事件驱动 (MemoryChanged 事件)

---

## 2. 值得直接复用

### 2.1 Kimi-CLI D-Mail 检查点系统 ⭐⭐⭐⭐⭐

**推荐指数**: 5/5

**理由**:
- 设计优雅 (Steins;Gate 风格命名)
- JSONL 流式存储 (高效追加)
- 完整回滚机制
- 文件旋转备份 (安全)

**复用方式**:
```python
# 直接采用其 JSONL 格式和检查点设计
# 可以简化为纯 Python 实现 (无需异步)
```

**JSONL 格式示例**:
```json
{"role": "_system_prompt", "content": "You are a helpful assistant."}
{"role": "user", "content": [{"type": "text", "text": "Hello"}]}
{"role": "assistant", "content": [{"type": "text", "text": "Hi!"}]}
{"role": "_checkpoint", "id": 0}
{"role": "_usage", "token_count": 1234}
```

---

### 2.2 IronClaw 三策略压缩器 ⭐⭐⭐⭐⭐

**推荐指数**: 5/5

**理由**:
- 三种策略覆盖不同场景
- 智能策略推荐 (基于超限程度)
- LLM 摘要 (高质量压缩)
- 失败保护机制

**复用方式**:
```rust
# 直接采用 CompactionStrategy 枚举
# 复用 ContextMonitor 的阈值判断逻辑
```

---

### 2.3 Gemini-CLI 分层记忆 ⭐⭐⭐⭐

**推荐指数**: 4/5

**理由**:
- 清晰的三层架构
- JIT 按需加载 (性能优化)
- 文件去重处理

**复用方式**:
```typescript
// 采用三层记忆设计
// 简化 JIT 逻辑 (仅项目根目录)
```

---

### 2.4 IronClaw Token 估算 ⭐⭐⭐⭐

**推荐指数**: 4/5

**理由**:
- 词数估算 (1.3 tokens/word)
- 简单高效 (无需加载 tokenizer)
- 误差可接受 (~10%)

**复用方式**:
```rust
const TOKENS_PER_WORD: f64 = 1.3;

fn estimate_tokens(text: &str) -> usize {
    let word_count = text.split_whitespace().count();
    (word_count as f64 * TOKENS_PER_WORD) as usize + 4
}
```

---

## 3. 需要垂直特化

### 3.1 领域特定保留区

**通用实现**: 保留最近 N 轮

**垂直特化需求**:

| 领域 | 保留内容 | 压缩策略 |
|------|----------|----------|
| **医疗** | 诊断结果、处方、过敏史 | 永久保留，不压缩 |
| **法律** | 案件编号、关键证据、法条引用 | 摘要保留引用 |
| **教育** | 知识点、错题记录 | 结构化存储 |
| **客服** | 订单号、问题分类 | 压缩为工单摘要 |

**特化方案**:
```rust
pub struct PreserveZone {
    pub name: String,
    pub trigger_keywords: Vec<String>,
    pub retention_policy: RetentionPolicy,
}

pub enum RetentionPolicy {
    Permanent,              // 永久保留 (医疗诊断)
    SummarizeOnly,          // 仅摘要 (法律引用)
    StructuredExtract,      // 结构化提取 (教育知识点)
    CompressToTicket,       // 压缩为工单 (客服)
}
```

---

### 3.2 多模态上下文

**通用实现**: 文本为主

**垂直特化需求**:
- 医疗：医学影像 (X 光/CT)、检验报告
- 教育：公式、图表、代码片段
- 法律：合同扫描件、证据照片

**特化方案**:
```typescript
type ContextItem =
  | { type: 'text'; content: string }
  | { type: 'image'; url: string; caption: string; ocr_text?: string }
  | { type: 'document'; path: string; summary: string; key_points: string[] }
  | { type: 'structured'; schema: string; data: Record<string, any> };

class ContextAssemblyPipeline {
  assemble(items: ContextItem[]): AssembledContext {
    // 1. 分类处理
    // 2. 提取文本 (OCR/解析)
    // 3. 生成摘要
    // 4. 按优先级排序
    // 5. Token 预算分配
  }
}
```

---

### 3.3 合规性审计

**通用实现**: 无审计日志

**垂直特化需求**:
- 医疗：HIPAA 合规 (访问日志)
- 金融：交易审计 (完整记录)
- 法律：证据链 (不可篡改)

**特化方案**:
```rust
pub struct AuditLogger {
    log_path: PathBuf,
}

impl AuditLogger {
    pub fn log_context_access(&self, context: &Context) {
        // 记录：时间、用户、访问的上下文片段
        // 医疗场景：记录患者 ID、访问原因
    }

    pub fn log_compaction(&self, result: &CompactionResult) {
        // 记录：压缩前后的 token 数、删除的内容哈希
    }
}
```

---

## 4. 推荐数据结构/接口/模块边界

### 4.1 核心数据结构

```rust
// 上下文项 (支持多模态)
#[derive(Clone, Debug)]
pub enum ContextItem {
    Text(String),
    Image { url: String, caption: String, ocr: Option<String> },
    Document { path: String, summary: String, key_points: Vec<String> },
    Structured { schema: String, data: serde_json::Value },
}

// 消息 (兼容 OpenAI 格式)
#[derive(Clone, Debug)]
pub struct Message {
    pub role: MessageRole,  // user/assistant/system/tool
    pub content: Vec<ContextItem>,
    pub tool_call_id: Option<String>,
    pub tool_calls: Option<Vec<ToolCall>>,
    pub metadata: MessageMetadata,
}

// 检查点
#[derive(Clone, Debug)]
pub struct Checkpoint {
    pub id: usize,
    pub timestamp: i64,
    pub token_count: usize,
    pub preserved_zones: Vec<PreserveZone>,
}

// 上下文组装结果
pub struct AssembledContext {
    pub system_prompt: String,
    pub messages: Vec<Message>,
    pub checkpoints: Vec<Checkpoint>,
    pub total_tokens: usize,
    pub preserve_zones: Vec<PreserveZone>,
    pub metadata: ContextMetadata,
}
```

### 4.2 核心接口

```rust
// 上下文组装管线
#[async_trait]
pub trait ContextAssemblyPipeline: Send + Sync {
    // 组装上下文
    async fn assemble(&self, input: AssemblyInput) -> Result<AssembledContext>;

    // 添加消息
    async fn add_message(&mut self, msg: Message) -> Result<()>;

    // 获取组装后的上下文 (用于 LLM 调用)
    fn get_context(&self, limit_tokens: usize) -> Vec<Message>;

    // Token 计数
    fn token_count(&self) -> usize;
}

// 压缩策略
#[async_trait]
pub trait CompactionStrategy: Send + Sync {
    // 判断是否需要压缩
    fn needs_compaction(&self, tokens: usize, limit: usize) -> bool;

    // 推荐压缩参数
    fn suggest_params(&self, tokens: usize, limit: usize) -> CompactionParams;

    // 执行压缩
    async fn compact(&self, context: &mut AssembledContext) -> Result<CompactionResult>;
}

// 检查点管理
#[async_trait]
pub trait CheckpointManager: Send + Sync {
    // 创建检查点
    async fn create(&mut self) -> Result<Checkpoint>;

    // 回滚到检查点
    async fn revert_to(&mut self, id: usize) -> Result<()>;

    // 列出检查点
    fn list(&self) -> Vec<Checkpoint>;

    // 清除检查点
    async fn clear(&mut self) -> Result<()>;
}
```

### 4.3 模块边界

```
context_assembly/
├── mod.rs
├── pipeline/                  # 组装管线
│   ├── mod.rs
│   ├── assembler.rs           # 主组装器
│   ├── builder.rs             # 构建器模式
│   └── validator.rs           # 上下文验证
├── items/                     # 上下文项
│   ├── mod.rs
│   ├── text.rs
│   ├── image.rs
│   ├── document.rs
│   └── structured.rs
├── compression/               # 压缩
│   ├── mod.rs
│   ├── strategies.rs          # 三策略实现
│   ├── monitor.rs             # 上下文监控
│   └── preserve_zones.rs      # 保留区管理
├── checkpoint/                # 检查点
│   ├── mod.rs
│   ├── manager.rs
│   ├── jsonl_store.rs         # JSONL 存储
│   └── rotation.rs            # 文件旋转
├── memory/                    # 分层记忆
│   ├── mod.rs
│   ├── global.rs
│   ├── project.rs
│   └── jit_loader.rs          # JIT 加载
└── audit/                     # 审计 (可选)
    ├── mod.rs
    └── logger.rs
```

---

## 5. MVP 最小实现方案

### 5.1 第一版目标

**范围**: 单轮对话 + 基础历史 + JSONL 存储

**核心组件**:
1. JSONL 上下文存储
2. 基础 Token 估算
3. 简单截断压缩

### 5.2 MVP 代码结构

```python
# 最小可运行版本 (~300 行)

import json
from pathlib import Path
from dataclasses import dataclass, field
from typing import List, Optional

@dataclass
class Message:
    role: str  # user/assistant/system
    content: str

@dataclass
class Context:
    file_path: Path
    messages: List[Message] = field(default_factory=list)
    token_count: int = 0

    def __post_init__(self):
        self.load()

    def load(self):
        """从 JSONL 文件加载"""
        if not self.file_path.exists():
            return
        with open(self.file_path) as f:
            for line in f:
                if not line.strip():
                    continue
                data = json.loads(line)
                if data['role'] == '_usage':
                    self.token_count = data['token_count']
                elif data['role'] in ('user', 'assistant', 'system'):
                    self.messages.append(Message(data['role'], data['content']))

    def save(self):
        """保存到 JSONL 文件"""
        with open(self.file_path, 'w') as f:
            for msg in self.messages:
                f.write(json.dumps({'role': msg.role, 'content': msg.content}) + '\n')
            f.write(json.dumps({'role': '_usage', 'token_count': self.token_count}) + '\n')

    def add(self, role: str, content: str):
        """添加消息"""
        self.messages.append(Message(role, content))
        self.token_count = self.estimate_tokens()
        self.save()

    def estimate_tokens(self) -> int:
        """Token 估算 (1.3 tokens/word)"""
        total = 0
        for msg in self.messages:
            words = len(msg.content.split())
            total += int(words * 1.3) + 4
        return total

    def truncate(self, keep_recent: int = 5):
        """截断保留最近 N 轮"""
        if len(self.messages) > keep_recent:
            self.messages = self.messages[-keep_recent:]
            self.token_count = self.estimate_tokens()
            self.save()

# 使用示例
ctx = Context(Path('session.jsonl'))
ctx.add('user', 'Hello')
ctx.add('assistant', 'Hi! How can I help?')
print(f'Tokens: {ctx.token_count}')

if ctx.token_count > 1000:
    ctx.truncate(keep_recent=3)
```

### 5.3 MVP 技术栈选择

| 组件 | 选择 | 理由 |
|------|------|------|
| 语言 | Python | 开发最快，原型验证 |
| 存储 | JSONL | 简单，流式，易调试 |
| Token 估算 | 词数×1.3 | 无需 tokenizer |
| 压缩 | 截断 | 最简单 |

---

## 6. 后续扩展路线

### 阶段一：基础增强 (1-2 周)

- [ ] LLM 摘要压缩 (采用 IronClaw 策略)
- [ ] 检查点系统 (采用 Kimi-CLI D-Mail)
- [ ] 文件旋转备份

### 阶段二：分层记忆 (2-3 周)

- [ ] 全局/项目记忆 (采用 Gemini-CLI)
- [ ] JIT 按需加载
- [ ] 记忆去重

### 阶段三：垂直特化 (3-4 周)

- [ ] 保留区管理 (医疗/法律/教育)
- [ ] 多模态支持 (图片/文档)
- [ ] 审计日志

### 阶段四：高级功能 (4-6 周)

- [ ] CJK 感知 Token 估算 (采用 PicoClaw)
- [ ] 动态策略推荐 (采用 IronClaw Monitor)
- [ ] 上下文可视化调试工具

---

## 附录：参考源码位置

| 项目 | 核心文件 | 行数 |
|------|----------|------|
| Kimi-CLI | `src/kimi_cli/soul/context.py` | ~350 |
| IronClaw | `src/agent/compaction.rs` | ~700 |
| IronClaw | `src/agent/context_monitor.rs` | ~250 |
| OpenClaw | `src/agents/pi-embedded-runner/compaction-runtime-context.ts` | ~80 |
| Gemini-CLI | `packages/core/src/services/contextManager.ts` | ~200 |

---

*文档版本：1.0*
*最后更新：2026-03-21*
