---
Task: T21-R
Status: DRAFT_DONE
Date: 2026-04-15
Chapter: Ch21 消息网关 (Message Gateway)
Keywords: 统一网关架构、会话上下文组装、消息投递、跨平台 slash 命令
---

# Ch21 消息网关 (Message Gateway) — Research Notes

## 1. 总体架构 (Overall Architecture)

Hermes 网关 (Gateway) 是一个**统一消息总线**，将来自 19 种平台 (Platform) 的消息归一化为内部 `MessageEvent`，经过认证 (Authorization)、会话管理 (Session Management)、命令分发 (Command Dispatch)、智能体调度 (Agent Invocation)，最终将回复投递回源平台。

### 核心组件关系

```
Platform Adapter (×21)         GatewayRunner              SessionStore
  ┌──────────────┐          ┌──────────────────┐       ┌──────────────┐
  │ TelegramAdapter│─────►  │ _handle_message()│──────►│get_or_create │
  │ DiscordAdapter │────►   │ slash commands   │       │load_transcript│
  │ SlackAdapter   │───►    │ agent dispatch   │       │append_to_...  │
  │ WhatsAppAdapter│──►     │ interrupt logic  │       └──────────────┘
  │ ...×21 total   │─►      └────────┬─────────┘
  └──────────────┘                   │
                                     ▼
                              DeliveryRouter ──► Platform Adapter.send()
```

**文件清单**（gateway/ 目录，41 个 .py 文件）:

| 文件 | 行数 | 职责 |
|------|------|------|
| `run.py` | 9654 | `GatewayRunner` 主控制器 (Main controller)，消息处理流水线 |
| `session.py` | 1086 | `SessionStore`, `SessionSource`, `SessionContext`, `build_session_key()` |
| `delivery.py` | 256 | `DeliveryTarget`, `DeliveryRouter` — 消息投递路由 |
| `config.py` | 1160 | `Platform` 枚举 (19 值), `GatewayConfig`, `SessionResetPolicy` |
| `hooks.py` | 170 | `HookRegistry` — 生命周期事件钩子系统 |
| `stream_consumer.py` | 744 | `GatewayStreamConsumer` — 流式 LLM 响应的渐进式编辑 |
| `platforms/base.py` | 2071 | `BasePlatformAdapter` ABC, `MessageEvent`, `SendResult` |
| `platforms/telegram.py` | 2825 | Telegram 适配器 (最大的平台适配器) |
| `platforms/discord.py` | 3020 | Discord 适配器，含 `VoiceReceiver` 语音接收 |
| `platforms/slack.py` | 1670 | Slack 适配器，含 Assistant API 支持 |

---

## 2. 事件模型 (Event Model) — MessageEvent

> **注意**：源码中不存在 `GatewayEvent` 类。核心事件模型是 `MessageEvent`（定义于 `platforms/base.py:656`）。

```python
@dataclass
class MessageEvent:
    text: str                               # 消息文本内容
    message_type: MessageType = TEXT        # TEXT/PHOTO/VOICE/AUDIO/DOCUMENT/...
    source: SessionSource = None            # 消息来源（平台、聊天、用户）
    raw_message: Any = None                 # 平台原始消息对象
    message_id: Optional[str] = None        # 平台消息 ID
    media_urls: List[str] = []              # 媒体附件的本地文件路径
    media_types: List[str] = []             # MIME 类型列表
    reply_to_message_id: Optional[str] = None  # 引用消息 ID
    reply_to_text: Optional[str] = None     # 引用消息文本
    auto_skill: Optional[str|list] = None   # 自动加载的 Skill
    internal: bool = False                  # 内部事件标志（跳过授权）
    timestamp: datetime = datetime.now()
```

### MessageType 枚举

```python
class MessageType(Enum):
    TEXT = "text"
    LOCATION = "location"
    PHOTO = "photo"
    VIDEO = "video"
    AUDIO = "audio"
    VOICE = "voice"
    DOCUMENT = "document"
    STICKER = "sticker"
    COMMAND = "command"
```

