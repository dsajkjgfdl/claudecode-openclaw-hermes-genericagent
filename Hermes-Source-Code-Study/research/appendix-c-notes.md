---
Task: TC-R
Title: "附录 C — 实战：接入一个新通信平台"
Keywords: 平台适配器开发, Platform Adapter Development
Status: DRAFT_DONE
Date: 2026-04-15
Sources:
  - gateway/platforms/base.py (2071 lines, ~82KB)
  - gateway/platforms/telegram.py (2825 lines, ~123KB — reference implementation)
  - gateway/platforms/discord.py (~24KB)
  - gateway/run.py (9654 lines, ~444KB — GatewayRunner._create_adapter)
  - gateway/session.py (1086 lines — SessionSource, build_session_key)
  - gateway/config.py (Platform enum, PlatformConfig)
---

# 附录 C — 实战：接入一个新通信平台（Research Notes）

## 1. 架构总览（Architecture Overview）

Hermes Gateway 采用 **适配器模式（Adapter Pattern）** 将多平台通信统一到一套抽象接口。所有平台适配器（Platform Adapter）继承自 `BasePlatformAdapter` 抽象基类（ABC），由 `GatewayRunner` 统一管理生命周期。

### 1.1 当前已接入的平台（19 个适配器子类）

| 适配器类 | 文件 | 平台 |
|---------|------|------|
| `TelegramAdapter` | telegram.py | Telegram |
| `DiscordAdapter` | discord.py | Discord |
| `WhatsAppAdapter` | whatsapp.py | WhatsApp |
| `SlackAdapter` | slack.py | Slack |
| `SignalAdapter` | signal.py | Signal |
| `MattermostAdapter` | mattermost.py | Mattermost |
| `MatrixAdapter` | matrix.py | Matrix |
| `HomeAssistantAdapter` | homeassistant.py | Home Assistant |
| `EmailAdapter` | email.py | Email (IMAP/SMTP) |
| `SmsAdapter` | sms.py | SMS (Twilio) |
| `DingTalkAdapter` | dingtalk.py | 钉钉 |
| `FeishuAdapter` | feishu.py | 飞书 |
| `WeComAdapter` | wecom.py | 企业微信 |
| `WecomCallbackAdapter` | wecom_callback.py | 企业微信回调 |
| `WeixinAdapter` | weixin.py | 微信 |
| `BlueBubblesAdapter` | bluebubbles.py | iMessage |
| `QQAdapter` | qqbot.py | QQ |
| `APIServerAdapter` | api_server.py | HTTP API |
| `WebhookAdapter` | webhook.py | Webhook |

---

## 2. 核心抽象基类：BasePlatformAdapter

**文件：** `gateway/platforms/base.py`，第 779–2071 行

### 2.1 构造函数 `__init__`（第 790–814 行）

```python
def __init__(self, config: PlatformConfig, platform: Platform):
    self.config = config
    self.platform = platform
    self._message_handler: Optional[MessageHandler] = None
    self._running = False
    self._fatal_error_code: Optional[str] = None
    self._fatal_error_message: Optional[str] = None
    self._fatal_error_retryable = True
    self._fatal_error_handler = None
    # 中断支持（Interrupt support）
    self._active_sessions: Dict[str, asyncio.Event] = {}
    self._pending_messages: Dict[str, MessageEvent] = {}
    self._background_tasks: set[asyncio.Task] = set()
    self._expected_cancelled_tasks: set[asyncio.Task] = set()
    self._busy_session_handler = None
    self._auto_tts_disabled_chats: set = set()
    self._typing_paused: set = set()
```

**关键字段说明：**

| 字段 | 用途 |
|------|------|
| `config` | `PlatformConfig` 实例，含 token/api_key/extra/reply_to_mode |
| `platform` | `Platform` 枚举值，标识平台类型 |
| `_message_handler` | 消息回调函数（Message Handler），由 `GatewayRunner` 注入 |
| `_running` | 连接状态标志（Connected flag） |
| `_active_sessions` | 当前正在处理的会话（Session key → asyncio.Event，支持中断） |
| `_pending_messages` | 中断时暂存的待处理消息（Pending messages during interrupt） |
| `_background_tasks` | 后台处理任务集合，shutdown 时统一取消 |
| `_typing_paused` | 暂停输入指示器的聊天集合（用于审批等待时） |

