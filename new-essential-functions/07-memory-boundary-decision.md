# 07 - Session/Workflow/Memory 边界 架构决策

**技术点**: Session State、Workflow State、Long-term Memory 边界划分

**分析日期**: 2026-03-21

---

## 1. 已有实现

### 1.1 IronClaw (Rust) — Session/Thread/Turn 三层模型

**位置**: `reference-source-code/personal-general-ai/ironclaw/src/agent/session.rs`

**核心设计**:

```rust
/// Session: 用户级会话容器
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Session {
    pub id: Uuid,
    pub user_id: String,
    pub active_thread: Option<Uuid>,
    pub threads: HashMap<Uuid, Thread>,
    pub created_at: DateTime<Utc>,
    pub last_active_at: DateTime<Utc>,
    pub auto_approved_tools: HashSet<String>,
}

/// Thread: 对话线程
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Thread {
    pub id: Uuid,
    pub session_id: Uuid,
    pub turns: Vec<Turn>,
    pub state: ThreadState,
    pub pending_approval: Option<PendingApproval>,
    pub pending_auth: Option<PendingAuth>,
}

/// Turn: 请求/响应对
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Turn {
    pub id: Uuid,
    pub user_input: String,
    pub response: Option<String>,
    pub tool_calls: Vec<ToolCall>,
    pub tool_results: Vec<ToolResult>,
    pub created_at: DateTime<Utc>,
}

/// 线程状态
pub enum ThreadState {
    Idle,             // 空闲
    Processing,       // 处理中
    AwaitingApproval, // 等待审批
    Completed,        // 完成
    Interrupted,      // 中断
}
```

**边界划分**:
```
Session (用户级)
  └── Thread 1 (对话线程)
        └── Turn 1 (请求/响应)
        └── Turn 2
  └── Thread 2
```

**特点**:
- Session: 用户级容器
- Thread: 支持多对话线程
- Turn: 完整的请求/响应周期
- 状态机驱动

---

### 1.2 OpenClaw (TypeScript) — Session 解析与生命周期

**位置**: `reference-source-code/personal-general-ai/openclaw/src/agents/command/session.ts`

**核心设计**:

```typescript
/// Session 解析结果
export type SessionResolution = {
  sessionId: string;
  sessionKey?: string;
  sessionEntry?: SessionEntry;
  sessionStore?: Record<string, SessionEntry>;
  storePath: string;
  isNewSession: boolean;
  persistedThinking?: ThinkLevel;
  persistedVerbose?: VerboseLevel;
};

/// Session 解析流程
export function resolveSession(opts: {
  cfg: OpenClawConfig;
  to?: string;
  sessionId?: string;
  sessionKey?: string;
  agentId?: string;
}): SessionResolution {
  // 1. 解析 Session Key
  const { sessionKey, sessionStore, storePath } = resolveSessionKeyForRequest(opts);
  
  // 2. 评估 Session 新鲜度
  const fresh = evaluateSessionFreshness({
    updatedAt: sessionEntry?.updatedAt,
    now: Date.now(),
    policy: resetPolicy,
  });
  
  // 3. 生成/复用 Session ID
  const sessionId = fresh 
    ? sessionEntry?.sessionId 
    : crypto.randomUUID();
  
  return { sessionId, sessionKey, ... };
}

/// Session Entry (存储)
export interface SessionEntry {
  sessionId: string;
  channel: string;
  lastChannel: string;
  updatedAt: number;
  thinkingLevel?: string;
  verboseLevel?: string;
}
```

**特点**:
- Session Key 派生 (基于发送者/通道)
- 新鲜度评估 (自动重置策略)
- 多 Agent Session 存储

---

### 1.3 三层记忆模型 (通用模式)

**来源**: 多个项目共同采用

