---
Task: T20-R
Status: DRAFT_DONE
Date: 2026-04-15
Chapter: 20
Title: CLI 交互界面 (CLI Interactive Interface)
Keywords: Rich + prompt_toolkit TUI、slash 命令注册表、KawaiiSpinner、皮肤引擎
Files:
  - cli.py (10039 lines, 447KB) — HermesCLI class, main interactive loop
  - hermes_cli/commands.py (1217 lines) — Slash command registry (CommandDef)
  - agent/display.py (1037 lines) — KawaiiSpinner, tool preview formatting
  - hermes_cli/skin_engine.py (816 lines) — Skin/theme engine
  - hermes_cli/banner.py (535 lines) — CLI banner, welcome screen
---

# Ch20 CLI 交互界面 — 研究笔记

## 1. 总体架构概览 (Architecture Overview)

Hermes CLI 是一个基于 **prompt_toolkit** 构建的全功能 TUI (Terminal User Interface)，使用 **Rich** 进行 Panel / Table / Markdown 等高级渲染输出。其核心设计模式：

```
┌────────────────────────────────────────────────────┐
│  HermesCLI (cli.py L1577)                          │
│  ├─ prompt_toolkit Application (TUI 框架层)         │
│  │   ├─ HSplit Layout (窗口布局)                    │
│  │   ├─ KeyBindings (按键绑定 Enter/Ctrl-C/Up/Down)│
│  │   └─ patch_stdout (输出重定向)                   │
│  ├─ Rich Console (渲染层)                           │
│  │   ├─ ChatConsole 适配器                          │
│  │   └─ Panel / Table / Markdown                   │
│  ├─ Skin Engine (皮肤引擎)                          │
│  │   ├─ SkinConfig dataclass                       │
│  │   └─ YAML 自定义皮肤                             │
│  ├─ Command Registry (命令注册表)                    │
│  │   ├─ CommandDef dataclass                       │
│  │   └─ SlashCommandCompleter                      │
│  └─ KawaiiSpinner (动画指示器)                      │
└────────────────────────────────────────────────────┘
```

**关键设计决策**：
- prompt_toolkit 管理输入、布局、按键；Rich 管理输出渲染
- `ChatConsole` 桥接 Rich → prompt_toolkit：Rich 渲染到 StringIO → `_cprint()` → prompt_toolkit ANSI renderer
- 皮肤引擎 (Skin Engine) 是一个数据驱动的主题系统，支持 YAML 自定义

## 2. prompt_toolkit 集成 (prompt_toolkit Integration)

### 2.1 导入层 (Import Layer)

`cli.py` 大量使用 prompt_toolkit，导入清单 (L39-53)：

```python
from prompt_toolkit.history import FileHistory
from prompt_toolkit.styles import Style as PTStyle
from prompt_toolkit.patch_stdout import patch_stdout
from prompt_toolkit.application import Application
from prompt_toolkit.layout import Layout, HSplit, Window, FormattedTextControl, ConditionalContainer
from prompt_toolkit.layout.processors import Processor, Transformation, PasswordProcessor, ConditionalProcessor
from prompt_toolkit.filters import Condition
from prompt_toolkit.layout.dimension import Dimension
from prompt_toolkit.layout.menus import CompletionsMenu
from prompt_toolkit.widgets import TextArea
from prompt_toolkit.key_binding import KeyBindings
from prompt_toolkit import print_formatted_text as _pt_print
from prompt_toolkit.formatted_text import ANSI as _PT_ANSI
from prompt_toolkit.cursor_shapes import CursorShape  # optional
```

### 2.2 TUI 布局 (Layout Architecture)

`_build_tui_layout_children()` (L8140-8183) 定义了 TUI 的 HSplit 布局顺序：

```
Window(height=0)         ← 占位/锚定
sudo_widget              ← sudo 密码输入
secret_widget            ← 密钥捕获
approval_widget          ← 危险命令审批
clarify_widget           ← 交互式澄清问答
model_picker_widget      ← /model 选择器
spinner_widget           ← TUI 内嵌 spinner 状态
spacer                   ← 弹性间距
*extra_widgets           ← 插件扩展点 (_get_extra_tui_widgets)
status_bar               ← 状态栏 (token/model/context)
input_rule_top           ← 输入区上分隔线
image_bar                ← 已附加图片指示
input_area               ← TextArea 用户输入
input_rule_bot           ← 输入区下分隔线
voice_status_bar         ← 语音状态
completions_menu         ← Tab 补全菜单
```