### 2.2 抽象方法（Abstract Methods — 必须实现）

共有 **4 个** `@abstractmethod` 方法，子类 **必须实现**：

#### 2.2.1 `connect()` → bool（第 942–949 行）

```python
@abstractmethod
async def connect(self) -> bool:
    """连接到平台并开始接收消息。成功返回 True。"""
```

**职责：**
- 初始化 SDK/HTTP 客户端
- 验证 token/凭据
- 注册消息回调（Message webhook/polling）
- 获取平台锁（`_acquire_platform_lock`）防止重复实例
- 调用 `_mark_connected()` 设置 `_running = True`

**参考实现（Telegram）：** 第 478–730 行
- 使用 `python-telegram-bot` 库的 `Application.builder().token(...).build()`
- 支持 polling 和 webhook 两种模式
- 调用 `_acquire_platform_lock('telegram-bot-token', self.config.token, ...)` 获取排他锁

#### 2.2.2 `disconnect()` → None（第 951–954 行）

```python
@abstractmethod
async def disconnect(self) -> None:
    """断开与平台的连接。"""
```

**职责：**
- 停止消息轮询/webhook 监听
- 关闭 HTTP 客户端/WebSocket
- 调用 `_release_platform_lock()` 释放平台锁
- 调用 `_mark_disconnected()` 设置 `_running = False`

#### 2.2.3 `send()` → SendResult（第 956–976 行）

```python
@abstractmethod
async def send(
    self,
    chat_id: str,
    content: str,
    reply_to: Optional[str] = None,
    metadata: Optional[Dict[str, Any]] = None
) -> SendResult:
    """发送消息到指定聊天。"""
```

**核心发送方法。** 所有其他发送方法（`send_image`, `send_voice` 等）默认回退（fallback）到此方法。

**`SendResult` 数据类（第 719–726 行）：**
```python
@dataclass
class SendResult:
    success: bool
    message_id: Optional[str] = None
    error: Optional[str] = None
    raw_response: Any = None
    retryable: bool = False  # True 标记可重试的瞬时网络错误
```

#### 2.2.4 `get_chat_info()` → Dict（第 1919–1928 行）

```python
@abstractmethod
async def get_chat_info(self, chat_id: str) -> Dict[str, Any]:
    """获取聊天/频道信息。返回至少包含 name 和 type 字段。"""
```

### 2.3 可选覆盖方法（Optional Override Methods）

这些方法在基类中有默认实现（通常是回退到 `send()` 或 no-op），子类可选择性覆盖：

| 方法 | 默认行为 | 覆盖原因 |
|------|---------|---------|
| `send_typing(chat_id, metadata)` | no-op | 发送输入指示器（typing indicator） |
| `stop_typing(chat_id)` | no-op | 停止持续性输入指示器 |
| `edit_message(chat_id, msg_id, content)` | 返回 `success=False` | 编辑已发送消息（用于流式响应） |
| `send_image(chat_id, image_url, caption)` | 发送 URL 文本 | 发送图片附件 |
| `send_animation(chat_id, url, caption)` | 调用 `send_image` | 发送 GIF 动画 |
| `send_voice(chat_id, audio_path, caption)` | 发送路径文本 | 发送语音消息 |
| `play_tts(chat_id, audio_path)` | 调用 `send_voice` | 自动 TTS 语音播放 |
| `send_video(chat_id, video_path, caption)` | 发送路径文本 | 发送视频 |
| `send_document(chat_id, file_path, caption)` | 发送路径文本 | 发送文件附件 |
| `send_image_file(chat_id, image_path)` | 发送路径文本 | 发送本地图片文件 |
| `format_message(content)` | 原文返回 | 平台特有格式化（如 Telegram MarkdownV2） |
| `on_processing_start(event)` | no-op | 处理开始钩子（如 Discord 添加 👀 reaction） |
| `on_processing_complete(event, outcome)` | no-op | 处理完成钩子（如 Discord 添加 ✅/❌ reaction） |

---

## 3. MessageEvent 消息事件格式

**文件：** `gateway/platforms/base.py`，第 655–717 行

