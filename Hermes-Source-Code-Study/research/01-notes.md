# Ch01 项目全景 — Research Notes

> Assignee: s-claude-01
> Task: T01-R
> 源码基线: commit `a4e1842f` (2026-04-15)
> 核心文件: `pyproject.toml`, `hermes_constants.py`, `AGENTS.md`, `CONTRIBUTING.md`

---

## 1. 项目概述

Hermes Agent 是 Nous Research 开发的**自我改进 AI 智能体**（self-improving AI agent），核心定位是"从经验中创建技能、在使用中改进技能、随处运行"。版本 v0.9.0，MIT 许可证。

**一句话总结：** 一个以 Python 编写、基于 OpenAI 兼容 API 的全栈 AI Agent 框架，支持 CLI 交互、多平台消息网关、工具系统、技能系统、记忆系统、子智能体委托等。

---

## 2. 技术栈选型分析

### 2.1 核心依赖 (`pyproject.toml` L12-37)

| 类别 | 依赖 | 版本约束 | 用途 |
|------|------|---------|------|
| **LLM SDK** | `openai>=2.21.0,<3` | 主接口 | 所有 LLM 调用走 OpenAI 兼容协议 |
| **LLM SDK** | `anthropic>=0.39.0,<1` | 原生适配 | Anthropic 模型的原生 API 适配（非 OpenAI 兼容模式） |
| **配置** | `python-dotenv>=1.2.1,<2` | 环境变量 | `.env` 文件加载 |
| **CLI 框架** | `fire>=0.7.1,<1` | 命令路由 | CLI 子命令路由（但实际 main.py 更多用 argparse 风格） |
| **HTTP** | `httpx[socks]>=0.28.1,<1` | 异步HTTP | 带 SOCKS 代理支持的 HTTP 客户端 |
| **TUI 渲染** | `rich>=14.3.3,<15` | 终端美化 | Banner、Panel、Markdown 渲染、Spinner |
| **TUI 输入** | `prompt_toolkit>=3.0.52,<4` | 交互输入 | 自动补全、多行编辑、键绑定 |
| **重试** | `tenacity>=9.1.4,<10` | 重试策略 | API 调用重试和回退 |
| **配置格式** | `pyyaml>=6.0.2,<7` | YAML解析 | `config.yaml` 读写 |
| **HTTP** | `requests>=2.33.0,<3` | 同步HTTP | 备用 HTTP 客户端（注释标注 CVE-2026-25645） |
| **模板** | `jinja2>=3.1.5,<4` | 模板引擎 | System prompt 模板渲染 |
| **数据校验** | `pydantic>=2.12.5,<3` | 数据模型 | 配置/数据校验 |
| **搜索** | `exa-py>=2.9.0,<3` | Web搜索 | Exa 搜索引擎 API |
| **爬取** | `firecrawl-py>=4.16.0,<5` | 网页提取 | Firecrawl 网页内容提取 |
| **搜索** | `parallel-web>=0.4.2,<1` | 并行搜索 | Parallel Web 搜索 |
| **生成** | `fal-client>=0.13.1,<1` | 图像生成 | Fal.ai 图像生成 |
| **TTS** | `edge-tts>=7.2.7,<8` | 语音合成 | Edge TTS（免费，无需 API key） |
| **认证** | `PyJWT[crypto]>=2.12.0,<3` | JWT | Skills Hub GitHub App JWT 认证 |

**设计选择要点：**
- 所有核心依赖都使用 **pinned ranges**（`>=x,<y`），注释说明是为了"限制供应链攻击面"（L14）
- 选用 OpenAI SDK 作为主接口，但同时保留 Anthropic 原生 SDK 适配——体现了"OpenAI 兼容优先，但不放弃原生优化"的策略
- `httpx[socks]` 支持 SOCKS 代理，说明考虑了中国等需要代理的使用场景
- Rich + prompt_toolkit 组合是 Python CLI TUI 的黄金搭配

### 2.2 可选依赖 (`pyproject.toml` L39-112)

可选依赖组织为 **17 个 extras**，体现了高度模块化设计：

