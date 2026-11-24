---
Task: T22-R
Status: DRAFT_DONE
Date: 2026-04-15
Chapter: Ch22 终端后端
Keywords: local/docker/ssh/daytona/modal/singularity 的统一抽象
Sources:
  - tools/environments/base.py (579 lines)
  - tools/environments/__init__.py (13 lines)
  - tools/environments/local.py (314 lines)
  - tools/environments/docker.py (578 lines)
  - tools/environments/ssh.py (258 lines)
  - tools/environments/modal.py (434 lines)
  - tools/environments/managed_modal.py (282 lines)
  - tools/environments/modal_utils.py (186 lines)
  - tools/environments/daytona.py (229 lines)
  - tools/environments/singularity.py (262 lines)
  - tools/environments/file_sync.py (168 lines)
  - tools/terminal_tool.py (1749 lines)
  - tools/process_registry.py (1172 lines)
---

# Ch22 终端后端 — Terminal Backend Architecture

## 1. 架构全景 (Architecture Overview)

Hermes Agent 的终端系统（terminal system）采用**三层架构**：

```
┌─────────────────────────────────────────────────────────────┐
│  terminal_tool()  — 前端编排层 (Frontend Orchestration)       │
│  • 环境创建/缓存/生命周期管理                                    │
│  • 安全检查 (dangerous command guard, tirith)                  │
│  • 前台/后台分发                                                │
├─────────────────────────────────────────────────────────────┤
│  BaseEnvironment ABC  — 统一抽象层 (Unified Abstraction)       │
│  • ProcessHandle protocol                                    │
│  • execute() orchestration pipeline                          │
│  • CWD tracking, session snapshots                           │
├──────────┬──────────┬────────┬─────────┬─────────┬──────────┤
│  Local   │  Docker  │  SSH   │  Modal  │ Daytona │Singularity│
│  Backend │  Backend │ Backend│ Backend │ Backend │  Backend  │
└──────────┴──────────┴────────┴─────────┴─────────┴──────────┘
                        ↕
┌─────────────────────────────────────────────────────────────┐
│  ProcessRegistry  — 后台进程管理 (Background Process Mgmt)    │
│  • Output buffering (200KB rolling window)                   │
│  • spawn_local / spawn_via_env                               │
│  • poll / wait / kill / stdin I/O                            │
│  • Crash recovery via JSON checkpoint                        │
└─────────────────────────────────────────────────────────────┘
```

### 核心设计原则 (Core Design Principles)

1. **Spawn-per-call 模型**: 每次 `execute()` 调用 spawn 一个新 shell 进程。CWD 跨调用跟踪，但 shell 环境不保留
2. **Backend 透明性**: `terminal_tool()` 对 LLM 呈现统一接口，LLM 无需知道底层是 Docker/Modal/本地
3. **安全隔离**: 非 local 后端在沙箱中执行，`_HERMES_PROVIDER_ENV_BLOCKLIST` 过滤敏感环境变量
4. **惰性初始化**: 环境在首次使用时创建，通过 `_active_environments` 字典缓存

## 2. 统一抽象层 — BaseEnvironment ABC

**文件**: `tools/environments/base.py` (579 lines)

### 2.1 ProcessHandle Protocol (进程句柄协议)

```python
class ProcessHandle(Protocol):
    """Duck type that every backend's _run_bash() must return.
    subprocess.Popen satisfies this natively. SDK backends return
    _ThreadedProcessHandle which adapts their blocking calls."""
    def poll(self) -> int | None: ...
    def kill(self) -> None: ...
    def wait(self, timeout: float | None = None) -> int: ...
    @property
    def stdout(self) -> IO[str] | None: ...
    @property
    def returncode(self) -> int | None: ...
```

关键洞察：`subprocess.Popen` 天然满足此协议；SDK 后端（Modal、Daytona）使用 `_ThreadedProcessHandle` 适配器。

### 2.2 _ThreadedProcessHandle (线程化进程适配器)