**设计模式**：使用 `ConditionalContainer` + `Condition` 过滤器实现条件可见性。每个 modal widget (sudo/approval/clarify) 只在对应 `_xxx_state` 非 None 时显示。

### 2.3 按键绑定 (Key Binding Architecture)

`run()` 方法 (L8185+) 中创建 `KeyBindings` (L8316)，核心 Enter 处理逻辑 (L8318-8384)：

```
Enter 按下 →
  ├─ _sudo_state?      → 提交密码到 response_queue
  ├─ _secret_state?    → 提交密钥
  ├─ _approval_state?  → 确认审批选择
  ├─ _model_picker?    → 确认模型选择
  ├─ _clarify_freetext? → 提交自由文本回答
  ├─ _clarify_state?   → 确认选项 / 切换到 freetext
  ├─ _agent_running?   → 放入 _interrupt_queue (中断消息)
  └─ else              → 放入 _pending_input (常规输入)
```

**注意**：以 `/` 开头的命令始终放入 `_pending_input`，确保它们作为命令处理而非中断文本。

### 2.4 双队列架构 (Dual Queue Architecture)

```python
self._pending_input = queue.Queue()    # 常规输入 (命令 + 新查询)
self._interrupt_queue = queue.Queue()  # agent 运行期间的中断消息
```

- `busy_input_mode == "interrupt"` (默认)：用户输入发送到 `_interrupt_queue`，agent 完成后重新排队
- `busy_input_mode == "queue"`：直接路由到 `_pending_input`

### 2.5 ChatConsole 桥接器 (ChatConsole Bridge)

`ChatConsole` (L1332-1372) 解决 Rich 与 prompt_toolkit `patch_stdout` 的兼容问题：

```python
class ChatConsole:
    """Rich Console adapter for prompt_toolkit's patch_stdout context."""
    def __init__(self):
        self._buffer = StringIO()
        self._inner = Console(
            file=self._buffer,
            force_terminal=True,
            color_system="truecolor",
            highlight=False,
        )

    def print(self, *args, **kwargs):
        self._buffer.seek(0)
        self._buffer.truncate()
        self._inner.width = shutil.get_terminal_size().columns
        self._inner.print(*args, **kwargs)
        output = self._buffer.getvalue()
        for line in output.rstrip("\n").split("\n"):
            _cprint(line)  # → prompt_toolkit 的 _pt_print(_PT_ANSI(text))
```

**问题背景**：`patch_stdout` 将 sys.stdout 替换为 `StdoutProxy`，会破坏 Rich 的直接输出。`ChatConsole` 先让 Rich 渲染到 StringIO，再通过 prompt_toolkit 的 `print_formatted_text(ANSI(...))` 管道输出，保持颜色正确。

## 3. Slash 命令注册表 (Slash Command Registry)

### 3.1 CommandDef 数据类 (CommandDef Dataclass)

```python
@dataclass(frozen=True)
class CommandDef:
    name: str                          # 规范名称: "background"
    description: str                   # 人类可读描述
    category: str                      # "Session", "Configuration" 等
    aliases: tuple[str, ...] = ()      # 别名: ("bg",)
    args_hint: str = ""                # 参数占位符: "<prompt>"
    subcommands: tuple[str, ...] = ()  # Tab 可补全子命令
    cli_only: bool = False             # 仅 CLI 可用
    gateway_only: bool = False         # 仅 Gateway 可用
    gateway_config_gate: str | None = None  # 配置门控路径
```

**frozen=True** — 不可变，线程安全。单一事实来源 (Single Source of Truth) 模式。

### 3.2 COMMAND_REGISTRY 命令清单

`COMMAND_REGISTRY` (L59-168) 包含全部命令定义，按类别分组：

