# Ch03 配置系统 — Research Notes

> Assignee: s-opus-03
> Task: T03-R
> 源码基线: hermes-agent v0.9.0 线
> 核心文件: `hermes_cli/config.py` (3415行), `hermes_cli/env_loader.py` (123行), `hermes_constants.py` (294行), `gateway/config.py` (~1160行), `gateway/run.py`, `cli-config.yaml.example` (881行), `.env.example` (391行), `hermes_cli/profiles.py` (1094行)

---

## 1. 配置系统全景

Hermes 的配置系统是一个**多层级、多源合并**的架构，核心设计理念是：

> **硬编码默认 → YAML 用户配置 → 环境变量覆盖 → CLI 参数**

这种分层设计在大型 CLI 应用中属于典型最佳实践，但 Hermes 在此基础上增加了以下独特特性：

- **Profile 机制**：支持多套独立配置（每个 profile 拥有独立的 `HERMES_HOME`）
- **Config Migration 系统**：版本化配置迁移（当前 `_config_version: 17`），自动升级旧配置
- **CLI↔Gateway 配置桥接**：CLI 配置通过环境变量传递给 Gateway 进程
- **Managed Mode**：受管模式下禁止写入配置文件，用于 SaaS/Docker 部署
- **安全加固**：`.env` 文件权限 600、凭证 ASCII 净化、secret 脱敏

---

## 2. 配置数据流——全链路时序

```
用户启动 `hermes` 命令
    │
    ▼
env_loader.load_hermes_dotenv()          ← 最早执行，加载 ~/.hermes/.env
    │  1. 清理 .env 腐败行 (_sanitize_env_file_if_needed)
    │  2. python-dotenv 加载到 os.environ (override=True)
    │  3. 凭证 ASCII 净化 (_sanitize_loaded_credentials)
    │  4. 可选：加载 project-level .env (override=False，仅填空)
    │
    ▼
config.load_config()                     ← main.py 中调用
    │  1. copy.deepcopy(DEFAULT_CONFIG)   — 3000+ 字段的默认值
    │  2. 读取 ~/.hermes/config.yaml (yaml.safe_load)
    │  3. _deep_merge(defaults, user_config) — 递归合并
    │  4. _normalize_root_model_keys()    — 迁移旧根级 provider/base_url
    │  5. _normalize_max_turns_config()   — 迁移根级 max_turns → agent.max_turns
    │  6. _expand_env_vars()              — 展开 ${VAR} 引用
    │
    ▼
CLI 参数覆盖（在 main.py 中）
    │  HERMES_MODEL → config["model"]
    │  HERMES_PROVIDER → config["provider"]
    │  HERMES_SKIN → config["display"]["skin"]
    │
    ▼
gateway/run.py: run_gateway()            ← 仅 gateway 模式
    │  读取 CLI config → 设置 os.environ (HERMES_MODEL, HERMES_PROVIDER, API keys)
    │  → gateway 进程通过 os.environ 获取配置
    │
    ▼
gateway/config.py: load_gateway_config()
    │  优先级: env vars > config.yaml > gateway.json(legacy) > 内置默认
    │  独立的 GatewayConfig dataclass，与 CLI config 结构不同
```

---

## 3. 核心文件详解

### 3.1 `hermes_cli/config.py` — 配置系统的心脏 (3415 行)

这是整个 Hermes 项目中最大的单文件之一，承载了配置的全部生命周期：

#### 3.1.1 DEFAULT_CONFIG — 硬编码默认值 (L314-707)

一个接近 400 行的巨型字典，定义了所有配置项的默认值。结构如下：