```python
@dataclass
class MessageEvent:
    text: str                                    # 消息文本内容
    message_type: MessageType = MessageType.TEXT  # 消息类型
    source: SessionSource = None                 # 来源信息
    raw_message: Any = None                      # 原始平台消息对象
    message_id: Optional[str] = None             # 消息 ID
    media_urls: List[str] = field(default_factory=list)   # 媒体本地路径
    media_types: List[str] = field(default_factory=list)  # 媒体 MIME 类型
    reply_to_message_id: Optional[str] = None    # 引用消息 ID
    reply_to_text: Optional[str] = None          # 引用消息文本
    auto_skill: Optional[str | list[str]] = None # 自动加载技能
    internal: bool = False                        # 内部合成事件标志
    timestamp: datetime = field(default_factory=datetime.now)
```

### 3.1 MessageType 枚举（第 634–644 行）

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
    COMMAND = "command"  # /command 风格
```

### 3.2 SessionSource 来源数据（session.py 第 66–137 行）

```python
@dataclass
class SessionSource:
    platform: Platform          # 平台枚举
    chat_id: str                # 聊天 ID
    chat_name: Optional[str]    # 聊天名称
    chat_type: str = "dm"       # "dm", "group", "channel", "thread"
    user_id: Optional[str]      # 用户 ID
    user_name: Optional[str]    # 用户名
    thread_id: Optional[str]    # 帖子/线程 ID
    chat_topic: Optional[str]   # 频道主题描述
    user_id_alt: Optional[str]  # 备用 ID（如 Signal UUID）
    chat_id_alt: Optional[str]  # 备用群组 ID
```

---

## 4. 会话键生成（Session Key Generation）

**文件：** `gateway/session.py`，`build_session_key()` 函数（第 435–491 行）

会话键（Session Key）是一个确定性字符串，用于将来自不同平台的消息路由到同一个对话会话。格式：

```
agent:main:{platform}:{chat_type}:{chat_id}[:{thread_id}][:{user_id}]
```

**规则：**
- **DM（私聊）：** `agent:main:telegram:dm:{chat_id}[:{thread_id}]`
- **群组（Group）：** `agent:main:telegram:group:{chat_id}[:{user_id}]`
  - `group_sessions_per_user=True`（默认）时加 `user_id`，每个用户独立会话
  - `thread_sessions_per_user=False`（默认）时，有 `thread_id` 的群组线程 **共享** 会话（不加 user_id）

---

## 5. 平台锁机制（Platform Token Lock）

**文件：** `gateway/platforms/base.py`，第 880–907 行

```python
def _acquire_platform_lock(self, scope: str, identity: str, resource_desc: str) -> bool:
    from gateway.status import acquire_scoped_lock
    acquired, existing = acquire_scoped_lock(scope, identity, metadata={'platform': self.platform.value})
    if acquired:
        return True
    # 锁定失败 → 设置 fatal error（不可重试）
    self._set_fatal_error(f'{scope}_lock', message, retryable=False)
    return False

def _release_platform_lock(self) -> None:
    from gateway.status import release_scoped_lock
    release_scoped_lock(self._platform_lock_scope, identity)
```

**设计意图：** 防止同一 Bot token 被多个进程同时使用。例如 Telegram 的 long-polling 只允许一个客户端连接。

---

## 6. 消息处理管线（Message Handling Pipeline）

**文件：** `gateway/platforms/base.py`，`handle_message()` 第 1482–1570 行，`_process_message_background()` 第 1593–1863 行

### 6.1 消息入口流程

```
平台 SDK 回调 → 子类构造 MessageEvent → handle_message(event)
  ├── 计算 session_key (build_session_key)
  ├── 检查 _active_sessions 是否有正在处理的会话
  │   ├── 是 + 特权命令（approve/deny/stop/new 等）→ 直接内联分发
  │   ├── 是 + 照片消息 → 合并到 _pending_messages（不中断）
  │   ├── 是 + 普通消息 → 存入 _pending_messages + 设置中断事件
  │   └── 否 → 设置 _active_sessions[key] = Event() + 创建后台任务
  └── asyncio.create_task(_process_message_background)
