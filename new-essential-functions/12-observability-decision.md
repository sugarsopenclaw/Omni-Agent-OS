# 12 - Observability/Tracing/Replay 架构决策

**技术点**: Observability, Tracing, Replay

**分析日期**: 2026-03-21

---

## 1. 已有实现

### 1.1 IronClaw (Rust) — TraceLlm + Recording

**位置**: `reference-source-code/personal-general-ai/ironclaw/tests/support/trace_llm.rs`

**核心设计**:

```rust
/// LLM Trace 结构
#[derive(Debug, Clone, Serialize)]
pub struct LlmTrace {
    pub model_name: String,
    pub turns: Vec<TraceTurn>,
    pub memory_snapshot: Vec<MemorySnapshotEntry>,
    pub http_exchanges: Vec<HttpExchange>,
    pub expects: TraceExpects,
}

/// Trace Turn
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TraceTurn {
    pub user_input: String,
    pub steps: Vec<TraceStep>,
    pub expects: TraceExpects,
}

/// Trace 期望
#[derive(Debug, Clone, Default, Serialize, Deserialize)]
pub struct TraceExpects {
    pub response_contains: Vec<String>,
    pub response_not_contains: Vec<String>,
    pub tools_used: Vec<String>,
    pub tools_not_used: Vec<String>,
    pub all_tools_succeeded: Option<bool>,
    pub max_tool_calls: Option<usize>,
}

/// TraceLlm Provider (用于测试回放)
pub struct TraceLlm {
    trace: LlmTrace,
    current_step: AtomicUsize,
}

#[async_trait]
impl LlmProvider for TraceLlm {
    async fn complete(&self, request: CompletionRequest) -> Result<CompletionResponse, LlmError> {
        // 从 trace 中读取预录响应
        let step = self.get_next_step()?;
        self.validate_request_hint(&request, &step)?;
        Ok(self.build_response(&step))
    }
}
```

**特点**:
- JSON Trace 格式
- 请求验证
- 期望断言
- 测试回放

---

### 1.2 IronClaw (Rust) — tracing 框架

**位置**: `reference-source-code/personal-general-ai/ironclaw/src/observability/log.rs`

**核心设计**:

```rust
// tracing 初始化
pub fn init(level: LogLevel) {
    tracing_subscriber::fmt()
        .with_env_filter(EnvFilter::from_default_env())
        .json() // 结构化 JSON 输出
        .init();
}

// 使用
tracing::info!(target: "ironclaw", session_id = %id, "Session started");
tracing::debug!(tool_name = %name, "Tool executed");
```

---

### 1.3 Kimi-CLI (Python) — Wire 协议

**位置**: `reference-source-code/coding-agents/kimi-cli/src/kimi_cli/wire/`

**核心设计**:

```python
# Wire 消息类型
class WireMessage:
    """Base class for all wire messages."""
    pass

class ToolCallRequest(WireMessage):
    tool_name: str
    params: dict

class ToolResultResponse(WireMessage):
    tool_call_id: str
    result: Any

class ApprovalRequest(WireMessage):
    tool_name: str
    description: str

# Wire 日志 (JSONL)
# ~/.kimi/sessions/{session_id}/wire.jsonl
{"type": "tool_call", "tool": "read_file", "params": {...}, "ts": 1234567890}
{"type": "tool_result", "tool_call_id": "...", "result": "...", "ts": 1234567891}
```

**特点**:
- JSONL 事件流
- 完整会话记录
- 支持回放

---

### 1.4 OpenClaw (TypeScript) — 会话记录

**位置**: `reference-source-code/personal-general-ai/openclaw/src/agents/`

**核心设计**:

```typescript
// 会话转录修复
export async function repairSessionTranscript(
  sessionKey: string,
): Promise<RepairResult> {
  // 从存储加载原始消息
  const messages = await loadSessionMessages(sessionKey);
  
  // 修复损坏的转录
  const repaired = messages.map(msg => ({
    ...msg,
    content: sanitizeContent(msg.content),
  }));
  
  // 保存修复后的转录
  await saveSessionMessages(sessionKey, repaired);
  
  return { success: true, repairedCount: messages.length };
}
```

---

## 2. 值得直接复用

### 2.1 IronClaw TraceLlm ⭐⭐⭐⭐⭐

**推荐指数**: 5/5

**理由**:
- JSON Trace 格式
- 测试回放
- 期望断言

**复用方式**:
```rust
let trace: LlmTrace = serde_json::from_str(trace_json)?;
let trace_llm = TraceLlm::new(trace);
let response = trace_llm.complete(request).await?;
```

---

### 2.2 tracing 框架 ⭐⭐⭐⭐⭐

**推荐指数**: 5/5

**理由**:
- 结构化日志
- 上下文传播
- 生态丰富

---

### 2.3 Kimi-CLI Wire ⭐⭐⭐⭐

**推荐指数**: 4/5

**理由**:
- JSONL 事件流
- 完整记录
- 可回放

---

## 3. 需要垂直特化

### 3.1 审计日志

- 医疗: HIPAA 审计
- 金融: 交易审计
- 法律: 证据链

### 3.2 合规报告

- 数据访问报告
- 模型使用报告
- 成本分析报告

---

## 4. 推荐接口

```rust
/// 追踪器
pub trait Tracer: Send + Sync {
    fn start_span(&self, name: &str) -> Span;
    fn record_event(&self, event: Event);
    fn finish_span(&self, span: Span);
}

/// 回放器
pub trait Replayer: Send + Sync {
    fn load_trace(&self, path: &Path) -> Result<Trace>;
    fn replay(&self, trace: &Trace) -> Result<Vec<Response>>;
    fn validate(&self, trace: &Trace, expects: &Expects) -> Result<()>;
}

/// 审计日志
pub trait AuditLogger: Send + Sync {
    fn log_access(&self, user: &str, resource: &str, action: &str);
    fn log_model_call(&self, model: &str, tokens: usize, cost: f64);
    fn log_tool_call(&self, tool: &str, params: &Value, result: &Value);
}
```

---

## 5. MVP 方案

```python
class SimpleTracer:
    def __init__(self):
        self.spans = []

    def start_span(self, name: str):
        span = {"name": name, "start": time.time()}
        self.spans.append(span)
        return span

    def finish_span(self, span):
        span["end"] = time.time()
        span["duration"] = span["end"] - span["start"]

    def to_json(self):
        return json.dumps(self.spans)

class SimpleReplayer:
    def __init__(self):
        self.traces = []

    def load_trace(self, path: Path):
        with open(path) as f:
            return json.load(f)

    def replay(self, trace):
        responses = []
        for step in trace["steps"]:
            response = self.execute_step(step)
            responses.append(response)
        return responses
```

---

## 6. 扩展路线

1. 结构化日志
2. 分布式追踪
3. 会话回放
4. 审计日志

---

*文档版本：1.0*
*最后更新：2026-03-21*
