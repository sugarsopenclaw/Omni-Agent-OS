# 01 - Agent Runtime 主循环架构决策

**技术点**: Agent Runtime 主循环 (Agent Loop / Core Execution Engine)

**分析日期**: 2026-03-21

---

## 1. 已有实现

### 1.1 IronClaw (Rust) — LoopDelegate 模式

**位置**: `reference-source-code/personal-general-ai/ironclaw/src/agent/agentic_loop.rs`

**核心设计**:

```rust
// 统一的 LoopDelegate trait
#[async_trait]
pub trait LoopDelegate: Send + Sync {
    async fn check_signals(&self) -> LoopSignal;
    async fn before_llm_call(&self, ctx: &mut ReasoningContext, iter: usize) -> Option<LoopOutcome>;
    async fn call_llm(&self, reasoning: &Reasoning, ctx: &mut ReasoningContext, iter: usize) -> Result<RespondOutput, Error>;
    async fn handle_text_response(&self, text: &str, ctx: &mut ReasoningContext) -> TextAction;
    async fn execute_tool_calls(&self, calls: Vec<ToolCall>, content: Option<String>, ctx: &mut ReasoningContext) -> Result<Option<LoopOutcome>, Error>;
    async fn after_iteration(&self, iteration: usize) {}
}

// 统一循环实现
pub async fn run_agentic_loop<D: LoopDelegate>(
    delegate: &dyn LoopDelegate,
    reasoning: &Reasoning,
    reason_ctx: &mut ReasoningContext,
    config: &AgenticLoopConfig,
) -> Result<LoopOutcome, Error> {
    for iteration in 1..=config.max_iterations {
        // 1. 检查信号 (停止/取消/用户消息)
        // 2. 前置钩子 (成本保护/工具刷新)
        // 3. LLM 调用
        // 4. 处理响应 (文本 vs 工具调用)
        // 5. 后置钩子
    }
}
```

**特点**:
- 一个循环，三个委托实现 (ChatDelegate, JobDelegate, ContainerDelegate)
- 代码复用率极高，核心逻辑只写一次
- 支持交互式聊天、后台任务、容器隔离三种执行模式

---

### 1.2 OpenClaw (TypeScript) — Gateway Server Loop

**位置**: `reference-source-code/personal-general-ai/openclaw/src/agents/command/session.ts`

**核心设计**:

```typescript
// 会话解析与状态管理
export function resolveSession(opts: {
  cfg: OpenClawConfig;
  to?: string;
  sessionId?: string;
  sessionKey?: string;
  agentId?: string;
}): SessionResolution {
  // 1. 解析会话键 (基于发送者/通道/显式指定)
  // 2. 加载会话存储 (JSONL 文件)
  // 3. 评估会话新鲜度 (是否过期需要重置)
  // 4. 生成/复用会话 ID
}

// 嵌入式 PI Agent 运行器
// 位置：src/agents/pi-embedded-runner.ts
// - 订阅消息流
// - 构建上下文 (系统提示 + 历史 + 工具)
// - 调用 LLM
// - 执行工具
// - 发送回复
```

**特点**:
- 多通道网关架构 (20+ 通道)
- 会话状态持久化 (JSONL + 会话存储)
- 支持子 Agent 委派 (sessions_spawn)

---

### 1.3 Aider (Python) — 结对编程循环

**位置**: `reference-source-code/coding-agents/aider/aider/main.py`

**核心设计**:

```python
# 主循环结构
def main(argv=None, input=None, output=None, args=None):
    # 1. 解析命令行参数
    # 2. 初始化 Git 仓库
    # 3. 创建 Coder 实例
    # 4. 进入交互循环
    while True:
        user_input = io.get_input()
        if user_input is None:
            break
        coder.run(user_input)
```

**特点**:
- 终端交互式结对编程
- Git 集成 (自动 commit)
- 文件编辑格式 (SEARCH/REPLACE)

---

### 1.4 Kimi-CLI (TypeScript) — SPMC 消息总线

**位置**: `reference-source-code/coding-agents/kimi-cli/src/kimi_cli/`

