---
Task: TA-R
Status: DRAFT_DONE
Date: 2026-04-15
Chapter: 附录 A — 实战：添加一个新工具
Keywords: 完整开发流程, registry.register, tool schema, toolset, auto-discovery, handler, check_fn
Sources:
  - hermes-source/tools/registry.py (ToolRegistry, ToolEntry, discover_builtin_tools)
  - hermes-source/toolsets.py (TOOLSETS, _HERMES_CORE_TOOLS, resolve_toolset)
  - hermes-source/model_tools.py (get_tool_definitions, handle_function_call, discover_builtin_tools)
  - hermes-source/tools/todo_tool.py (最简注册示例)
  - hermes-source/tools/web_tools.py (async 工具 + check_fn + requires_env 完整示例)
  - hermes-source/tools/homeassistant_tool.py (多工具单 toolset 示例)
  - hermes-source/tools/browser_tool.py (10 个工具单 toolset 示例)
  - hermes-source/tools/clarify_tool.py (回调模式示例)
  - hermes-source/AGENTS.md (官方指南 §Adding New Tools)
---

# 附录 A — 实战：添加一个新工具 (Adding a New Tool: Step-by-Step)

## 1. 全局概览 (Overview)

在 Hermes Agent 中添加一个新工具**只需修改 2 个文件**：

| 步骤 | 文件 | 做什么 |
|------|------|--------|
| ① | `tools/your_tool.py` | 创建工具文件：定义 schema、handler、check_fn，调用 `registry.register()` |
| ② | `toolsets.py` | 将工具名加入 `_HERMES_CORE_TOOLS` 列表或定义新的 toolset |

**不需要修改 `model_tools.py`**。自动发现机制 (auto-discovery) 会扫描 `tools/` 目录下的所有 `.py` 文件，找到包含顶层 `registry.register()` 调用的模块并自动导入。

### 文件依赖链 (File Dependency Chain)

```
tools/registry.py     ← 无依赖，所有工具文件导入它
       ↑
tools/*.py             ← 每个文件在 module-import 时调用 registry.register()
       ↑
model_tools.py         ← 导入 tools.registry，触发 discover_builtin_tools()
       ↑
run_agent.py / cli.py  ← 消费 get_tool_definitions() 和 handle_function_call()
```

这个链条是**防循环导入 (circular-import safe)** 的：`registry.py` 不导入任何工具文件或 `model_tools`。

---

## 2. 核心概念解析 (Core Concepts)

### 2.1 ToolEntry — 工具元数据对象

`ToolEntry` 是注册表中每个工具的元数据结构，使用 `__slots__` 优化内存：

```python
class ToolEntry:
    __slots__ = (
        "name",                  # 工具名，如 "web_search"
        "toolset",               # 所属工具集，如 "web"
        "schema",                # OpenAI function-calling schema (dict)
        "handler",               # 处理函数 Callable
        "check_fn",              # 可用性检查函数 Callable -> bool
        "requires_env",          # 需要的环境变量列表，如 ["API_KEY"]
        "is_async",              # 是否为异步处理函数
        "description",           # 描述（从 schema 中提取的备用值）
        "emoji",                 # 显示用 emoji，如 "🔍"
        "max_result_size_chars",  # 结果截断阈值（可选）
    )
```

### 2.2 ToolRegistry — 单例注册表 (Singleton Registry)

`ToolRegistry` 是**模块级单例** (module-level singleton)：

```python
# tools/registry.py 底部
registry = ToolRegistry()
```

所有工具文件通过 `from tools.registry import registry` 获取同一实例。注册表内部使用 `threading.RLock` 保证线程安全——特别是 MCP 动态刷新时的并发安全。

核心数据结构：
- `_tools: Dict[str, ToolEntry]` — 工具名到元数据的映射
- `_toolset_checks: Dict[str, Callable]` — 工具集到可用性检查函数的映射
- `_toolset_aliases: Dict[str, str]` — 别名到规范名的映射（主要用于 MCP）

### 2.3 工具响应序列化辅助函数 (Response Helpers)

`registry.py` 提供了两个辅助函数，消除了工具文件中反复出现的 `json.dumps()` 样板代码：

