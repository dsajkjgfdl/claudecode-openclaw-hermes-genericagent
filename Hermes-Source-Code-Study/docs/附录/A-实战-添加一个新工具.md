---
title: "附录 A：实战——添加一个新工具"
appendix: A
status: DRAFT
---

# 附录 A：实战——添加一个新工具

> **本附录是一份手把手的实战教程。** 读完后，你将掌握在 Hermes Agent 中从零开始添加一个全新工具的完整流程——从创建文件、定义 Schema、编写 Handler，到注册进 Toolset、验证自动发现，最终让 LLM 可以调用它。

---

## A.1 概览：只需修改两个文件

在 Hermes Agent 中添加一个新工具，**核心只需要修改两个文件**：

| 步骤 | 文件 | 做什么 |
|------|------|--------|
| ① | `tools/your_tool.py` | 创建工具文件：定义 Schema、Handler、check_fn，调用 `registry.register()` |
| ② | `toolsets.py` | 将工具名加入 `_HERMES_CORE_TOOLS` 列表或定义新的 Toolset |

**你不需要修改 `model_tools.py`。** 自动发现机制（Auto-Discovery）会通过 AST 扫描 `tools/` 目录下的所有 `.py` 文件，找到包含顶层 `registry.register()` 调用的模块并自动导入。

### 文件依赖链

理解这条链对避免循环导入至关重要：

```
tools/registry.py     ← 无依赖，所有工具文件导入它
       ↑
tools/*.py             ← 每个文件在模块导入时调用 registry.register()
       ↑
model_tools.py         ← 导入 tools.registry，触发 discover_builtin_tools()
       ↑
run_agent.py / cli.py  ← 消费 get_tool_definitions() 和 handle_function_call()
```

这条链是**防循环导入安全**的——`registry.py` 不导入任何工具文件或 `model_tools`。你的工具文件只需从 `tools.registry` 导入，绝不要从 `model_tools` 导入。

---

## A.2 核心概念速览

在动手之前，先了解三个关键概念。

### A.2.1 ToolEntry——工具元数据对象

`ToolEntry` 是注册表中每个工具的元数据结构，使用 `__slots__` 优化内存：

```python
class ToolEntry:
    __slots__ = (
        "name",                  # 工具名，如 "web_search"
        "toolset",               # 所属工具集，如 "web"
        "schema",                # OpenAI function-calling schema (dict)
        "handler",               # 处理函数 Callable
        "check_fn",              # 可用性检查函数 Callable -> bool
        "requires_env",          # 需要的环境变量列表
        "is_async",              # 是否为异步处理函数
        "description",           # 描述（备用值）
        "emoji",                 # 显示用 emoji
        "max_result_size_chars",  # 结果截断阈值（可选）
    )
```

每一个你注册的工具，最终都变成一个 `ToolEntry` 实例存储在注册表中。

### A.2.2 ToolRegistry——模块级单例

`ToolRegistry` 是一个**模块级单例**：

```python
# tools/registry.py 底部
registry = ToolRegistry()
```

所有工具文件通过 `from tools.registry import registry` 获取同一实例。注册表内部使用 `threading.RLock` 保证线程安全。核心数据结构包括：

- `_tools: Dict[str, ToolEntry]` —— 工具名到元数据的映射
- `_toolset_checks: Dict[str, Callable]` —— 工具集到可用性检查函数的映射
- `_toolset_aliases: Dict[str, str]` —— 别名到规范名的映射

### A.2.3 响应辅助函数——tool_error 与 tool_result

`registry.py` 提供了两个辅助函数，省去手写 `json.dumps()` 的样板代码：

```python
from tools.registry import registry, tool_error, tool_result

# 错误响应
return tool_error("something went wrong")
# → '{"error": "something went wrong"}'

# 成功响应
return tool_result(success=True, data=payload)
# → '{"success": true, "data": ...}'
```

**关键规则：所有 Handler 函数必须返回 JSON 字符串。** 这是 Hermes Agent 工具系统的铁律。

---

## A.3 Step-by-Step：开发一个天气查询工具

下面以一个完整的天气查询工具为例，走一遍从零到上线的全流程。

### 步骤 1：创建工具文件

在 `tools/` 目录下创建 `weather_tool.py`：

```python
#!/usr/bin/env python3
"""Weather Tool — Query current weather conditions."""

import json
import os
import logging
import httpx

logger = logging.getLogger(__name__)
```

