# 04 - Tool Registry 与结果后处理 架构决策

**技术点**: Tool Registry 与 Tool Result Post-Processing

**分析日期**: 2026-03-21

---

## 1. 已有实现

### 1.1 IronClaw (Rust) — Trait 驱动工具系统

**位置**: `reference-source-code/personal-general-ai/ironclaw/src/tools/tool.rs`

**核心设计**:

```rust
/// 工具 Trait
#[async_trait]
pub trait Tool: Send + Sync {
    fn name(&self) -> &str;
    fn description(&self) -> &str;
    fn parameters_schema(&self) -> serde_json::Value;
    
    async fn execute(&self, params: serde_json::Value, ctx: &JobContext) 
        -> Result<ToolOutput, ToolError>;
    
    fn requires_approval(&self, _params: &serde_json::Value) -> ApprovalRequirement;
    fn domain(&self) -> ToolDomain;
    fn rate_limit_config(&self) -> Option<ToolRateLimitConfig>;
}

pub enum ApprovalRequirement { Never, UnlessAutoApproved, Always }
pub enum ToolDomain { Orchestrator, Container }

pub struct ToolRegistry {
    tools: RwLock<HashMap<String, Arc<dyn Tool>>>,
}
```

**特点**: Trait 驱动、审批分级、执行域隔离、速率限制

---

### 1.2 OpenClaw (TypeScript) — 策略管道

**位置**: `reference-source-code/personal-general-ai/openclaw/src/agents/pi-tools.ts`

**核心设计**:

```typescript
export function applyToolPolicyPipeline(
  tools: AnyAgentTool[],
  steps: ToolPolicyPipelineStep[],
): AnyAgentTool[] {
  return steps.reduce((acc, step) => step(acc), tools);
}

// 工具包装器
export function wrapToolWithBeforeToolCallHook(
  tool: AnyAgentTool, hook: BeforeToolCallHook,
): AnyAgentTool {
  return {
    ...tool,
    execute: async (args, ctx) => {
      const result = await hook(tool.name, args, ctx);
      if (result.blocked) throw new ToolBlockedError(result.reason);
      return tool.execute(result.modifiedArgs || args, ctx);
    },
  };
}
```

**特点**: 策略管道、工具包装、多层级策略

---

### 1.3 Kimi-CLI (Python) — 装饰器注册

**位置**: `reference-source-code/coding-agents/kimi-cli/src/kimi_cli/tools/`

**核心设计**:

```python
@tool
def read_file(file_path: str, offset: int | None = None) -> str:
    """Read a file from disk."""
    ...

def get_tools() -> list[Callable]:
    return [obj for name, obj in globals().items() 
            if callable(obj) and hasattr(obj, "_is_tool")]
```

**特点**: 装饰器注册、自动发现、文档字符串 → Schema

---

### 1.4 Codex (Rust) — 结果截断

**位置**: `reference-source-code/coding-agents/codex-rs/core/src/truncate.rs`

```rust
pub enum TruncationStrategy {
    Head { max_chars: usize },
    Tail { max_chars: usize },
    Middle { max_chars: usize, head_percent: f64 },
}
```

**特点**: 三种截断策略、保留关键信息

---

## 2. 值得直接复用

| 项目 | 特性 | 推荐指数 |
|------|------|----------|
| IronClaw | Trait + 审批 + 域隔离 | ⭐⭐⭐⭐⭐ |
| OpenClaw | 策略管道 | ⭐⭐⭐⭐⭐ |
| Kimi-CLI | 装饰器注册 | ⭐⭐⭐⭐ |
| Codex | 结果截断 | ⭐⭐⭐⭐ |

---

## 3. 需要垂直特化

### 3.1 审批工作流

**通用**: Never/UnlessAutoApproved/Always

**垂直特化**:
- 医疗: 处方药 → 药师审批
- 金融: 大额交易 → 双重审批
- 法律: 合同签署 → 律所审批

### 3.2 结果后处理

**通用**: 截断/格式化

**垂直特化**:
- 医疗: 检验报告 → 结构化提取
- 金融: 交易结果 → 对账格式
- 法律: 法条引用 → 引用验证

---

## 4. 推荐接口

```rust
#[async_trait]
pub trait Tool: Send + Sync {
    fn name(&self) -> &str;
    fn description(&self) -> &str;
    fn parameters_schema(&self) -> Value;
    async fn execute(&self, params: Value, ctx: &Context) -> Result<ToolOutput>;
    fn requires_approval(&self, params: &Value) -> ApprovalRequirement;
}

pub trait ToolRegistry: Send + Sync {
    fn register(&self, tool: Arc<dyn Tool>);
    fn get(&self, name: &str) -> Option<Arc<dyn Tool>>;
    fn list(&self) -> Vec<Arc<dyn Tool>>;
}

pub trait ToolResultProcessor: Send + Sync {
    async fn process(&self, result: ToolOutput) -> ProcessedOutput;
}
```

---

## 5. MVP 方案

```python
class SimpleToolRegistry:
    def __init__(self):
        self._tools: dict[str, Callable] = {}
    
    def register(self, tool: Callable):
        self._tools[tool.__name__] = tool
    
    def get(self, name: str) -> Callable | None:
        return self._tools.get(name)
    
    def list(self) -> list[str]:
        return list(self._tools.keys())

# 使用
registry = SimpleToolRegistry()
registry.register(read_file)
registry.register(write_file)
```

---

## 6. 扩展路线

1. 审批工作流
2. 结果后处理管道
3. 速率限制
4. 执行域隔离

---

*文档版本：1.0*
*最后更新：2026-03-21*