```

### 6.2 后台处理流程

```
_process_message_background(event, session_key):
  1. 启动持续 typing 指示器（_keep_typing, 每 2s 刷新）
  2. 调用 on_processing_start 钩子
  3. 调用 _message_handler(event) — 这是 GatewayRunner 注入的回调
  4. 解析响应：
     a. extract_media() — 提取 MEDIA:<path> 标签
     b. extract_images() — 提取 Markdown/HTML 图片
     c. extract_local_files() — 自动检测本地文件路径
  5. 自动 TTS（如果是语音消息且未禁用）
  6. 发送文本部分（_send_with_retry，含重试 + plain-text 回退）
  7. 发送图片/视频/音频/文档附件
  8. 调用 on_processing_complete 钩子
  9. 检查 _pending_messages 中是否有中断期间排队的消息
  10. 清理：取消 typing 任务 + 移除 _active_sessions
```

### 6.3 重试机制（_send_with_retry，第 1384–1464 行）

- 首次发送失败后，检查错误是否为 **瞬时网络错误**（retryable）
- 如果是 → 指数退避重试（2s, 4s...），最多 `max_retries=2` 次
- 如果非网络错误 → 尝试 **plain-text 回退**（去掉格式化）
- 超时错误（timed out）**不重试**（可能已发送，避免重复投递）
- 所有重试失败 → 发送 ⚠️ 投递失败通知给用户

---

## 7. 媒体支持（Media Support）

### 7.1 媒体提取链

基类提供三个静态方法，在发送前自动从响应文本中提取媒体：

1. **`extract_media(content)`**（第 1192 行）：提取 `MEDIA:/path/to/file` 标签 + `[[audio_as_voice]]` 指令
2. **`extract_images(content)`**（第 1051 行）：提取 `![alt](url)` 和 `<img src="url">` 格式的图片 URL
3. **`extract_local_files(content)`**（第 1234 行）：自动检测 `/path/to/file.png` 格式的本地文件路径
   - 忽略代码块中的路径
   - 使用 `os.path.isfile()` 验证文件存在
   - 防止 SSRF/URL 误匹配

### 7.2 媒体文件路由

```python
# _process_message_background 中，按文件扩展名路由到不同发送方法：
_AUDIO_EXTS = {'.ogg', '.opus', '.mp3', '.wav', '.m4a'}  → send_voice()
_VIDEO_EXTS = {'.mp4', '.mov', '.avi', '.mkv', '.webm'}  → send_video()
_IMAGE_EXTS = {'.jpg', '.jpeg', '.png', '.webp', '.gif'}  → send_image_file()
其他                                                       → send_document()
```

### 7.3 消息分割（Message Truncation）

`truncate_message(content, max_length=4096)` 静态方法（第 1941–2071 行）：
- 按自然边界（换行/空格）分割长消息
- **保持代码块完整**：分割在代码块内时，自动关闭/重新打开 ` ``` ` 围栏
- 避免分割在内联代码 `` ` `` 中间（奇数反引号检测）
- 支持自定义长度函数 `len_fn`（Telegram 使用 UTF-16 编码长度计算）
- 多段消息添加 `(1/3)` 指示器

---

## 8. 连接/断开生命周期（Connect/Disconnect Lifecycle）

### 8.1 生命周期状态管理

```python
# 基类提供的状态管理方法：
_mark_connected()      # _running=True, 清除 fatal_error, 写入 status 文件
_mark_disconnected()   # _running=False, 写入 status 文件
_set_fatal_error(code, message, retryable=bool)  # 致命错误，写入 status 文件
_notify_fatal_error()  # 通知外层错误处理器
```

### 8.2 GatewayRunner 中的适配器生命周期

**文件：** `gateway/run.py`

```
1. GatewayRunner.__init__()
   └── adapters: Dict[Platform, BasePlatformAdapter] = {}

2. _create_adapter(platform, config) → BasePlatformAdapter
   ├── 根据 Platform 枚举值 import 对应模块
   ├── 调用 check_xxx_requirements() 验证依赖
   └── 返回 XxxAdapter(config) 实例

3. 启动流程:
   ├── adapter.set_message_handler(handler)  # 注入消息处理回调
   ├── adapter.connect()                      # 连接平台
   └── self.adapters[platform] = adapter      # 注册到适配器字典

