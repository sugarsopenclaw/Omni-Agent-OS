# 10 - Channel Adapter 架构决策

**技术点**: Channel Adapter (飞书/钉钉/QQ/Web)

**分析日期**: 2026-03-21

---

## 1. 已有实现

### 1.1 IronClaw (Rust) — Channel Trait

**位置**: `reference-source-code/personal-general-ai/ironclaw/src/channels/channel.rs`

**核心设计**:

```rust
/// 传入消息
#[derive(Debug, Clone)]
pub struct IncomingMessage {
    pub id: Uuid,
    pub channel: String,
    pub user_id: String,
    pub sender_id: String,
    pub content: String,
    pub thread_id: Option<String>,
    pub conversation_scope_id: Option<String>,
    pub received_at: DateTime<Utc>,
    pub metadata: serde_json::Value,
    pub timezone: Option<String>,
    pub attachments: Vec<IncomingAttachment>,
}

/// 附件
#[derive(Debug, Clone)]
pub struct IncomingAttachment {
    pub id: String,
    pub kind: AttachmentKind,  // Audio/Image/Document
    pub mime_type: String,
    pub filename: Option<String>,
    pub source_url: Option<String>,
    pub extracted_text: Option<String>,
    pub data: Vec<u8>,
}

/// Channel Trait
#[async_trait]
pub trait Channel: Send + Sync {
    /// Channel 名称
    fn name(&self) -> &str;

    /// 启动监听
    async fn start(&self, handler: MessageHandler) -> Result<(), ChannelError>;

    /// 发送消息
    async fn send(&self, message: OutgoingMessage) -> Result<(), ChannelError>;

    /// 检查用户权限
    fn is_allowed(&self, user_id: &str) -> bool;
}

/// 传出消息
#[derive(Debug, Clone)]
pub struct OutgoingMessage {
    pub target: String,
    pub content: String,
    pub thread_id: Option<String>,
    pub attachments: Vec<OutgoingAttachment>,
}
```

**特点**:
- 统一的消息格式
- 附件抽象
- 线程支持
- 时区传递

---

### 1.2 CoPaw (Python) — BaseChannel

**位置**: 基于 AgentScope

**核心设计**:

```python
class BaseChannel(ABC):
    """Unified channel interface for all messaging platforms."""

    def __init__(
        self,
        process: ProcessHandler,
        dm_policy: str = "open",  # "open", "closed", "whitelist"
        group_policy: str = "open",
        allow_from: Optional[List[str]] = None,
        require_mention: bool = False,
    ):
        self.process = process
        self.dm_policy = dm_policy
        self.group_policy = group_policy
        self.allow_from = set(allow_from or [])
        self.require_mention = require_mention

    @abstractmethod
    async def start(self) -> None:
        """Start listening for messages."""
        pass

    @abstractmethod
    async def send_reply(
        self,
        request: AgentRequest,
        contents: List[OutgoingContentPart],
    ) -> None:
        """Send response back to the channel."""
        pass

    async def _should_process(self, msg: IncomingMessage) -> bool:
        """Permission check: whitelist + mention policy."""
        if self.allow_from and msg.sender_id not in self.allow_from:
            return False
        if msg.is_group and self.require_mention and not msg.is_mentioned:
            return False
        return True
```

**支持通道**:
- Feishu/Lark (WebSocket)
- DingTalk (Stream SDK)
- Discord (Gateway WebSocket)
- QQ (HTTP POST)
- Telegram (Bot API)
- Matrix (Client-Server API)
- iMessage (macOS Scripting Bridge)

---

### 1.3 OpenClaw (TypeScript) — Channel Registry

**位置**: `reference-source-code/personal-general-ai/openclaw/src/channels/`

**核心设计**:

```typescript
// 通道注册表
export class ChannelRegistry {
  private channels: Map<string, Channel> = new Map();

  register(name: string, channel: Channel): void {
    this.channels.set(name, channel);
  }

  async initializeEnabled(config: Config): Promise<Channel[]> {
    const enabled: Channel[] = [];
    for (const [name, channel] of this.channels) {
      if (config.channels[name]?.enabled) {
        await channel.start();
        enabled.push(channel);
      }
    }
    return enabled;
  }
}

// 统一消息格式
export interface InboundMessage {
  channel: string;
  senderId: string;
  content: string;
  threadId?: string;
  attachments?: Attachment[];
  timestamp: Date;
}

export interface OutboundMessage {
  target: string;
  content: string;
  threadId?: string;
  attachments?: Attachment[];
}
```

---

## 2. 值得直接复用

### 2.1 IronClaw Channel Trait ⭐⭐⭐⭐⭐

**推荐指数**: 5/5

**理由**:
- 类型安全
- 统一消息格式
- 附件抽象

**复用方式**:
```rust
#[async_trait]
impl Channel for FeishuChannel {
    fn name(&self) -> &str { "feishu" }
    async fn start(&self, handler: MessageHandler) -> Result<()> { ... }
    async fn send(&self, message: OutgoingMessage) -> Result<()> { ... }
}
```

---

### 2.2 CoPaw 权限策略 ⭐⭐⭐⭐⭐

**推荐指数**: 5/5

**理由**:
- DM/Group 策略
- 白名单
- @提及要求

---

### 2.3 OpenClaw Channel Registry ⭐⭐⭐⭐

**推荐指数**: 4/5

**理由**:
- 动态注册
- 配置驱动

---

## 3. 需要垂直特化

### 3.1 国内平台适配

- 飞书: 卡片消息、审批流
- 钉钉: 工作通知、考勤
- 企业微信: 部门同步

### 3.2 行业特定集成

- 医疗: 医院 HIS 系统
- 金融: 银行消息网关
- 政务: 政务微信

---

## 4. 推荐接口

```rust
#[async_trait]
pub trait Channel: Send + Sync {
    fn name(&self) -> &str;
    async fn start(&self, handler: MessageHandler) -> Result<()>;
    async fn send(&self, message: OutgoingMessage) -> Result<()>;
    fn is_allowed(&self, user_id: &str) -> bool;
}

pub struct ChannelRegistry {
    channels: HashMap<String, Box<dyn Channel>>,
}

impl ChannelRegistry {
    pub fn register(&mut self, channel: Box<dyn Channel>) {
        self.channels.insert(channel.name().to_string(), channel);
    }

    pub async fn start_all(&self) -> Result<()> {
        for (_, channel) in &self.channels {
            channel.start(handler).await?;
        }
        Ok(())
    }
}
```

---

## 5. MVP 方案

```python
class SimpleChannel(ABC):
    @abstractmethod
    async def start(self): pass

    @abstractmethod
    async def send(self, target: str, content: str): pass

class ChannelManager:
    def __init__(self):
        self.channels: dict[str, SimpleChannel] = {}

    def register(self, name: str, channel: SimpleChannel):
        self.channels[name] = channel

    async def start_all(self):
        for name, channel in self.channels.items():
            await channel.start()
            print(f"Channel {name} started")
```

---

## 6. 扩展路线

1. Channel Trait
2. 权限策略
3. 附件处理
4. 国内平台适配

---

*文档版本：1.0*
*最后更新：2026-03-21*
