---
title: "实战：接入一个新通信平台"
appendix: C
status: DRAFT
---

# 附录 C　实战：接入一个新通信平台

> **目标读者：** 希望为 Hermes Agent 添加新通信平台（如 LINE、Rocket.Chat、自研 IM）的开发者。
>
> **学完本附录你将获得：** 从零完成一个可投入使用的 `PlatformAdapter`，涵盖消息收发、媒体处理、会话管理与测试验证。

---

## C.1　架构速览——"一个基类统治一切"

Hermes Gateway 采用 **适配器模式（Adapter Pattern）** 将 19 个通信平台统一到一套抽象接口。所有适配器继承自同一个抽象基类 `BasePlatformAdapter`（定义在 `gateway/platforms/base.py`，约 2071 行），由 `GatewayRunner`（`gateway/run.py`）统一管理创建、连接与销毁。

```
┌─────────────────────────────────────────────────┐
│                 GatewayRunner                    │
│  ┌────────────┐ ┌───────────┐ ┌──────────────┐  │
│  │ Telegram   │ │ Discord   │ │ MyPlatform   │  │
│  │ Adapter    │ │ Adapter   │ │ Adapter (新)  │  │
│  └─────┬──────┘ └─────┬─────┘ └──────┬───────┘  │
│        │              │              │           │
│        └──────────────┼──────────────┘           │
│                       ▼                          │
│            BasePlatformAdapter (ABC)              │
│   connect() | disconnect() | send() | get_chat_info() │
└─────────────────────────────────────────────────┘
```

**关键数据流：**

```
平台 SDK 回调
  → 子类构造 MessageEvent
  → handle_message()              ← 基类提供
    → build_session_key()         ← 会话路由
    → _process_message_background()
      → _message_handler()        ← GatewayRunner 注入的业务回调
      → send() / send_image() …  ← 子类实现的发送方法
```

你需要做的，就是 **填充子类中的 4 个抽象方法 + 1 个消息接收回调**，其余流程（会话管理、中断处理、typing 指示器、重试、消息分割）全由基类接管。

---

## C.2　你需要实现什么？

### C.2.1　4 个必须实现的抽象方法

| 方法签名 | 职责 |
|---------|------|
| `async def connect(self) -> bool` | 初始化 SDK、验证凭据、注册回调、获取平台锁、标记连接 |
| `async def disconnect(self) -> None` | 停止监听、关闭客户端、释放平台锁、标记断开 |
| `async def send(self, chat_id, content, reply_to?, metadata?) -> SendResult` | 发送文本消息到指定聊天 |
| `async def get_chat_info(self, chat_id) -> Dict` | 返回聊天名称和类型（至少 `name` + `type`） |

### C.2.2　可选覆盖方法（按需增强）

| 方法 | 默认行为 | 何时覆盖 |
|------|---------|---------|
| `send_typing(chat_id, metadata)` | 静默跳过 | 平台支持"对方正在输入"时 |
| `stop_typing(chat_id)` | 静默跳过 | 平台的 typing 是持续性状态时 |
| `edit_message(chat_id, msg_id, content)` | 返回失败 | 支持编辑已发消息（流式响应） |
| `send_image(chat_id, url, caption)` | 发送 URL 文本 | 支持图片嵌入 |
| `send_voice(chat_id, path, caption)` | 发送路径文本 | 支持语音消息 |
| `send_video(chat_id, path, caption)` | 发送路径文本 | 支持视频消息 |
| `send_document(chat_id, path, caption)` | 发送路径文本 | 支持文件附件 |
| `format_message(content)` | 原文返回 | 平台有特殊格式化（如 MarkdownV2） |
| `on_processing_start(event)` | 静默跳过 | 需要处理开始钩子（如 Discord 的 👀 反应） |
| `on_processing_complete(event, outcome)` | 静默跳过 | 需要处理完成钩子（如 ✅/❌ 反应） |

### C.2.3　核心数据结构