### SendResult — 发送结果

```python
@dataclass
class SendResult:
    success: bool
    message_id: Optional[str] = None
    error: Optional[str] = None
    raw_response: Any = None
    retryable: bool = False  # 可重试的瞬态错误
```

### ProcessingOutcome — 处理结果分类

```python
class ProcessingOutcome(Enum):
    SUCCESS = "success"
    FAILURE = "failure"
    CANCELLED = "cancelled"
```

---

## 3. SessionSource — 消息来源标识

```python
@dataclass
class SessionSource:
    platform: Platform          # 平台枚举
    chat_id: str               # 聊天 ID
    chat_name: Optional[str]   # 聊天名称
    chat_type: str = "dm"      # "dm" | "group" | "channel" | "thread"
    user_id: Optional[str]     # 用户 ID
    user_name: Optional[str]   # 用户名
    thread_id: Optional[str]   # 论坛话题 / Discord 线程
    chat_topic: Optional[str]  # 频道主题描述
    user_id_alt: Optional[str] # Signal UUID 备选 ID
    chat_id_alt: Optional[str] # Signal group 内部 ID
```

---

## 4. 会话管理 (Session Management)

### 4.1 会话键构建 — `build_session_key()`

**格式**: `agent:main:{platform}:{chat_type}:{chat_id}:{thread_id}:{user_id}`

规则：
- **DM**: `agent:main:telegram:dm:{chat_id}[:thread_id]`
- **群组**: `agent:main:discord:group:{chat_id}[:thread_id][:user_id]`
- **用户隔离 (User Isolation)**: `group_sessions_per_user=True` 时，群组中每个用户有独立会话
- **线程共享 (Thread Sharing)**: `thread_sessions_per_user=False`（默认），同一线程内所有用户共享会话

### 4.2 SessionStore — 会话存储

**双层存储架构**:
1. **SQLite** (`hermes_state.SessionDB`) — 主存储，结构化查询
2. **JSONL** — 遗留兼容，文件级别的消息日志

```python
class SessionStore:
    def __init__(self, sessions_dir, config):
        self._db = SessionDB()          # SQLite
        self._entries: Dict[str, SessionEntry] = {}  # 内存索引
        self._lock = threading.Lock()   # 线程安全

    def get_or_create_session(source) -> SessionEntry
    def load_transcript(session_id) -> List[Dict]      # 优先用 SQLite，回退 JSONL
    def append_to_transcript(session_id, message)       # 双写 SQLite + JSONL
    def rewrite_transcript(session_id, messages)        # 用于 /retry, /undo, /compress
    def reset_session(session_key) -> SessionEntry      # 强制重置
    def switch_session(key, target_id) -> SessionEntry  # /resume 切换
    def list_sessions(active_minutes) -> List[SessionEntry]
    def suspend_active_sessions(max_age_seconds)        # 启动时挂起上次活跃会话
```

**Transcript 加载优先级**: 比较 SQLite 和 JSONL 的消息条数，取多的那个。这解决了 SQLite 迁移期间的消息丢失问题。

### 4.3 SessionEntry — 会话条目

包含丰富的 Token 统计字段:
- `input_tokens`, `output_tokens`, `cache_read_tokens`, `cache_write_tokens`
- `total_tokens`, `last_prompt_tokens`
- `estimated_cost_usd`, `cost_status`
- `memory_flushed`, `suspended`

### 4.4 SessionResetPolicy — 会话重置策略

四种模式：`daily`（每日定时）、`idle`（空闲超时）、`both`（两者皆用）、`none`（不重置）

### 4.5 SessionContext — 会话上下文组装

```python
@dataclass
class SessionContext:
    source: SessionSource               # 消息来源
    connected_platforms: List[Platform]  # 已连接的平台列表
    home_channels: Dict[Platform, HomeChannel]  # 各平台主频道
    session_key: str
    session_id: str
    created_at: datetime
    updated_at: datetime
```

`build_session_context_prompt()` 将上下文转换为自然语言，注入系统提示词 (System Prompt)，让 Agent 知道：
- 当前消息来自哪个平台/聊天
- 哪些平台已连接
- 定时任务输出应投递到哪里