4. 关闭流程:
   ├── adapter.cancel_background_tasks()      # 取消所有后台任务
   └── adapter.disconnect()                   # 断开连接
```

### 8.3 致命错误恢复

```python
# 致命错误分类：
# - retryable=True → GatewayRunner 将尝试后台重连
# - retryable=False → 需要人工干预（如 token 冲突）

# _failed_platforms 跟踪重连状态：
self._failed_platforms: Dict[Platform, Dict] = {
    Platform.TELEGRAM: {
        "config": platform_config,
        "attempts": 0,
        "next_retry": float  # timestamp
    }
}
```

---

## 9. PlatformConfig 配置（config.py 第 144–187 行）

```python
@dataclass
class PlatformConfig:
    enabled: bool = False
    token: Optional[str] = None       # Bot token
    api_key: Optional[str] = None     # API key（与 token 不同时）
    home_channel: Optional[HomeChannel] = None  # 默认投递频道
    reply_to_mode: str = "first"      # "off" / "first" / "all"
    extra: Dict[str, Any] = field(default_factory=dict)  # 平台特有配置
```

### 9.1 Platform 枚举（config.py 第 48–69 行）

```python
class Platform(Enum):
    LOCAL = "local"
    TELEGRAM = "telegram"
    DISCORD = "discord"
    WHATSAPP = "whatsapp"
    SLACK = "slack"
    SIGNAL = "signal"
    MATTERMOST = "mattermost"
    MATRIX = "matrix"
    HOMEASSISTANT = "homeassistant"
    EMAIL = "email"
    SMS = "sms"
    DINGTALK = "dingtalk"
    API_SERVER = "api_server"
    WEBHOOK = "webhook"
    FEISHU = "feishu"
    WECOM = "wecom"
    WECOM_CALLBACK = "wecom_callback"
    WEIXIN = "weixin"
    BLUEBUBBLES = "bluebubbles"
    QQBOT = "qqbot"
```

---

## 10. 接入新平台：分步教程（Step-by-Step Tutorial）

### Step 1：添加 Platform 枚举值

```python
# gateway/config.py
class Platform(Enum):
    ...
    MY_PLATFORM = "my_platform"  # 新增
```

### Step 2：创建适配器文件

```python
# gateway/platforms/my_platform.py

import logging
from typing import Dict, Any, Optional
from gateway.config import Platform, PlatformConfig
from gateway.platforms.base import (
    BasePlatformAdapter, MessageEvent, MessageType, SendResult
)
from gateway.session import SessionSource

logger = logging.getLogger(__name__)

# 标准模式：顶层尝试导入 SDK，失败则标记不可用
try:
    import my_platform_sdk
    MY_PLATFORM_AVAILABLE = True
except ImportError:
    MY_PLATFORM_AVAILABLE = False


def check_my_platform_requirements() -> bool:
    """检查平台依赖是否已安装。"""
    return MY_PLATFORM_AVAILABLE