```python
from tools.registry import registry, tool_error, tool_result

# 错误响应
return tool_error("something went wrong")        # → '{"error": "something went wrong"}'
return tool_error("bad input", code=400)          # → '{"error": "bad input", "code": 400}'

# 成功响应
return tool_result(success=True, data=payload)    # → '{"success": true, "data": ...}'
return tool_result({"key": "value"})              # → '{"key": "value"}'
```

**关键规则：所有 handler 函数必须返回 JSON 字符串 (All handlers MUST return a JSON string)**。

---

## 3. 完整开发流程 (Step-by-Step Development Flow)

### 步骤 1：创建工具文件 (Create Tool File)

在 `tools/` 目录下创建 `your_tool.py`。以下是一个完整模板：

```python
#!/usr/bin/env python3
"""
Your Tool Module - Brief description

详细说明这个工具做什么、为什么需要它。
"""

import json
import os
import logging
from typing import Dict, Any, Optional

logger = logging.getLogger(__name__)


# ─── 可用性检查 (Availability Check) ──────────────────────────
def check_requirements() -> bool:
    """检查工具运行的前置条件是否满足。
    
    返回 True 表示可用，False 表示不可用。
    当 check_fn 返回 False 时，工具不会出现在 LLM 的 tool schemas 中。
    """
    return bool(os.getenv("YOUR_API_KEY"))


# ─── 工具核心逻辑 (Tool Core Logic) ──────────────────────────
def your_tool(param: str, task_id: str = None) -> str:
    """工具的核心实现。
    
    Args:
        param: 工具参数
        task_id: 任务隔离 ID（由 terminal/browser 等工具使用）
    
    Returns:
        JSON string — 必须是 JSON 字符串！
    """
    try:
        # ... 核心逻辑 ...
        result = {"success": True, "data": "your result here"}
        return json.dumps(result, ensure_ascii=False)
    except Exception as e:
        logger.error("your_tool failed: %s", e)
        return tool_error(f"Tool execution failed: {e}")


# ─── OpenAI Function-Calling Schema ─────────────────────────
YOUR_TOOL_SCHEMA = {
    "name": "your_tool",
    "description": (
        "一段清晰的描述，告诉 LLM 这个工具做什么、什么时候用它。"
        "行为指导也可以放在描述中（baked into schema, cached）。"
    ),
    "parameters": {
        "type": "object",
        "properties": {
            "param": {
                "type": "string",
                "description": "参数说明"
            }
        },
        "required": ["param"]
    }
}


# ─── 注册 (Registration) ─────────────────────────────────────
from tools.registry import registry, tool_error

registry.register(
    name="your_tool",           # 工具名，必须全局唯一
    toolset="your_toolset",     # 所属工具集
    schema=YOUR_TOOL_SCHEMA,    # OpenAI function-calling schema
    handler=lambda args, **kw: your_tool(
        param=args.get("param", ""),
        task_id=kw.get("task_id"),
    ),
    check_fn=check_requirements,         # 可选，检查前置条件
    requires_env=["YOUR_API_KEY"],       # 可选，元数据用
    emoji="⚡",                          # 可选，CLI 显示用
    # is_async=False,                    # 可选，True 表示 handler 返回 coroutine
    # max_result_size_chars=100_000,     # 可选，结果截断阈值
)
```

### 步骤 2：定义 Schema (Define the Schema)

Schema 遵循 **OpenAI function-calling** 规范：

```python
SCHEMA = {
    "name": "tool_name",          # 必须与 registry.register(name=...) 一致
    "description": "...",          # LLM 看到的描述，是决定何时调用工具的关键
    "parameters": {
        "type": "object",
        "properties": {
            "param1": {
                "type": "string",      # string | integer | number | boolean | array | object
                "description": "...",
                "enum": ["a", "b"],    # 可选：限制取值范围
            },
            "param2": {
                "type": "boolean",
                "description": "...",
                "default": False       # 可选：默认值
            }
        },
        "required": ["param1"]         # 必填参数列表
    }
}
```

**Schema 设计要点**：
- description 是 LLM 决策的核心依据——写清楚什么时候用、什么时候不用
- 行为指导 (behavioral guidance) 放在 description 里，不放在 system prompt 中（todo_tool 的做法）
- Schema 在 import 时生成，是静态的、可缓存的

**参数类型强制转换 (Argument Type Coercion)**：`model_tools.py` 中的 `coerce_tool_args()` 会自动将 LLM 返回的字符串类型参数转换为 schema 声明的类型。例如 `"42"` → `42`、`"true"` → `True`。这意味着你的 schema 声明类型很重要。