文件开头很简单——标准的 Python 模块头。注意**不要在这里导入 `model_tools`**，否则会触发循环导入。

### 步骤 2：定义可用性检查函数

```python
def check_weather_requirements() -> bool:
    """检查 WEATHER_API_KEY 环境变量是否已设置。
    
    返回 False 时，工具不会出现在 LLM 的工具列表中。
    """
    return bool(os.getenv("WEATHER_API_KEY"))
```

`check_fn` 是工具的"门禁"。它的行为规则：

- 返回 `True` → 工具对 LLM 可见
- 返回 `False` → 工具被隐藏，LLM 看不到它
- 抛出异常 → 被捕获，视为不可用
- **同一 Toolset 的多个工具共享 `check_fn`**：第一个注册的 `check_fn` 被存储在 `_toolset_checks[toolset]`
- 结果在同一次 `get_definitions()` 调用中被缓存

常见的检查模式：

```python
# 模式 A：无外部依赖，始终可用
def check_requirements() -> bool:
    return True

# 模式 B：检查 API 密钥
def check_requirements() -> bool:
    return bool(os.getenv("YOUR_API_KEY"))

# 模式 C：检查外部服务可达性
def check_requirements() -> bool:
    url = os.getenv("SERVICE_URL", "").strip()
    token = os.getenv("SERVICE_TOKEN", "").strip()
    return bool(url and token)
```

### 步骤 3：实现 Handler

```python
def weather_tool(city: str) -> str:
    """查询指定城市的当前天气。"""
    api_key = os.getenv("WEATHER_API_KEY")
    if not api_key:
        return tool_error("WEATHER_API_KEY not configured")
    
    try:
        resp = httpx.get(
            "https://api.weatherapi.com/v1/current.json",
            params={"key": api_key, "q": city},
            timeout=10,
        )
        resp.raise_for_status()
        data = resp.json()
        return tool_result(
            city=city,
            temp_c=data["current"]["temp_c"],
            condition=data["current"]["condition"]["text"],
            humidity=data["current"]["humidity"],
        )
    except httpx.HTTPError as e:
        return tool_error(f"Weather API request failed: {e}")
```

几个要点：

1. **返回值必须是 JSON 字符串**——使用 `tool_result()` 和 `tool_error()` 辅助函数
2. **异常要捕获**——不要让异常从 Handler 中逃逸，用 `tool_error()` 包装
3. **参数是强类型的**——`model_tools.py` 中的 `coerce_tool_args()` 会自动将 LLM 返回的字符串类型参数转换为 Schema 声明的类型（例如 `"42"` → `42`、`"true"` → `True`）

### 步骤 4：定义 OpenAI Function-Calling Schema

```python
WEATHER_SCHEMA = {
    "name": "weather",
    "description": "Get current weather conditions for a city.",
    "parameters": {
        "type": "object",
        "properties": {
            "city": {
                "type": "string",
                "description": "City name (e.g., 'Tokyo', 'San Francisco')"
            }
        },
        "required": ["city"]
    }
}
```

**Schema 设计要点**：

- `description` 是 LLM 决策的核心依据——写清楚什么时候用、什么时候不用
- 行为指导（behavioral guidance）也可以放在 description 里
- Schema 在 import 时生成，是静态的、可缓存的
- Schema 中的 `name` 必须与 `registry.register(name=...)` 一致

参数类型支持：`string`、`integer`、`number`、`boolean`、`array`、`object`。还可以通过 `enum` 限制取值范围、`default` 指定默认值。

### 步骤 5：调用 registry.register()

```python
from tools.registry import registry, tool_error, tool_result

registry.register(
    name="weather",                    # 工具名，必须全局唯一
    toolset="weather",                 # 所属工具集
    schema=WEATHER_SCHEMA,             # OpenAI function-calling schema
    handler=lambda args, **kw: weather_tool(
        city=args.get("city", ""),
    ),
    check_fn=check_weather_requirements,
    requires_env=["WEATHER_API_KEY"],
    emoji="🌤️",
)
```

**注意 Handler 的桥接模式。** Registry 的调用签名是 `handler(args: dict, **kwargs)`，其中 `kwargs` 包含 `task_id`、`user_task`、`store`、`callback` 等上下文。你通过 `lambda` 适配器将其解构为你的函数参数：