class MyPlatformAdapter(BasePlatformAdapter):
    """适配器实现。"""

    MAX_MESSAGE_LENGTH = 4096  # 平台消息长度限制

    def __init__(self, config: PlatformConfig):
        super().__init__(config, Platform.MY_PLATFORM)
        self._client = None

    # ── 必须实现的抽象方法 ──

    async def connect(self) -> bool:
        if not MY_PLATFORM_AVAILABLE:
            logger.error("[%s] SDK not installed", self.name)
            return False
        if not self.config.token:
            logger.error("[%s] No token configured", self.name)
            return False

        # 1. 获取平台锁
        if not self._acquire_platform_lock(
            'my-platform-token', self.config.token, 'My Platform bot token'
        ):
            return False

        try:
            # 2. 初始化 SDK 客户端
            self._client = my_platform_sdk.Client(self.config.token)

            # 3. 注册消息回调
            self._client.on_message(self._on_message)

            # 4. 启动连接
            await self._client.start()

            # 5. 标记连接成功
            self._mark_connected()
            logger.info("[%s] Connected successfully", self.name)
            return True
        except Exception as e:
            logger.error("[%s] Failed to connect: %s", self.name, e)
            self._set_fatal_error("connect_failed", str(e), retryable=True)
            await self._notify_fatal_error()
            return False

    async def disconnect(self) -> None:
        try:
            await self.cancel_background_tasks()
            if self._client:
                await self._client.stop()
        finally:
            self._release_platform_lock()
            self._mark_disconnected()
            logger.info("[%s] Disconnected", self.name)

    async def send(
        self,
        chat_id: str,
        content: str,
        reply_to: Optional[str] = None,
        metadata: Optional[Dict[str, Any]] = None,
    ) -> SendResult:
        try:
            # 格式化 + 分割
            formatted = self.format_message(content)
            chunks = self.truncate_message(formatted, self.MAX_MESSAGE_LENGTH)

            last_msg_id = None
            for chunk in chunks:
                result = await self._client.send_message(
                    chat_id=chat_id,
                    text=chunk,
                    reply_to=reply_to if last_msg_id is None else None,
                )
                last_msg_id = result.id

            return SendResult(success=True, message_id=str(last_msg_id))
        except Exception as e:
            return SendResult(success=False, error=str(e))

    async def get_chat_info(self, chat_id: str) -> Dict[str, Any]:
        chat = await self._client.get_chat(chat_id)
        return {
            "name": chat.name,
            "type": "dm" if chat.is_direct else "group",
        }

    # ── 消息接收回调 ──

    async def _on_message(self, raw_msg):
        """SDK 消息回调 → 构造 MessageEvent → 分发。"""
        source = self.build_source(
            chat_id=str(raw_msg.chat_id),
            chat_name=raw_msg.chat_name,
            chat_type="dm" if raw_msg.is_direct else "group",
            user_id=str(raw_msg.author_id),
            user_name=raw_msg.author_name,
            thread_id=str(raw_msg.thread_id) if raw_msg.thread_id else None,
        )

        event = MessageEvent(
            text=raw_msg.text,
            message_type=MessageType.TEXT,
            source=source,
            raw_message=raw_msg,
            message_id=str(raw_msg.id),
        )

        # 分发到基类的 handle_message（自动管理会话+中断+typing）
        await self.handle_message(event)

    # ── 可选覆盖 ──

    async def send_typing(self, chat_id: str, metadata=None) -> None:
        await self._client.trigger_typing(chat_id)

    async def send_image(
        self, chat_id, image_url, caption=None, reply_to=None, metadata=None
    ) -> SendResult:
        try:
            await self._client.send_photo(chat_id, url=image_url, caption=caption)
            return SendResult(success=True)
        except Exception as e:
            return SendResult(success=False, error=str(e))
```

### Step 3：注册到 GatewayRunner

```python
# gateway/run.py → _create_adapter() 方法中添加：
elif platform == Platform.MY_PLATFORM:
    from gateway.platforms.my_platform import (
        MyPlatformAdapter, check_my_platform_requirements
    )
    if not check_my_platform_requirements():
        logger.warning("My Platform: SDK not installed")
        return None
    return MyPlatformAdapter(config)
```

### Step 4：添加用户授权映射

```python
# gateway/run.py → _is_user_authorized() 方法中的映射字典：
platform_env_map = {
    ...
    Platform.MY_PLATFORM: "MY_PLATFORM_ALLOWED_USERS",
}
platform_allow_all_map = {
    ...
    Platform.MY_PLATFORM: "MY_PLATFORM_ALLOW_ALL_USERS",
}
```

### Step 5：配置环境变量

```bash
# .env 文件
MY_PLATFORM_TOKEN=xxx
MY_PLATFORM_ALLOWED_USERS=user1,user2
# 或 MY_PLATFORM_ALLOW_ALL_USERS=true
```

### Step 6：配置 YAML（可选）

```yaml
# config.yaml → platforms 节
platforms:
  my_platform:
    enabled: true
    token: "${MY_PLATFORM_TOKEN}"
    home_channel:
      chat_id: "123456"
      name: "Main Channel"
    extra:
      group_sessions_per_user: true
```

---

## 11. 测试方法（Testing Approach）

### 11.1 单元测试模式

```python
# 构造最小适配器实例
config = PlatformConfig(enabled=True, token="test-token")
adapter = MyPlatformAdapter(config)

# Mock _message_handler
async def mock_handler(event):
    return "Hello!"