```python
class _ThreadedProcessHandle:
    """Wraps a blocking exec_fn() -> (output_str, exit_code)
    in a background thread and exposes ProcessHandle-compatible interface."""
    def __init__(self, exec_fn, cancel_fn=None):
        # Background thread runs exec_fn()
        # Output piped through os.pipe() for stdout compatibility
        # cancel_fn invoked on kill() for backend-specific cancellation
```

**设计模式**: Adapter Pattern（适配器模式）— 将 SDK 的阻塞式 API 适配为 Popen 兼容的非阻塞接口。

### 2.3 execute() Pipeline (执行管道)

```
execute(command, cwd, timeout, stdin_data)
    → _before_execute()      # FileSyncManager hook (远程后端同步凭据文件)
    → _prepare_command()     # sudo password transform
    → _wrap_command()        # 添加 CWD 标记、环境变量、bash 包装
    → _run_bash()            # Backend-specific: 实际执行命令
    → _wait_for_process()    # 等待完成，带 interrupt 检查
    → _update_cwd()          # 从输出中提取新 CWD
```

### 2.4 CWD Tracking (工作目录跟踪)

CWD 使用 **marker 技术**跟踪：命令完成后在输出中打印特殊标记 `HERMES_CWD_MARKER`，然后 `_extract_cwd_from_output()` 从输出中提取。

- **Local 后端**: 使用 `mktemp` 创建临时文件，将 CWD 写入文件再读取
- **其他后端**: 在 bash 包装命令的末尾追加 `; echo "HERMES_CWD_MARKER$(pwd)"`

### 2.5 Session Snapshots (会话快照)

`_load_json_store()` / `_save_json_store()` 管理基于 JSON 文件的持久状态，用于 Docker 容器 ID、Modal sandbox ID、Singularity overlay 路径等。

## 3. 各后端实现详解 (Backend Implementation Details)

### 3.1 LocalEnvironment (本地后端)

**文件**: `tools/environments/local.py` (314 lines)

| 特性 | 值 |
|------|-----|
| 执行方式 | `subprocess.Popen` with login shell (`bash -lic`) |
| 安全机制 | `_HERMES_PROVIDER_ENV_BLOCKLIST` 过滤 ~60+ 敏感环境变量 |
| 进程隔离 | `os.setsid()` 创建新进程组 (process group) |
| CWD 跟踪 | 临时文件写入 + 读取 |
| 持久性 | 文件系统就是宿主机，天然持久 |
| stdin 模式 | pipe (默认) |

**环境变量屏蔽列表** (`_HERMES_PROVIDER_ENV_BLOCKLIST`):
- API 密钥: `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `GOOGLE_API_KEY` 等
- 消息平台凭据: `DISCORD_BOT_TOKEN`, `SLACK_BOT_TOKEN`, `TELEGRAM_BOT_TOKEN`
- Hermes 内部: `HERMES_*` 系列变量
- 数据库/存储: `REDIS_URL`, `DATABASE_URL`, `SUPABASE_*`

```python
def _sanitize_subprocess_env(base_env, extra_vars=None):
    """Create a sanitized copy of the environment, removing blocked variables."""
    env = {k: v for k, v in base_env.items()
           if k not in _HERMES_PROVIDER_ENV_BLOCKLIST}
    if extra_vars:
        env.update(extra_vars)
    return env