### 步骤 3：实现 Handler (Implement Handler)

Handler 函数通过 `lambda` 适配器桥接 registry 的调用约定：

```python
# Registry 调用签名：handler(args: dict, **kwargs) -> str
# kwargs 包含：task_id, user_task, store, callback 等上下文

# 简单工具
handler=lambda args, **kw: your_tool(args.get("param", ""))

# 需要 task_id 的工具（用于终端/浏览器会话隔离）
handler=lambda args, **kw: your_tool(
    param=args.get("param", ""),
    task_id=kw.get("task_id"),
)

# 需要 agent 级状态的工具（如 todo 的 store）
handler=lambda args, **kw: todo_tool(
    todos=args.get("todos"),
    merge=args.get("merge", False),
    store=kw.get("store"),     # 由 run_agent.py 注入
)

# 需要回调的工具（如 clarify）
handler=lambda args, **kw: clarify_tool(
    question=args.get("question", ""),
    choices=args.get("choices"),
    callback=kw.get("callback"),  # 由 run_agent.py 注入
)
```

**handler 调用路径**：
```
run_agent.py → handle_function_call() → registry.dispatch(name, args, **kwargs)
    → entry.handler(args, task_id=task_id, user_task=user_task, ...)
```

对于**异步 handler**（`is_async=True`），registry 自动通过 `_run_async()` 桥接：
```python
if entry.is_async:
    from model_tools import _run_async
    return _run_async(entry.handler(args, **kwargs))
```

### 步骤 4：实现 check_fn (Implement Availability Check)

`check_fn` 决定工具是否对 LLM **可见**：

```python
# 模式 A：无外部依赖，始终可用
def check_requirements() -> bool:
    return True

# 模式 B：检查 API 密钥
def check_requirements() -> bool:
    return bool(os.getenv("YOUR_API_KEY"))

# 模式 C：检查多个条件（如 web_tools）
def check_web_api_key() -> bool:
    backend = _get_backend()
    return _is_backend_available(backend)

# 模式 D：检查外部服务可达性（如 homeassistant）
def _check_ha_available() -> bool:
    url = os.getenv("HASS_URL", "").strip().rstrip("/")
    token = os.getenv("HASS_TOKEN", "").strip()
    return bool(url and token)
```

**重要行为**：
- `check_fn` 返回 `False` → 工具不出现在 `get_definitions()` 结果中 → LLM 看不到它
- `check_fn` 抛异常 → 被捕获，视为不可用（`_evaluate_toolset_check` 的 try/except）
- **同一 toolset 的多个工具共享 check_fn**：第一个注册的 `check_fn` 被存储在 `_toolset_checks[toolset]`
- `check_fn` 结果在 `get_definitions()` 调用中被缓存（同一 check_fn 对象只执行一次）

### 步骤 5：注册到 Toolset (Register in Toolsets)

在 `toolsets.py` 中，有两种方式让工具对用户可用：

#### 方式 A：加入 `_HERMES_CORE_TOOLS`（所有平台共享）

```python
_HERMES_CORE_TOOLS = [
    # ... 现有工具 ...
    "your_tool",          # ← 添加你的工具名
]
```

这会让工具出现在 CLI、Telegram、Discord、Slack 等所有平台的工具集中。

#### 方式 B：定义新的独立 Toolset

```python
TOOLSETS = {
    # ... 现有 toolsets ...
    
    "your_toolset": {
        "description": "描述这个工具集做什么",
        "tools": ["your_tool", "your_other_tool"],
        "includes": []    # 可选：包含其他 toolset
    },
}
```

然后用户可以通过 `--enabled-toolsets your_toolset` 选择性启用。

#### 方式 C：组合型 Toolset（包含其他 Toolsets）

```python
"debugging": {
    "description": "Debugging and troubleshooting toolkit",
    "tools": ["terminal", "process"],
    "includes": ["web", "file"]  # 递归解析，支持循环检测
},
```

### 步骤 6：自动发现生效 (Auto-Discovery Takes Effect)

**无需任何额外操作！** `discover_builtin_tools()` 在 `model_tools.py` 导入时自动执行：

```python
# model_tools.py 顶层代码
from tools.registry import discover_builtin_tools, registry
discover_builtin_tools()  # 扫描 tools/*.py 并导入
```