| 类别 (Category) | 命令 (Commands) |
|---|---|
| **Session** | new/reset, clear, history, save, retry, undo, title, branch/fork, compress, rollback, snapshot/snap, stop, approve, deny, background/bg, btw, queue/q, status, resume |
| **Configuration** | config, model, provider, personality, statusbar/sb, verbose, yolo, reasoning, fast, skin, voice |
| **Tools & Skills** | tools, toolsets, skills, cron, reload, reload-mcp, browser, plugins |
| **Info** | commands, help, usage, insights, platforms/gateway, paste, image, update, debug, profile |
| **Exit** | quit/exit/q |

### 3.3 派生查找表 (Derived Lookups)

从 `COMMAND_REGISTRY` 自动派生：

```python
_COMMAND_LOOKUP: dict[str, CommandDef]     # 名称+别名 → CommandDef
COMMANDS: dict[str, str]                    # "/cmd" → 描述 (向后兼容)
COMMANDS_BY_CATEGORY: dict[str, dict]       # 分类 → {"/cmd": 描述}
SUBCOMMANDS: dict[str, list[str]]           # "/cmd" → ["sub1", "sub2"]
GATEWAY_KNOWN_COMMANDS: frozenset[str]      # Gateway 识别的命令集
```

**别名解析**：`resolve_command("bg")` → 返回 `CommandDef(name="background", ...)`

### 3.4 子命令提取

两种途径：
1. 显式 `subcommands` 字段：`("search", "browse", "inspect", "install")`
2. 自动从 `args_hint` 中提取管道分隔模式：`"[on|off|tts|status]"` → `["on", "off", "tts", "status"]`

### 3.5 SlashCommandCompleter

`SlashCommandCompleter` (L~1001+) 继承 `prompt_toolkit.completion.Completer`：

```python
class SlashCommandCompleter(Completer):
    def get_completions(self, document, complete_event):
        # 1. 补全命令名 (以 / 开头)
        # 2. 补全子命令 (/reasoning → none, minimal, low...)
        # 3. 模糊文件路径补全 (非命令位置)
```

- **命令补全**：匹配 `COMMAND_REGISTRY` 中的命令名
- **子命令补全**：检查 `SUBCOMMANDS` 字典
- **文件路径补全**：使用模糊评分算法 (`_fuzzy_file_score`)，支持路径前缀匹配

### 3.6 SlashCommandAutoSuggest

`SlashCommandAutoSuggest` (L~1100+) 继承 `AutoSuggest`：

```python
class SlashCommandAutoSuggest(AutoSuggest):
    def get_suggestion(self, buffer, document):
        # 从命令历史中提供 ghost-text 自动建议
```

### 3.7 命令分发 (Command Dispatch)

`process_command()` (L5262+) 的分发流程：

```python
def process_command(self, command: str) -> bool:
    # 1. 小写化命令进行匹配
    cmd_lower = command.lower().strip()
    # 2. 通过 resolve_command 解析别名 → 规范名
    _cmd_def = resolve_command(_base_word)
    canonical = _cmd_def.name if _cmd_def else _base_word
    # 3. if-elif 链分发到具体处理函数
    if canonical in ("quit", "exit", "q"):
        return False
    elif canonical == "help":
        self.show_help()
    elif canonical == "clear":
        self.new_session(silent=True)
        # 使用 prompt_toolkit 的 output 清屏
        if self._app:
            out = self._app.output
            out.erase_screen()
            out.cursor_goto(0, 0)
    ...
```

**返回值**：`True` 继续循环，`False` 退出。

## 4. KawaiiSpinner 动画指示器

### 4.1 类定义 (Class Definition)

`KawaiiSpinner` (display.py L571-756) 是一个多线程动画指示器：

```python
class KawaiiSpinner:
    """Animated spinner with kawaii faces for CLI feedback during tool execution."""

    SPINNERS = {
        'dots':    ['⠋', '⠙', '⠹', '⠸', '⠼', '⠴', '⠦', '⠧', '⠇', '⠏'],
        'bounce':  ['⠁', '⠂', '⠄', '⡀', '⢀', '⠠', '⠐', '⠈'],
        'grow':    ['▁', '▂', '▃', '▄', '▅', '▆', '▇', '█', '▇', ...],
        'arrows':  ['←', '↖', '↑', '↗', '→', '↘', '↓', '↙'],
        'star':    ['✶', '✷', '✸', '✹', '✺', '✹', '✸', '✷'],
        'moon':    ['🌑', '🌒', '🌓', '🌔', '🌕', '🌖', '🌗', '🌘'],
        'pulse':   ['◜', '◠', '◝', '◞', '◡', '◟'],
        'brain':   ['🧠', '💭', '💡', '✨', '💫', '🌟', '💡', '💭'],
        'sparkle': ['⁺', '˚', '*', '✧', '✦', '✧', '*', '˚'],
    }
```