```

**`_find_shell()` 函数**: 按优先级查找 shell — `$SHELL` → `/bin/bash` → `/usr/bin/bash` → `shutil.which("bash")` → `/bin/sh`。

### 3.2 DockerEnvironment (Docker 后端)

**文件**: `tools/environments/docker.py` (578 lines)

| 特性 | 值 |
|------|-----|
| 安全加固 | `--cap-drop ALL`, `--security-opt no-new-privileges`, PID limits |
| 容器发现 | `find_docker()` 搜索 PATH + `/usr/local/bin/docker` + `/opt/homebrew/bin/docker` |
| 持久性 | Bind mount 目录到宿主机（workspace + home 分离） |
| 环境变量 | `docker_forward_env` 列表 + 自动规范化键名 |
| 磁盘限制 | `--storage-opt size=` 仅 overlay2+XFS+pquota 支持 |

**容器创建安全参数**:
```python
cmd = [docker, "run", "-d",
       "--cap-drop", "ALL",           # 丢弃所有 Linux capabilities
       "--security-opt", "no-new-privileges",  # 禁止提权
       "--pids-limit", "256",         # 进程数限制
       "-w", cwd,                     # 工作目录
       "-v", f"{workspace}:/workspace",  # bind mount
       "-v", f"{home}:/root",         # home 持久化
]
```

**清理策略**: `cleanup()` 使用 `timeout 60 docker stop` + 后台 `docker rm -f`，非阻塞设计。

### 3.3 SSHEnvironment (SSH 后端)

**文件**: `tools/environments/ssh.py` (258 lines)

| 特性 | 值 |
|------|-----|
| 连接持久化 | SSH ControlMaster + ControlPath + ControlPersist=300 |
| 文件同步 | FileSyncManager 使用 scp 上传 |
| 远程 home | `_detect_remote_home()` 自动检测 |
| 目录初始化 | `_ensure_remote_dirs()` 创建 `.hermes/` 目录结构 |

**ControlMaster 连接池**:
```python
def _build_ssh_command(self, command):
    return ["ssh",
            "-o", "BatchMode=yes",
            "-o", f"ControlPath={self._control_path}",
            "-o", "ControlMaster=auto",
            "-o", "ControlPersist=300",   # 保持连接 5 分钟
            "-p", str(self.port),
            f"{self.user}@{self.host}",
            command]
```

**FileSyncManager 集成**: SSH 后端通过 `_before_execute()` hook 在每次命令执行前同步凭据文件（credentials/skills/cache）到远程。

### 3.4 ModalEnvironment (Modal SDK 后端)

**文件**: `tools/environments/modal.py` (434 lines)

| 特性 | 值 |
|------|-----|
| SDK 调用 | `modal.Sandbox.create()` + `sandbox.exec()` |
| 异步桥接 | `_AsyncWorker` 单线程事件循环，sync→async 桥接 |
| 镜像解析 | `_resolve_modal_image()` 支持 Docker Hub / Dockerfile / snapshot ID |
| 持久性 | Filesystem snapshot (`sandbox.snapshot_filesystem()`) |
| 文件同步 | tar-based bulk upload |
| 快照存储 | `modal_snapshots.json` 记录 task_id → snapshot_id 映射 |

**_AsyncWorker (异步工作者)**:
```python
class _AsyncWorker:
    """Single-thread event loop for sync→async bridging.
    All Modal SDK calls go through this to avoid blocking the main thread."""
    def run(self, coro) -> Any:
        future = asyncio.run_coroutine_threadsafe(coro, self._loop)
        return future.result()
```

**镜像解析逻辑** (`_resolve_modal_image`):
- 输入以 `sha256:` 开头 → Modal snapshot ID 恢复
- 输入包含 `/` → Docker Hub 镜像 (`modal.Image.from_registry`)
- 输入是 Dockerfile 路径 → `modal.Image.from_dockerfile`
- Ubuntu/Debian 镜像 → 自动 `add_python()` 确保 Python 可用

### 3.5 ManagedModalEnvironment (托管 Modal 后端)

**文件**: `tools/environments/managed_modal.py` (282 lines)

| 特性 | 值 |
|------|-----|
| 与 Modal SDK 区别 | 通过 tool-gateway HTTP API 间接调用，不直接使用 Modal SDK |
| 执行模型 | Poll-based: 发起请求 → 轮询完成状态 |
| 超时配置 | `_request_timeout_env` 环境变量可配置 |
| 幂等性 | `idempotency_key` 防止重复创建 sandbox |
| 凭据保护 | `_guard_unsupported_credential_passthrough` 检查 |

**BaseModalExecutionEnvironment (共享基类)**:
```python
class BaseModalExecutionEnvironment(BaseEnvironment):
    """Override BaseEnvironment.execute() because gateway handles
    command preparation, CWD tracking, and snapshot management."""
    _stdin_mode = "payload"           # 直接传递
    _poll_interval_seconds = 0.25
    # Template Method pattern: subclass 实现 _start/_poll/_cancel
    @abstractmethod
    def _start_modal_exec(self, prepared) -> ModalExecStart: ...
    @abstractmethod
    def _poll_modal_exec(self, handle) -> dict | None: ...
    @abstractmethod
    def _cancel_modal_exec(self, handle) -> None: ...
