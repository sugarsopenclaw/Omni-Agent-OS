# 06 - Model Provider 抽象与路由 架构决策

**技术点**: Model Provider 抽象与路由

**分析日期**: 2026-03-21

---

## 1. 已有实现

### 1.1 IronClaw (Rust) — Provider Trait + 包装器链

**位置**: `reference-source-code/personal-general-ai/ironclaw/src/llm/`

**核心设计**:

```rust
#[async_trait]
pub trait LlmProvider: Send + Sync {
    async fn complete(&self, request: CompletionRequest) -> Result<CompletionResponse>;
    async fn complete_stream(&self, request: CompletionRequest) -> Result<CompletionStream>;
    fn metadata(&self) -> ModelMetadata;
}

/// 包装器: 重试
pub struct RetryProvider { inner: Arc<dyn LlmProvider>, config: RetryConfig }

/// 包装器: 故障转移
pub struct FailoverProvider { providers: Vec<Arc<dyn LlmProvider>> }

/// 包装器: 智能路由
pub struct SmartRoutingProvider {
    providers: HashMap<TaskComplexity, Arc<dyn LlmProvider>>,
}
```

**包装器链**:
```
FailoverProvider(
  RetryProvider(
    CircuitBreakerProvider(
      CachedProvider(BaseProvider)
    )
  )
)
```

**特点**: Trait 抽象、包装器链、故障转移、智能路由

---

### 1.2 OpenClaw (TypeScript) — 模型回退

**位置**: `reference-source-code/personal-general-ai/openclaw/src/agents/model-fallback.ts`

```typescript
async function runWithModelFallback<T>({
  run,
  candidates,
}: {
  run: ModelFallbackRunFn<T>;
  candidates: ModelCandidate[];
}): Promise<ModelFallbackRunResult<T>> {
  for (const candidate of candidates) {
    const result = await run(candidate.provider, candidate.model);
    if (result.ok) return result;
    // 记录失败，继续下一个
  }
  throw new Error("All candidates failed");
}
```

**特点**: 候选模型排序、失败统计、认证配置管理

---

### 1.3 CoPaw (Python) — 路由模型

```python
class RoutingChatModel:
    def __init__(self, models: Dict[str, ChatModelBase]):
        self.models = models
        self.token_usage = TokenUsageManager()

    def select_model(self, complexity: TaskComplexity) -> ChatModelBase:
        if complexity == TaskComplexity.Simple:
            return self.models["gpt-3.5-turbo"]
        return self.models["gpt-4"]
```

**特点**: 任务复杂度分析、Token 用量追踪

---

### 1.4 Aider (Python) — LiteLLM

```python
from litellm import completion

class Model:
    def complete(self, messages: List[Dict]) -> str:
        response = completion(model=self.name, messages=messages)
        return response.choices[0].message.content
```

**特点**: LiteLLM 统一接口、100+ 模型支持

---

## 2. 值得直接复用

| 项目 | 特性 | 推荐指数 |
|------|------|----------|
| IronClaw | Provider Trait + 包装器链 | ⭐⭐⭐⭐⭐ |
| OpenClaw | 模型回退机制 | ⭐⭐⭐⭐⭐ |
| IronClaw | 智能路由 | ⭐⭐⭐⭐ |
| CoPaw | Token 用量追踪 | ⭐⭐⭐⭐ |

---

## 3. 需要垂直特化

### 3.1 领域特定模型选择

- 医疗: 临床诊断 → GPT-4/Claude-3-Opus
- 法律: 合同审查 → 法律专用模型
- 金融: 风险评估 → 金融专用模型

### 3.2 合规性要求

- 医疗: 数据不出境 → 本地模型
- 金融: 审计要求 → 完整日志

---

## 4. 推荐接口

```rust
#[async_trait]
pub trait LlmProvider: Send + Sync {
    async fn complete(&self, request: CompletionRequest) -> Result<CompletionResponse>;
    async fn complete_stream(&self, request: CompletionRequest) -> Result<CompletionStream>;
}

pub struct ProviderRegistry {
    providers: HashMap<String, Arc<dyn LlmProvider>>,
}

pub struct ModelRouter {
    registry: ProviderRegistry,
    fallback_chain: Vec<String>,
}
```

---

## 5. MVP 方案

```python
class SimpleModelProvider:
    def __init__(self, api_key: str, base_url: str):
        self.client = OpenAI(api_key=api_key, base_url=base_url)

    def complete(self, messages: list) -> str:
        response = self.client.chat.completions.create(
            model="gpt-3.5-turbo",
            messages=messages,
        )
        return response.choices[0].message.content

class SimpleRouter:
    def __init__(self, providers: dict):
        self.providers = providers
        self.fallback_order = list(providers.keys())

    def complete(self, messages: list) -> str:
        for name in self.fallback_order:
            try:
                return self.providers[name].complete(messages)
            except Exception as e:
                print(f"{name} failed: {e}")
        raise Exception("All providers failed")
```

---

## 6. 扩展路线

1. Provider Trait 抽象
2. 包装器链 (Retry/Failover)
3. 智能路由
4. Token 用量追踪

---

*文档版本：1.0*
*最后更新：2026-03-21*