### 4.2 Kawaii 表情库

```python
KAWAII_WAITING = [
    "(｡◕‿◕｡)", "(◕‿◕✿)", "٩(◕‿◕｡)۶", "(✿◠‿◠)", "( ˘▽˘)っ",
    "♪(´ε` )", "(◕ᴗ◕✿)", "ヾ(＾∇＾)", "(≧◡≦)", "(★ω★)",
]
KAWAII_THINKING = [
    "(｡•́︿•̀｡)", "(◔_◔)", "(¬‿¬)", "( •_•)>⌐■-■", "(⌐■_■)",
    "(´･_･`)", "◉_◉", "(°ロ°)", "( ˘⌣˘)♡", "ヽ(>∀<☆)☆",
    "٩(๑❛ᴗ❛๑)۶", "(⊙_⊙)", "(¬_¬)", "( ͡° ͜ʖ ͡°)", "ಠ_ಠ",
]
THINKING_VERBS = [
    "pondering", "contemplating", "musing", "cogitating", "ruminating",
    "deliberating", "mulling", "reflecting", "processing", "reasoning",
    "analyzing", "computing", "synthesizing", "formulating", "brainstorming",
]
```

### 4.3 动画引擎 (_animate)

```python
def _animate(self):
    # 非 TTY (Docker/pipe) → 静态输出，无动画
    if not self._is_tty:
        self._write(f"  [tool] {self.message}", flush=True)
        while self.running: time.sleep(0.5)
        return

    # StdoutProxy (prompt_toolkit patch_stdout) → 跳过 \r 动画
    # CLI 通过 TUI widget (_spinner_text) 显示 spinner
    if self._is_patch_stdout_proxy():
        while self.running: time.sleep(0.1)
        return

    # 正常 TTY → \r 覆写动画
    skin = _get_skin()
    wings = skin.get_spinner_wings() if skin else []
    while self.running:
        frame = self.spinner_frames[self.frame_idx % len(self.spinner_frames)]
        elapsed = time.time() - self.start_time
        if wings:
            left, right = wings[self.frame_idx % len(wings)]
            line = f"  {left} {frame} {self.message} {right} ({elapsed:.1f}s)"
        else:
            line = f"  {frame} {self.message} ({elapsed:.1f}s)"
        self._write(f"\r{line}{' ' * pad}", end='', flush=True)
        self.frame_idx += 1
        time.sleep(0.12)  # ~8 FPS
```

**三种输出模式**：
1. **非 TTY** (Docker/pipe/systemd)：单行静态输出，避免日志膨胀 (log bloat)
2. **StdoutProxy** (TUI 内)：静默循环，CLI 通过专用 TUI widget 渲染状态
3. **原生 TTY**：`\r` 覆写动画，支持皮肤翅膀 (wings) 装饰

### 4.4 上下文管理器协议

```python
def __enter__(self):
    self.start()
    return self

def __exit__(self, exc_type, exc_val, exc_tb):
    self.stop()
    return False