```

### 3.6 DaytonaEnvironment (Daytona 后端)

**文件**: `tools/environments/daytona.py` (229 lines)

| 特性 | 值 |
|------|-----|
| SDK | `daytona.Daytona` SDK |
| 持久性 | Sandbox stop/resume by name + label 查找 |
| stdin 模式 | `_stdin_mode = "heredoc"` (SDK 不支持 pipe stdin) |
| 磁盘限制 | 硬编码 10GB 上限 (`min(disk, 10240)`) |
| 进程适配 | `_ThreadedProcessHandle` 包装阻塞式 SDK exec |

**Sandbox 恢复逻辑**:
```python
# 1. 按名称查找已有 sandbox
# 2. 如找不到，按 label 查找
# 3. 如都找不到，创建新 sandbox
# 4. 如找到已停止的 sandbox，恢复它
```

### 3.7 SingularityEnvironment (Singularity/Apptainer 后端)

**文件**: `tools/environments/singularity.py` (262 lines)

| 特性 | 值 |
|------|-----|
| 可执行文件 | `_find_singularity_executable()`: apptainer 优先，然后 singularity |
| 安全隔离 | `--containall` + `--no-home` |
| 持久性 | Writable overlay directories |
| Scratch 目录 | `_get_scratch_dir()`: TERMINAL_SCRATCH_DIR → /scratch → sandbox dir |
| 实例执行 | `singularity instance start` + `singularity exec instance://` |

**Overlay 目录结构**:
```
<scratch_dir>/hermes-<task_id>/
  ├── overlay/          # Writable overlay (挂载为容器的可写层)
  ├── workspace/        # /workspace bind mount
  └── home/             # /root bind mount
```

## 4. 后端分类学 (Backend Taxonomy)

### 4.1 按文件同步需求分类

| 类型 | 后端 | 文件同步 | 原因 |
|------|------|----------|------|
| **Bind-mount** | Local, Docker, Singularity | ❌ 不需要 | 宿主机文件系统直接可见 |
| **Remote/SDK** | SSH, Modal, Daytona | ✅ FileSyncManager | 沙箱文件系统与宿主机隔离 |

### 4.2 按 stdin 模式分类

| 模式 | 后端 | 说明 |
|------|------|------|
| `pipe` (默认) | Local, Docker, SSH, Singularity | stdin 通过管道传递 |
| `heredoc` | Daytona | SDK 不支持管道，stdin 嵌入为 heredoc |
| `payload` | ManagedModal | stdin 作为 HTTP payload 传递给 gateway |

### 4.3 按持久性策略分类

| 后端 | 持久性机制 | 配置项 |
|------|-----------|--------|
| Local | 宿主文件系统 | N/A (天然持久) |
| Docker | Bind mount 目录 | `container_persistent` |
| Singularity | Overlay 目录 | `persistent_filesystem` |
| Modal | Filesystem snapshot (`snapshot_filesystem()`) | `persistent_filesystem` |
| Daytona | SDK sandbox stop/resume | `persistent_filesystem` |
| SSH | 远程文件系统 | N/A (天然持久) |

## 5. 前端编排层 — terminal_tool.py

**文件**: `tools/terminal_tool.py` (1749 lines)

### 5.1 环境生命周期管理 (Environment Lifecycle)

```python
# 全局状态
_active_environments: Dict[str, Any] = {}     # task_id → env 实例
_last_activity: Dict[str, float] = {}          # task_id → last_used_time
_creation_locks: Dict[str, threading.Lock] = {} # 防止并发创建同一 task 的沙箱
```

**双重检查锁 (Double-checked Locking)**:
```python
# 1. 快速路径: 在全局锁下检查缓存
with _env_lock:
    if effective_task_id in _active_environments:
        env = _active_environments[effective_task_id]
        needs_creation = False

# 2. 慢速路径: 获取 per-task lock
if needs_creation:
    with task_lock:
        # 再次检查（另一个线程可能已创建）
        with _env_lock:
            if effective_task_id in _active_environments:
                needs_creation = False
        if needs_creation:
            new_env = _create_environment(...)
```