```python
# 简单工具——只需参数
handler=lambda args, **kw: your_tool(args.get("param", ""))

# 需要 task_id——用于终端/浏览器会话隔离
handler=lambda args, **kw: your_tool(
    param=args.get("param", ""),
    task_id=kw.get("task_id"),
)

# 需要 agent 级状态——如 todo 的 store
handler=lambda args, **kw: todo_tool(
    todos=args.get("todos"),
    store=kw.get("store"),
)
```

> ⚠️ **`registry.register()` 必须在模块顶层调用。** 不能放在函数内部。自动发现的 AST 扫描只检查 `tree.body`（模块体），不会递归进入函数体。

### 步骤 6：注册到 Toolset

打开 `toolsets.py`，有两种方式：

**方式 A：加入 `_HERMES_CORE_TOOLS`（推荐，所有平台可用）**

```python
_HERMES_CORE_TOOLS = [
    # ... 现有工具 ...
    "weather",     # ← 新增
]
```

这会让工具出现在 CLI、Telegram、Discord、Slack 等所有平台的工具集中。

**方式 B：定义独立 Toolset（可选启用）**

```python
TOOLSETS = {
    # ... 现有 toolsets ...
    "weather": {
        "description": "Weather information tool",
        "tools": ["weather"],
        "includes": []
    },
}
```

用户通过 `--enabled-toolsets weather` 选择性启用。如果你的工具是实验性的或只在特定场景使用，推荐这种方式。

**方式 C：组合型 Toolset（包含其他 Toolsets）**

```python
"outdoor_assistant": {
    "description": "Outdoor activity planning toolkit",
    "tools": ["weather"],
    "includes": ["web", "file"]  # 递归解析，支持循环检测
},
```

### 步骤 7：自动发现——你不需要做任何事

**无需任何额外操作！** `discover_builtin_tools()` 在 `model_tools.py` 导入时自动执行：

```python
# model_tools.py 顶层代码
from tools.registry import discover_builtin_tools, registry
discover_builtin_tools()  # 扫描 tools/*.py 并导入
```

自动发现的工作原理：

1. **AST 静态扫描**：`_module_registers_tools(path)` 用 Python AST 解析每个 `tools/*.py` 文件，只检查模块体中是否有 `registry.register()` 调用
2. **排除列表**：`__init__.py`、`registry.py`、`mcp_tool.py` 被跳过
3. **动态导入**：通过 `importlib.import_module(mod_name)` 导入，导入时 `registry.register()` 自动执行
4. **错误容忍**：导入失败只 log warning，不影响其他工具

---

## A.4 完整代码汇总

### `tools/weather_tool.py`（完整版）

```python
#!/usr/bin/env python3
"""Weather Tool — Query current weather conditions."""

import json
import os
import logging
import httpx

logger = logging.getLogger(__name__)


# ─── 可用性检查 ──────────────────────────────────────────────
def check_weather_requirements() -> bool:
    """检查 WEATHER_API_KEY 是否已设置。"""
    return bool(os.getenv("WEATHER_API_KEY"))


# ─── 工具核心逻辑 ────────────────────────────────────────────
def weather_tool(city: str) -> str:
    """查询指定城市的当前天气。"""
    api_key = os.getenv("WEATHER_API_KEY")
    if not api_key:
        return tool_error("WEATHER_API_KEY not configured")
    
    try:
        resp = httpx.get(
            "https://api.weatherapi.com/v1/current.json",
            params={"key": api_key, "q": city},
            timeout=10,
        )
        resp.raise_for_status()
        data = resp.json()
        return tool_result(
            city=city,
            temp_c=data["current"]["temp_c"],
            condition=data["current"]["condition"]["text"],
            humidity=data["current"]["humidity"],
        )
    except httpx.HTTPError as e:
        logger.error("Weather API failed: %s", e)
        return tool_error(f"Weather API request failed: {e}")


# ─── OpenAI Function-Calling Schema ─────────────────────────
WEATHER_SCHEMA = {
    "name": "weather",
    "description": "Get current weather conditions for a city.",
    "parameters": {
        "type": "object",
        "properties": {
            "city": {
                "type": "string",
                "description": "City name (e.g., 'Tokyo', 'San Francisco')"
            }
        },
        "required": ["city"]
    }
}


# ─── 注册 ────────────────────────────────────────────────────
from tools.registry import registry, tool_error, tool_result

registry.register(
    name="weather",
    toolset="weather",
    schema=WEATHER_SCHEMA,
    handler=lambda args, **kw: weather_tool(city=args.get("city", "")),
    check_fn=check_weather_requirements,
    requires_env=["WEATHER_API_KEY"],
    emoji="🌤️",
)
```