**PII 脱敏**: 对 WhatsApp、Signal、Telegram、BlueBubbles 平台，支持 `redact_pii` 选项，用 SHA-256 哈希替换用户 ID 和聊天 ID。

---

## 5. 核心消息处理流水线 (Core Message Pipeline)

### 5.1 `_handle_message()` — 核心处理器

位于 `run.py:2655`，是网关的**心脏**。处理步骤:

```
1. 授权检查 (Authorization Check)
   ├── internal 事件跳过授权
   ├── 无 user_id 的消息静默丢弃
   └── 未授权用户 → DM 中发送配对码 (pairing code)

2. 更新提示拦截 (/update 流程)

3. 运行中的 Agent 优先处理 (Running Agent Priority)
   ├── /status → 查询状态
   ├── /stop → 强制中断 + 释放会话锁
   ├── /new → 中断 + 重置会话
   ├── /queue → 排队不打断
   ├── /approve, /deny → 绕过中断，直接处理审批
   ├── PHOTO 消息 → 排队合并（相册批处理）
   └── 其他文本 → interrupt() 中断当前 Agent

4. Slash 命令分发 (Command Dispatch)
   ├── 内建命令 (30+ 个): /new, /help, /model, /provider, ...
   ├── Quick Commands (用户自定义): exec / alias 类型
   ├── Plugin Commands (插件注册)
   └── Skill Commands (技能加载)

5. 会话获取 + Agent 调用
   ├── _running_agents[key] = PENDING_SENTINEL  ← 占位防并发
   └── → _handle_message_with_agent()
```

### 5.2 `_handle_message_with_agent()` — Agent 调用

```
1. get_or_create_session() → SessionEntry
2. build_session_context() → SessionContext
3. build_session_context_prompt() → 系统提示词注入
4. 自动重置通知（如果会话过期被重置）
5. Auto-skill 加载（话题/频道绑定）
6. load_transcript() → 会话历史
7. 会话卫生检查（自动压缩过大的 transcript）
8. _prepare_inbound_message_text()
   ├── 共享线程中添加发送者标识 [username]
   ├── 图片 → Vision 增强
   ├── 音频 → STT 转文字
   └── 文档 → 内容提取
9. 创建 AIAgent + GatewayStreamConsumer
10. 在线程池中运行 Agent
11. 通过 StreamConsumer 渐进式发送响应
```

### 5.3 并发控制 — `_running_agents`

```python
self._running_agents: Dict[str, Any]  # session_key → Agent 或 PENDING_SENTINEL
self._running_agents_ts: Dict[str, float]  # 时间戳，用于过期检测
```

**PENDING_SENTINEL 模式**：在 Agent 创建前就占位，防止在 `await` 点之间的竞态条件导致重复 Agent。

**过期回收**: 基于 Agent 的空闲时间 (`get_activity_summary()`) 而非纯墙钟时间，支持长时间运行的 Agent。

---

## 6. 消息投递 (Message Delivery)

### 6.1 DeliveryTarget — 投递目标

```python
@dataclass
class DeliveryTarget:
    platform: Platform
    chat_id: Optional[str] = None       # None 表示使用主频道
    thread_id: Optional[str] = None
    is_origin: bool = False             # 回复到消息来源
    is_explicit: bool = False           # chat_id 是显式指定的
```

解析格式:
- `"origin"` → 回复到来源
- `"local"` → 保存到本地文件
- `"telegram"` → Telegram 主频道
- `"telegram:123456"` → 指定 Telegram 聊天
- `"telegram:123456:789"` → 指定聊天和线程

### 6.2 DeliveryRouter — 投递路由器

```python
class DeliveryRouter:
    async def deliver(content, targets, job_id, job_name, metadata) -> Dict
    def _deliver_local(content, job_id, ...)     # 保存到 ~/.hermes/cron/output/
    async def _deliver_to_platform(target, ...)  # 调用 adapter.send()
```