### 5.2 _create_environment() — 工厂方法 (Factory Method)

```python
def _create_environment(env_type, image, cwd, timeout, ssh_config, container_config, ...):
    """Factory function — creates the appropriate backend based on TERMINAL_ENV."""
    if env_type == "local":    return _LocalEnvironment(...)
    elif env_type == "docker":  return _DockerEnvironment(...)
    elif env_type == "singularity": return _SingularityEnvironment(...)
    elif env_type == "modal":   # Direct vs Managed 选择逻辑
    elif env_type == "daytona": return _DaytonaEnvironment(...)
    elif env_type == "ssh":     return _SSHEnvironment(...)
```

### 5.3 环境配置 — _get_env_config()

所有配置来自环境变量：

| 环境变量 | 默认值 | 说明 |
|---------|--------|------|
| `TERMINAL_ENV` | `local` | 后端类型 |
| `TERMINAL_DOCKER_IMAGE` | `nikolaik/python-nodejs:python3.11-nodejs20` | Docker 镜像 |
| `TERMINAL_TIMEOUT` | `180` | 默认超时秒数 |
| `TERMINAL_LIFETIME_SECONDS` | `300` | 空闲环境清理阈值 |
| `TERMINAL_CWD` | 取决于后端 | 工作目录 |
| `TERMINAL_CONTAINER_CPU` | `1` | CPU 核数 |
| `TERMINAL_CONTAINER_MEMORY` | `5120` | 内存 MB |
| `TERMINAL_CONTAINER_DISK` | `51200` | 磁盘 MB |
| `TERMINAL_SSH_HOST/USER/PORT/KEY` | — | SSH 连接配置 |

**CWD 默认值逻辑**:
- `local` → `os.getcwd()` (宿主当前目录)
- `ssh` → `~` (远程 home)
- 容器后端 → `/root`

### 5.4 空闲清理线程 (Idle Cleanup Thread)

```python
def _cleanup_inactive_envs(lifetime_seconds=300):
    # 1. 检查 ProcessRegistry — 有活跃后台进程的环境不清理
    # 2. 收集超过 lifetime_seconds 的过期环境
    # 3. 在锁外执行 env.cleanup()（避免阻塞其他调用）
```

- 后台守护线程，每 60 秒运行一次
- 两阶段清理：Phase 1 在锁内收集，Phase 2 在锁外执行 cleanup（Modal/Docker 可能阻塞 10-15s）
- `atexit` 注册 `_atexit_cleanup()` 确保进程退出时关闭所有沙箱

### 5.5 Per-Task 环境覆盖 (Task-level Environment Overrides)

```python
_task_env_overrides: Dict[str, Dict[str, Any]] = {}

def register_task_env_overrides(task_id, overrides):
    """Called by Atropos environments before agent loop.
    Supported keys: modal_image, docker_image, cwd"""
```

用途：TerminalBench2 等评测环境可以为每个 rollout 指定自定义 Docker 镜像。

### 5.6 前台命令执行流 (Foreground Execution Flow)

```
terminal_tool(command, timeout=180)
  → _get_env_config()              # 读取环境变量配置
  → get/create environment          # 双重检查锁
  → _check_all_guards(command)      # 安全检查 (tirith + 危险命令)
  → _validate_workdir(workdir)      # 防止 shell injection
  → env.execute(command, timeout)   # 调用后端执行
  → _handle_sudo_failure(output)    # sudo 失败友好提示
  → truncate output (50KB max)      # 40% head + 60% tail
  → strip_ansi(output)             # 去除 ANSI 转义码
  → redact_sensitive_text(output)  # 密钥脱敏
  → _interpret_exit_code()         # 常见退出码语义注释
  → JSON response                   # {output, exit_code, error}
```

**输出截断策略**: 超过 50,000 字符时保留 40% 头部 + 60% 尾部，因为错误信息常在开头，最新输出在末尾。