```
┌─────────────────────────────────────────────────────────────────┐
│ Layer 1: Working Memory (Session Context)                       │
│          └── 当前对话、工具调用、即时状态                        │
│          └── RAM，临时，最高带宽                                 │
├─────────────────────────────────────────────────────────────────┤
│ Layer 2: Short-Term Memory (Recent History)                     │
│          └── 每日日志、会话摘要、近期事实                        │
│          └── JSONL / YYYY-MM-DD.md                              │
├─────────────────────────────────────────────────────────────────┤
│ Layer 3: Long-Term Memory (Persistent Knowledge)                │
│          └── 精选事实、用户偏好、决策                            │
│          └── MEMORY.md + Vector DB + FTS                        │
└─────────────────────────────────────────────────────────────────┘
```

**写入策略**:

| 层级 | 触发条件 | 存储 | 同步方式 |
|------|---------|------|---------|
| Working | 实时 | RAM | 同步 |
| Short-Term | 阈值/定时 | JSONL | 异步 |
| Long-Term | 手动/压缩 | SQLite/Vector | 异步 |

---

## 2. 值得直接复用

### 2.1 IronClaw Session/Thread/Turn 模型 ⭐⭐⭐⭐⭐

**推荐指数**: 5/5

**理由**:
- 清晰的层级边界
- 支持多线程
- 状态机驱动
- 完整的生命周期

**复用方式**:
```rust
// Session → Thread → Turn
let session = Session::new(user_id);
let thread = session.create_thread();
let turn = thread.add_turn(user_input);
```

---

### 2.2 OpenClaw Session 解析 ⭐⭐⭐⭐⭐

**推荐指数**: 5/5

**理由**:
- Session Key 派生算法
- 新鲜度评估
- 多 Agent 支持

---

### 2.3 三层记忆模型 ⭐⭐⭐⭐⭐

**推荐指数**: 5/5

**理由**:
- 业界通用模式
- 清晰的职责划分
- 渐进式持久化

---

## 3. 需要垂直特化

### 3.1 Workflow State 边界

**通用**: Session 级状态

**垂直特化**:
- 医疗: 病例级状态 (跨 Session)
- 法律: 案件级状态 (跨 Session)
- 金融: 交易级状态 (原子性)

**特化方案**:
```rust
pub struct WorkflowState {
    pub workflow_id: String,
    pub workflow_type: WorkflowType,  // MedicalCase/LegalCase/Transaction
    pub session_ids: Vec<String>,     // 关联的 Sessions
    pub state: WorkflowStatus,
}
```

---

### 3.2 Memory 访问控制

**通用**: 用户级隔离

**垂直特化**:
- 医疗: 医生-患者隔离
- 法律: 律师-客户隔离
- 企业: 部门级隔离

---

## 4. 推荐接口

```rust
/// Session 管理
pub trait SessionManager: Send + Sync {
    fn create(&self, user_id: &str) -> Session;
    fn get(&self, session_id: &str) -> Option<Session>;
    fn update(&self, session: &Session);
    fn delete(&self, session_id: &str);
}

/// Workflow 管理
pub trait WorkflowManager: Send + Sync {
    fn create(&self, workflow_type: WorkflowType) -> Workflow;
    fn attach_session(&self, workflow_id: &str, session_id: &str);
    fn get_state(&self, workflow_id: &str) -> WorkflowState;
}

/// Memory 管理
pub trait MemoryManager: Send + Sync {
    fn working(&self) -> &mut WorkingMemory;
    fn short_term(&self) -> &ShortTermMemory;
    fn long_term(&self) -> &LongTermMemory;
}
```

---

## 5. MVP 方案

```python
@dataclass
class Session:
    id: str
    user_id: str
    messages: List[Message]
    created_at: datetime

@dataclass
class Workflow:
    id: str
    workflow_type: str
    session_ids: List[str]
    state: str

class SimpleSessionManager:
    def __init__(self):
        self.sessions: Dict[str, Session] = {}
    
    def create(self, user_id: str) -> Session:
        session = Session(
            id=str(uuid.uuid4()),
            user_id=user_id,
            messages=[],
            created_at=datetime.now(),
        )
        self.sessions[session.id] = session
        return session
```

---

## 6. 扩展路线

1. Session/Thread/Turn 层级
2. Workflow 跨 Session 状态
3. 三层记忆模型
4. 访问控制

---

*文档版本：1.0*
*最后更新：2026-03-21*