### `toolsets.py`（修改部分）

```python
_HERMES_CORE_TOOLS = [
    # ... 现有工具 ...
    "weather",     # ← 新增
]
```

---

## A.5 调度与执行流

当 LLM 决定调用你的工具时，完整的执行路径如下：

```
LLM 返回 tool_call（name="weather", arguments={"city": "Tokyo"}）
    ↓
run_agent.py: _invoke_tool(tool_call)
    ↓
model_tools.py: handle_function_call(name, args, task_id, ...)
    ↓
    1. coerce_tool_args()           — 类型强制转换
    2. 检查 _AGENT_LOOP_TOOLS      — 拦截 agent 级工具
    3. 检查 plugin hook             — pre_tool_call 插件可阻止执行
    4. notify_other_tool_call()     — 重置读循环计数器
    ↓
registry.dispatch("weather", {"city": "Tokyo"}, task_id=..., ...)
    ↓
    1. get_entry("weather")         — 查找工具
    2. 异步桥接判断                  — is_async → _run_async()
    3. entry.handler(args, **kw)    — 执行你的 handler
    4. 异常捕获 → JSON error         — 保证不抛异常
    ↓
结果字符串返回给 agent loop → 追加到 messages
```

对于**异步 Handler**（`is_async=True`），Registry 自动通过 `_run_async()` 桥接，你无需手动处理事件循环：

```python
if entry.is_async:
    from model_tools import _run_async
    return _run_async(entry.handler(args, **kwargs))
```

---

## A.6 测试你的工具

### 单元测试

```python
# tests/test_weather_tool.py
import json
from tools.weather_tool import weather_tool, check_weather_requirements

def test_check_requirements_missing_key(monkeypatch):
    monkeypatch.delenv("WEATHER_API_KEY", raising=False)
    assert check_weather_requirements() is False

def test_check_requirements_with_key(monkeypatch):
    monkeypatch.setenv("WEATHER_API_KEY", "test-key-123")
    assert check_weather_requirements() is True
```

### 注册测试

```python
def test_tool_registered():
    from tools.registry import registry
    entry = registry.get_entry("weather")
    assert entry is not None
    assert entry.toolset == "weather"
    assert entry.emoji == "🌤️"

def test_schema_valid():
    from tools.registry import registry
    schema = registry.get_schema("weather")
    assert schema["name"] == "weather"
    assert "parameters" in schema
    assert "required" in schema["parameters"]
```

### 集成测试

```python
def test_dispatch():
    from tools.registry import registry
    result = registry.dispatch("weather", {"city": "Tokyo"})
    data = json.loads(result)
    assert "error" not in data  # 需要真实 API Key 才能通过
```

### Toolset 测试

```python
def test_toolset_includes_tool():
    from toolsets import resolve_toolset
    tools = resolve_toolset("weather")
    assert "weather" in tools
```

### 快速验证命令

```bash
# 1. 检查注册成功
python -c "
from tools.weather_tool import *
from tools.registry import registry
print(registry.get_entry('weather'))
"

# 2. 检查 toolset 解析
python -c "
from toolsets import resolve_toolset
print(resolve_toolset('weather'))
"

# 3. 运行测试
pytest tests/test_weather_tool.py -v
```

---

## A.7 不同复杂度的工具模式

根据你的需求，选择合适的模式：

### 模式 1：最简工具（始终可用）

以 `todo_tool.py` 为代表，无外部依赖，`check_fn` 始终返回 `True`：

```python
registry.register(
    name="todo",
    toolset="todo",
    schema=TODO_SCHEMA,
    handler=lambda args, **kw: todo_tool(
        todos=args.get("todos"),
        merge=args.get("merge", False),
        store=kw.get("store"),
    ),
    check_fn=check_todo_requirements,
    emoji="📋",
)
```

### 模式 2：API 密钥依赖 + 异步工具

以 `web_tools.py` 为代表，异步 Handler 需要设置 `is_async=True`：