**Exit Code 语义注释**: 为 grep(1=no matches)、diff(1=files differ)、curl(28=timeout) 等添加说明，防止 LLM 误判为错误。

### 5.7 前台超时上限 (Foreground Timeout Cap)

```python
FOREGROUND_MAX_TIMEOUT = 600  # 前台命令最大 600 秒
```

超过此值的 timeout 请求直接被拒绝，引导模型使用 `background=true`。

## 6. 后台进程管理 — ProcessRegistry

**文件**: `tools/process_registry.py` (1172 lines)

### 6.1 ProcessSession 数据类

```python
@dataclass
class ProcessSession:
    id: str               # "proc_xxxxxxxxxxxx" (UUID-based)
    command: str           # 原始命令
    task_id: str           # 任务隔离键
    session_key: str       # Gateway 会话键
    pid: int               # OS PID 或 sandbox PID
    process: Popen         # 本地 Popen 句柄 (local only)
    env_ref: Any           # 环境对象引用 (非 local)
    pid_scope: str         # "host" | "sandbox"
    output_buffer: str     # 滚动输出 (200KB 窗口)
    detached: bool         # 崩溃恢复后无管道
    _pty: Any              # ptyprocess 句柄 (PTY mode)
    notify_on_complete: bool     # 完成时通知 agent
    watch_patterns: List[str]    # 输出模式匹配
```

### 6.2 两种 Spawn 模式

#### spawn_local() — 本地后台进程
```python
def spawn_local(self, command, cwd, task_id, env_vars, use_pty=False):
    if use_pty:
        # PTY mode: ptyprocess.PtyProcess.spawn()
        # _pty_reader_loop 读取 PTY 输出
    else:
        # Pipe mode: subprocess.Popen()
        # _reader_loop 读取 stdout pipe
```

#### spawn_via_env() — 沙箱后台进程
```python
def spawn_via_env(self, env, command, cwd, task_id):
    # 在沙箱中启动 nohup 进程:
    # nohup bash -lc <command> > /tmp/hermes_bg_<session_id>.log 2>&1
    # 写 PID 到 .pid 文件，退出码到 .exit 文件
    # _env_poller_loop 每 2 秒轮询 log 文件和进程状态
```

### 6.3 三种 Reader 线程

| 线程 | 适用场景 | 机制 |
|------|---------|------|
| `_reader_loop` | 本地 Popen (pipe mode) | 直接读 `stdout.read(4096)` |
| `_pty_reader_loop` | 本地 PTY mode | 读 `pty.read(4096)` |
| `_env_poller_loop` | 非本地沙箱 | 每 2s `cat log_file` + `kill -0 PID` |

### 6.4 Watch Patterns — 输出模式匹配

```python
# 速率限制参数
WATCH_MAX_PER_WINDOW = 8        # 每窗口最大通知数
WATCH_WINDOW_SECONDS = 10       # 窗口长度
WATCH_OVERLOAD_KILL_SECONDS = 45 # 持续过载后永久禁用
```

模型可设置 `watch_patterns=["ERROR", "FAIL"]`，当后台进程输出匹配时自动触发通知。防止刷屏的三层保护：
1. 每 10 秒窗口最多 8 条通知
2. 超过限制时抑制并计数
3. 持续 45 秒过载后永久禁用该进程的 watch

### 6.5 Crash Recovery (崩溃恢复)

```python
CHECKPOINT_PATH = get_hermes_home() / "processes.json"

def _write_checkpoint(self):
    """Atomic JSON write of all running process metadata."""
    # 每次 spawn/move/kill 后更新

def recover_from_checkpoint(self) -> int:
    """Gateway startup: probe host PIDs from checkpoint file."""
    # 只恢复 pid_scope="host" 的进程
    # sandbox-backed 进程跳过（sandbox handle 已丢失）
    # 恢复为 detached=True（无管道，但可查状态/kill）
```

### 6.6 Completion Queue (完成通知队列)

```python
self.completion_queue: queue.Queue = queue.Queue()
# 两种事件类型:
# 1. type="completion" — 进程退出通知
# 2. type="watch_match" — 输出模式匹配
# 3. type="watch_disabled" — 过载后禁用通知
```

