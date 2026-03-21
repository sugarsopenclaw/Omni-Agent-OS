# 14 - Storage Schema 与核心对象模型 架构决策

**技术点**: Storage Schema 与核心对象模型

**分析日期**: 2026-03-21

---

## 1. 已有实现

### 1.1 IronClaw (Rust) — PostgreSQL + libSQL

**位置**: `reference-source-code/personal-general-ai/ironclaw/migrations/`

**核心 Schema**:

```sql
-- Routines: 定时任务系统
CREATE TABLE routines (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    description TEXT NOT NULL DEFAULT '',
    user_id TEXT NOT NULL,
    enabled BOOLEAN NOT NULL DEFAULT true,
    
    -- 触发器定义
    trigger_type TEXT NOT NULL,
    trigger_config JSONB NOT NULL,
    
    -- 动作定义
    action_type TEXT NOT NULL,
    action_config JSONB NOT NULL,
    
    -- Guardrails
    cooldown_secs INTEGER NOT NULL DEFAULT 300,
    max_concurrent INTEGER NOT NULL DEFAULT 1,
    
    -- 运行时状态
    state JSONB NOT NULL DEFAULT '{}',
    last_run_at TIMESTAMPTZ,
    next_fire_at TIMESTAMPTZ,
    run_count BIGINT NOT NULL DEFAULT 0,
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- 执行记录
CREATE TABLE routine_runs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    routine_id UUID NOT NULL REFERENCES routines(id) ON DELETE CASCADE,
    trigger_type TEXT NOT NULL,
    started_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at TIMESTAMPTZ,
    status TEXT NOT NULL DEFAULT 'running',
    tokens_used INTEGER
);
```

**对象模型**:
```rust
User → Workspace → Session → Thread → Turn → Message
```

---

### 1.2 OpenClaw (TypeScript) — TypeBox Schema

**位置**: `reference-source-code/personal-general-ai/openclaw/src/gateway/protocol/schema/`

**核心 Schema**:

```typescript
export const ProtocolSchemas = {
  AgentParams: AgentParamsSchema,
  SessionsCreateParams: SessionsCreateParamsSchema,
  ToolsCatalogParams: ToolsCatalogParamsSchema,
  CronAddParams: CronAddParamsSchema,
};
```

---

### 1.3 存储分层模型

```
Runtime Memory → Session Store → Persistent Store
     (RAM)          (JSONL)         (SQLite/PostgreSQL)
```

---

## 2. 值得直接复用

| 项目 | 特性 | 推荐指数 |
|------|------|----------|
| IronClaw | 对象模型 | ⭐⭐⭐⭐⭐ |
| OpenClaw | TypeBox Schema | ⭐⭐⭐⭐⭐ |
| 通用 | 存储分层 | ⭐⭐⭐⭐⭐ |

---

## 3. 推荐接口

```rust
#[async_trait]
pub trait StorageBackend: Send + Sync {
    async fn init(&self) -> Result<()>;
    async fn begin_transaction(&self) -> Result<Transaction>;
}

#[async_trait]
pub trait ObjectStore<T: Object>: Send + Sync {
    async fn create(&self, obj: &T) -> Result<T>;
    async fn read(&self, id: &str) -> Result<Option<T>>;
    async fn update(&self, obj: &T) -> Result<T>;
    async fn delete(&self, id: &str) -> Result<()>;
}
```

---

## 4. MVP 方案

```python
@dataclass
class Session:
    id: str
    user_id: str
    messages: List[Message]
    created_at: datetime

class SimpleStorage:
    def __init__(self, db_path: str):
        self.conn = sqlite3.connect(db_path)
    
    def save_session(self, session: Session):
        # 保存到 SQLite
        pass
```

---

## 5. 完成总结

**14 个技术分析文档全部完成！**

| 编号 | 技术点 | 状态 |
|------|--------|------|
| 01 | Agent Runtime 主循环 | ✅ |
| 02 | Context Assembly Pipeline | ✅ |
| 03 | 上下文压缩/摘要/保留区策略 | ✅ |
| 04 | Tool Registry 与结果后处理 | ✅ |
| 05 | Skills 体系设计 | ✅ |
| 06 | Model Provider 抽象与路由 | ✅ |
| 07 | Session/Workflow/Memory 边界 | ✅ |
| 08 | RAG Orchestration 与引用机制 | ✅ |
| 09 | Document Ingestion Pipeline | ✅ |
| 10 | Channel Adapter | ✅ |
| 11 | Workflow/Task Engine | ✅ |
| 12 | Observability/Tracing/Replay | ✅ |
| 13 | Plugin Architecture | ✅ |
| 14 | Storage Schema 与核心对象模型 | ✅ |

---

*文档版本：1.0*
*最后更新：2026-03-21*
*总计：14 篇技术分析文档*