**核心设计**:
- Single Producer, Multiple Consumer 架构
- 双队列设计 (输入队列 + 输出队列)
- 异步消息传递

---

### 1.5 PicoClaw (Go) — Worker Pool

**位置**: `reference-source-code/personal-general-ai/picoclaw/pkg/`

**核心设计**:
- Go routine worker pool
- 通道 (channel) 通信
- 轻量级并发模型

---

## 2. 值得直接复用

### 2.1 IronClaw LoopDelegate 模式 ⭐⭐⭐⭐⭐

**推荐指数**: 5/5

**理由**:
- 代码复用率最高 (一套循环，多种执行模式)
- 类型安全 (Rust trait 系统)
- 测试友好 (Mock delegate 单元测试)
- 扩展性强 (新增执行模式只需实现 trait)

**复用方式**:
```rust
// 直接采用其 trait 设计和循环结构
// 可以简化部分复杂逻辑 (如 cost guard)
```

---

### 2.2 OpenClaw 会话解析逻辑 ⭐⭐⭐⭐

**推荐指数**: 4/5

**理由**:
- 成熟的会话键派生算法
- 多通道会话隔离
- 会话新鲜度评估 (自动重置策略)

**复用方式**:
```typescript
// 提取 session.ts 中的核心函数
// - resolveSessionKeyForRequest
// - evaluateSessionFreshness
// - resolveSessionResetPolicy
```

---

### 2.3 Aider Git 集成 ⭐⭐⭐

**推荐指数**: 3/5

**理由**:
- 成熟的 Git 工作流集成
- 自动 commit 消息生成
- 适用于代码类 Agent

**复用方式**:
- 仅在需要代码编辑功能时引入
- 可作为可选模块

---

## 3. 需要垂直特化

### 3.1 上下文窗口管理

**通用实现**: 固定 token 限制 (如 128K)

**垂直特化需求**:
- 医疗/法律场景需要更长的上下文保留
- 教育场景需要知识点关联记忆
- 客服场景需要会话摘要压缩

**特化方案**:
```rust
pub struct ContextWindowConfig {
    pub max_tokens: usize,
    pub compression_threshold: f32,  // 70% 触发压缩
    pub preserve_zones: Vec<PreserveZone>,  // 保留区 (诊断结果/处方等)
    pub domain_specific_compressor: DomainCompressor,
}
```

---

### 3.2 工具执行安全

**通用实现**: 基础权限检查

**垂直特化需求**:
- 医疗：处方药调用需要二次确认
- 金融：交易操作需要多因素认证
- 法律：文件访问需要审计日志

**特化方案**:
```rust
pub enum ToolSafetyLevel {
    Normal,      // 普通工具
    Sensitive,   // 敏感工具 (记录日志)
    Critical,    // 关键工具 (需要审批)
}
```

---

### 3.3 错误恢复策略

**通用实现**: 重试 + 降级

**垂直特化需求**:
- 医疗：错误时需要明确告知用户"无法确定"
- 金融：错误时回滚事务
- 教育：错误时提供提示而非直接答案

---

## 4. 推荐数据结构/接口/模块边界

### 4.1 核心 Trait/Interface

```rust
// Agent Loop 核心接口
#[async_trait]
pub trait AgentLoop: Send + Sync {
    // 生命周期
    async fn initialize(&mut self) -> Result<()>;
    async fn run(&mut self, input: UserInput) -> Result<AgentOutput>;
    async fn shutdown(&mut self) -> Result<()>;
    
    // 状态查询
    fn status(&self) -> LoopStatus;
    fn capabilities(&self) -> Vec<Capability>;
}

// 上下文管理接口
pub trait ContextManager: Send + Sync {
    fn add_message(&mut self, msg: Message);
    fn get_context(&self, limit_tokens: usize) -> Vec<Message>;
    fn compress(&mut self, strategy: CompressionStrategy) -> Result<()>;
    fn preserve(&mut self, zone: PreserveZone);
}

// 工具执行接口
#[async_trait]
pub trait ToolExecutor: Send + Sync {
    async fn execute(&self, call: ToolCall) -> Result<ToolResult>;
    fn list_tools(&self) -> Vec<ToolDefinition>;
    fn safety_level(&self, tool_id: &str) -> ToolSafetyLevel;
}
```