**大消息截断**: 超过 4000 字符的 cron 输出被截断，完整内容保存到文件。

### 6.3 `_send_with_retry()` — 重试机制

```
send() 失败
  ├── 瞬态网络错误 (retryable) → 指数退避重试 (max 2 次)
  │   └── 全部失败 → 发送失败通知给用户
  ├── 超时错误 (timeout) → 不重试（可能已送达）
  └── 格式/权限错误 → 纯文本回退 (plain-text fallback)
```

可重试错误模式: `connecterror`, `connectionreset`, `network`, `broken pipe`, `remotedisconnected`, `eoferror`

---

## 7. 流式响应消费 — GatewayStreamConsumer

`stream_consumer.py` 实现了从同步 Agent 线程到异步平台投递的桥接:

```python
class GatewayStreamConsumer:
    def on_delta(text: str)    # 线程安全回调（sync Agent 线程调用）
    async def run()            # 异步任务（在事件循环中运行）
    def finish()               # 信号流完成
    def on_segment_break()     # 工具边界 → 新消息段
    def on_commentary(text)    # 中间评论消息
```

**设计模式**: 使用 `queue.Queue` 作为跨线程通道。Agent 线程同步写入 delta，异步 `run()` 任务消费并渐进式编辑平台消息。

**关键配置**:
- `edit_interval`: 1.0s（编辑频率）
- `buffer_threshold`: 40 字符（缓冲阈值）
- `cursor`: ` ▉`（打字光标）

**Think-block 过滤**: 状态机过滤 `<think>`, `<reasoning>`, `<THINKING>` 等模型推理标签，防止原始推理内容泄露到用户界面。

**Flood Control**: 连续 3 次编辑失败后永久禁用渐进式编辑，回退到最终一次性发送。

---

## 8. 平台适配器 (Platform Adapters)

### 8.1 BasePlatformAdapter — 抽象基类

```python
class BasePlatformAdapter(ABC):
    # 必须实现的抽象方法
    async def connect() -> bool
    async def disconnect()
    async def send(chat_id, content, reply_to, metadata) -> SendResult
    async def send_typing(chat_id, metadata)

    # 可选覆盖（提供默认实现）
    async def edit_message(...)     # 默认返回 "Not supported"
    async def send_image(...)      # 默认回退为文本 URL
    async def send_animation(...)  # 默认回退为 send_image
    async def send_voice(...)      # 默认回退为文本
    async def send_video(...)      # 默认回退为文本
    async def send_document(...)   # 默认回退为文本
    async def send_image_file(...) # 本地图片文件
    async def play_tts(...)        # TTS 播放，默认 → send_voice
    async def stop_typing(...)     # 默认 no-op

    # 生命周期钩子
    async def on_processing_start(event)
    async def on_processing_complete(event, outcome)
```

### 8.2 媒体处理 (Media Handling)

**图片提取** `extract_images()`:
- 识别 Markdown `![alt](url)` 和 HTML `<img src="url">`
- 仅提取看起来像图片的 URL（`.png`, `.jpg`, `.gif`, `.webp`, `fal.media`, `fal-cdn`, `replicate.delivery`）
- 清理提取后的文本

**MEDIA 标签** `extract_media()`:
- 解析 `MEDIA:/path/to/file` 标签
- 支持 `[[audio_as_voice]]` 指令
- 处理带引号/反引号的路径

**本地文件检测** `extract_local_files()`:
- 检测 `/path/to/image.png` 或 `~/image.jpg` 格式
- 忽略代码块中的路径
- 用 `os.path.isfile()` 验证文件存在

**缓存系统**:
- `IMAGE_CACHE_DIR` — 图片缓存
- `AUDIO_CACHE_DIR` — 音频缓存
- `DOCUMENT_CACHE_DIR` — 文档缓存
- 支持 SSRF 防护 (`is_safe_url()`)
- 支持的文档类型: `.pdf`, `.md`, `.txt`, `.log`, `.zip`, `.docx`, `.xlsx`, `.pptx`

### 8.3 Typing 指示器 (Typing Indicators)