```python
registry.register(
    name="web_extract",
    toolset="web",
    schema=WEB_EXTRACT_SCHEMA,
    handler=lambda args, **kw: web_extract_tool(
        args.get("urls", [])[:5] if isinstance(args.get("urls"), list) else [],
        "markdown",
    ),
    check_fn=check_web_api_key,
    requires_env=_web_requires_env(),
    is_async=True,               # ← handler 返回 coroutine
    emoji="📄",
    max_result_size_chars=100_000,
)
```

### 模式 3：多工具共享一个 Toolset

以 `homeassistant_tool.py` 为代表，4 个工具共享同一个 Toolset 和 `check_fn`：

```python
_check = _check_ha_available

registry.register(name="ha_list_entities", toolset="homeassistant",
                  schema=HA_LIST_ENTITIES_SCHEMA, handler=_handle_list_entities,
                  check_fn=_check, emoji="🏠")

registry.register(name="ha_get_state", toolset="homeassistant",
                  schema=HA_GET_STATE_SCHEMA, handler=_handle_get_state,
                  check_fn=_check, emoji="🏠")
# ... 共 4 个工具
```

### 模式 4：大规模 Toolset

以 `browser_tool.py` 为代表，10 个工具使用 Schema 映射表减少重复：

```python
_BROWSER_SCHEMA_MAP = {s["name"]: s for s in BROWSER_TOOL_SCHEMAS}

registry.register(
    name="browser_navigate",
    toolset="browser",
    schema=_BROWSER_SCHEMA_MAP["browser_navigate"],
    handler=lambda args, **kw: browser_navigate(
        url=args.get("url", ""), task_id=kw.get("task_id")),
    check_fn=check_browser_requirements,
    emoji="🌐",
)
```

---

## A.8 常见陷阱与排错

### ❌ 陷阱 1：Handler 返回非 JSON 字符串

```python
# 错误 ❌
handler=lambda args, **kw: "Success!"

# 正确 ✅
handler=lambda args, **kw: tool_result(success=True, message="Success!")
```

### ❌ 陷阱 2：registry.register() 放在函数内部

```python
# 错误 ❌ — AST 扫描检测不到，工具不会被自动发现
def setup():
    registry.register(name="my_tool", ...)

# 正确 ✅ — 放在模块顶层
registry.register(name="my_tool", ...)
```

### ❌ 陷阱 3：循环导入

```python
# 错误 ❌ — 在工具文件中从 model_tools 导入
from model_tools import handle_function_call  # 循环导入！

# 正确 ✅ — 只从 tools.registry 导入
from tools.registry import registry, tool_error, tool_result
```

### ❌ 陷阱 4：忘记在 toolsets.py 中注册

工具被 `registry.register()` 注册后**仅表示它存在于注册表中**。要让它对用户可用，必须在 `toolsets.py` 的某个 Toolset 中列出它，否则 `get_tool_definitions()` 不会包含它。

### ❌ 陷阱 5：Schema name 与 register name 不一致

```python
# 危险 ⚠️
SCHEMA = {"name": "my_tool_v2", ...}
registry.register(name="my_tool", schema=SCHEMA, ...)
# → registry 会用 entry.name 覆盖 schema 中的 name
#   虽然不会报错，但会造成困惑
```

### ❌ 陷阱 6：异步 Handler 但没设 is_async=True

```python
# 错误 ❌ — handler 返回 coroutine 但 is_async=False
async def my_handler(args):
    result = await do_something()
    return json.dumps(result)

registry.register(
    handler=lambda args, **kw: my_handler(args),
    # is_async=False,  # 默认值！ → 返回 coroutine 对象而不是字符串
)

# 正确 ✅
registry.register(
    handler=lambda args, **kw: my_handler(args),
    is_async=True,  # ← registry.dispatch 自动通过 _run_async() 桥接
)
```

### ❌ 陷阱 7：忽略 task_id

如果工具涉及终端操作或浏览器会话，必须透传 `task_id` 以实现会话隔离（特别是子代理场景）：

```python
handler=lambda args, **kw: your_tool(
    param=args.get("param", ""),
    task_id=kw.get("task_id"),  # ← 不要忘记
)
```

### ❌ 陷阱 8：使用硬编码路径

```python
# 错误 ❌
state_dir = Path.home() / ".hermes" / "my_tool_data"

# 正确 ✅ — 支持多 profile
from hermes_cli.config import get_hermes_home
state_dir = get_hermes_home() / "my_tool_data"
```

---

## A.9 注册防护机制

Hermes Agent 的注册表内置了防护机制，防止工具命名冲突：