### 4.2 模块边界

```
agent_runtime/
├── loop/                    # 核心循环
│   ├── mod.rs
│   ├── delegate.rs          # LoopDelegate trait
│   ├── state.rs             # 循环状态机
│   └── config.rs            # 循环配置
├── context/                 # 上下文管理
│   ├── mod.rs
│   ├── manager.rs           # ContextManager 实现
│   ├── compression.rs       # 压缩策略
│   └── preserve.rs          # 保留区管理
├── tools/                   # 工具执行
│   ├── mod.rs
│   ├── executor.rs          # ToolExecutor 实现
│   ├── registry.rs          # 工具注册表
│   └── safety.rs            # 安全检查
└── delegates/               # 委托实现
    ├── mod.rs
    ├── chat.rs              # 聊天模式
    ├── job.rs               # 后台任务
    └── container.rs         # 容器隔离
```

---

## 5. MVP 最小实现方案

### 5.1 第一版目标

**范围**: 单 Agent，交互式聊天，基础工具调用

**核心组件**:
1. 简化版 LoopDelegate (仅 ChatDelegate)
2. 基础上下文管理 (无压缩)
3. 工具注册与执行 (无安全分级)

### 5.2 MVP 代码结构

```rust
// 最小可运行版本 (~500 行)

pub struct SimpleAgent {
    llm: Box<dyn LLM>,
    tools: ToolRegistry,
    context: Vec<Message>,
    max_iterations: usize,
}

impl SimpleAgent {
    pub async fn run(&mut self, user_input: &str) -> Result<String> {
        self.context.push(Message::user(user_input));
        
        for _ in 0..self.max_iterations {
            // 1. 调用 LLM
            let response = self.llm.chat(&self.context).await?;
            
            // 2. 处理响应
            match response {
                RespondOutput::Text(text) => return Ok(text),
                RespondOutput::ToolCalls(calls) => {
                    for call in calls {
                        let result = self.tools.execute(call).await?;
                        self.context.push(Message::tool_result(result));
                    }
                }
            }
        }
        
        Err(Error::MaxIterationsReached)
    }
}
```

### 5.3 MVP 技术栈选择

| 组件 | 选择 | 理由 |
|------|------|------|
| 语言 | TypeScript | 开发效率高，生态丰富 |
| LLM 调用 | OpenAI SDK | 最简单，文档完善 |
| 工具注册 | 内存 Map | MVP 无需持久化 |
| 上下文管理 | 数组截断 | 暂不实现压缩 |

---

## 6. 后续扩展路线

### 阶段一：基础增强 (1-2 周)

- [ ] 实现上下文压缩 (摘要策略)
- [ ] 添加会话持久化 (JSONL)
- [ ] 工具安全分级

### 阶段二：多执行模式 (2-3 周)

- [ ] 实现 JobDelegate (后台任务)
- [ ] 实现 ContainerDelegate (容器隔离)
- [ ] 添加子 Agent 委派

### 阶段三：高级功能 (3-4 周)

- [ ] 多模型路由与故障转移
- [ ] 垂直领域特化 (医疗/金融/教育)
- [ ] 可观测性 (tracing/replay)

### 阶段四：生产就绪 (4-6 周)

- [ ] 性能优化 (并发/缓存)
- [ ] 多通道适配 (飞书/钉钉/微信)
- [ ] 插件系统

---

## 附录：参考源码位置

| 项目 | 核心文件 | 行数 |
|------|----------|------|
| IronClaw | `src/agent/agentic_loop.rs` | ~550 |
| OpenClaw | `src/agents/command/session.ts` | ~200 |
| OpenClaw | `src/agents/pi-embedded-runner.ts` | ~800 |
| Aider | `aider/main.py` | ~1300 |
| Kimi-CLI | `src/kimi_cli/agent/` | ~500 |

---

*文档版本：1.0*
*最后更新：2026-03-21*