```python
async def _keep_typing(chat_id, interval=2.0):
    # 每 2 秒刷新 typing 状态
    # 支持暂停（审批等待时）
    # 取消时自动调用 stop_typing()
```

`pause_typing_for_chat()` / `resume_typing_for_chat()`: 线程安全（CPython GIL），可从同步 Agent 线程调用。

### 8.4 21 个平台适配器一览

| # | 适配器 | 平台 | 特色 |
|---|--------|------|------|
| 1 | `TelegramAdapter` | Telegram | MarkdownV2, 网络容错, 冲突检测, 最大(2825 行) |
| 2 | `DiscordAdapter` | Discord | `VoiceReceiver` 语音捕获, Opus 解码, 表情反应 |
| 3 | `SlackAdapter` | Slack | Socket Mode, 多工作区, Assistant API, `_ThreadContextCache` |
| 4 | `WhatsAppAdapter` | WhatsApp | Meta Business API |
| 5 | `SignalAdapter` | Signal | UUID 备选 ID |
| 6 | `WeComAdapter` | 企业微信 | 企业级消息 |
| 7 | `WecomCallbackAdapter` | 企业微信回调 | 回调模式 |
| 8 | `FeishuAdapter` | 飞书/Lark | 1042+ 行, 复杂 API |
| 9 | `WeixinAdapter` | 微信 | 1046+ 行 |
| 10 | `DingTalkAdapter` | 钉钉 | 阿里巴巴 |
| 11 | `QQAdapter` | QQ | 腾讯 QQ 机器人 |
| 12 | `MatrixAdapter` | Matrix | 开放协议 |
| 13 | `MattermostAdapter` | Mattermost | 开源 Slack 替代 |
| 14 | `HomeAssistantAdapter` | Home Assistant | 智能家居集成 |
| 15 | `BlueBubblesAdapter` | iMessage | macOS 桥接 |
| 16 | `SmsAdapter` | SMS | 短信 |
| 17 | `EmailAdapter` | Email | 电子邮件 |
| 18 | `APIServerAdapter` | REST API | HTTP API 接口 |
| 19 | `WebhookAdapter` | Webhook | 通用 Webhook |

---

## 9. 跨平台 Slash 命令 (Cross-platform Slash Commands)

### 9.1 命令解析

```python
def get_command() -> Optional[str]:
    # 1. 检查消息是否以 / 开头
    # 2. 提取第一个词并去掉 /
    # 3. 处理 Telegram @botname 后缀: /help@mybot → help
    # 4. 拒绝文件路径 (包含 /)
```

### 9.2 命令分发层次

1. **内建命令 (Built-in)**: 30+ 个，集中注册于 `hermes_cli/commands.py` 的 `COMMAND_REGISTRY`
2. **Quick Commands**: 用户在 `config.yaml` 中定义
   - `exec` 类型: 执行 shell 命令（30s 超时）
   - `alias` 类型: 映射到另一个命令
3. **Plugin Commands**: 插件注册的命令，通过 `get_plugin_command_handler()` 查找
4. **Skill Commands**: 技能目录中的命令，自动扫描 `~/.hermes/skills/`

### 9.3 内建命令完整列表

```
/new (aliases: /reset)     — 重置会话
/help                      — 帮助信息
/commands                  — 命令列表
/profile                   — 用户 Profile
/status                    — Agent 状态
/restart                   — 重启网关
/stop                      — 停止当前 Agent
/reasoning                 — 推理模式切换
/fast                      — 快速模式
/verbose                   — 详细模式
/yolo                      — YOLO 模式（跳过确认）
/model                     — 切换模型
/provider                  — 切换提供商
/personality               — 切换人格
/plan                      — 规划模式
/retry                     — 重试上次
/undo                      — 撤销上次
/sethome                   — 设置主频道
/compress                  — 压缩会话历史
/usage                     — Token 使用统计
/insights                  — 洞察
/reload-mcp                — 重新加载 MCP 服务器
/approve, /deny            — 危险命令审批
/update                    — 更新 Hermes
/debug                     — 调试信息
/title                     — 设置会话标题
/resume                    — 恢复历史会话
/branch                    — 分支会话
/rollback                  — 回滚会话
/background                — 后台任务
/btw                       — 插话 (side message)
/voice                     — 语音模式
/queue (alias: /q)         — 排队消息
```