自动发现的工作原理：

1. **AST 静态扫描**：`_module_registers_tools(path)` 用 Python AST 解析每个 `tools/*.py` 文件，检查是否有**顶层** `registry.register()` 调用

```python
def _module_registers_tools(module_path: Path) -> bool:
    source = module_path.read_text(encoding="utf-8")
    tree = ast.parse(source, filename=str(module_path))
    # 只检查 module body 的顶层语句
    return any(_is_registry_register_call(stmt) for stmt in tree.body)
```

2. **排除列表**：`__init__.py`、`registry.py`、`mcp_tool.py` 被跳过
3. **动态导入**：通过 `importlib.import_module(mod_name)` 导入，导入时 `registry.register()` 自动执行
4. **错误容忍**：导入失败只 log warning，不影响其他工具

**关键细节**：`registry.register()` 必须在**模块顶层**调用，不能在函数内部调用。AST 扫描只检查 `tree.body`（模块体），不会递归进入函数体。

---

## 4. 不同复杂度的工具模式 (Tool Patterns by Complexity)

### 模式 1：最简工具（Always Available）— todo_tool.py

```python
# 无外部依赖，check_fn 始终返回 True
# handler 需要 agent 级状态 (store)
# 被 run_agent.py 拦截处理（_AGENT_LOOP_TOOLS）

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

### 模式 2：API 密钥依赖工具 — web_tools.py

```python
# 需要 API 密钥，check_fn 检查多个后端
# 异步 handler (is_async=True)
# 自定义结果大小限制 (max_result_size_chars)

registry.register(
    name="web_search",
    toolset="web",
    schema=WEB_SEARCH_SCHEMA,
    handler=lambda args, **kw: web_search_tool(args.get("query", ""), limit=5),
    check_fn=check_web_api_key,
    requires_env=_web_requires_env(),  # 动态计算需要的环境变量列表
    emoji="🔍",
    max_result_size_chars=100_000,
)

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
    is_async=True,              # ← handler 返回 coroutine
    emoji="📄",
    max_result_size_chars=100_000,
)
```

### 模式 3：多工具单 Toolset — homeassistant_tool.py

```python
# 4 个工具共享同一个 toolset 和 check_fn
_check = _check_ha_available

registry.register(name="ha_list_entities", toolset="homeassistant",
                  schema=HA_LIST_ENTITIES_SCHEMA, handler=_handle_list_entities,
                  check_fn=_check, emoji="🏠")

registry.register(name="ha_get_state", toolset="homeassistant",
                  schema=HA_GET_STATE_SCHEMA, handler=_handle_get_state,
                  check_fn=_check, emoji="🏠")
# ... 共 4 个工具
```

### 模式 4：大规模 Toolset — browser_tool.py

```python
# 10 个工具，使用 schema 映射表减少重复
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
# ... 共 10 个 register 调用
```

### 模式 5：Agent-Loop 拦截工具 — todo, memory, session_search, delegate_task

这些工具在 `model_tools.py` 中被特殊标记：

```python
_AGENT_LOOP_TOOLS = {"todo", "memory", "session_search", "delegate_task"}
```

它们的 `handle_function_call()` 返回占位错误，实际处理由 `run_agent.py` 的 agent loop 完成（因为需要 agent 级状态如 TodoStore、MemoryStore 等）。

---

## 5. 注册防护机制 (Registration Safeguards)

### 名称冲突检测 (Name Collision Detection)

```python
def register(self, name, toolset, schema, handler, ...):
    existing = self._tools.get(name)
    if existing and existing.toolset != toolset:
        # MCP-to-MCP 覆盖是允许的（服务器刷新场景）
        both_mcp = (existing.toolset.startswith("mcp-") and toolset.startswith("mcp-"))
        if both_mcp:
            logger.debug("MCP toolset overwrite: %s → %s", existing.toolset, toolset)
        else:
            # 内置工具不能被插件/MCP 覆盖
            logger.error("Tool registration REJECTED: '%s' would shadow '%s'", name, existing.toolset)
            return  # 静默拒绝，不抛异常
```

### 注销机制 (Deregistration)

```python
def deregister(self, name: str) -> None:
    """移除工具，清理关联的 toolset check 和 alias。"""
    # 用于 MCP 的 nuke-and-repave 模式