| Extra | 用途 | 关键依赖 |
|-------|------|---------|
| `modal` | Modal 云计算后端 | `modal>=1.0.0` |
| `daytona` | Daytona 开发环境 | `daytona>=0.148.0` |
| `dev` | 开发和测试 | `pytest`, `debugpy`, `mcp` |
| `messaging` | 消息平台（Telegram/Discord/Slack） | `python-telegram-bot`, `discord.py`, `slack-bolt` |
| `cron` | 定时任务 | `croniter>=6.0.0` |
| `matrix` | Matrix 协议（仅 Linux） | `mautrix[encryption]` |
| `cli` | CLI 交互菜单 | `simple-term-menu` |
| `tts-premium` | 高级语音合成 | `elevenlabs>=1.0` |
| `voice` | 语音输入（STT） | `faster-whisper`, `sounddevice` |
| `pty` | 伪终端支持 | `ptyprocess`(Unix)/`pywinpty`(Win) |
| `honcho` | Honcho AI 记忆集成 | `honcho-ai>=2.0.1` |
| `mcp` | MCP 协议 | `mcp>=1.2.0` |
| `homeassistant` | 智能家居 | `aiohttp` |
| `mistral` | Mistral 原生 API | `mistralai>=2.3.0` |
| `termux` | Android/Termux 环境 | 组合多个核心 extras |
| `dingtalk`/`feishu` | 钉钉/飞书 | 各自 SDK |
| `rl` | 强化学习训练 | `atroposlib`, `tinker`, `wandb` |

**设计洞察：**
- `termux` extra 专门为 Android 优化，排除了依赖 C 扩展的 `voice` extra
- `matrix` 限定 `sys_platform == 'linux'`，因为 macOS 上 `python-olm` 存在编译问题
- `pty` 做了跨平台分支：Unix 用 `ptyprocess`，Windows 用 `pywinpty`
- `rl` extra 通过 git URL 引入 Atropos 和 Tinker，表明 RL 训练是实验性功能

### 2.3 入口点 (`pyproject.toml` L114-118)

```toml
[project.scripts]
hermes = "hermes_cli.main:main"          # 主 CLI 入口
hermes-agent = "run_agent:main"           # 直接启动 Agent（跳过 CLI TUI）
hermes-acp = "acp_adapter.entry:main"    # ACP 服务器（编辑器集成）
```

三个入口体现三种使用模式：交互式 CLI、编程式调用、编辑器集成。

---

## 3. 模块结构全景

### 3.1 顶层文件 (根目录 .py 文件)

| 文件 | 行数 | 职责 |
|------|------|------|
| `run_agent.py` | 11,024 | AIAgent 类 — 核心对话循环、tool dispatch、会话持久化 |
| `cli.py` | 10,039 | HermesCLI 类 — 交互式 TUI、Rich/prompt_toolkit 集成 |
| `hermes_state.py` | 1,238 | SessionDB — SQLite WAL + FTS5 全文搜索 |
| `toolsets.py` | 702 | 工具集定义和解析 |
| `model_tools.py` | 562 | 工具编排薄层（发现、分发、schema 管理） |
| `hermes_logging.py` | 390 | 结构化日志 |
| `hermes_constants.py` | 294 | 共享常量（零依赖，可从任何地方安全导入） |
| `batch_runner.py` | - | 并行批处理 |
| `trajectory_compressor.py` | - | 轨迹压缩 |
| `toolset_distributions.py` | - | 工具集分发配置 |
| `mcp_serve.py` | - | MCP 服务端 |

**代码量统计：** 276 个 Python 文件（不含 tests/venv/website），总计约 206,778 行。

### 3.2 包结构

```
hermes-agent/
├── agent/                # Agent 内部模块（prompt 构建、上下文压缩、模型元数据等）
├── tools/                # 工具实现（自注册模式）
│   └── environments/     # 终端执行后端（local/docker/ssh/modal/daytona/singularity）
├── hermes_cli/           # CLI 子命令和配置管理
├── gateway/              # 消息平台网关
│   └── platforms/        # 平台适配器（24 个文件）
├── acp_adapter/          # ACP 服务器（VS Code/Zed/JetBrains）
├── cron/                 # 定时调度器
├── plugins/              # 插件系统（如 memory providers）
├── skills/               # 捆绑技能（26 个类别目录）
├── optional-skills/      # 官方可选技能
├── environments/         # RL 训练环境（Atropos 集成）
├── tests/                # ~3000 个测试
├── scripts/              # 安装脚本
└── website/              # 文档站点
```

### 3.3 消息平台支持（`gateway/platforms/`）