### 9.4 Telegram 特殊处理

Telegram 的 BotCommand 菜单不支持连字符，命令名使用下划线。`resolve_skill_command_key()` 处理下划线到连字符的转换: `/claude_code` → `claude-code`。

---

## 10. Hook 系统 (Event Hooks)

```python
class HookRegistry:
    events = [
        "gateway:startup",
        "session:start", "session:end", "session:reset",
        "agent:start", "agent:step", "agent:end",
        "command:*"  # 通配符，匹配所有命令
    ]
```

- Hook 从 `~/.hermes/hooks/` 目录自动发现 (`HOOK.yaml` + `handler.py`)
- 内建 hook: `boot-md` — 启动时运行 `~/.hermes/BOOT.md`
- **Fire-and-forget**: 错误被捕获并记录，**永不阻塞**主流水线
- 支持同步和异步处理器

---

## 11. 设计模式 (Design Patterns)

### 11.1 适配器模式 (Adapter Pattern)
`BasePlatformAdapter` ABC 定义统一接口，21 个具体适配器各自实现平台特定逻辑。消息被归一化为 `MessageEvent`，响应通过 `SendResult` 返回。

### 11.2 模板方法 (Template Method)
`handle_message()` 在基类中定义了消息处理骨架（会话查找 → 去重 → 后台任务分发），子类通过 `on_processing_start()` / `on_processing_complete()` 钩子注入平台特定行为（如 Discord 添加 👀/✅/❌ 表情反应）。

### 11.3 策略模式 (Strategy Pattern)
消息发送使用策略层次: `send()` → `_send_with_retry()` → 纯文本回退。媒体发送使用 graceful degradation: `send_image()` 有默认回退到纯文本 URL。

### 11.4 发布-订阅 (Pub-Sub)
`HookRegistry` 实现了事件发布-订阅模式。事件名支持通配符匹配 (`command:*`)。

### 11.5 生产者-消费者 (Producer-Consumer)
`GatewayStreamConsumer` 使用 `queue.Queue` 实现跨线程的生产者-消费者模式: Agent 线程生产 delta token，异步任务消费并编辑平台消息。

### 11.6 哨兵模式 (Sentinel Pattern)
`_AGENT_PENDING_SENTINEL` 对象在 Agent 创建前占位，防止并发消息导致重复 Agent 实例化。类似的哨兵还有 `_DONE`, `_NEW_SEGMENT`, `_COMMENTARY` 用于流控制。

### 11.7 双写存储 (Dual-Write Storage)
`SessionStore` 同时写入 SQLite 和 JSONL，加载时选择消息更多的源。这是迁移期间的兼容策略。

### 11.8 平台锁互斥 (Platform Lock)
`_acquire_platform_lock()` 防止多个网关实例使用同一个平台 Token，避免 Telegram/Discord 的 webhook 冲突。

---

## 12. Platform 枚举 — 19 种平台

```python
class Platform(Enum):
    LOCAL = "local"
    TELEGRAM = "telegram"
    DISCORD = "discord"
    SLACK = "slack"
    WHATSAPP = "whatsapp"
    SIGNAL = "signal"
    WECOM = "wecom"
    WECOM_CALLBACK = "wecom_callback"  # 独立于 WECOM
    FEISHU = "feishu"
    WEIXIN = "weixin"
    DINGTALK = "dingtalk"
    QQ = "qq"
    MATRIX = "matrix"
    MATTERMOST = "mattermost"
    HOMEASSISTANT = "homeassistant"
    BLUEBUBBLES = "bluebubbles"
    SMS = "sms"
    EMAIL = "email"
    API_SERVER = "api_server"
    # WEBHOOK 可能也有，待确认
```

---

## 13. 配置系统 (Configuration)