```

---

## 6. 调度与执行流 (Dispatch & Execution Flow)

```
LLM 返回 tool_call
    ↓
run_agent.py: _invoke_tool(tool_call)
    ↓
model_tools.py: handle_function_call(name, args, task_id, ...)
    ↓
    1. coerce_tool_args()           — 类型强制转换
    2. 检查 _AGENT_LOOP_TOOLS      — 拦截 agent 级工具
    3. 检查 plugin hook (pre_tool_call) — 插件可阻止执行
    4. notify_other_tool_call()     — 重置读循环计数器
    ↓
registry.dispatch(name, args, task_id=..., user_task=...)
    ↓
    1. get_entry(name)              — 查找工具
    2. 异步桥接判断                  — is_async → _run_async()
    3. entry.handler(args, **kwargs) — 执行 handler
    4. 异常捕获 → JSON error         — 保证不抛异常
    ↓
结果字符串返回给 agent loop → 追加到 messages
```

---

## 7. 测试方法 (Testing Approach)

### 7.1 单元测试

```python
# tests/test_your_tool.py
import json
from tools.your_tool import your_tool, check_requirements

def test_your_tool_basic():
    result = your_tool(param="test")
    data = json.loads(result)
    assert data["success"] is True

def test_check_requirements_missing_key(monkeypatch):
    monkeypatch.delenv("YOUR_API_KEY", raising=False)
    assert check_requirements() is False

def test_check_requirements_with_key(monkeypatch):
    monkeypatch.setenv("YOUR_API_KEY", "test-key-123")
    assert check_requirements() is True
```

### 7.2 注册测试

```python
def test_tool_registered():
    from tools.registry import registry
    entry = registry.get_entry("your_tool")
    assert entry is not None
    assert entry.toolset == "your_toolset"
    assert entry.emoji == "⚡"

def test_schema_valid():
    from tools.registry import registry
    schema = registry.get_schema("your_tool")
    assert schema["name"] == "your_tool"
    assert "parameters" in schema
    assert "required" in schema["parameters"]
```

### 7.3 集成测试

```python
def test_dispatch():
    from tools.registry import registry
    result = registry.dispatch("your_tool", {"param": "test"})
    data = json.loads(result)
    assert "error" not in data
```

### 7.4 Toolset 测试

```python
def test_toolset_includes_tool():
    from toolsets import resolve_toolset
    tools = resolve_toolset("your_toolset")
    assert "your_tool" in tools
```

---

## 8. 常见陷阱与注意事项 (Common Pitfalls)

### ❌ 陷阱 1：handler 返回非 JSON 字符串

```python
# 错误 ❌
handler=lambda args, **kw: "Success!"

# 正确 ✅
handler=lambda args, **kw: json.dumps({"success": True, "message": "Success!"})
# 或使用 tool_result()
handler=lambda args, **kw: tool_result(success=True, message="Success!")
```

### ❌ 陷阱 2：registry.register() 放在函数内部

```python
# 错误 ❌ — AST 扫描不会检测到，工具不会被自动发现
def setup():
    registry.register(name="my_tool", ...)

# 正确 ✅ — 放在模块顶层
registry.register(name="my_tool", ...)
```

### ❌ 陷阱 3：循环导入

```python
# 错误 ❌ — 在文件顶部从 model_tools 导入
from model_tools import handle_function_call  # 循环导入！