**MessageEvent** — 消息事件（`base.py` 第 655–717 行）：

```python
@dataclass
class MessageEvent:
    text: str                                    # 消息文本
    message_type: MessageType = MessageType.TEXT  # TEXT/PHOTO/VOICE/COMMAND...
    source: SessionSource = None                 # 来源信息
    raw_message: Any = None                      # 平台原始消息对象
    message_id: Optional[str] = None             # 消息 ID
    media_urls: List[str] = field(default_factory=list)   # 媒体本地路径
    media_types: List[str] = field(default_factory=list)  # MIME 类型
    reply_to_message_id: Optional[str] = None    # 引用消息 ID
    reply_to_text: Optional[str] = None          # 引用消息文本
    timestamp: datetime = field(default_factory=datetime.now)
```

**SendResult** — 发送结果：

```python
@dataclass
class SendResult:
    success: bool
    message_id: Optional[str] = None
    error: Optional[str] = None
    raw_response: Any = None
    retryable: bool = False  # 标记可重试的瞬时网络错误
```

**SessionSource** — 会话来源（`session.py`）：

```python
@dataclass
class SessionSource:
    platform: Platform          # 平台枚举
    chat_id: str                # 聊天 ID
    chat_name: Optional[str]    # 聊天名称
    chat_type: str = "dm"       # "dm" / "group" / "channel" / "thread"
    user_id: Optional[str]      # 用户 ID
    user_name: Optional[str]    # 用户名
    thread_id: Optional[str]    # 帖子/线程 ID
```

---

## C.3　六步接入指南

下面以接入一个虚构平台 **"MyPlatform"** 为例，完整演示从零到可用的全过程。

### Step 1：添加 Platform 枚举值

在 `gateway/config.py` 的 `Platform` 枚举中添加新值：

```python
# gateway/config.py
class Platform(Enum):
    LOCAL = "local"
    TELEGRAM = "telegram"
    DISCORD = "discord"
    # ... 其他平台 ...
    MY_PLATFORM = "my_platform"  # ← 新增
```

> **命名约定：** 枚举名全大写下划线，值全小写下划线。这个值会出现在会话键和日志中。

### Step 2：创建适配器文件（核心工作）

创建 `gateway/platforms/my_platform.py`：