```python
def register(self, name, toolset, schema, handler, ...):
    existing = self._tools.get(name)
    if existing and existing.toolset != toolset:
        # MCP-to-MCP 覆盖是允许的（服务器刷新场景）
        both_mcp = (existing.toolset.startswith("mcp-")
                    and toolset.startswith("mcp-"))
        if both_mcp:
            logger.debug("MCP toolset overwrite: %s → %s",
                         existing.toolset, toolset)
        else:
            # 内置工具不能被覆盖
            logger.error("Tool registration REJECTED: '%s'", name)
            return  # 静默拒绝，不抛异常
```

这意味着：

- **内置工具不会被插件或 MCP 工具覆盖**——如果名称冲突，后注册的会被拒绝
- **MCP 工具之间可以覆盖**——支持服务器刷新场景
- Registry 还提供 `deregister()` 方法用于 MCP 的 nuke-and-repave 模式

---

## A.10 高级话题

### MCP 动态工具注册

MCP 工具通过 `tools/mcp_tool.py` 动态注册，不参与 AST 静态扫描（`mcp_tool.py` 在排除列表中）。MCP 工具的 Toolset 名格式为 `mcp-{server_name}`，并注册别名：

```python
registry.register(
    name=tool_name,
    toolset=f"mcp-{server_name}",
    schema=schema,
    handler=handler,
    check_fn=check_fn,
    is_async=True,
)
registry.register_toolset_alias(server_name, f"mcp-{server_name}")
```

### 结果截断

工具可通过 `max_result_size_chars` 控制结果大小。未设置时使用全局默认值（来自 `tools/budget_config.py` 的 `DEFAULT_RESULT_SIZE_CHARS`）。

### 线程安全

`ToolRegistry` 使用 `threading.RLock` 保护所有写操作，读操作通过 `_snapshot_state()` 返回稳定快照。这在 MCP 动态刷新和并行工具执行时尤为重要。

---

## A.11 关键源码定位

| 概念 | 文件 | 位置 |
|------|------|------|
| ToolEntry 定义 | `tools/registry.py` | class ToolEntry |
| ToolRegistry.register() | `tools/registry.py` | L176-227 |
| discover_builtin_tools() | `tools/registry.py` | L56-73 |
| _module_registers_tools() | `tools/registry.py` | L41-53 |
| tool_error() / tool_result() | `tools/registry.py` | L456-482 |
| dispatch() | `tools/registry.py` | L292-309 |
| get_definitions() | `tools/registry.py` | L258-286 |
| _HERMES_CORE_TOOLS | `toolsets.py` | L31-63 |
| TOOLSETS dict | `toolsets.py` | L68-397 |
| resolve_toolset() | `toolsets.py` | L447-497 |
| get_tool_definitions() | `model_tools.py` | L196-315 |
| handle_function_call() | `model_tools.py` | L421-533 |
| coerce_tool_args() | `model_tools.py` | L334-370 |

---

## A.12 Checklist：添加新工具前的自查清单

在提交代码之前，对照这份清单检查：

- [ ] 工具文件放在 `tools/` 目录下
- [ ] `registry.register()` 在模块**顶层**调用（不在函数内部）
- [ ] Schema 的 `name` 与 `registry.register(name=...)` 一致
- [ ] Handler 返回 **JSON 字符串**（使用 `tool_result()` / `tool_error()`）
- [ ] 异步 Handler 设置了 `is_async=True`
- [ ] `check_fn` 在条件不满足时返回 `False`（而非抛异常）
- [ ] 工具名已加入 `toolsets.py` 的某个 Toolset
- [ ] 工具文件**不**从 `model_tools` 导入（避免循环导入）
- [ ] 涉及终端/浏览器操作时透传了 `task_id`
- [ ] 编写了单元测试和注册测试

---

## 小结

> 在 Hermes Agent 中添加新工具 = **一个文件**定义 Schema + Handler + check_fn 并调用 `registry.register()`，**另一个文件**（`toolsets.py`）中将工具名加入对应的工具集列表。自动发现机制通过 AST 扫描完成其余一切。

这套机制的设计体现了 Hermes Agent 在工具系统上的两个核心原则：

1. **约定优于配置**——文件放对位置、顶层注册，就能被自动发现
2. **关注点分离**——工具文件只负责自身逻辑，注册表负责管理，`model_tools` 负责调度，三者互不耦合