覆盖 **17+ 个通信平台**：
Telegram, Discord, Slack, WhatsApp, Signal, BlueBubbles (iMessage), Home Assistant, Email (IMAP/SMTP), Mattermost, Matrix, DingTalk, Feishu/Lark, Weixin (微信), QQBot, WeCom (企业微信), WeCom Callback, SMS (Twilio), Webhook, API Server

---

## 4. 文件依赖链分析

### 4.1 核心依赖链（AGENTS.md L68-78）

```
tools/registry.py  (零依赖 — 被所有工具文件导入)
       ↑
tools/*.py  (每个文件在 import 时调用 registry.register())
       ↑
model_tools.py  (导入 tools/registry + 触发工具发现)
       ↑
run_agent.py, cli.py, batch_runner.py, environments/
```

**关键设计：自注册模式（Self-Registration Pattern）**
- `tools/registry.py` 作为中央注册表，无任何外部依赖
- 每个工具文件在模块级别调用 `registry.register()`
- `model_tools.py` 通过导入所有工具模块触发注册（而非扫描）
- `run_agent.py` 等上层模块通过 `model_tools.py` 获取工具定义

### 4.2 关键接口

**AIAgent 类** (`run_agent.py`) — 两个主要接口：
```python
class AIAgent:
    def chat(self, message: str) -> str:           # 简单接口
    def run_conversation(self, ...) -> dict:        # 完整接口
```

**核心循环**（同步设计）：
```python
while api_call_count < max_iterations and iteration_budget.remaining > 0:
    response = client.chat.completions.create(...)
    if response.tool_calls:
        for tool_call in response.tool_calls:
            result = handle_function_call(tool_call.name, tool_call.args, task_id)
            messages.append(tool_result_message(result))
    else:
        return response.content
```

---

## 5. hermes_constants.py 深度分析 (294行)

### 5.1 设计定位

文件头注释明确声明：**"Import-safe module with no dependencies — can be imported from anywhere without risk of circular imports."**

这是典型的 **常量/工具模块反循环依赖** 模式：将共享常量和工具函数放在一个零依赖模块中，其他任何模块都可安全导入。

### 5.2 核心函数

| 函数 | 行号 | 用途 |
|------|------|------|
| `get_hermes_home()` | L11-17 | 返回 Hermes Home 目录（`HERMES_HOME` env → `~/.hermes`） |
| `get_default_hermes_root()` | L20-56 | Profile 模式下获取根目录（处理 Docker/Profile 嵌套路径） |
| `get_optional_skills_dir()` | L59-70 | 可选技能目录（支持包管理器覆写） |
| `get_hermes_dir()` | L73-91 | **向后兼容的目录解析** — 新旧路径自动选择 |
| `display_hermes_home()` | L94-111 | 用户友好的路径显示（`~/` 缩写） |
| `get_subprocess_home()` | L114-137 | 子进程 HOME 隔离（Docker 持久化 + Profile 隔离） |
| `parse_reasoning_effort()` | L143-158 | 推理努力级别解析 |
| `is_termux()` | L161-168 | Termux (Android) 环境检测 |
| `is_wsl()` | L174-189 | WSL 环境检测（缓存结果） |
| `is_container()` | L195-220 | Docker/Podman 容器检测（多策略） |
| `get_config_path()` | L226-232 | `config.yaml` 路径 |
| `get_skills_dir()` | L235-237 | 技能目录路径 |
| `get_env_path()` | L241-243 | `.env` 文件路径 |
| `apply_ipv4_preference()` | L249-288 | **猴子补丁 socket.getaddrinfo** 强制 IPv4 |

### 5.3 关键设计模式

1. **Profile-safe 路径解析**：所有路径通过 `get_hermes_home()` 获取，绝不硬编码 `~/.hermes`。AGENTS.md 强调这是**最重要的规则**，曾因违反导致 5 个 bug（PR #3575）。

2. **向后兼容目录迁移** (`get_hermes_dir()` L73-91)：
   ```python
   def get_hermes_dir(new_subpath, old_name):
       old_path = home / old_name
       if old_path.exists(): return old_path    # 旧路径存在则继续使用
       return home / new_subpath                 # 否则用新路径
   ```

3. **环境检测缓存**：`is_wsl()` 和 `is_container()` 使用模块级全局变量缓存结果，避免重复文件系统访问。