```python
"""MyPlatform 适配器 — 完整参考实现。"""

import logging
from typing import Dict, Any, Optional

from gateway.config import Platform, PlatformConfig
from gateway.platforms.base import (
    BasePlatformAdapter, MessageEvent, MessageType, SendResult
)
from gateway.session import SessionSource

logger = logging.getLogger(__name__)

# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# 标准模式：顶层尝试导入 SDK，失败则标记不可用
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
try:
    import my_platform_sdk
    MY_PLATFORM_AVAILABLE = True
except ImportError:
    MY_PLATFORM_AVAILABLE = False


def check_my_platform_requirements() -> bool:
    """检查平台依赖是否已安装。GatewayRunner 在创建适配器前调用。"""
    return MY_PLATFORM_AVAILABLE


class MyPlatformAdapter(BasePlatformAdapter):
    """MyPlatform 通信平台适配器。"""

    MAX_MESSAGE_LENGTH = 4096  # 平台消息长度限制

    def __init__(self, config: PlatformConfig):
        super().__init__(config, Platform.MY_PLATFORM)
        self._client = None

    # ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    # 抽象方法 #1: connect — 连接到平台
    # ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    async def connect(self) -> bool:
        # 前置检查
        if not MY_PLATFORM_AVAILABLE:
            logger.error("[MyPlatform] SDK 未安装: pip install my-platform-sdk")
            return False
        if not self.config.token:
            logger.error("[MyPlatform] 未配置 token")
            return False

        # 1. 获取平台锁 —— 防止同一 token 被多进程使用
        if not self._acquire_platform_lock(
            'my-platform-token',        # 锁作用域
            self.config.token,          # 锁标识（token 值）
            'My Platform bot token'     # 人类可读描述
        ):
            return False  # 锁获取失败，_set_fatal_error 已在内部设置

        try:
            # 2. 初始化 SDK 客户端
            self._client = my_platform_sdk.Client(self.config.token)

            # 3. 注册消息回调
            self._client.on_message(self._on_message)

            # 4. 启动连接（SDK 特有逻辑）
            await self._client.start()

            # 5. 标记连接成功
            self._mark_connected()
            logger.info("[MyPlatform] ✓ 连接成功")
            return True

        except Exception as e:
            logger.error("[MyPlatform] 连接失败: %s", e)
            # retryable=True → GatewayRunner 将尝试后台重连
            self._set_fatal_error("connect_failed", str(e), retryable=True)
            await self._notify_fatal_error()
            return False

    # ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    # 抽象方法 #2: disconnect — 断开连接
    # ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    async def disconnect(self) -> None:
        try:
            # 先取消所有后台处理任务
            await self.cancel_background_tasks()
            if self._client:
                await self._client.stop()
        finally:
            # 无论如何都要释放锁并标记断开
            self._release_platform_lock()
            self._mark_disconnected()
            logger.info("[MyPlatform] 已断开连接")

    # ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    # 抽象方法 #3: send — 发送消息
    # ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    async def send(
        self,
        chat_id: str,
        content: str,
        reply_to: Optional[str] = None,
        metadata: Optional[Dict[str, Any]] = None,
    ) -> SendResult:
        try:
            # 1. 格式化（如平台有特殊语法，覆盖 format_message）
            formatted = self.format_message(content)

            # 2. 分割超长消息 — 基类提供，自动保持代码块完整
            chunks = self.truncate_message(formatted, self.MAX_MESSAGE_LENGTH)

            last_msg_id = None
            for chunk in chunks:
                result = await self._client.send_message(
                    chat_id=chat_id,
                    text=chunk,
                    # 仅第一段消息带回复引用
                    reply_to=reply_to if last_msg_id is None else None,
                )
                last_msg_id = result.id

            return SendResult(success=True, message_id=str(last_msg_id))

        except my_platform_sdk.NetworkError as e:
            # 网络错误标记为可重试
            return SendResult(
                success=False, error=str(e), retryable=True
            )
        except Exception as e:
            return SendResult(success=False, error=str(e))

    # ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    # 抽象方法 #4: get_chat_info — 获取聊天信息
    # ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    async def get_chat_info(self, chat_id: str) -> Dict[str, Any]:
        chat = await self._client.get_chat(chat_id)
        return {
            "name": chat.name,
            "type": "dm" if chat.is_direct else "group",
        }

    # ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    # 消息接收回调 — 平台消息 → MessageEvent
    # ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    async def _on_message(self, raw_msg):
        """SDK 消息回调：将平台原始消息转换为 MessageEvent。"""

        # 构造 SessionSource — 告诉基类"这条消息从哪来"
        source = self.build_source(
            chat_id=str(raw_msg.chat_id),
            chat_name=raw_msg.chat_name,
            chat_type="dm" if raw_msg.is_direct else "group",
            user_id=str(raw_msg.author_id),
            user_name=raw_msg.author_name,
            thread_id=(
                str(raw_msg.thread_id) if raw_msg.thread_id else None
            ),
        )

        # 构造统一消息事件
        event = MessageEvent(
            text=raw_msg.text,
            message_type=MessageType.TEXT,
            source=source,
            raw_message=raw_msg,
            message_id=str(raw_msg.id),
        )

        # 分发到基类 handle_message
        # 基类自动处理：会话路由、中断管理、typing、重试
        await self.handle_message(event)

    # ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    # 可选覆盖 — 按需增强
    # ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    async def send_typing(self, chat_id: str, metadata=None) -> None:
        """发送"正在输入"指示器。"""
        await self._client.trigger_typing(chat_id)

    async def send_image(
        self, chat_id, image_url, caption=None,
        reply_to=None, metadata=None
    ) -> SendResult:
        """发送图片消息。"""
        try:
            await self._client.send_photo(
                chat_id, url=image_url, caption=caption
            )
            return SendResult(success=True)
        except Exception as e:
            return SendResult(success=False, error=str(e))
```