| 顶层 Key | 用途 | 典型子字段 |
|-----------|------|-----------|
| `model` | 主模型 | 字符串或 `{default, provider, base_url}` |
| `providers` | 自定义 provider 注册 | `{name: {api, api_key, default_model}}` |
| `fallback_providers` | 备选 provider 列表 | `[{provider, model}]` |
| `agent` | Agent 运行参数 | `max_turns=90`, `gateway_timeout=1800`, `tool_use_enforcement="auto"` |
| `terminal` | 终端执行配置 | `backend="local"`, `timeout=180`, Docker/Modal/SSH 配置 |
| `browser` | 浏览器工具配置 | `inactivity_timeout`, `command_timeout`, Camofox |
| `checkpoints` | 文件系统快照 | `enabled=True`, `max_snapshots=50` |
| `compression` | 上下文压缩 | `threshold=0.50`, `target_ratio=0.20`, `protect_last_n=20` |
| `smart_model_routing` | 智能路由 | `enabled=False`, `max_simple_chars=160` |
| `auxiliary` | 辅助模型配置 | vision, web_extract, compression, session_search, skills_hub, approval, mcp, flush_memories |
| `display` | 显示设置 | `personality="kawaii"`, `skin="default"`, `streaming=False` |
| `privacy` | 隐私设置 | `redact_pii=False` |
| `tts` | 文字转语音 | edge/elevenlabs/openai/mistral/neutts 五种后端 |
| `stt` | 语音转文字 | local(whisper)/groq/openai/mistral |
| `voice` | 语音交互 | `record_key="ctrl+b"`, `max_recording_seconds=120` |
| `human_delay` | 模拟人类延迟 | `mode="off"`, `min_ms=800` |
| `context` | 上下文引擎 | `engine="compressor"` |
| `memory` | 持久记忆 | `memory_enabled=True`, `memory_char_limit=2200` |
| `delegation` | 子 Agent 委托 | model/provider/base_url/max_iterations |
| `skills` | 技能系统 | `external_dirs=[]` |
| `discord` / `whatsapp` | 平台设置 | 各平台特有选项 |
| `approvals` | 命令审批 | `mode="manual"`, `timeout=60` |
| `security` | 安全设置 | `redact_secrets=True`, tirith 扫描 |
| `logging` | 日志设置 | `level="INFO"`, `max_size_mb=5` |
| `network` | 网络设置 | `force_ipv4=False` |
| `_config_version` | 配置版本号 | 当前 `17` |

**设计亮点：**
- `auxiliary` 子系统使用统一的 `{provider, model, base_url, api_key, timeout}` 接口，8 个辅助任务复用同一结构
- `tts` 和 `stt` 按 provider 嵌套配置（`tts.edge.voice`, `tts.elevenlabs.voice_id`），每个 provider 有独立参数
- `delegation` 允许子 Agent 使用独立的模型/provider，实现主 Agent 用强模型、子 Agent 用便宜模型的成本优化

#### 3.1.2 `_deep_merge()` — 递归字典合并 (L2386-2403)

```python
def _deep_merge(base: dict, override: dict) -> dict:
    result = base.copy()
    for key, value in override.items():
        if key in result and isinstance(result[key], dict) and isinstance(value, dict):
            result[key] = _deep_merge(result[key], value)
        else:
            result[key] = value
    return result
```

**关键行为：**
- 当用户只覆盖 `tts.elevenlabs.voice_id` 时，`tts.elevenlabs.model_id` 保持默认值
- 非 dict 类型的值直接覆盖（不会合并列表）
- 用户新增的非默认 key 也会被保留

#### 3.1.3 `_expand_env_vars()` — 环境变量展开 (L2406-2423)

支持 `${VAR}` 语法在 config.yaml 中引用环境变量：
```python
def _expand_env_vars(obj):
    if isinstance(obj, str):
        return re.sub(r"\${([^}]+)}", lambda m: os.environ.get(m.group(1), m.group(0)), obj)
    # 递归处理 dict 和 list ...
```

未解析的 `${VAR}` 保持原样（不报错），方便检测缺失变量。

#### 3.1.4 `load_config()` — 配置加载主入口 (L2491-2515)

```python
def load_config() -> Dict[str, Any]:
    config = copy.deepcopy(DEFAULT_CONFIG)
    if config_path.exists():
        user_config = yaml.safe_load(f) or {}
        # 兼容：根级 max_turns → agent.max_turns
        config = _deep_merge(config, user_config)
    return _expand_env_vars(_normalize_root_model_keys(_normalize_max_turns_config(config)))
```

**处理管线：** `deepcopy(DEFAULT) → merge(user) → normalize_model_keys → normalize_max_turns → expand_env_vars`

#### 3.1.5 `save_config()` — 配置保存 (L2615-2641)

```python
def save_config(config: Dict[str, Any]):
    if is_managed():
        managed_error("save configuration")
        return
    from utils import atomic_yaml_write
    normalized = _normalize_root_model_keys(_normalize_max_turns_config(config))
    atomic_yaml_write(config_path, normalized, extra_content=parts)
    _secure_file(config_path)
```