4. **IPv4 偏好猴子补丁** (`apply_ipv4_preference()` L249-288)：在 IPv6 不可达的服务器上，Python 默认尝试 AAAA 记录会导致超时。通过替换 `socket.getaddrinfo` 强制 IPv4，同时保留纯 IPv6 主机的兼容性。带防重复补丁保护 (`_hermes_ipv4_patched`)。

### 5.4 常量定义

```python
VALID_REASONING_EFFORTS = ("minimal", "low", "medium", "high", "xhigh")
OPENROUTER_BASE_URL = "https://openrouter.ai/api/v1"
OPENROUTER_MODELS_URL = f"{OPENROUTER_BASE_URL}/models"
AI_GATEWAY_BASE_URL = "https://ai-gateway.vercel.sh/v1"
```

---

## 6. 工具集架构 (toolsets.py)

### 6.1 _HERMES_CORE_TOOLS（共享工具列表）

`_HERMES_CORE_TOOLS` (L31-63) 定义了 CLI 和所有消息平台共享的核心工具列表（35 个工具），分为 10 个类别：
- Web: `web_search`, `web_extract`
- Terminal: `terminal`, `process`
- File: `read_file`, `write_file`, `patch`, `search_files`
- Vision: `vision_analyze`, `image_generate`
- Skills: `skills_list`, `skill_view`, `skill_manage`
- Browser: 10 个浏览器自动化工具
- TTS: `text_to_speech`
- Planning: `todo`, `memory`
- Session: `session_search`
- Other: `clarify`, `execute_code`, `delegate_task`, `cronjob`, `send_message`, Home Assistant 工具

### 6.2 TOOLSETS 字典

定义了 **30+ 个工具集**（L68-397），分为：
- **基础工具集**：`web`, `search`, `vision`, `terminal`, `file`, `skills`, `browser` 等
- **场景工具集**：`debugging`（组合 terminal + web + file）、`safe`（无终端访问）
- **平台工具集**：`hermes-cli`, `hermes-telegram`, `hermes-discord` 等 17 个平台全部使用 `_HERMES_CORE_TOOLS`
- **聚合工具集**：`hermes-gateway` 通过 `includes` 组合所有消息平台工具集

### 6.3 工具集解析（组合模式）

`resolve_toolset()` (L447-497) 实现了递归解析，支持：
- 工具集组合（`includes` 字段引用其他工具集）
- 循环检测（`visited` 集合）
- 菱形依赖去重
- 特殊别名 `all` / `*` 解析所有工具

---

## 7. 关键设计模式汇总

### 7.1 自注册工具模式（Self-Registering Tools）

每个工具文件在模块级别调用 `registry.register()`，不需要任何中央注册文件。CONTRIBUTING.md 提供了标准模板：

```python
from tools.registry import registry

def my_tool(param1, param2, **kwargs):
    ...

registry.register(
    name="my_tool",
    toolset="my_toolset",
    schema=MY_TOOL_SCHEMA,
    handler=lambda args, **kw: my_tool(**args, **kw),
    check_fn=_check_requirements,
)
```

### 7.2 Provider 抽象

- 所有 LLM 调用走 OpenAI 兼容 API
- Provider 解析在 init 时完成（Nous Portal OAuth / OpenRouter API key / 自定义 endpoint）
- Anthropic 模型有原生适配器，但仍可走 OpenAI 兼容层

### 7.3 会话持久化

- SQLite WAL 模式 + FTS5 全文搜索
- JSON 日志保存到 `~/.hermes/sessions/`
- 临时注入（system prompt、prefill）不持久化

### 7.4 Prompt 缓存保护

AGENTS.md 特别强调：**禁止在对话中途改变上下文、工具集、记忆或系统提示词**，因为这会打破 prompt 缓存，导致成本剧增。唯一允许的上下文修改是自动压缩。

### 7.5 安全纵深

六层安全防护（CONTRIBUTING.md L560-570）：
1. Sudo 密码注入使用 `shlex.quote()` 防 shell 注入
2. 危险命令检测 + 用户审批流程
3. Cron prompt injection 扫描
4. 写入拒绝列表（受保护路径，解析符号链接）
5. 技能安全卫士（Hub 安装的技能安全扫描）
6. 代码执行沙箱（剥离 API key 环境变量）
7. Docker 容器加固（全部 capability drop、禁止提权）

### 7.6 跨平台兼容