### Step 3：注册到 GatewayRunner

在 `gateway/run.py` 的 `_create_adapter()` 工厂方法中添加分支：

```python
# gateway/run.py → _create_adapter()
elif platform == Platform.MY_PLATFORM:
    from gateway.platforms.my_platform import (
        MyPlatformAdapter, check_my_platform_requirements
    )
    if not check_my_platform_requirements():
        logger.warning("MyPlatform: SDK 未安装，跳过")
        return None
    return MyPlatformAdapter(config)
```

> **延迟导入模式：** 注意 `import` 在 `elif` 分支内部。这是 Hermes 的标准做法——只有真正启用的平台才会加载对应 SDK，避免所有平台的依赖成为硬性要求。

### Step 4：添加用户授权映射

在 `gateway/run.py` 的 `_is_user_authorized()` 方法中，将新平台映射到环境变量：

```python
# 允许列表映射
platform_env_map = {
    Platform.TELEGRAM: "TELEGRAM_ALLOWED_USERS",
    Platform.DISCORD: "DISCORD_ALLOWED_USERS",
    # ... 其他平台 ...
    Platform.MY_PLATFORM: "MY_PLATFORM_ALLOWED_USERS",  # ← 新增
}

# 全部放行映射
platform_allow_all_map = {
    Platform.TELEGRAM: "TELEGRAM_ALLOW_ALL_USERS",
    # ...
    Platform.MY_PLATFORM: "MY_PLATFORM_ALLOW_ALL_USERS",  # ← 新增
}
```

### Step 5：配置环境变量

```bash
# .env 文件
MY_PLATFORM_TOKEN=your-bot-token-here
MY_PLATFORM_ALLOWED_USERS=user_id_1,user_id_2
# 或开放所有用户（仅限开发测试！）：
# MY_PLATFORM_ALLOW_ALL_USERS=true
```

### Step 6：配置 YAML（可选）

```yaml
# config.yaml → platforms 段
platforms:
  my_platform:
    enabled: true
    token: "${MY_PLATFORM_TOKEN}"
    home_channel:
      chat_id: "123456"
      name: "Main Channel"
    reply_to_mode: "first"   # "off" / "first" / "all"
    extra:
      group_sessions_per_user: true   # 群组中每用户独立会话
```

---

## C.4　深入理解——基类帮你做了什么

### C.4.1　会话键与消息路由

当你的 `_on_message` 回调调用 `handle_message(event)` 后，基类会用 `build_session_key()` 生成一个确定性会话键：

```
agent:main:{platform}:{chat_type}:{chat_id}[:{thread_id}][:{user_id}]
```

路由规则：

- **私聊（DM）：** `agent:main:my_platform:dm:12345`
- **群组（Group）：** `agent:main:my_platform:group:67890:user42`
  - 默认每个用户在群组中有独立会话（`group_sessions_per_user=True`）
  - 群组中的线程（thread）默认共享会话，不按用户分隔

### C.4.2　中断与排队机制

当同一会话正在处理消息时，新消息到达的处理逻辑：

```python
# handle_message() 中的关键判断（简化）：
if session_key in self._active_sessions:
    if is_privileged_command(event):   # /stop, /approve, /deny, /new
        # 直接内联分发，不排队
        await self._dispatch_privileged(event)
    elif event.message_type == MessageType.PHOTO:
        # 合并到 pending_messages 的 media_urls（照片合并）
        self.merge_pending_message_event(session_key, event)
    else:
        # 存入待处理队列 + 设置中断信号
        self._pending_messages[session_key] = event
        self._active_sessions[session_key].set()  # 触发中断
else:
    # 首条消息 → 同步标记活跃 + 创建后台任务
    self._active_sessions[session_key] = asyncio.Event()
    task = asyncio.create_task(
        self._process_message_background(event, session_key)
    )
```

