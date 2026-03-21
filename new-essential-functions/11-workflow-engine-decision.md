# 11 - Workflow/Task Engine 架构决策

**技术点**: Workflow/Task Engine

**分析日期**: 2026-03-21

---

## 1. 已有实现

### 1.1 IronClaw (Rust) — Task 枚举 + Handler

**位置**: `reference-source-code/personal-general-ai/ironclaw/src/agent/task.rs`

**核心设计**:

```rust
/// Task 枚举
#[derive(Clone)]
pub enum Task {
    /// LLM 驱动的 Job
    Job {
        id: Uuid,
        title: String,
        description: String,
    },

    /// 工具执行 (子任务)
    ToolExec {
        parent_id: Uuid,
        tool_name: String,
        params: serde_json::Value,
    },

    /// 后台计算
    Background {
        id: Uuid,
        handler: Arc<dyn TaskHandler>,
    },
}

/// Task Handler Trait
#[async_trait]
pub trait TaskHandler: Send + Sync {
    async fn run(&self, ctx: TaskContext) -> Result<TaskOutput, Error>;
    fn description(&self) -> &str;
}

/// Task 上下文
#[derive(Debug, Clone)]
pub struct TaskContext {
    pub task_id: Uuid,
    pub parent_id: Option<Uuid>,
    pub metadata: serde_json::Value,
}

/// Task 输出
#[derive(Debug, Clone)]
pub struct TaskOutput {
    pub result: serde_json::Value,
    pub duration: Duration,
}
```

**特点**:
- 三种 Task 类型
- Handler Trait 抽象
- 父子任务关系

---

### 1.2 Kimi-CLI (Python) — Subagent Task

**位置**: `reference-source-code/coding-agents/kimi-cli/src/kimi_cli/tools/multiagent/task.py`

**核心设计**:

```python
class Task(CallableTool2[Params]):
    """Task tool for running subagents."""

    async def __call__(self, params: Params) -> ToolReturnValue:
        # 1. 获取子 Agent
        agent = self._labor_market.subagents.get(params.subagent_name)
        if not agent:
            return ToolError(message=f"Subagent not found: {params.subagent_name}")

        # 2. 创建子 Agent 上下文
        subagent_context_file = await self._get_subagent_context_file()
        context = Context(file_backend=subagent_context_file)
        soul = KimiSoul(agent, context=context)

        # 3. 运行子 Agent
        try:
            await run_soul(soul, params.prompt, _ui_loop_fn, asyncio.Event())
        except MaxStepsReached as e:
            return ToolError(message=f"Max steps {e.n_steps} reached")

        # 4. 返回结果
        final_response = context.history[-1].extract_text()
        return ToolOk(output=final_response)
```

**特点**:
- 子 Agent 委派
- 独立上下文
- 步骤限制

---

### 1.3 Edict (Python) — 状态机工作流

**位置**: `reference-source-code/personal-general-ai/edict/`

**核心设计**:

```python
class TaskState(str, enum.Enum):
    Taizi = "Taizi"       # 太子分拣
    Zhongshu = "Zhongshu" # 中书省起草
    Menxia = "Menxia"     # 门下省审议
    Assigned = "Assigned" # 尚书省派发
    Doing = "Doing"       # 六部执行
    Review = "Review"     # 审查汇总
    Done = "Done"         # 完成

# 状态转换规则
STATE_TRANSITIONS = {
    TaskState.Taizi: {TaskState.Zhongshu, TaskState.Cancelled},
    TaskState.Zhongshu: {TaskState.Menxia, TaskState.Cancelled},
    TaskState.Menxia: {TaskState.Assigned, TaskState.Zhongshu},
    TaskState.Assigned: {TaskState.Next, TaskState.Cancelled},
    TaskState.Doing: {TaskState.Review, TaskState.Cancelled},
    TaskState.Review: {TaskState.Done, TaskState.Zhongshu},
}
```

**特点**:
- 严格状态机
- 状态转换规则
- 审批流程

---

## 2. 值得直接复用

### 2.1 IronClaw Task 枚举 ⭐⭐⭐⭐⭐

**推荐指数**: 5/5

**理由**:
- 三种任务类型
- Handler 抽象
- 父子关系

**复用方式**:
```rust
enum Task {
    Job { id, title, description },
    ToolExec { parent_id, tool_name, params },
    Background { id, handler },
}
```

---

### 2.2 Kimi-CLI Subagent ⭐⭐⭐⭐⭐

**推荐指数**: 5/5

**理由**:
- 子 Agent 委派
- 独立上下文
- 步骤限制

---

### 2.3 Edict 状态机 ⭐⭐⭐⭐

**推荐指数**: 4/5

**理由**:
- 严格状态转换
- 审批流程

---

## 3. 需要垂直特化

### 3.1 领域特定工作流

- 医疗: 诊断流程 (挂号→问诊→检查→诊断→处方)
- 法律: 案件流程 (立案→调查→开庭→判决→执行)
- 金融: 审批流程 (申请→风控→审批→放款)

### 3.2 SLA 监控

- 超时提醒
- 升级机制
- 统计报表

---

## 4. 推荐接口

```rust
#[async_trait]
pub trait WorkflowEngine: Send + Sync {
    async fn start(&self, workflow: Workflow) -> Result<WorkflowId>;
    async fn transition(&self, id: WorkflowId, state: State) -> Result<()>;
    async fn get_status(&self, id: WorkflowId) -> Result<WorkflowStatus>;
}

pub struct Workflow {
    pub id: Uuid,
    pub workflow_type: String,
    pub current_state: State,
    pub transitions: Vec<Transition>,
}

pub struct Transition {
    pub from: State,
    pub to: State,
    pub condition: Box<dyn Fn(&Context) -> bool>,
}
```

---

## 5. MVP 方案

```python
class SimpleWorkflow:
    def __init__(self):
        self.state = "pending"
        self.transitions = {
            "pending": ["running"],
            "running": ["completed", "failed"],
        }

    def transition(self, to_state: str):
        if to_state not in self.transitions.get(self.state, []):
            raise ValueError(f"Invalid transition: {self.state} -> {to_state}")
        self.state = to_state

class TaskQueue:
    def __init__(self):
        self.queue = []
        self.running = set()

    async def submit(self, task: Callable):
        self.queue.append(task)

    async def run(self):
        while self.queue:
            task = self.queue.pop(0)
            asyncio.create_task(self._execute(task))

    async def _execute(self, task: Callable):
        self.running.add(task)
        await task()
        self.running.remove(task)
```

---

## 6. 扩展路线

1. Task 类型扩展
2. 状态机工作流
3. 子 Agent 委派
4. SLA 监控

---

*文档版本：1.0*
*最后更新：2026-03-21*