adapter.set_message_handler(mock_handler)
```

### 11.2 集成测试要点

1. **connect/disconnect 生命周期**：验证 `_running` 状态变化
2. **消息往返**：构造 MessageEvent → handle_message → 验证 send 被调用
3. **中断支持**：两条消息快速到达，验证第二条进入 `_pending_messages`
4. **平台锁**：两个实例使用同一 token，验证第二个获取锁失败
5. **消息分割**：发送超长消息，验证正确分割 + 代码块保持完整
6. **重试逻辑**：Mock send 返回 retryable 错误，验证重试 + 回退

### 11.3 手动测试

```bash
hermes start --platform my_platform
# 然后在平台上发送消息，观察 gateway 日志
```

---

## 12. 设计模式总结（Design Patterns）

| 模式 | 应用场景 |
|------|---------|
| **模板方法（Template Method）** | `handle_message` → `_process_message_background` 定义完整流程，子类仅覆盖 `connect/send` 等步骤 |
| **策略（Strategy）** | `_message_handler` 回调由外部（GatewayRunner）注入，适配器不关心业务逻辑 |
| **适配器（Adapter）** | 每个子类将平台特有 SDK 适配到统一 `MessageEvent` / `SendResult` 接口 |
| **工厂方法（Factory Method）** | `GatewayRunner._create_adapter()` 根据 `Platform` 枚举创建适配器实例 |
| **观察者（Observer）** | 生命周期钩子 `on_processing_start/complete`，子类可选订阅 |
| **排他锁（Mutex/Lock）** | `_acquire_platform_lock` 防止同一 token 被多进程使用 |
| **指数退避重试（Exponential Backoff）** | `_send_with_retry` 和 Telegram 网络错误恢复 |
| **优雅降级（Graceful Degradation）** | 媒体方法默认回退到 `send()` 文本消息；格式化失败回退 plain-text |
| **命令绕行（Command Bypass）** | 特权命令 (`/stop`, `/approve`) 绕过 active-session 守卫直接分发 |

---

## 13. 关键设计决策与注意事项

### 13.1 为什么 `handle_message` 同步设置 `_active_sessions`？

```python
# 第 1557-1558 行：
# Mark session as active BEFORE spawning background task to close
# the race window where a second message arriving before the task
# starts would also pass the _active_sessions check and spawn a
# duplicate task.
self._active_sessions[session_key] = asyncio.Event()
task = asyncio.create_task(self._process_message_background(...))
```

这是 grammY sequentialize / aiogram EventIsolation 模式的体现——在异步事件循环的同一 tick 内同步地设置守卫标志，确保快速连续到达的消息不会绕过检查。

### 13.2 照片合并（Photo Burst Merging）

多张照片在短时间内到达时（Telegram 相册/iPhone Live Photo），通过 `merge_pending_message_event()` 合并 `media_urls`，而非中断当前处理。

### 13.3 PII 脱敏（PII Redaction）

`build_session_context_prompt()` 在构建系统提示词时，对 WhatsApp/Signal/Telegram 等平台的用户 ID 进行哈希处理（SHA256 前 12 字符），但 Discord 除外（因为 mention 需要原始 ID）。

### 13.4 typing 暂停机制

在等待危险命令审批时（`/approve`, `/deny`），通过 `pause_typing_for_chat()` 暂停 typing 指示器。这对 Slack 尤其重要——Slack 的 `assistant_threads_setStatus` 会禁用输入框，暂停 typing 才能让用户输入审批命令。

---

## 14. 文件清单与代码行数

| 文件 | 行数 | 大小 | 核心内容 |
|------|------|------|---------|
| `base.py` | 2071 | 82KB | BasePlatformAdapter ABC + MessageEvent + SendResult + 媒体提取 + 消息分割 |
| `telegram.py` | 2825 | 123KB | 参考实现（polling/webhook, MarkdownV2, 相册合并, forum topics） |
| `discord.py` | ~600 | 24KB | Discord.py 集成（reaction hooks, embed formatting） |
| `config.py` | 1160 | 53KB | Platform 枚举, PlatformConfig, SessionResetPolicy |
| `session.py` | 1086 | 42KB | SessionSource, build_session_key, SessionStore |
| `run.py` | 9654 | 444KB | GatewayRunner 总控（创建/连接/路由/授权） |