**为什么 `_active_sessions` 在同步上下文中设置？** 因为 Python asyncio 的事件循环在同一 tick 内不会被中断。如果在 `create_task` 之后才设置，两条快速连续到达的消息可能都通过 `if` 检查，产生重复任务。

### C.4.3　消息分割——truncate_message

基类的 `truncate_message(content, max_length=4096)` 提供智能消息分割：

- 按自然边界（换行 → 空格）分割
- **保持代码块完整**：如果分割点在 ` ``` ` 围栏内，自动关闭旧块、在下一段重新打开
- 避免在内联代码 `` ` `` 中间断开
- 多段消息自动添加 `(1/3)` 页码指示器
- 支持自定义长度函数 `len_fn`（Telegram 使用 UTF-16 编码长度计算）

你只需在 `send()` 中调用即可：

```python
chunks = self.truncate_message(formatted, self.MAX_MESSAGE_LENGTH)
for chunk in chunks:
    await self._client.send_message(chat_id, chunk)
```

### C.4.4　重试与降级

基类的 `_send_with_retry()` 提供完整的发送容错：

1. 首次发送失败后，检查 `SendResult.retryable`
2. 可重试 → 指数退避重试（2s、4s…），最多 2 次
3. 不可重试 → 尝试 **plain-text 降级**（去掉 Markdown 格式化重发）
4. 超时错误不重试（消息可能已到达，避免重复投递）
5. 全部失败 → 发送 ⚠️ 投递失败通知

### C.4.5　媒体自动提取

基类在发送前自动从 AI 响应文本中提取媒体内容：

| 提取器 | 识别格式 | 示例 |
|--------|---------|------|
| `extract_media()` | `MEDIA:/path/to/file` | `MEDIA:/tmp/chart.png` |
| `extract_images()` | `![alt](url)` / `<img src="url">` | `![](https://example.com/img.png)` |
| `extract_local_files()` | 自动检测本地文件路径 | `/home/user/output.pdf` |

提取后按扩展名路由到对应发送方法：

| 扩展名 | 调用方法 |
|--------|---------|
| `.jpg`, `.png`, `.webp`, `.gif` | `send_image_file()` |
| `.ogg`, `.opus`, `.mp3`, `.wav` | `send_voice()` |
| `.mp4`, `.mov`, `.avi`, `.webm` | `send_video()` |
| 其他 | `send_document()` |

如果你没有覆盖这些方法，基类会回退到 `send()` 发送文件路径文本——不会报错，只是用户体验略差。

---

## C.5　平台锁机制详解

平台锁是一个容易被忽视但非常重要的机制。许多平台（如 Telegram long-polling）不允许同一 Bot token 被多个客户端同时连接。

```python
# connect() 中：
if not self._acquire_platform_lock(
    'my-platform-token',    # scope: 锁的命名空间
    self.config.token,      # identity: 用 token 值作为唯一标识
    'My Platform bot token' # resource_desc: 冲突时的人类可读提示
):
    return False  # 内部已设置 fatal_error(retryable=False)

# disconnect() 中：
self._release_platform_lock()  # 必须在 finally 中调用
```

**致命错误分类：**

| `retryable` | 行为 | 典型场景 |
|-------------|------|---------|
| `True` | GatewayRunner 后台自动重连 | 网络抖动、SDK 超时 |
| `False` | 需要人工干预 | Token 冲突、凭据过期 |

---

## C.6　处理 Webhook 模式

有些平台（如企业微信、飞书）不使用 long-polling，而是通过 **Webhook 回调** 接收消息。基类同样支持，你需要在 `connect()` 中启动一个 HTTP 服务：