CLI `process_loop` 和 Gateway 在每次 agent turn 后 drain 此队列。

### 6.7 Process Tool Schema

```python
PROCESS_SCHEMA = {
    "name": "process",
    "parameters": {
        "action": ["list", "poll", "log", "wait", "kill", "write", "submit", "close"],
        "session_id": str,
        "data": str,      # write/submit 的 stdin 数据
        "timeout": int,   # wait 的超时
        "offset": int,    # log 分页
        "limit": int,     # log 行数限制
    }
}
```

## 7. PTY 支持 (伪终端支持)

### 7.1 PTY Mode 使用场景

- 交互式 CLI 工具: Codex, Claude Code, Python REPL
- `pty=true` 参数仅在 `local` 后端生效
- 使用 `ptyprocess.PtyProcess` (Unix) 或 `winpty.PtyProcess` (Windows)

### 7.2 PTY vs Pipe Mode

| 特性 | Pipe Mode | PTY Mode |
|------|-----------|----------|
| stdin | `process.stdin.write()` / `close()` | `pty.write()` / `sendeof()` |
| stdout | `process.stdout.read()` | `pty.read()` |
| 终端尺寸 | N/A | 30×120 |
| Shell | `bash -lic` | `bash -lic` |
| 回退 | 默认模式 | 降级到 pipe (如 ptyprocess 未安装) |

### 7.3 PTY 自动禁用

```python
def _command_requires_pipe_stdin(command):
    """gh auth login --with-token expects piped stdin + EOF"""
    return command.startswith("gh auth login") and "--with-token" in command
```

## 8. FileSyncManager — 文件同步管理器

**文件**: `tools/environments/file_sync.py` (168 lines)

### 8.1 设计目的

远程后端（SSH/Modal/Daytona）需要将宿主机的凭据、技能、缓存文件同步到沙箱。

### 8.2 同步机制

```python
class FileSyncManager:
    _SYNC_INTERVAL_SECONDS = 5.0    # 最小同步间隔
    
    def iter_sync_files(self):
        """Yield (host_path, container_path) for all syncable files."""
        # credentials, skills, cache files
        # container_base 重映射路径
    
    def sync(self, upload_fn, delete_fn, bulk_upload_fn=None):
        """Transactional sync with rollback on failure."""
        # mtime + size 变更检测
        # 增量上传 (只传变化的文件)
        # 失败时回滚
```

### 8.3 各后端的上传实现

| 后端 | 单文件上传 | 批量上传 | 删除 |
|------|-----------|---------|------|
| SSH | `scp` | `_ssh_bulk_upload` | `ssh rm` |
| Modal | sandbox.exec(`cat > file`) | tar pipeline | sandbox.exec(`rm`) |
| Daytona | SDK file write | — | SDK file remove |

## 9. 设计模式总结 (Design Patterns)

| 模式 | 应用位置 | 说明 |
|------|---------|------|
| **Strategy Pattern** (策略模式) | Backend 选择 | `TERMINAL_ENV` 决定使用哪个后端 |
| **Factory Method** (工厂方法) | `_create_environment()` | 根据 env_type 创建对应实例 |
| **Template Method** (模板方法) | `BaseEnvironment.execute()` | 骨架流程固定，子类覆写 `_run_bash()` |
| **Adapter Pattern** (适配器) | `_ThreadedProcessHandle` | 将 SDK 阻塞 API 适配为 Popen 接口 |
| **Singleton** (单例) | `process_registry` | 模块级别单例，全局唯一 |
| **Observer Pattern** (观察者) | Watch patterns + completion_queue | 后台进程事件通知 |
| **Double-Checked Locking** (双重检查锁) | 环境创建 | 避免并发重复创建沙箱 |
| **Protocol / Duck Typing** | `ProcessHandle` | 行为协议，不要求继承 |
| **Two-Phase Cleanup** (两阶段清理) | `_cleanup_inactive_envs()` | 锁内收集 → 锁外清理 |

## 10. 关键数值参数 (Key Constants)