# 正确 ✅ — 只从 tools.registry 导入
from tools.registry import registry, tool_error, tool_result
```

依赖链：`registry.py → tools/*.py → model_tools.py`。工具文件不能导入 `model_tools`。

### ❌ 陷阱 4：忘记在 toolsets.py 中注册

工具被 `registry.register()` 注册后**仅表示它存在于注册表中**。要让它对用户可用，必须在 `toolsets.py` 的某个 toolset 中列出它。否则 `get_tool_definitions()` 不会包含它。

### ❌ 陷阱 5：Schema name 与 register name 不一致

```python
# 危险 ⚠️
SCHEMA = {"name": "my_tool_v2", ...}
registry.register(name="my_tool", schema=SCHEMA, ...)
# → get_definitions() 会用 entry.name 覆盖 schema 中的 name
#   schema_with_name = {**entry.schema, "name": entry.name}
```

虽然 registry 会自动修正，但保持一致是最佳实践。

### ❌ 陷阱 6：忽略 task_id

如果工具涉及终端操作或浏览器会话，必须透传 `task_id` 以实现会话隔离（特别是子代理场景）。

### ❌ 陷阱 7：使用硬编码路径

```python
# 错误 ❌
state_dir = Path.home() / ".hermes" / "my_tool_data"

# 正确 ✅ — 支持多 profile
from hermes_cli.config import get_hermes_home
state_dir = get_hermes_home() / "my_tool_data"
```

### ❌ 陷阱 8：async handler 但没设 is_async=True

```python
# 错误 ❌ — handler 返回 coroutine 但 is_async=False
async def my_handler(args):
    result = await do_something()
    return json.dumps(result)

registry.register(
    handler=lambda args, **kw: my_handler(args),  # 返回 coroutine，不是 str！
    # is_async=False,  # 默认值！
)

# 正确 ✅
registry.register(
    handler=lambda args, **kw: my_handler(args),
    is_async=True,  # ← registry.dispatch 会自动通过 _run_async() 桥接
)
```

---

## 9. 端到端示例：添加一个天气查询工具 (End-to-End Example: Weather Tool)

### 文件 1：`tools/weather_tool.py`

```python
#!/usr/bin/env python3
"""Weather Tool — Query current weather conditions."""

import json
import os
import logging
import httpx

logger = logging.getLogger(__name__)

def check_weather_requirements() -> bool:
    return bool(os.getenv("WEATHER_API_KEY"))

def weather_tool(city: str) -> str:
    api_key = os.getenv("WEATHER_API_KEY")
    if not api_key:
        return tool_error("WEATHER_API_KEY not configured")
    
    try:
        resp = httpx.get(
            f"https://api.weatherapi.com/v1/current.json",
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

### 文件 2：`toolsets.py` 修改

```python
# 方式 A：加入核心工具列表（所有平台可用）
_HERMES_CORE_TOOLS = [
    # ... existing tools ...
    "weather",     # ← 新增
]

# 方式 B：作为独立 toolset（可选启用）
TOOLSETS = {
    # ... existing toolsets ...
    "weather": {
        "description": "Weather information tool",
        "tools": ["weather"],
        "includes": []
    },
}
```

### 验证

```bash
# 1. 检查注册成功
python -c "from tools.weather_tool import *; from tools.registry import registry; print(registry.get_entry('weather'))"

# 2. 检查 toolset 解析
python -c "from toolsets import resolve_toolset; print(resolve_toolset('weather'))"

# 3. 运行测试
pytest tests/test_weather_tool.py -v
```

---

## 10. 高级话题 (Advanced Topics)

### 10.1 MCP 动态工具注册

MCP 工具通过 `tools/mcp_tool.py` 动态注册，不参与 AST 静态扫描（`mcp_tool.py` 在排除列表中）。MCP 工具的 toolset 名格式为 `mcp-{server_name}`，并注册别名：

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

### 10.2 插件工具注册

插件通过 `hermes_cli/plugins.py` 的 `discover_plugins()` 加载，在 `model_tools.py` 的 discover 阶段执行。插件注册的工具集名称不在 `TOOLSETS` 字典中，但会被 `_get_plugin_toolset_names()` 识别。

### 10.3 结果截断

工具可通过 `max_result_size_chars` 控制结果大小。未设置时使用全局默认值（来自 `tools/budget_config.py` 的 `DEFAULT_RESULT_SIZE_CHARS`）。

### 10.4 线程安全

`ToolRegistry` 使用 `threading.RLock` 保护所有写操作，读操作通过 `_snapshot_state()` 返回稳定快照。这在 MCP 动态刷新和并行工具执行时尤为重要。

---

## 11. 关键源码定位 (Key Source Locations)

| 概念 | 文件 | 行号 / 位置 |
|------|------|-------------|
| ToolEntry 定义 | `tools/registry.py` | class ToolEntry, L76-98 |
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
| _run_async() 桥接 | `model_tools.py` | L81-125 |

---

## 12. 一句话总结 (One-Line Summary)

> 在 Hermes Agent 中添加新工具 = **一个文件**定义 schema + handler + check_fn 并调用 `registry.register()`，**另一个文件** (`toolsets.py`) 中将工具名加入对应的工具集列表。自动发现机制通过 AST 扫描完成其余一切。

---

*研究笔记完成于 2026-04-15。基于 hermes-source/ 当前代码库。*