**关键点：**
- **Managed Mode 保护**：受管模式下直接拒绝写入
- **原子写入**：通过 `atomic_yaml_write` 实现（先写临时文件，再 `os.replace`）
- **附加注释**：在 YAML 末尾追加被注释掉的 security/fallback 配置模板
- **安全加固**：`_secure_file()` 限制文件权限

#### 3.1.6 `read_raw_config()` — 轻量级原始读取 (L2473-2488)

不走 deep-merge 和 migration 管线的快速读取函数。用于迁移逻辑中检查用户实际写入的内容（vs 默认值），避免循环依赖。

---

### 3.2 环境变量系统

#### 3.2.1 `.env` 文件管理

- **路径**：`~/.hermes/.env`（由 `get_env_path()` 返回）
- **权限**：`0600`（owner-only read/write），因为包含 API keys
- **读取函数**：`get_env_value(key)` — 先查 `os.environ`，再 fallback 到 `.env` 文件
- **写入函数**：`save_env_value(key, value)` — 原子写入 `.env` 文件 + 同步更新 `os.environ`
- **删除函数**：`remove_env_value(key)` — 从 `.env` 文件和 `os.environ` 中移除
- **净化函数**：`_sanitize_env_lines()` — 修复腐败的 `.env` 文件（拼接行分割、清除 `***` 占位符）

#### 3.2.2 `env_loader.py` — 最早期加载 (123 行)

`load_hermes_dotenv()` 在 CLI 启动链路的最早期被调用：

```
加载优先级:
1. ~/.hermes/.env (override=True) — 用户配置，覆盖 shell 中已有的同名变量
2. project/.env (override=False)  — 开发用 fallback，只填空缺变量
```

**防腐败机制：**
- `_sanitize_env_file_if_needed()` — 加载前预处理，修复拼接行（issue #8908）
- `_load_dotenv_with_fallback()` — UTF-8 失败自动 fallback 到 Latin-1
- `_sanitize_loaded_credentials()` — 加载后净化凭证中的非 ASCII 字符（PDF 复制产生的 Unicode 替身字符）

#### 3.2.3 `ENV_VARS` 字典 (L724+)

定义了所有可选环境变量的元数据：

```python
OPTIONAL_ENV_VARS = {
    "OPENAI_API_KEY": {
        "description": "OpenAI API key",
        "prompt": "OpenAI API key",
        "url": "https://platform.openai.com/api-keys",
        "password": True,
    },
    # ... 20+ 个 provider 的 API key 定义
}
```

每个条目包含：`description`（说明）、`prompt`（交互提示）、`url`（获取地址）、`password`（是否隐藏输入）、`advanced`（是否仅高级用户）。

#### 3.2.4 `ENV_VARS_BY_VERSION` — 版本化环境变量引入 (L715+)

记录每个 config version 新增的环境变量，用于 migration 时只提示新增的 key：

```python
ENV_VARS_BY_VERSION: Dict[int, List[str]] = {
    3: ["FIRECRAWL_API_KEY", "BROWSERBASE_API_KEY", ...],
    4: ["VOICE_TOOLS_OPENAI_KEY", "ELEVENLABS_API_KEY"],
    5: ["WHATSAPP_ENABLED", "WHATSAPP_MODE", ...],
    # ...
}
```

---

### 3.3 Provider 系统

#### 3.3.1 `PROVIDER_ALIASES` (L200+)

将缩写/常见变体映射到规范 provider 名称：

```python
PROVIDER_ALIASES = {
    "or": "openrouter",
    "gpt": "openai",
    "claude": "anthropic",
    # ...
}
```

#### 3.3.2 自定义 Provider 注册

用户可在 `config.yaml` 的 `providers:` 下注册自定义 OpenAI 兼容端点：

```yaml
providers:
  my-local-llm:
    api: http://localhost:8080/v1
    api_key: no-key
    default_model: my-model
    name: "My Local LLM"
```

---

### 3.4 Gateway 配置桥接

#### 3.4.1 `gateway/run.py` — CLI→Gateway 环境变量桥 