```

使用方式：`with KawaiiSpinner("Thinking...") as s: ...`

### 4.5 线程安全设计

- `_write()` 捕获创建时的 `sys.stdout`，免受后续 `redirect_stdout(devnull)` 影响
- `print_above()` 可在动画运行时安全打印一行文本（清除 spinner 行、打印、下一 tick 重绘）
- `stop()` 使用空格覆盖而非 `\033[K`，避免在 patch_stdout 环境下的 ANSI 乱码

## 5. 皮肤引擎 (Skin Engine)

### 5.1 SkinConfig 数据结构

```python
@dataclass
class SkinConfig:
    name: str
    description: str = ""
    colors: Dict[str, str] = field(default_factory=dict)
    spinner: Dict[str, Any] = field(default_factory=dict)
    branding: Dict[str, str] = field(default_factory=dict)
    tool_prefix: str = "┊"
    tool_emojis: Dict[str, str] = field(default_factory=dict)
    banner_logo: str = ""      # Rich markup ASCII art
    banner_hero: str = ""      # Rich markup hero art (caduceus 替代)
```

**color 键集合** (18+个)：

| 键 (Key) | 用途 (Usage) |
|---|---|
| `banner_border` | 面板边框颜色 |
| `banner_title` | 面板标题颜色 |
| `banner_accent` | 区段标题 (Available Tools 等) |
| `banner_dim` | 暗淡文本 (分隔线/标签) |
| `banner_text` | 正文 (工具名/技能名) |
| `ui_accent` | 通用 UI 强调色 |
| `ui_ok` / `ui_error` / `ui_warn` | 成功/错误/警告指示 |
| `prompt` | 提示符颜色 |
| `input_rule` | 输入区水平分隔线 |
| `response_border` | 响应框边框 (ANSI) |
| `status_bar_bg` | 状态栏背景 |
| `completion_menu_bg` | 补全菜单背景 |

### 5.2 内置皮肤 (Built-in Skins)

`_BUILTIN_SKINS` 字典定义了 6+ 套内置皮肤：

| 皮肤 (Skin) | 描述 (Description) | 风格 |
|---|---|---|
| `default` | Classic Hermes — gold and kawaii | 金色/可爱 |
| `ares` | War-god theme — crimson and bronze | 红铜战神 |
| `mono` | Clean grayscale monochrome | 灰度极简 |
| `slate` | Cool blue developer-focused | 冷蓝开发者 |
| `daylight` | Light background with dark text | 浅色主题 |
| `warm-lightmode` | Warm brown/gold for light terminals | 暖色浅色 |
| `boulder` | Granite grey monumental theme | 花岗岩灰 |
| `charizard` | Volcanic — burnt orange and ember | 火焰龙主题 |

每套皮肤可自定义：
- **colors** — 颜色映射
- **spinner** — 自定义 waiting_faces / thinking_faces / thinking_verbs / wings
- **branding** — agent_name / welcome / goodbye / response_label / prompt_symbol / help_header
- **banner_logo / banner_hero** — 自定义 ASCII art
- **tool_prefix** — 工具输出行前缀字符

### 5.3 YAML 自定义皮肤

用户可在 `~/.hermes/skins/<name>.yaml` 放置 YAML 文件：

```yaml
name: mytheme
description: My custom theme
colors:
  banner_border: "#CD7F32"
  banner_title: "#FFD700"
  # ... 缺失的键继承自 default
spinner:
  waiting_faces: ["(⚔)", "(⛨)"]
  wings: [["⟪⚔", "⚔⟫"]]
branding:
  agent_name: "My Agent"
  prompt_symbol: "⚔ ❯ "
```

**加载优先级**：用户皮肤 (`~/.hermes/skins/`) > 内置皮肤 > default 回退

### 5.4 皮肤加载流程

```python
def load_skin(name: str) -> SkinConfig:
    # 1. 检查 ~/.hermes/skins/<name>.yaml
    # 2. 检查 _BUILTIN_SKINS[name]
    # 3. 回退到 _BUILTIN_SKINS["default"]
    → _build_skin_config(data)  # 合并 default 基础 + 皮肤覆盖

def init_skin_from_config(config: dict):
    """CLI 启动时从配置初始化皮肤。"""
    skin_name = config.get("display", {}).get("skin", "default")
    set_active_skin(skin_name)
```

**缓存策略**：`_active_skin` 全局缓存，首次调用 `get_active_skin()` 时加载。`/skin <name>` 命令通过 `set_active_skin()` 立即切换。

### 5.5 prompt_toolkit 样式集成

`get_prompt_toolkit_style_overrides()` (L751-816) 将皮肤颜色转换为 prompt_toolkit 的样式字典：

```python
def get_prompt_toolkit_style_overrides() -> Dict[str, str]:
    return {
        "input-area": prompt_color,
        "placeholder": f"{dim} italic",
        "status-bar": f"bg:{status_bg} {text}",
        "status-bar-strong": f"bg:{status_bg} {title} bold",
        "completion-menu": f"bg:{menu_bg} {text}",
        "clarify-border": input_rule,
        "sudo-prompt": f"{error} bold",
        "approval-border": input_rule,
        "voice-status": f"bg:{voice_bg} {label}",
        # 共 30+ 样式键
    }
```

这实现了 `/skin` 切换时 TUI 即时刷新（不需重建 Application）。

## 6. 欢迎横幅系统 (Banner System)

### 6.1 banner.py 结构

`hermes_cli/banner.py` (535 行) 包含：

- **ASCII Art 常量**：`HERMES_AGENT_LOGO` (大号文字)、`HERMES_CADUCEUS` (医神杖图案)
- **`build_welcome_banner()`** (L315-535)：完整横幅构建函数
- **`_build_compact_banner()`** (cli.py L1401)：终端窄于 80 列时的紧凑横幅
- **`get_available_skills()`**：扫描已安装技能
- **`cprint()`**：ANSI 文本通过 prompt_toolkit 渲染

### 6.2 横幅布局

`build_welcome_banner()` 使用 Rich `Table.grid()` 双列布局：

```
╭──────────────────────────────────────────────────╮
│  ██╗  ██╗███████╗██████╗...  (ASCII Art Logo)    │
│                                                  │
│  [Caduceus Art]    │  Available Tools            │
│  model · context   │    core: terminal, read...  │
│  /path/to/cwd      │    browser: navigate...     │
│  Session: abc123   │  MCP Servers                │
│                    │    server_name (stdio)       │
│                    │  Available Skills            │
│                    │    category: skill1, skill2  │
│                    │  45 tools · 12 skills · /help│
╰──────────────────────────────────────────────────╯
```

- 左列：Caduceus / 自定义 hero art + 模型/路径/Session 信息
- 右列：工具列表 (分 toolset) + MCP 服务器 + 技能 + 更新提示
- 外框使用 Rich `Panel`，标题/边框颜色从皮肤引擎获取

### 6.3 皮肤自定义横幅

皮肤可替换：
- `banner_logo` — 替换 `HERMES_AGENT_LOGO` (ASCII 大字)
- `banner_hero` — 替换 `HERMES_CADUCEUS` (Braille art 图案)

例如 charizard 皮肤有自定义火焰龙图案和 "CHARIZARD AGENT" 文字。

## 7. 响应渲染 (Response Rendering)

### 7.1 三个渲染点

响应在 cli.py 中有三个渲染入口：

1. **流式响应框打开** (L2530-2609)：`_open_stream_box()` — 实时 token 流输出
2. **非流式渲染** (L5752 附近)：完整响应一次性渲染
3. **chat() 最终渲染** (L7854-7887)：统一的 Rich Panel 输出

### 7.2 响应框结构

```python
# 从皮肤获取标签和颜色
from hermes_cli.skin_engine import get_active_skin
_skin = get_active_skin()
label = _skin.get_branding("response_label", "⚕ Hermes")
_resp_color = _skin.get_color("response_border", "#CD7F32")
_resp_text = _skin.get_color("banner_text", "#FFF8DC")

# Rich Panel 渲染
_chat_console.print(Panel(
    _rich_text_from_ansi(response),
    title=f"[{_resp_color} bold]{label}[/]",
    title_align="left",
    border_style=_resp_color,
    style=_resp_text,
    box=rich_box.HORIZONTALS,
    padding=(1, 4),
))
```

### 7.3 流式 vs 非流式路径

```
流式路径 (streaming=True):
  _open_stream_box() → 逐 token 写入 → _flush_stream() 关闭框
  
非流式路径 (streaming=False):
  agent.chat() 返回完整文本 → Rich Panel 一次性渲染

TTS 流式路径:
  逐句写入 + 逐句合成语音 → 手动 ╰───╯ 关闭框
```

## 8. 工具输出格式化 (Tool Output Formatting)

### 8.1 get_cute_tool_message()

`display.py` 中的 `get_cute_tool_message()` (L796-950) 为每种工具生成带 emoji 的单行摘要：

```
┊ 🔍 search    "how to parse JSON"          2.3s
┊ 📖 read      /src/main.py                 0.4s
┊ 💻 $         npm install express           5.1s
┊ 🔧 patch     /src/config.ts               0.2s
┊ 🧠 memory    +facts: "user prefers dark"   0.3s
┊ 🔀 delegate  3 parallel tasks             12.5s
```

**特性**：
- `┊` 前缀可由皮肤的 `tool_prefix` 替换
- 路径截断 (`_path()`)、内容截断 (`_trunc()`)
- 失败检测 (`_detect_tool_failure`)：红色前缀 + `[exit 1]` / `[error]` / `[full]` 后缀
- 皮肤前缀 `get_skin_tool_prefix()` 集成

### 8.2 内联 Diff 渲染

`render_edit_diff_with_delta()` (L542-564) 在 CLI 中内联显示文件变更：
- 最多 6 个文件 (`_MAX_INLINE_DIFF_FILES`)
- 最多 80 行 (`_MAX_INLINE_DIFF_LINES`)
- 颜色从皮肤引擎获取 (`_diff_ansi()`)，自动适配浅/深色主题

## 9. 上下文压力显示 (Context Pressure Display)

`format_context_pressure()` (L978-1012) 显示 context window 使用进度：

```
  ⚠ context ▰▰▰▰▰▰▰▰▰▰▱▱▱▱▱▱▱▱▱▱ 52% to compaction
    128k threshold (80%) · compaction approaching
```

- 使用 `▰` (filled) / `▱` (empty) 字符绘制进度条
- 颜色随压力等级变化
- `format_context_pressure_gateway()` 提供无 ANSI 的纯文本版本 (用于 Telegram/Discord)

## 10. 设计模式总结 (Design Patterns Summary)

### 10.1 数据驱动配置 (Data-Driven Configuration)
- 皮肤 = YAML/dict 数据 → SkinConfig dataclass → 运行时查询
- 命令 = CommandDef dataclass → 注册表 → 派生查找表

### 10.2 桥接器模式 (Bridge Pattern)
- `ChatConsole`：Rich Console API → prompt_toolkit ANSI renderer
- `_cprint()` / `cprint()`：ANSI 字符串 → `_pt_print(_PT_ANSI(text))`

### 10.3 状态机 UI (State Machine UI)
- Enter 键的路由根据当前 UI 状态 (`_sudo_state`, `_approval_state`, `_clarify_state`, `_agent_running`) 切换行为
- 每种模态状态 (modal state) 使用 `response_queue` 与后台线程通信

### 10.4 优雅降级 (Graceful Degradation)
- KawaiiSpinner 在 3 种环境 (TTY/pipe/StdoutProxy) 自动选择输出策略
- 皮肤加载失败 → 回退 default
- prompt_toolkit 缺失 → commands.py 中的 Completer/AutoSuggest 降级为 object

### 10.5 单一事实来源 (Single Source of Truth)
- `COMMAND_REGISTRY` → 所有命令的唯一定义点
- `_BUILTIN_SKINS` → 所有内置皮肤的唯一定义点
- 派生数据（`_COMMAND_LOOKUP`, `SUBCOMMANDS`, `GATEWAY_KNOWN_COMMANDS`）在导入时自动构建

### 10.6 线程安全 (Thread Safety)
- KawaiiSpinner 在 daemon 线程中运行动画
- 捕获创建时的 stdout 引用，不受后续重定向影响
- `_approval_lock` 序列化并发审批提示（delegation race fix）
- Voice 状态使用 `_voice_lock` 保护跨线程访问

## 11. 文件规模参考 (File Size Reference)

| 文件 | 行数 | 大小 | 主要内容 |
|---|---|---|---|
| `cli.py` | 10,039 | 447KB | HermesCLI 完整实现，TUI 布局，chat()，process_command() |
| `commands.py` | 1,217 | 49KB | CommandDef，COMMAND_REGISTRY，Completer，AutoSuggest |
| `display.py` | 1,037 | 40KB | KawaiiSpinner，tool preview，diff 渲染，context pressure |
| `skin_engine.py` | 816 | 39KB | SkinConfig，内置皮肤定义，YAML 加载，PT 样式桥 |
| `banner.py` | 535 | 22KB | ASCII art，横幅构建，技能扫描，更新检查 |