```python
async def connect(self) -> bool:
    # ... 前置检查和平台锁 ...

    from aiohttp import web

    # 创建 webhook 路由
    app = web.Application()
    app.router.add_post('/webhook/my_platform', self._handle_webhook)

    # 在 extra 配置中获取端口
    port = self.config.extra.get('webhook_port', 8080)

    runner = web.AppRunner(app)
    await runner.setup()
    site = web.TCPSite(runner, '0.0.0.0', port)
    await site.start()
    self._runner = runner

    self._mark_connected()
    return True

async def _handle_webhook(self, request):
    """处理平台推送的 webhook 请求。"""
    data = await request.json()

    # 验证签名（重要！防止伪造请求）
    if not self._verify_signature(request, data):
        return web.Response(status=403)

    # 构造 MessageEvent 并分发
    event = self._parse_webhook_event(data)
    if event:
        await self.handle_message(event)

    return web.Response(text='ok')
```

---

## C.7　完整检查清单

在提交你的适配器之前，逐项检查：

### 必做项 ✅

- [ ] 继承 `BasePlatformAdapter`，传入正确的 `Platform` 枚举值
- [ ] 实现 `connect()` — 含平台锁获取 + `_mark_connected()`
- [ ] 实现 `disconnect()` — 含 `cancel_background_tasks()` + `_release_platform_lock()` + `_mark_disconnected()`
- [ ] 实现 `send()` — 含消息分割 + 正确设置 `SendResult.retryable`
- [ ] 实现 `get_chat_info()` — 至少返回 `name` 和 `type`
- [ ] 编写 `_on_message` 回调 — 构造完整的 `SessionSource` 和 `MessageEvent`
- [ ] 添加 `check_xxx_requirements()` 函数
- [ ] 在 `Platform` 枚举中添加新值
- [ ] 在 `_create_adapter()` 中添加工厂分支
- [ ] 在 `_is_user_authorized()` 中添加环境变量映射

### 应做项 🔧

- [ ] 覆盖 `send_typing()` — 提升用户体验
- [ ] 覆盖 `send_image()` / `send_document()` — 支持富媒体
- [ ] 处理平台特有的消息类型（图片、语音、文件等）
- [ ] 实现 `format_message()` — 适配平台 Markdown 方言
- [ ] 正确区分 `chat_type`（`"dm"` / `"group"` / `"channel"`）
- [ ] 在 `_on_message` 中处理 `thread_id`（如果平台支持线程）

### 避免项 ⚠️

- [ ] 不要在 `send()` 中自行重试——基类的 `_send_with_retry` 会处理
- [ ] 不要在 `_on_message` 中直接调用 `_message_handler`——始终走 `handle_message()`
- [ ] 不要忘记在 `disconnect()` 的 `finally` 中释放平台锁
- [ ] 不要在 `connect()` 中阻塞事件循环（使用 `await`，避免同步 I/O）

---

## C.8　测试你的适配器

### C.8.1　单元测试骨架

```python
import pytest
from unittest.mock import AsyncMock, MagicMock
from gateway.config import PlatformConfig, Platform
from gateway.platforms.my_platform import MyPlatformAdapter


@pytest.fixture
def adapter():
    config = PlatformConfig(enabled=True, token="test-token-123")
    adapter = MyPlatformAdapter(config)
    adapter.set_message_handler(AsyncMock(return_value="Hello!"))
    return adapter


class TestMyPlatformAdapter:
    """适配器单元测试。"""

    def test_platform_enum(self, adapter):
        """验证平台枚举值正确。"""
        assert adapter.platform == Platform.MY_PLATFORM

    @pytest.mark.asyncio
    async def test_connect_without_token(self):
        """无 token 时连接应失败。"""
        config = PlatformConfig(enabled=True, token=None)
        adapter = MyPlatformAdapter(config)
        result = await adapter.connect()
        assert result is False

    @pytest.mark.asyncio
    async def test_send_splits_long_message(self, adapter):
        """超长消息应被分割。"""
        long_text = "A" * 10000
        chunks = adapter.truncate_message(long_text, 4096)
        assert len(chunks) >= 3
        # 验证所有内容都被保留
        assert sum(len(c) for c in chunks) >= 10000

    @pytest.mark.asyncio
    async def test_message_event_construction(self, adapter):
        """验证消息事件构造正确。"""
        raw_msg = MagicMock(
            chat_id=123, chat_name="Test",
            is_direct=True, author_id=456,
            author_name="Alice", thread_id=None,
            text="hello", id=789
        )
        # 直接调用回调逻辑中的 source 构造
        source = adapter.build_source(
            chat_id="123", chat_name="Test",
            chat_type="dm", user_id="456",
            user_name="Alice", thread_id=None,
        )
        assert source.platform == Platform.MY_PLATFORM
        assert source.chat_type == "dm"
```