当 Hermes 以 Gateway 模式运行时（Telegram/Discord/Slack/WhatsApp），CLI config 需要转换为环境变量传递给 Gateway 进程：

```python
# run_gateway() 中的核心逻辑（简化）
config = load_config()
os.environ["HERMES_MODEL"] = config.get("model", "")
os.environ["HERMES_PROVIDER"] = config.get("provider", "")
# 桥接 API keys
for key_name in ("OPENAI_API_KEY", "ANTHROPIC_API_KEY", "OPENROUTER_API_KEY", ...):
    if val := get_env_value(key_name):
        os.environ[key_name] = val
```

#### 3.4.2 `gateway/config.py` — GatewayConfig dataclass

Gateway 有独立的配置数据模型 `GatewayConfig`，不直接复用 CLI 的 DEFAULT_CONFIG dict：

```python
@dataclass
class GatewayConfig:
    model: str = ""
    provider: str = ""
    base_url: str = ""
    platforms: Dict[str, PlatformConfig] = field(default_factory=dict)
    # ... 30+ 字段
```

**`load_gateway_config()` 多源加载 (L435+)：**
```
优先级（高→低）:
1. 环境变量 (os.environ)
2. ~/.hermes/config.yaml (YAML 各 section 映射到 GatewayConfig 字段)
3. ~/.hermes/gateway.json (legacy fallback)
4. 内置默认值
```

`config.yaml` 到 `GatewayConfig` 的映射不是自动的，而是手动逐字段桥接（如 `yaml_cfg.get("stt")` → `gw_data["stt"]`），这保证了清晰的转换语义。

---

### 3.5 Config Migration 系统

#### 3.5.1 版本化迁移架构 (L1768-2383)

`migrate_config()` 函数是迁移入口，检查 `_config_version` 并执行增量迁移：

```python
def migrate_config(interactive=True, quiet=False) -> Dict:
    current_ver, latest_ver = check_config_version()
    # Version 1 → 2: migrate display settings...
    if current_ver < 2:
        # ...
    # Version 2 → 3: migrate tool_progress...
    if current_ver < 3:
        # ...
    # ... 一直到 version 16 → 17
```

**已实现的迁移列表：**

| 版本跳转 | 迁移内容 |
|----------|----------|
| 1→2 | display.tool_progress 设置迁移 |
| 4→5 | 新增 timezone 字段 |
| 8→9 | 清除 ANTHROPIC_TOKEN（旧认证流程） |
| 11→12 | `custom_providers` 列表 → `providers` 字典 |
| 12→13 | 清除 LLM_MODEL/OPENAI_MODEL（已废弃 env vars） |
| 13→14 | STT flat `model` → provider-specific 嵌套 |
| 14→15 | 新增 display.interim_assistant_messages |
| 15→16 | tool_progress_overrides → display.platforms |
| 16→17 | 移除 compression.summary_* → auxiliary.compression |

**设计特点：**
- 使用 `if current_ver < N:` 而非 switch/case，保证从任意旧版本可一次性升级到最新
- 迁移中使用 `read_raw_config()` 检查用户实际配置（不含默认值），避免误迁移
- 每步迁移后调用 `save_config()` 持久化
- 支持 `quiet` 模式（后台迁移不输出）和 `interactive` 模式（交互式补充缺失 key）

#### 3.5.2 缺失字段自动补充

```python
missing_config = get_missing_config_fields()
for field in missing_config:
    _set_nested(config, field["key"], field["default"])
```

使用 `_set_nested()` 支持点分路径（如 `display.skin`）设置嵌套字典值。

#### 3.5.3 技能声明的配置变量

```python
missing_skill_config = get_missing_skill_config_vars()
# Skills 在 SKILL.md frontmatter 的 metadata.hermes.config 中声明需要的 config 变量
# migration 时交互式提示用户配置
```

---

### 3.6 Profile 系统 (`profiles.py`, 1094 行)

#### 3.6.1 核心概念

Profile 是**独立的 HERMES_HOME 目录**，每个 profile 拥有自己的 `config.yaml`、`.env`、sessions、skills：

```
~/.hermes/                    ← "default" profile
~/.hermes/profiles/work/      ← "work" profile 的 HERMES_HOME
~/.hermes/profiles/personal/  ← "personal" profile
~/.hermes/active_profile      ← 当前活跃 profile 名称
```