CONTRIBUTING.md 列出关键规则：
- `termios`/`fcntl` 是 Unix-only，必须 catch `ImportError` 和 `NotImplementedError`
- 文件编码处理（Windows cp1252）
- 进程管理的平台差异（`os.setsid()` 等）
- 使用 `pathlib.Path` 代替字符串拼接

---

## 8. 用户配置体系

| 路径 | 用途 |
|------|------|
| `~/.hermes/config.yaml` | 设置（模型、终端、工具集、压缩等） |
| `~/.hermes/.env` | API key 和密钥 |
| `~/.hermes/auth.json` | OAuth 凭据 (Nous Portal) |
| `~/.hermes/skills/` | 所有活跃技能 |
| `~/.hermes/memories/` | 持久记忆 (MEMORY.md, USER.md) |
| `~/.hermes/state.db` | SQLite 会话数据库 |
| `~/.hermes/sessions/` | JSON 会话日志 |
| `~/.hermes/cron/` | 定时任务数据 |
| `~/.hermes/skins/` | 自定义皮肤 (YAML) |

支持 **Profile 模式**：每个 profile 有独立的 `HERMES_HOME`，实现完全隔离。

---

## 9. Skill vs Tool 哲学

CONTRIBUTING.md 明确了一个重要的设计决策：**大部分功能应该是 Skill 而非 Tool**。

- **Skill**：指令 + shell 命令 + 现有工具的组合，无需自定义 Python 集成
- **Tool**：需要端到端的 API 集成、自定义处理逻辑、二进制数据处理

Skill 系统三层：
1. **Bundled skills** (`skills/`)：捆绑发布，广泛有用
2. **Optional skills** (`optional-skills/`)：官方但不默认激活
3. **Skills Hub**：社区贡献，通过 `hermes skills install` 安装

---

## 10. 最小运行验证

### 10.1 入口验证

```bash
# 验证三个入口点都注册成功
python -c "from hermes_cli.main import main; print('CLI entry OK')"
python -c "from run_agent import main; print('Agent entry OK')"
python -c "from acp_adapter.entry import main; print('ACP entry OK')"
```

### 10.2 常量模块零依赖验证

```bash
python -c "
from hermes_constants import get_hermes_home, is_termux, is_wsl, is_container, VALID_REASONING_EFFORTS
print(f'HERMES_HOME: {get_hermes_home()}')
print(f'is_termux: {is_termux()}')
print(f'is_wsl: {is_wsl()}')
print(f'is_container: {is_container()}')
print(f'reasoning efforts: {VALID_REASONING_EFFORTS}')
"
```

### 10.3 工具集解析验证

```bash
python -c "
from toolsets import resolve_toolset, get_toolset_names
cli_tools = resolve_toolset('hermes-cli')
print(f'CLI toolset: {len(cli_tools)} tools')
print(f'Tools: {cli_tools[:5]}...')
all_toolsets = get_toolset_names()
print(f'Total toolsets: {len(all_toolsets)}')
"
```

---

## 11. 与 Claude Code 的对比

Hermes Agent 与 Claude Code (Anthropic 官方 CLI Agent) 有显著的架构差异：

| 维度 | Hermes Agent | Claude Code |
|------|-------------|-------------|
| 语言 | Python | TypeScript |
| LLM 接口 | OpenAI 兼容（多模型） | Anthropic 原生 |
| 对话循环 | 同步 | 异步 |
| 工具注册 | 自注册模式（registry.register()） | 静态定义 |
| 平台支持 | 17+ 消息平台 + CLI | CLI only |
| 技能系统 | 有（SKILL.md + Hub） | 无 |
| 记忆系统 | MEMORY.md + 多 provider | 无 |
| 配置系统 | YAML + .env + Profile | 配置文件 |
| 代码量 | ~207K行 Python | ~50K行 TypeScript |

---

## 12. 待深入研究的方向

- [ ] `run_agent.py` 的完整 Agent 循环细节（T02-R/T05-R）
- [ ] `agent/prompt_builder.py` 的 System Prompt 构建（T06-R）
- [ ] `hermes_state.py` 的 SQLite 持久化设计（T04-R）
- [ ] `tools/registry.py` 的注册表内部实现（T11-R）
- [ ] `agent/context_compressor.py` 的上下文压缩策略（T07-R）
- [ ] `gateway/run.py` 的消息网关架构（T21-R）