### C.8.2　集成测试要点

| 测试场景 | 验证内容 |
|---------|---------|
| 连接/断开生命周期 | `_running` 状态变化正确 |
| 消息往返 | MessageEvent → handle_message → send 被调用 |
| 中断支持 | 两条消息快速到达，第二条进入 `_pending_messages` |
| 平台锁 | 两个实例同 token，第二个获取锁失败 |
| 消息分割 | 超长消息正确分割，代码块保持完整 |
| 重试逻辑 | Mock send 返回 `retryable=True`，验证重试 + 降级 |

### C.8.3　手动端到端测试

```bash
# 启动 Gateway，仅启用新平台
hermes start --platform my_platform

# 观察日志输出
# 应看到：[MyPlatform] ✓ 连接成功

# 在平台上向 Bot 发送消息
# 观察日志中的 MessageEvent 和 send 调用
```

---

## C.9　设计模式回顾

接入过程中，你实际运用了以下设计模式：

| 模式 | 体现位置 |
|------|---------|
| **模板方法（Template Method）** | `handle_message` → `_process_message_background` 定义完整流程骨架，子类只填充 `connect` / `send` 等步骤 |
| **策略（Strategy）** | `_message_handler` 回调由 `GatewayRunner` 注入，适配器不关心上层业务逻辑 |
| **适配器（Adapter）** | 你写的子类就是适配器——将平台 SDK 转换为统一的 `MessageEvent` / `SendResult` |
| **工厂方法（Factory Method）** | `_create_adapter()` 根据 `Platform` 枚举动态创建正确的适配器实例 |
| **观察者（Observer）** | `on_processing_start/complete` 钩子，子类可选订阅处理事件 |
| **排他锁（Mutex）** | `_acquire_platform_lock` 防止 token 被多进程竞争使用 |
| **优雅降级（Graceful Degradation）** | 媒体方法默认回退到文本发送；格式化失败回退 plain-text |

---

## C.10　真实案例参考

如果你在实现过程中遇到困难，推荐参考以下已有适配器的源码：

| 场景 | 参考文件 | 关键学习点 |
|------|---------|-----------|
| 最完整的实现 | `telegram.py`（2825 行） | polling/webhook 双模式、MarkdownV2 格式化、相册合并、forum topics |
| Webhook 模式 | `dingtalk.py` / `feishu.py` | 签名验证、回调解析 |
| WebSocket 模式 | `discord.py` | discord.py 库集成、reaction 钩子 |
| 最简实现 | `webhook.py` | 仅 HTTP 端点，适合理解最小可用适配器 |
| 企业级回调 | `wecom_callback.py` | 加解密、XML 解析 |

> **Tip：** 从 `webhook.py` 开始阅读——它是最简单的适配器，剥离了所有平台特有的复杂度，只保留核心接口实现。理解它之后再看 `telegram.py` 会事半功倍。

---

**恭喜！** 完成以上步骤后，你就拥有了一个功能完整的新平台适配器。基类的 1300+ 行通用逻辑（会话管理、中断处理、typing 指示器、消息分割、重试降级、媒体提取）全部自动生效，你只需要专注于平台 SDK 的对接即可。