#### 3.6.2 切换机制

- `hermes -p <profile>` — 单次指定 profile
- `hermes profile switch <name>` — 持久切换（写入 `active_profile` 文件）
- 切换时设置 `HERMES_HOME` 环境变量指向 profile 目录

#### 3.6.3 安全约束

```python
_RESERVED_NAMES = frozenset({"hermes", "default", "test", "tmp", "root", "sudo"})
_HERMES_SUBCOMMANDS = frozenset({"chat", "model", "gateway", "setup", ...})
```

Profile 名称不能与保留名或子命令冲突。名称验证：`[a-z0-9][a-z0-9_-]{0,63}`。

#### 3.6.4 Alias/Wrapper Scripts

`~/.local/bin/<profile>` — 自动生成的 shell wrapper，`<alias>` 等同于 `hermes -p <profile>`。

---

### 3.7 `hermes_constants.py` — 路径与常量 (294 行)

提供全局路径计算：

```python
def get_hermes_home() -> Path:
    """返回 HERMES_HOME（默认 ~/.hermes）"""
    return Path(os.getenv("HERMES_HOME", Path.home() / ".hermes"))

def get_default_hermes_root() -> Path:
    """返回默认根目录（忽略 profile 切换的影响）"""

def get_config_path() -> Path:
    return get_hermes_home() / "config.yaml"

def get_env_path() -> Path:
    return get_hermes_home() / ".env"
```

还定义了 base URL 常量（Nous Portal、Skills Hub 等）和版本信息。

---

### 3.8 Managed Mode — 受管模式

```python
def is_managed() -> bool:
    """True when running as a managed service (SaaS/Docker)."""
    return os.getenv("HERMES_MANAGED", "").lower() in ("1", "true", "yes")

def managed_error(action: str):
    """Raise error for write operations in managed mode."""
    raise RuntimeError(f"Cannot {action}: running in managed mode")
```

受管模式下，`save_config()`、`save_env_value()`、`remove_env_value()` 等写操作全部被拦截，配置完全由 orchestrator 通过环境变量注入。

---

## 4. 配置安全机制

### 4.1 文件权限

```python
def _secure_file(path: Path):
    if not _IS_WINDOWS:
        os.chmod(path, stat.S_IRUSR | stat.S_IWUSR)  # 0600
```

`.env` 和 `config.yaml` 写入后立即设置为 owner-only 权限。

### 4.2 凭证净化

```python
def _check_non_ascii_credential(key, value):
    """Strip non-ASCII from API keys (they must be valid HTTP header values)."""

def _sanitize_loaded_credentials():
    """After dotenv load: strip non-ASCII from *_TOKEN, *_SECRET, *_KEY vars."""
```

### 4.3 原子文件写入

所有配置写入都通过 `tempfile.mkstemp → write → os.fsync → os.replace` 模式，保证崩溃安全：

```python
fd, tmp_path = tempfile.mkstemp(dir=str(env_path.parent), suffix='.tmp', prefix='.env_')
with os.fdopen(fd, 'w') as f:
    f.writelines(lines)
    f.flush()
    os.fsync(f.fileno())
os.replace(tmp_path, env_path)  # 原子替换
```

### 4.4 .env 文件腐败修复

`_sanitize_env_lines()` 处理两种已知腐败模式：
1. **拼接行**：多个 `KEY=VALUE` 被写在同一行（缺少换行符），#8908
2. **占位符残留**：`KEY=***` 由不完整的 setup 留下

---

## 5. 配置项交互——`setup_env()` 和 `setup_config()` 向导

交互式配置向导在首次运行或版本升级时触发：

1. **必填环境变量检查**：`get_missing_env_vars(required_only=True)`
2. **新增可选变量提示**：`ENV_VARS_BY_VERSION` 中当前版本新增的变量
3. **缺失配置字段补充**：`get_missing_config_fields()` 自动补上默认值
4. **技能声明配置**：`get_missing_skill_config_vars()` 检查 SKILL.md 中声明的配置需求

---

## 6. 设计模式与值得学习的实践

### 6.1 分层配置合并（Layered Configuration）

经典的 4 层合并：默认值 → 用户 YAML → 环境变量 → CLI 参数。通过 `_deep_merge()` 实现嵌套 dict 的智能合并，用户只需覆盖想改的字段。