| 常量 | 值 | 文件 | 说明 |
|------|-----|------|------|
| `MAX_OUTPUT_CHARS` | 200,000 | process_registry.py | 输出滚动缓冲区大小 |
| `FINISHED_TTL_SECONDS` | 1,800 | process_registry.py | 完成进程保留 30 分钟 |
| `MAX_PROCESSES` | 64 | process_registry.py | 最大并发跟踪进程数 |
| `FOREGROUND_MAX_TIMEOUT` | 600 | terminal_tool.py | 前台命令最大超时 |
| `MAX_OUTPUT_CHARS` (tool) | 50,000 | terminal_tool.py | 工具返回给 LLM 的最大输出 |
| `_SYNC_INTERVAL_SECONDS` | 5.0 | file_sync.py | 文件同步最小间隔 |
| `ControlPersist` | 300 | ssh.py | SSH 连接保持 5 分钟 |
| `Daytona disk cap` | 10,240 MB | daytona.py | Daytona 磁盘硬限制 |
| `WATCH_MAX_PER_WINDOW` | 8 | process_registry.py | Watch 通知速率窗口 |
| `WATCH_OVERLOAD_KILL_SECONDS` | 45 | process_registry.py | Watch 过载禁用阈值 |

## 11. 安全层 (Security Layers)

### 11.1 命令安全

- **Tirith 策略引擎**: 外部策略规则检查
- **危险命令检测**: 内置模式匹配 (`rm -rf /` 等)
- **用户审批流**: `approval_required` 状态 → 等待用户确认
- **Smart Approval**: AI 自动判断安全性

### 11.2 Workdir 注入防护

```python
def _validate_workdir(workdir):
    """Block shell injection via malicious workdir paths."""
    # 检查 shell 元字符: $, `, ;, |, &, (, ) 等
```

### 11.3 输出安全

- `strip_ansi()`: 去除终端转义码，防止 LLM 复制到文件
- `redact_sensitive_text()`: 密钥脱敏（`env`/`printenv` 泄露防护）
- `_HERMES_PROVIDER_ENV_BLOCKLIST`: 源头屏蔽敏感环境变量

## 12. 线程安全 (Thread Safety)

### 12.1 锁层次

```
_env_lock (全局环境锁)
  └── _creation_locks_lock (创建锁的锁)
       └── _creation_locks[task_id] (每任务创建锁)

ProcessRegistry._lock (注册表锁)
  └── ProcessSession._lock (每会话锁)
```

### 12.2 并发访问场景

- **Executor 线程**: terminal_tool, process tool handlers
- **Gateway asyncio loop**: watcher tasks, session reset checks
- **Cleanup 线程**: sandbox reaping coordination
- **Reader 线程**: per-process output reading

## 13. Modal 后端选择逻辑 (Modal Backend Selection)

```python
# TERMINAL_MODAL_MODE: "auto" | "direct" | "managed"
#
# auto (默认):
#   1. 有直接 Modal 凭据 → direct
#   2. managed gateway 可用 → managed
#   3. 都没有 → 报错
#
# direct: 必须有 Modal SDK 凭据
# managed: 必须有 tool-gateway
```

`ManagedModalEnvironment` vs `ModalEnvironment`:
- **Direct**: 在进程内使用 Modal SDK，拥有完整的 sandbox 控制
- **Managed**: 通过 tool-gateway HTTP API 间接操作，gateway 拥有 sandbox 生命周期

## 14. 重试与错误处理 (Retry & Error Handling)

### 前台命令重试

```python
max_retries = 3
retry_count = 0
while retry_count <= max_retries:
    try:
        result = env.execute(command, timeout=effective_timeout)
    except Exception as e:
        if "timeout" in str(e).lower():
            return timeout_error  # 超时不重试
        wait_time = 2 ** retry_count  # 指数退避: 2, 4, 8 秒
        retry_count += 1
        time.sleep(wait_time)
```

### 沙箱环境创建失败

- `ImportError` → 优雅降级 (`status: "disabled"`)
- 其他异常 → 重试或报错

### 后台进程环境丢失

`_env_poller_loop` 捕获所有异常 → 将进程标记为 `exit_code=-1`，防止孤儿线程。