### config.yaml 关键配置项

```yaml
# 会话重置策略
session_reset:
  mode: both          # daily | idle | both | none
  at_hour: 4          # 每日重置时间
  idle_minutes: 120   # 空闲超时分钟数
  notify: true        # 是否通知用户
  notify_exclude_platforms: [api_server, webhook]

# 用户自定义快捷命令
quick_commands:
  deploy:
    type: exec
    command: "deploy.sh"
  d:
    type: alias
    target: deploy

# STT 配置
stt:
  provider: whisper

# 流式响应
streaming:
  enabled: true
  edit_interval: 1.0

# 会话隔离
group_sessions_per_user: true   # 群组中每用户独立会话
thread_sessions_per_user: false # 线程中共享会话（默认）
```

---

## 14. 关键流程图：消息生命周期

```
[用户] → [Platform API] → [PlatformAdapter.handle_message()]
                                    │
                                    ▼
                          build_session_key()  →  check _active_sessions
                                    │
                                    ▼ (背景任务)
                          GatewayRunner._handle_message()
                                    │
                      ┌─────────────┼─────────────┐
                      ▼             ▼              ▼
               Auth Check    Slash Command?    Running Agent?
                      │             │              │
                      ▼             ▼              ▼
              unauthorized →    dispatch      interrupt()
              pairing code    /new /model...   or queue
                                    │
                                    ▼
                     _handle_message_with_agent()
                                    │
                      ┌─────────────┼─────────────┐
                      ▼             ▼              ▼
              get_or_create   build_context   load_transcript
              session()       prompt()        ()
                                    │
                                    ▼
                          _prepare_inbound_message_text()
                          (vision/STT/document enrichment)
                                    │
                                    ▼
                         AIAgent.run() [线程池]
                         + GatewayStreamConsumer [异步]
                                    │
                                    ▼
                    StreamConsumer.run() → adapter.send() / edit_message()
                                    │
                                    ▼
                          [Platform API] → [用户]
```

---

## 15. 网关特殊能力

### 15.1 照片批处理 (Photo Burst Merging)
`merge_pending_message_event()` 将短时间内连续到达的多张照片合并为一个事件，防止每张照片独立触发 Agent。

### 15.2 SSL 证书自动检测
`_ensure_ssl_certs()` 在任何 HTTP 库导入前运行，支持 NixOS 等非标准系统的证书路径发现。

### 15.3 Config → Env 桥接
`run.py` 启动时将 `config.yaml` 的配置桥接到环境变量，支持 terminal/auxiliary/agent 等嵌套配置的自动展开。

### 15.4 Discord 语音接收
`VoiceReceiver` 类解密 RTP 包（NaCl + DAVE E2EE），解码 Opus 到 PCM，按用户缓冲语音，检测静默后递交完整语句。

### 15.5 Slack Assistant API
`SlackAdapter` 支持 Slack 的 AI Assistant 生命周期事件 (`assistant_thread_started`, `assistant_thread_context_changed`)，multi-workspace (多工作区) 通过逗号分隔的 bot token 实现。

---

## 16. 安全机制 (Security)

- **用户授权**: `_is_user_authorized()` 检查用户白名单
- **配对系统 (Pairing)**: 未授权 DM 用户获得配对码，管理员批准后授权
- **SSRF 防护**: `is_safe_url()` 验证 URL 安全性
- **速率限制**: 配对请求有频率限制
- **平台锁**: 防止多实例 token 冲突
- **PII 脱敏**: 可选的用户 ID 哈希化

---

## 17. 待深入研究

- [ ] `run.py` 剩余 5000+ 行: 各具体命令处理器实现细节
- [ ] `_enrich_message_with_vision()`: 图片自动描述的完整流程
- [ ] Agent 调用的完整参数组装
- [ ] `channel_directory.py`: 频道目录构建
- [ ] 各平台适配器的 `connect()` 和 `send()` 具体实现差异
- [ ] `status.py`: 运行时状态写入和监控
- [ ] `pairing_store.py`: 配对码存储和验证