### 6.2 版本化迁移（Versioned Migration）

类似数据库 migration 的模式：`_config_version` 跟踪当前版本，`migrate_config()` 执行增量升级。关键优势：
- 老用户升级无需手动修改配置
- 新字段自动补充默认值
- 废弃字段自动清理

### 6.3 配置与运行时隔离（Config→Env Bridge）

CLI config（YAML dict）通过显式的环境变量桥传递给 Gateway，而非共享内存。这使得 Gateway 可以独立进程运行，也方便容器化部署。

### 6.4 防御性编程

- 每个读取都有 try/except 包裹
- `.env` 解析支持 UTF-8 → Latin-1 fallback
- 凭证自动净化非 ASCII 字符
- 原子写入防止断电数据丢失
- Windows 兼容：显式指定 `encoding="utf-8"`

### 6.5 Profile 实现的巧妙之处

Profile 不是"切换配置文件"，而是**切换整个 HERMES_HOME 目录**——每个 profile 有独立的 sessions、skills、memories。这避免了选择性合并的复杂性。

---

## 7. 与其他系统的交互点

| 消费方 | 配置入口 | 说明 |
|--------|---------|------|
| Agent 主循环 | `load_config()` | 获取 model、agent 参数 |
| 工具系统 | `config["terminal"]`, `config["browser"]` | 工具特定配置 |
| Gateway | `load_gateway_config()` | 独立的 dataclass 加载 |
| TTS/STT | `config["tts"]`, `config["stt"]` | 按 provider 嵌套 |
| 记忆系统 | `config["memory"]` | 是否启用、char limit |
| 压缩系统 | `config["compression"]` | 阈值、保护消息数 |
| 辅助模型 | `config["auxiliary"]` | 8 个子任务的独立模型配置 |
| 安全扫描 | `config["security"]` | tirith 扫描、secret 脱敏 |
| Profiles | `profiles.py` | 切换 HERMES_HOME |

---

## 8. 潜在的写作要点

1. **多层合并流程图**——可用 ASCII art 或 Mermaid 展示 4 层覆盖
2. **DEFAULT_CONFIG 字段表**——按功能分组的完整表格
3. **Migration 系统设计**——与数据库 schema migration 的类比
4. **Profile 的 HERMES_HOME 切换**——为什么是目录级隔离而非文件级选择
5. **CLI→Gateway 配置桥**——为什么不直接传 dict 而是走环境变量
6. **安全加固实践**——chmod 600、原子写入、凭证净化

---

## 9. 源码行号索引

| 函数/常量 | 文件 | 行号 |
|-----------|------|------|
| `DEFAULT_CONFIG` | `config.py` | L314-707 |
| `ENV_VARS_BY_VERSION` | `config.py` | L715-720+ |
| `PROVIDER_ALIASES` | `config.py` | L200+ |
| `_deep_merge()` | `config.py` | L2386-2403 |
| `_expand_env_vars()` | `config.py` | L2406-2423 |
| `read_raw_config()` | `config.py` | L2473-2488 |
| `load_config()` | `config.py` | L2491-2515 |
| `save_config()` | `config.py` | L2615-2641 |
| `load_env()` | `config.py` | L2644-2670 |
| `_sanitize_env_lines()` | `config.py` | L2673+ |
| `get_env_value()` | `config.py` | ~L1550 |
| `save_env_value()` | `config.py` | L2810-2872 |
| `remove_env_value()` | `config.py` | L2874-2916 |
| `sanitize_env_file()` | `config.py` | L2724 |
| `migrate_config()` | `config.py` | L1978-2383 |
| `check_config_version()` | `config.py` | L1768 |
| `is_managed()` | `config.py` | 早期定义 |
| `load_hermes_dotenv()` | `env_loader.py` | L92-123 |
| `_sanitize_loaded_credentials()` | `env_loader.py` | L18-31 |
| `get_hermes_home()` | `hermes_constants.py` | 早期 |
| `get_config_path()` | `hermes_constants.py` | 路径 helper |
| `GatewayConfig` | `gateway/config.py` | dataclass |
| `load_gateway_config()` | `gateway/config.py` | L435+ |
| `ProfileManager` / path helpers | `profiles.py` | L120-220 |
