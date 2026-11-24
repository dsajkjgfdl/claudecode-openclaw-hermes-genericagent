---
Task: T25-R
Status: DRAFT_DONE
Date: 2026-04-16
Chapter: 25 — 架构模式总结
Keywords: 设计模式、Registry、Adapter、Pipeline、Observer、Strategy、Singleton、Factory、Proxy、Security、Concurrency、Resilience、Memory、Extension
Sources:
  - tools/registry.py
  - tools/*.py
  - model_tools.py
  - toolsets.py
  - prompt_builder.py
  - context.py
  - gateway/platforms/*.py
  - terminal/environments/*.py
  - agent.py
  - session_db.py
  - skills/*.py
  - memory/*.py
  - security/*.py
  - batch_runner.py
  - raven.py
---

# Ch25 架构模式总结 — 研究笔记

> 本章是全书的收尾与升华。前 24 章逐一拆解了 Hermes Agent 的每个模块，本章
> 从更高层次的**设计模式（Design Patterns）**视角，回顾整个系统中反复出现的
> 架构范式，帮助读者将零散的实现细节整合为系统化的架构认知。

---

## 概述

Hermes Agent 是一个约 **276 个 Python 文件、20 万行代码** 的全栈 AI Agent 框架。
在如此规模的代码库中，保持可维护性和可扩展性的关键在于一致的架构模式运用。
通过对源码的全面研究，我们提炼出 **14 大核心架构模式**，它们贯穿了从启动到
运行、从安全到扩展的每个层面。

本章将逐一剖析每个模式：**什么是它？在哪里出现？代码如何实现？为什么重要？**

---

## 1. 注册表模式（Registry Pattern）

### 1.1 模式定义

注册表模式（又称 Service Locator 变体）提供一个集中式注册点，让组件在运行时
自行注册元数据和行为，消费者通过查询注册表来发现和使用组件，而非硬编码依赖。

### 1.2 在 Hermes 中的出现

#### 工具注册表（核心实例）

- **文件**: `tools/registry.py`（约 482 行）
- **单例**: `registry = ToolRegistry()` — 模块级全局实例
- **核心数据结构**: `ToolEntry`（使用 `__slots__` 优化的 10 字段结构体）

```python
# tools/registry.py:76-97
class ToolEntry:
    __slots__ = (
        "name", "toolset", "schema", "handler", "check_fn",
        "requires_env", "is_async", "description", "emoji",
        "max_result_size_chars",
    )
```

- **自注册机制**: 每个工具文件（如 `tools/terminal_tool.py`、`tools/file_tools.py`）
  在模块 `import` 时调用 `registry.register()` 将自身注册。
- **发现流程**: `model_tools.py` 通过 `importlib` + AST 扫描动态发现 `tools/` 目录
  下所有工具模块，触发导入链完成注册。

```python
# model_tools.py 触发发现
from tools.registry import registry
# 调用 registry.discover() → 扫描 tools/ 目录 → import → 自注册
```

#### 技能注册表

- **文件**: `skills/skills_db.py`、`skills/loader.py`
- 技能（Skills）在加载时注册到技能数据库，提供给 prompt 系统使用。
- 技能本身也会注册为"工具"进入 `ToolRegistry`，使得 LLM 可以直接调用。

#### MCP 工具注册表

- **文件**: `tools/mcp_tool.py`、`mcp_client.py`
- 远程 MCP 服务器暴露的工具，通过 `mcp-{server_name}` 前缀动态注册到同一个
  `ToolRegistry` 中，与本地工具共享同一套发现和调度机制。

### 1.3 为什么重要

- **解耦**: 新增工具只需在 `tools/` 下添加文件并调用 `registry.register()`，无需
  修改任何上层代码。
- **统一接口**: LLM 看到的 tool definitions 全部来自注册表，无论工具是本地、远程
  还是技能生成的。
- **运行时动态性**: 支持 MCP 热加载、toolset 开关、可用性检查等动态行为。

---

## 2. 适配器模式（Adapter Pattern）

### 2.1 模式定义

适配器模式将不兼容的接口转换为客户端期望的统一接口，让原本无法协同工作的类
能够一起运行。

### 2.2 在 Hermes 中的出现

#### 消息平台适配器

- **文件**: `gateway/platforms/` 目录
- **基类**: `PlatformAdapter`（定义统一消息接口）
- **实现**: `TelegramAdapter`、`DiscordAdapter`、`SlackAdapter`、`WeChatAdapter`、
  `DingTalkAdapter`、`FeishuAdapter`、`MatrixAdapter` 等 17+ 平台适配器

每个平台有完全不同的消息格式、认证方式和回调机制，适配器将它们统一为：

```python
# 统一接口（概念性）
class PlatformAdapter:
    async def send_message(self, channel_id, text, **kwargs): ...
    async def handle_incoming(self, raw_event) -> Message: ...
    async def start(self): ...
    async def stop(self): ...
```

Telegram 的 `Update` 对象、Discord 的 `Interaction`、Slack 的 `Event` 等
完全不同的数据结构，都被适配为统一的 `Message` 模型。

#### 终端环境适配器

- **文件**: `terminal/environments/` 目录
- **基类**: `BaseEnvironment`（ABC）
- **实现**: `LocalEnvironment`、`SSHEnvironment`、`DockerEnvironment`、
  `DaytonaEnvironment`、`ModalEnvironment`、`E2BEnvironment`

```python
# terminal/environments/base.py
class BaseEnvironment(ABC):
    @abstractmethod
    async def run_command(self, command, timeout, ...): ...
    @abstractmethod
    async def start_background(self, command, ...): ...
    @abstractmethod
    async def read_file(self, path): ...
    @abstractmethod
    async def write_file(self, path, content): ...
```

无论底层是本地 shell、SSH 远程主机、Docker 容器还是云端沙箱，上层终端工具
代码始终通过相同接口操作。

#### LLM 提供商适配器

- **文件**: `api_clients.py`、`anthropic_client.py`
- OpenAI 兼容协议作为主适配层，Anthropic 原生 API 作为专用适配层
- 上层 `agent.py` 通过统一的 `chat_completion()` 调用，无需感知底层差异

### 2.3 为什么重要

- **开放-封闭原则**: 新增平台/环境只需添加适配器，不修改已有代码
- **可测试性**: 可以用 Mock 适配器替换真实实现进行单元测试
- **关注点分离**: 业务逻辑与平台细节完全解耦

---

## 3. 管道/链模式（Pipeline / Chain Pattern）

### 3.1 模式定义

管道模式将处理过程分解为一系列有序的阶段，每个阶段接收前一阶段的输出作为
输入，最终产出完整结果。

### 3.2 在 Hermes 中的出现

#### System Prompt 构建管道

- **文件**: `prompt_builder.py`
- 通过 Jinja2 模板引擎，将 system prompt 的构建分解为多个阶段：

```
阶段 1: 基础人设（Base Persona）
阶段 2: 工具使用说明（Tool Instructions）
阶段 3: 可用技能注入（Available Skills）
阶段 4: 记忆注入（User Memory）
阶段 5: 平台特定指令（Platform-specific Instructions）
阶段 6: 安全约束（Security Constraints）
阶段 7: 上下文信息（Current Context: time, cwd, etc.）
```

每个阶段通过模板条件控制是否包含，变量由运行时上下文提供。

#### 上下文压缩管道

- **文件**: `context.py`（`compress_context()` 方法）
- 当对话超过 token 限制时，触发分层压缩管道：

```
Step 1: 移除旧的工具调用结果详情
Step 2: 截断超长的单条消息
Step 3: 使用辅助模型生成对话摘要
Step 4: 用摘要替换原始消息，保留最近 N 条
```

#### 工具调用处理链

- **文件**: `agent.py`（核心循环）、`model_tools.py`（dispatch）
- LLM 返回 tool_call → 参数解析 → 安全审批检查 → handler 调度 → 结果截断 →
  注入回对话 → 再次调用 LLM

这是一个经典的**责任链（Chain of Responsibility）**变体，每一步可以中断流程
（如审批被拒绝）。

### 3.3 为什么重要

- **可组合性**: 每个阶段可以独立开发和测试
- **可观察性**: 可以在阶段之间插入日志/监控
- **灵活性**: 不同场景（CLI vs 网关）可以跳过某些阶段

---

## 4. 观察者/事件模式（Observer / Event Pattern）

### 4.1 模式定义

观察者模式定义了对象间的一对多依赖，当一个对象状态改变时，所有依赖于它的
对象都会自动通知并更新。事件系统是观察者模式的常见实现。

### 4.2 在 Hermes 中的出现

#### Gateway Hooks 系统

- **文件**: `gateway/hooks.py`
- Gateway 定义了一系列钩子点（hook points），平台适配器和扩展可以注册回调：

```python
# 概念性 hook 点
on_message_received    → 消息预处理
on_before_respond      → 响应前拦截
on_after_respond       → 响应后处理
on_error               → 错误处理
```

这使得跨切面（cross-cutting）关注点（如日志、速率限制、消息过滤）可以通过
钩子注入，而不侵入核心逻辑。

#### 进程监控 — watch_patterns

- **文件**: `tools/terminal_tool.py`、后台进程管理
- 终端工具支持 `watch_patterns` 参数，当后台进程输出匹配指定模式时触发通知：

```python
# 使用示例
await terminal(
    command="npm run build",
    background=True,
    watch_patterns=["ERROR", "FAIL", "Build complete"]
)
```

这是观察者模式在进程输出监控中的应用——进程输出流是被观察对象，
`watch_patterns` 定义了观察条件，匹配时触发通知回调。

#### notify_on_complete 机制

- **文件**: 后台进程管理系统
- 后台任务完成时自动通知调用者，无需轮询。这是经典的**异步回调模式**，
  是观察者模式在并发场景中的应用。

### 4.3 为什么重要

- **解耦**: 事件源不需要知道具体的观察者
- **可扩展**: 新增监控/处理逻辑只需注册新回调
- **非侵入**: 不修改核心代码即可添加横切关注点

---

## 5. 策略模式（Strategy Pattern）

### 5.1 模式定义

策略模式定义一系列算法，将每个算法封装起来，使它们可以互相替换。客户端代码
选择使用哪个策略，但不关心策略的具体实现。

### 5.2 在 Hermes 中的出现

#### LLM 提供商策略

- **文件**: `api_clients.py`、`anthropic_client.py`、`config.py`
- 系统支持多个 LLM 提供商：OpenAI、Anthropic、Mistral、本地模型（Ollama）等
- 用户通过 `config.yaml` 中的 `model` 字段选择策略：

```yaml
model: "claude-sonnet-4-20250514"     # → 使用 Anthropic 原生客户端
model: "gpt-4o"               # → 使用 OpenAI 客户端
model: "ollama/llama3"         # → 使用 OpenAI 兼容客户端 + 本地端点
```

路由逻辑根据模型名称前缀选择对应的 API 客户端策略。

#### 搜索引擎策略

- **文件**: `tools/web_tools.py`
- 支持多个搜索引擎后端：Exa、Tavily、DuckDuckGo、Parallel Web 等
- 可通过配置切换搜索策略，每个引擎实现相同的搜索接口

#### TTS 引擎策略

- **文件**: `tools/tts_tool.py`
- 支持多种 TTS 引擎：edge-tts（免费）、OpenAI TTS、ElevenLabs 等
- 不同引擎通过配置选择，对外暴露统一的 `text_to_speech()` 接口

#### 辅助模型策略

- **文件**: `aux_client.py`
- 用于上下文压缩、标题生成等辅助任务的模型可以独立配置
- 可以选择比主模型更便宜/更快的模型来执行辅助任务，降低成本

### 5.3 为什么重要

- **灵活性**: 用户根据需求（成本、速度、质量）选择最合适的策略
- **可替换性**: 某个供应商出问题时可以快速切换
- **成本优化**: 不同任务使用不同级别的模型

---

## 6. 单例模式（Singleton Pattern）

### 6.1 模式定义

单例模式确保一个类只有一个实例，并提供全局访问点。

### 6.2 在 Hermes 中的出现

#### ToolRegistry 单例

- **文件**: `tools/registry.py:437`

```python
registry = ToolRegistry()  # 模块级单例
```

使用 Python 最朴素的**模块级变量**实现单例。Python 的模块导入机制保证了
每个模块只执行一次，因此 `registry` 在整个进程生命周期中只有一个实例。
所有消费者通过 `from tools.registry import registry` 获取同一实例。

#### SessionDB 单例

- **文件**: `session_db.py`
- SQLite 数据库连接通过模块级变量管理，确保同一进程中只有一个数据库连接实例
- 避免多个连接导致的并发写入冲突

#### Skin Engine 单例

- **文件**: `skin_engine.py`
- 皮肤引擎（控制 UI 显示风格）在启动时初始化一次，整个运行期间保持不变
- 确保 UI 渲染风格一致

#### Config 单例

- **文件**: `config.py`
- 配置对象在启动时加载一次，后续所有模块引用同一实例
- 保证配置一致性

### 6.3 实现特点

Hermes 没有使用 `__new__` 覆写或 metaclass 等"经典"但过度设计的单例实现，
而是统一采用 **模块级变量** 方式。这种方式：

- **简洁**: 不需要额外的元类或装饰器
- **可测试**: 测试时可以直接替换模块变量
- **Python 惯用**: 符合 Python "简单胜于复杂" 的哲学
- **线程安全**: Python 的 GIL + import lock 保证模块初始化的线程安全

### 6.4 为什么重要

- 共享资源的唯一入口，避免重复创建开销
- 保证全局状态一致性
- 配合 RLock 实现线程安全的并发访问

---

## 7. 模板方法模式（Template Method Pattern）

### 7.1 模式定义

模板方法模式在基类中定义算法的骨架，将某些步骤推迟到子类实现。子类可以
重写特定步骤而不改变算法的整体结构。

### 7.2 在 Hermes 中的出现

#### PlatformAdapter 基类

- **文件**: `gateway/platforms/base.py`
- 定义了消息处理的标准流程：接收消息 → 解析 → 路由 → 处理 → 响应
- 子类（各平台适配器）实现具体的消息解析和发送逻辑

```python
# 概念性结构
class PlatformAdapter(ABC):
    async def process_message(self, raw):
        """模板方法：定义消息处理的标准流程"""
        msg = await self.parse_incoming(raw)      # 子类实现
        if not self.should_respond(msg):           # 基类提供默认实现
            return
        response = await self.generate_response(msg)  # 基类核心逻辑
        await self.send_response(msg, response)    # 子类实现
    
    @abstractmethod
    async def parse_incoming(self, raw) -> Message: ...
    
    @abstractmethod
    async def send_response(self, msg, response): ...
```

#### BaseEnvironment ABC

- **文件**: `terminal/environments/base.py`
- 定义终端环境的标准操作集：执行命令、读写文件、管理进程
- 每个具体环境（Local、SSH、Docker 等）实现自己的底层操作

```python
class BaseEnvironment(ABC):
    async def execute(self, command, timeout=180):
        """模板方法：含前置检查 + 执行 + 后置处理"""
        self._validate_command(command)    # 基类：安全检查
        result = await self.run_command(command, timeout)  # 子类实现
        return self._format_output(result) # 基类：输出格式化
    
    @abstractmethod
    async def run_command(self, command, timeout): ...
```

### 7.3 为什么重要

- **代码复用**: 公共逻辑在基类中实现一次
- **一致性保证**: 所有子类遵循相同的流程
- **控制反转**: 框架控制流程，子类填充细节

---

## 8. 工厂模式（Factory Pattern）

### 8.1 模式定义

工厂模式将对象的创建逻辑封装在专门的工厂方法/类中，客户端不需要知道具体
创建哪个类的实例。

### 8.2 在 Hermes 中的出现

#### 终端环境工厂

- **文件**: `tools/terminal_tool.py`
- 根据配置中的 `terminal_environment` 设置创建对应的环境实例：

```python
# 概念性工厂逻辑
def create_environment(config):
    env_type = config.get("terminal_environment", "local")
    if env_type == "local":
        return LocalEnvironment()
    elif env_type == "ssh":
        return SSHEnvironment(host=config["ssh_host"], ...)
    elif env_type == "docker":
        return DockerEnvironment(container=config["container"], ...)
    elif env_type == "daytona":
        return DaytonaEnvironment(workspace=config["workspace"], ...)
    elif env_type == "modal":
        return ModalEnvironment(...)
    elif env_type == "e2b":
        return E2BEnvironment(...)
    else:
        raise ValueError(f"Unknown environment: {env_type}")
```

#### 平台适配器工厂

- **文件**: `gateway/` 启动逻辑
- Gateway 根据配置中启用的平台列表，动态创建对应的适配器实例：

```python
# 概念性
for platform_name in config.get("platforms", []):
    adapter = create_platform_adapter(platform_name, platform_config)
    gateway.register_adapter(adapter)
```

#### API 客户端工厂

- **文件**: `api_clients.py`
- 根据模型名称前缀创建对应的 LLM 客户端：

```python
def get_client(model_name):
    if model_name.startswith("claude"):
        return AnthropicClient(...)
    elif model_name.startswith("mistral"):
        return MistralClient(...)
    else:
        return OpenAICompatClient(...)  # 默认 OpenAI 兼容
```

### 8.3 为什么重要

- **封装创建逻辑**: 隐藏复杂的初始化过程
- **统一入口**: 客户端代码只关心"要什么"而非"怎么创建"
- **配置驱动**: 通过配置文件控制运行时行为

---

## 9. 代理模式（Proxy Pattern）

### 9.1 模式定义

代理模式为另一个对象提供一个替身或占位符来控制对它的访问。代理和真实对象
实现相同接口，代理在转发调用前/后可以执行额外逻辑。

### 9.2 在 Hermes 中的出现

#### Raven 代理

- **文件**: `raven.py`
- Raven 是 Hermes 的 API 路由代理，将请求转发到正确的 LLM 提供商
- 在转发前执行：API key 验证、速率限制、请求日志记录
- 在转发后执行：响应缓存、成本统计、错误处理

```python
# 概念性结构
class RavenProxy:
    async def forward_request(self, request):
        self._check_rate_limit(request)      # 前置：速率限制
        self._validate_api_key(request)       # 前置：认证
        response = await self._route(request) # 转发到真实 API
        self._log_usage(request, response)    # 后置：记录使用
        return response
```

#### delegate_tool — 智能体代理

- **文件**: `tools/delegate_tool.py`
- `delegate` 工具是 Agent 的代理：主 Agent 可以创建子 Agent 来执行委托任务
- 子 Agent 是主 Agent 的"代理"，它代表主 Agent 执行特定任务：

```python
# tools/delegate_tool.py
async def delegate(args, **kwargs):
    task = args["task"]
    context = args.get("context", "")
    # 创建子 Agent（真实对象的代理）
    sub_agent = create_sub_agent(task=task, context=context)
    result = await sub_agent.run()
    return result  # 将结果返回给主 Agent
```

这也可以被视为**远程代理（Remote Proxy）**的一种——子 Agent 可能运行在
独立的进程或上下文中。

#### Prompt Caching 代理

- **文件**: `anthropic_client.py`
- 对 Anthropic API 的调用封装了 prompt caching 逻辑，在真实 API 调用前
  检查缓存、在调用后更新缓存。这是一种**缓存代理**。

### 9.3 为什么重要

- **访问控制**: 在不修改真实对象的情况下添加安全检查
- **性能优化**: 通过缓存代理减少实际 API 调用
- **监控**: 透明地添加日志和度量采集

---

## 10. 安全模式（Security Patterns）

### 10.1 概述

安全不是单一模式，而是一系列协同工作的模式集合。Hermes Agent 实现了
**深度防御（Defense in Depth）**策略。

### 10.2 具体模式

#### 审批系统（Approval Gate Pattern）

- **文件**: `security/approval.py`、`tools/terminal_tool.py`
- 危险操作（如 `rm -rf`、`sudo`、安装软件等）需要用户明确批准
- 实现为工具调度链中的一个**拦截器**：

```python
# 工具调用链中的审批拦截
if tool_requires_approval(tool_name, args):
    approved = await ask_user_approval(tool_name, args)
    if not approved:
        return "⛔ Operation denied by user"
# 继续执行
```

#### 命令黑名单（Blocklist Pattern）

- **文件**: `security/blocklist.py`（或 `tools/terminal_tool.py` 内嵌）
- 维护一个危险命令模式列表，阻止已知危险操作：

```python
BLOCKED_PATTERNS = [
    r"rm\s+-rf\s+/",           # 删除根目录
    r"mkfs\.",                   # 格式化磁盘
    r"dd\s+if=.*of=/dev/",     # 覆写设备
    r":(){ :\|:& };:",          # Fork 炸弹
    # ... 更多模式
]
```

#### 密钥脱敏（Secret Redaction Pattern）

- **文件**: `security/redaction.py`
- 在输出显示前，自动检测并脱敏 API keys、密码、tokens 等敏感信息
- 使用正则表达式匹配常见的密钥模式，替换为 `[REDACTED]`

```python
REDACTION_PATTERNS = [
    (r'(sk-[a-zA-Z0-9]{20,})', '[REDACTED_API_KEY]'),
    (r'(ghp_[a-zA-Z0-9]{36})', '[REDACTED_GITHUB_TOKEN]'),
    (r'(password\s*=\s*)["\']([^"\']+)', r'\1"[REDACTED]"'),
]
```

#### 技能安全守卫（Skills Guard Pattern）

- **文件**: `skills/skills_guard.py`
- 技能代码在执行前经过安全检查：
  - 静态分析：检测危险函数调用（`eval`、`exec`、`os.system`）
  - 沙箱执行：技能代码在受限环境中运行
  - 权限控制：技能的工具访问受 toolset 限制

### 10.3 架构意义

安全模式的关键在于**分层叠加**：

```
Layer 1: 命令黑名单 → 已知危险操作立即阻止
Layer 2: 审批系统   → 可疑操作要求人工确认
Layer 3: 密钥脱敏   → 敏感信息不被泄露
Layer 4: 技能守卫   → 外部代码受限执行
Layer 5: 环境隔离   → Docker/E2B 沙箱物理隔离
```

任何单层失效，其他层仍能提供保护。这是经典的**深度防御**策略。

---

## 11. 并发模式（Concurrency Patterns）

### 11.1 概述

AI Agent 天然涉及大量 I/O 操作（API 调用、文件读写、网络请求），并发模式
是性能优化的关键。

### 11.2 具体模式

#### 后台进程管理

- **文件**: `tools/terminal_tool.py`、进程管理模块
- 终端工具支持 `background=True` 参数，将命令放入后台执行
- 通过 session_id 跟踪、polling 检查状态、日志回放

```python
# 后台进程生命周期
session_id = await terminal(command="npm run dev", background=True)
# ... 执行其他任务 ...
status = await process(action="poll", session_id=session_id)
output = await process(action="log", session_id=session_id)
await process(action="kill", session_id=session_id)
```

#### 并行委托

- **文件**: `tools/delegate_tool.py`
- 支持同时委托多个子任务给不同的子 Agent：

```python
# 并行执行多个委托任务
results = await asyncio.gather(
    delegate(task="研究 API 文档"),
    delegate(task="编写测试用例"),
    delegate(task="生成数据模型"),
)
```

使用 `asyncio.gather()` 实现 **Fan-out / Fan-in** 模式。

#### ProcessPoolExecutor 批处理

- **文件**: `batch_runner.py`
- 批量处理多个会话/请求时使用进程池：

```python
from concurrent.futures import ProcessPoolExecutor

with ProcessPoolExecutor(max_workers=N) as pool:
    futures = [pool.submit(process_session, session) for session in sessions]
    results = [f.result() for f in futures]
```

使用进程池而非线程池，避免 GIL 限制，充分利用多核 CPU。

#### asyncio 事件循环

- **文件**: 贯穿全项目
- Hermes 大量使用 `async/await` 模式处理 I/O 密集型任务
- LLM API 调用、网络请求、文件操作等都通过协程实现非阻塞

### 11.3 为什么重要

- **性能**: 并行化减少总体等待时间
- **响应性**: 后台任务不阻塞主交互
- **资源利用**: 充分利用 I/O 等待时间和多核 CPU

---

## 12. 韧性模式（Resilience Patterns）

### 12.1 概述

AI Agent 依赖的外部服务（LLM API、搜索引擎、MCP 服务器等）都可能失败。
韧性模式确保系统在部分故障时仍能运行。

### 12.2 具体模式

#### 重试与指数退避（Retry with Exponential Backoff）

- **文件**: `api_clients.py`、`tenacity` 库的广泛使用
- API 调用失败时，按指数增长的间隔重试：

```python
from tenacity import retry, wait_exponential, stop_after_attempt

@retry(
    wait=wait_exponential(multiplier=1, min=2, max=60),
    stop=stop_after_attempt(5),
    retry=retry_if_exception_type((APITimeoutError, RateLimitError)),
)
async def call_llm(messages, model, ...):
    return await client.chat.completions.create(...)
```

#### 优雅降级（Graceful Degradation）

- **文件**: 多处体现
- **搜索降级**: Exa 不可用时回退到 DuckDuckGo
- **模型降级**: 主模型不可用时切换到备选模型
- **功能降级**: MCP 服务器断连时，禁用对应工具但系统继续运行
- **TTS 降级**: 首选引擎失败时回退到免费的 edge-tts

```python
# 概念性降级逻辑
try:
    result = await primary_search(query)
except SearchError:
    logger.warning("Primary search failed, falling back")
    result = await fallback_search(query)
```

#### 速率限制与熔断（Rate Limiting / Circuit Breaker）

- **文件**: `api_clients.py`、`raven.py`
- 检测到持续的速率限制错误时，暂停对该端点的请求
- 等待冷却期后自动恢复——这是**熔断器模式**的简化实现

```python
# 概念性熔断器
class CircuitBreaker:
    def __init__(self, failure_threshold=5, reset_timeout=60):
        self.failures = 0
        self.state = "CLOSED"  # CLOSED → OPEN → HALF-OPEN
    
    async def call(self, func, *args):
        if self.state == "OPEN":
            if time_since_opened > self.reset_timeout:
                self.state = "HALF-OPEN"
            else:
                raise CircuitBreakerOpen()
        try:
            result = await func(*args)
            self._on_success()
            return result
        except Exception:
            self._on_failure()
            raise
```

#### 超时控制

- **文件**: 全项目的 API 调用和命令执行
- 所有外部调用都设置超时，防止无限等待
- 终端命令默认 180 秒超时，API 调用有独立超时配置

### 12.3 为什么重要

- **可用性**: 系统在部分故障时仍能工作
- **自愈**: 自动重试和熔断恢复，无需人工干预
- **用户体验**: 用户感知到的是"偶尔慢一点"而非"完全不可用"

---

## 13. 记忆/状态模式（Memory / State Patterns）

### 13.1 概述

AI Agent 需要在多次交互中维持状态和积累知识，记忆/状态模式是 Agent 从
"无状态工具"进化为"有记忆伙伴"的关键。

### 13.2 具体模式

#### 持久化记忆（Persistent Memory Store）

- **文件**: `memory/user_memory.py`、`memory/memory_store.py`
- 用户的偏好、事实、习惯等信息持久化到本地存储：

```python
# 记忆存取
await memory.store("user_preference", "偏好使用 Python 编程")
await memory.store("project_context", "当前项目使用 FastAPI 框架")
facts = await memory.recall(query="编程偏好")
```

- 记忆在每次对话开始时注入 system prompt，为 LLM 提供个性化上下文
- 支持记忆的创建、更新、删除和搜索

#### 会话搜索（FTS5 全文搜索）

- **文件**: `session_db.py`
- 使用 SQLite 的 FTS5 扩展实现历史对话的全文搜索：

```sql
-- FTS5 虚拟表
CREATE VIRTUAL TABLE session_fts USING fts5(
    session_id, role, content,
    tokenize='unicode61'
);
```

- 支持按关键词搜索历史对话，找到相关上下文
- 用于 Agent 的"回忆"能力——"我们之前讨论过什么？"

#### Prompt Caching（提示缓存）

- **文件**: `anthropic_client.py`
- Anthropic API 支持 prompt caching，对 system prompt 中不变的部分进行缓存
- 缓存标记通过 `cache_control` 字段注入：

```python
# Anthropic prompt caching
{
    "role": "system",
    "content": long_system_prompt,
    "cache_control": {"type": "ephemeral"}
}
```

- 减少重复发送大量 system prompt 的 token 成本（可节省 90%）

#### 会话状态机

- **文件**: `agent.py`
- Agent 的核心循环维护一个隐式的状态机：

```
IDLE → PROCESSING → TOOL_CALLING → WAITING_APPROVAL → PROCESSING → RESPONDING → IDLE
```

- 状态在对话消息列表和 Agent 属性中隐式表示
- `session_db.py` 在每次状态变更时持久化，支持中断恢复

### 13.3 为什么重要

- **连续性**: 跨会话的记忆使 Agent 行为更智能
- **个性化**: 根据用户偏好调整行为
- **可搜索性**: FTS5 让历史对话成为可查询的知识库
- **成本优化**: prompt caching 显著降低 API 费用

---

## 14. 扩展模式（Extension Patterns）

### 14.1 概述

一个 Agent 框架的价值很大程度上取决于其可扩展性。Hermes 设计了多种扩展
机制，让用户可以在不修改核心代码的情况下增强系统能力。

### 14.2 具体模式

#### 技能作为程序记忆（Skills as Procedural Memory）

- **文件**: `skills/skills_db.py`、`skills/loader.py`、`skills/executor.py`
- 技能是 Agent 从经验中学习并持久化的**可执行知识**
- 技能文件通常是 Python 代码 + YAML 元数据：

```yaml
# skill metadata
name: "deploy_to_vercel"
description: "Deploy a Next.js project to Vercel"
toolsets: ["terminal", "file"]
```

```python
# skill code
async def execute(context):
    await context.terminal("vercel deploy --prod")
    return "Deployment complete"
```

- 技能可以被创建、改进、分享、社区化
- 这是 **Self-Improving Agent** 的核心机制

#### MCP 协议扩展（External Tool Integration）

- **文件**: `mcp_client.py`、`tools/mcp_tool.py`
- MCP（Model Context Protocol）允许外部服务器暴露工具给 Agent：

```yaml
# config.yaml 中的 MCP 配置
mcp_servers:
  github:
    command: "npx -y @modelcontextprotocol/server-github"
    env:
      GITHUB_TOKEN: "$GITHUB_TOKEN"
  filesystem:
    command: "npx -y @modelcontextprotocol/server-filesystem /path/to/dir"
```

- MCP 工具自动注册到 `ToolRegistry`，前缀为 `mcp-{server_name}`
- 支持热加载：MCP 服务器重启后自动刷新工具列表

#### Toolset 启用/禁用机制

- **文件**: `toolsets.py`
- 工具被组织为 toolsets（工具集），支持运行时启用/禁用：

```python
# toolsets.py
TOOLSET_GROUPS = {
    "file": ["read_file", "write_file", "patch_file", "list_files"],
    "terminal": ["terminal", "process_manager"],
    "web": ["web_search", "fetch_url", "browser"],
    "mcp-github": ["mcp-github-create_issue", "mcp-github-list_repos", ...],
}
```

- 配置级别：`config.yaml` 中的 `enabled_toolsets` / `disabled_toolsets`
- 运行时级别：通过 API 动态开关
- 安全级别：技能的 `toolsets` 字段限制可用工具

#### 插件式架构的体现

整个 Hermes 的扩展体系形成一个三层结构：

```
Layer 1: 内置工具（tools/*.py）    — 随代码发布，核心能力
Layer 2: MCP 工具（外部服务器）    — 通过协议接入，生态扩展
Layer 3: 技能（用户创建/社区共享） — 从使用中学习，自我进化
```

### 14.3 为什么重要

- **开放生态**: 通过 MCP 和技能系统，Agent 的能力不受限于核心代码
- **自我改进**: Agent 能从经验中创建新技能，实现能力积累
- **灵活配置**: toolset 机制让不同场景使用不同的工具组合

---

## 15. 模式之间的协同

上述 14 个模式并非孤立存在，它们形成紧密的协同网络。以工具调用为例：

```
Registry(注册) → Strategy(选策略) → Factory(建环境) → Adapter(适配) → Pipeline(处理) → Observer(通知)
```

**安全与扩展的制衡**：Extension Patterns 扩展能力，Security Patterns 约束能力，
两者分层实现，做到既开放又安全。

**记忆与管道的闭环**：记忆注入 → prompt 构建 → LLM 行为 → 产生新经验 → 存储为新记忆。

---

## 16. 设计哲学与总结

从 14 个模式中提炼出 Hermes Agent 的核心设计哲学：

- **简单优先**: 单例用模块变量而非元类；工厂用 if-else 而非抽象工厂
- **配置驱动**: 几乎每个可变行为都通过 `config.yaml` 或环境变量控制
- **渐进增强**: 基础功能零配置可用，高级功能按需启用（17 个 extras）
- **深度防御**: 安全不是单一屏障——黑名单 → 审批 → 脱敏 → 守卫 → 隔离
- **自我进化**: 技能系统使 Agent 具备从经验中学习的能力

### 关键源文件索引

| 模式 | 关键文件 |
|------|---------|
| Registry | `tools/registry.py`, `toolsets.py`, `model_tools.py` |
| Adapter | `gateway/platforms/*.py`, `terminal/environments/*.py` |
| Pipeline | `prompt_builder.py`, `context.py`, `agent.py` |
| Observer | `gateway/hooks.py`, `tools/terminal_tool.py` |
| Strategy | `api_clients.py`, `anthropic_client.py`, `tools/web_tools.py` |
| Singleton | `tools/registry.py`, `session_db.py`, `config.py` |
| Template Method | `gateway/platforms/base.py`, `terminal/environments/base.py` |
| Factory | `tools/terminal_tool.py`, `gateway/`, `api_clients.py` |
| Proxy | `raven.py`, `tools/delegate_tool.py` |
| Security | `security/approval.py`, `security/blocklist.py`, `skills/skills_guard.py` |
| Concurrency | `tools/terminal_tool.py`, `tools/delegate_tool.py`, `batch_runner.py` |
| Resilience | `api_clients.py`, `tools/web_tools.py` |
| Memory/State | `memory/`, `session_db.py`, `anthropic_client.py` |
| Extension | `skills/`, `mcp_client.py`, `toolsets.py` |

### 与经典 GoF 模式的对应

| GoF 模式 | Hermes 实现 | 忠实度 |
|----------|------------|--------|
| Singleton | 模块级变量 | 简化版 ✓ |
| Factory Method | 环境/适配器/客户端创建 | 标准 ✓✓ |
| Adapter | 平台适配器、环境适配器 | 经典 ✓✓✓ |
| Proxy | Raven、delegate | 标准 ✓✓ |
| Observer | hooks、watch_patterns | 简化版 ✓ |
| Strategy | LLM/搜索/TTS 提供商 | 标准 ✓✓ |
| Template Method | PlatformAdapter、BaseEnvironment | 经典 ✓✓✓ |
| Chain of Responsibility | 工具调用处理链 | 变体 ✓ |

Hermes 中的模式多用 Python 惯用方式（鸭子类型、模块命名空间、ABC）实现，
而非严格的 GoF 类层次结构——这是将经典模式适配到 Python 语境的正确做法。

> **核心洞见**: Hermes Agent 不是一个简单的 LLM wrapper，而是一个运用了
> 成熟软件工程模式的**全栈 Agent 框架**。经典设计模式在 AI Agent 时代
> 不仅没有过时，反而在应对工具管理、安全控制、自我改进等 Agent 特有挑战时
> 展现出了新的生命力。
