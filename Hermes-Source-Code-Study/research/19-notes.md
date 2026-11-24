---
Task: T19-R
Status: DRAFT_DONE
Date: 2026-04-15
Chapter: 19
Title: 安全体系 (Security Architecture)
Keywords: Tirith 安全引擎, path traversal 防护, prompt injection 扫描, URL 安全, SSRF 防护, 审批工作流, 秘密脱敏, Skills Guard, MCP 安全
Sources:
  - tools/tirith_security.py (670 lines)
  - tools/approval.py (926 lines)
  - tools/skills_guard.py (928 lines)
  - tools/url_safety.py (97 lines)
  - tools/path_security.py (43 lines)
  - tools/credential_files.py (407 lines)
  - tools/env_passthrough.py (101 lines)
  - tools/mcp_tool.py (2273 lines) — security helpers section
  - tools/web_tools.py — SSRF pre-flight filtering
  - tools/vision_tools.py — SSRF redirect guard
  - agent/redact.py (181 lines)
  - gateway/platforms/base.py — _ssrf_redirect_guard
---

# Ch19 安全体系 (Security Architecture) — 研究笔记

## 1. 安全架构总览 (Security Architecture Overview)

Hermes Agent 采用**纵深防御 (Defense-in-Depth)** 架构，安全机制分布在多个层次：

```
┌─────────────────────────────────────────────────┐
│                   用户交互层                      │
│  Approval Workflow / Smart Approve / YOLO Mode  │
├─────────────────────────────────────────────────┤
│                  预执行扫描层                     │
│  Tirith Engine / Dangerous Pattern Detection    │
├─────────────────────────────────────────────────┤
│                  内容安全层                       │
│  Skills Guard / MCP Injection Scanner           │
├─────────────────────────────────────────────────┤
│                  网络安全层                       │
│  URL Safety (SSRF) / SSRF Redirect Guard        │
│  Website Policy / robots.txt                    │
├─────────────────────────────────────────────────┤
│                  路径安全层                       │
│  Path Traversal Prevention / Credential Files   │
├─────────────────────────────────────────────────┤
│                  输出安全层                       │
│  Secret Redaction / Env Passthrough Filtering   │
├─────────────────────────────────────────────────┤
│                  沙箱/隔离层                      │
│  Container backends / Env var stripping         │
└─────────────────────────────────────────────────┘
```

**设计原则 (Design Principles):**
- **Fail-closed (默认关闭):** 大多数安全检查在异常情况下默认拒绝（url_safety、path_security）
- **Fail-open 可配置:** Tirith 引擎支持 fail_open/fail_closed 配置，适应不同风险容忍度
- **分层验证 (Layered validation):** 同一请求经过多层独立检查
- **最小权限 (Least privilege):** MCP subprocess 环境变量过滤，sandbox 环境变量剥离

## 2. Tirith 安全引擎 (Tirith Security Engine)

**文件:** `tools/tirith_security.py` (670 lines)

### 2.1 架构设计

Tirith 是一个外部 Rust 二进制程序，作为**预执行安全扫描器 (pre-exec security scanner)** 运行。它以 subprocess 方式被调用，扫描 shell 命令中的内容级威胁。

```python
# 核心调用流程
result = subprocess.run(
    [tirith_path, "check", "--json", "--non-interactive",
     "--shell", "posix", "--", command],
    capture_output=True, text=True, timeout=timeout,
)
```

### 2.2 Exit Code 判决协议 (Verdict Protocol)

Exit code 是**判决的唯一权威来源 (single source of truth)**，JSON stdout 仅作为富化信息：

| Exit Code | Action | 含义 |
|-----------|--------|------|
| 0 | allow | 安全通过 |
| 1 | block | 阻止执行 |
| 2 | warn | 发出警告但允许 |
| 其他 | 取决于 fail_open | 未知状态 |

```python
# Exit code → action 映射
if exit_code == 0: action = "allow"
elif exit_code == 1: action = "block"
elif exit_code == 2: action = "warn"
else:
    # Unknown exit code — respect fail_open
    action = "allow" if fail_open else "block"
```

### 2.3 自动安装与供应链安全 (Auto-Install & Supply Chain)

Tirith 支持**自动安装 (auto-install)**，安装流程有严格的供应链验证：

1. **SHA-256 校验和验证** — 始终执行，比对 checksums.txt
2. **Cosign 签名验证** — 可选但推荐，验证 GitHub Actions workflow 签名
3. **失败标记机制 (Failure Marker)** — 24h TTL 的磁盘持久化标记，避免重复网络尝试

```python
# 安装验证层级
_COSIGN_IDENTITY_REGEXP = f"^https://github.com/{_REPO}/\\.github/workflows/release\\.yml@refs/tags/v"
_COSIGN_ISSUER = "https://token.actions.githubusercontent.com"
_MARKER_TTL = 86400  # 24小时
```

**后台安装设计:** 使用 daemon thread 后台下载，启动不阻塞 (startup never blocks)。`ensure_installed()` 安全地多次调用。

### 2.4 Fail-Open / Fail-Closed 策略

```python
# 配置层级: env var > config.yaml > default
defaults = {
    "tirith_enabled": True,
    "tirith_path": "tirith",
    "tirith_timeout": 5,       # 5秒超时
    "tirith_fail_open": True,  # 默认 fail-open
}
```

**Fail-open 覆盖场景:**
- OSError (FileNotFoundError, PermissionError)
- subprocess.TimeoutExpired
- 未知 exit code

## 3. 危险命令检测 (Dangerous Command Detection)

**文件:** `tools/approval.py` (926 lines)

### 3.1 模式匹配引擎

`DANGEROUS_PATTERNS` 列表包含 **38+ 个危险模式 (dangerous patterns)**，按类别分组：

**文件系统破坏 (Filesystem Destruction):**
- `rm -rf /`, `rm --recursive` — 递归删除
- `mkfs` — 格式化文件系统
- `dd if=` — 磁盘复制

**权限滥用 (Permission Abuse):**
- `chmod 777/666` — 全局可写权限
- `chown -R root` — 递归 root 所有权

**数据库破坏 (Database Destruction):**
- `DROP TABLE/DATABASE`
- `DELETE FROM` without WHERE
- `TRUNCATE TABLE`

**命令注入 (Command Injection):**
- `bash -c`, `python -e` — 脚本执行
- `curl | sh` — 远程脚本执行
- heredoc 执行: `python3 << 'EOF'`

**自我保护 (Self-Protection):**
- `hermes gateway stop/restart` — 网关生命周期保护
- `pkill hermes` — 自终止保护
- `kill $(pgrep ...)` — 通过扩展自终止

**Git 破坏性操作 (Git Destructive Operations):**
- `git reset --hard` — 丢弃未提交更改
- `git push --force` — 重写远程历史
- `git clean -f` — 删除未跟踪文件

### 3.2 命令规范化 (Command Normalization)

检测前执行**反混淆处理 (anti-obfuscation)**：

```python
def _normalize_command_for_detection(command: str) -> str:
    command = strip_ansi(command)           # 剥离 ANSI 转义序列
    command = command.replace('\x00', '')   # 删除 null 字节
    command = unicodedata.normalize('NFKC', command)  # Unicode 规范化（全角→半角）
    return command
```

关键对抗技术：
- **ANSI 转义注入:** 通过 ECMA-48 完整解析器 (`tools/ansi_strip`) 剥离
- **Null 字节注入:** 删除 `\x00`
- **Unicode 全角绕过:** NFKC 规范化（如全角 `ｒｍ` → 半角 `rm`）

### 3.3 敏感写入目标 (Sensitive Write Targets)

专门检测对系统关键路径的写入，包括 shell 展开形式：

```python
_SSH_SENSITIVE_PATH = r'(?:~|\$home|\$\{home\})/\.ssh(?:/|$)'
_HERMES_ENV_PATH = r'(?:~/\.hermes/|...|...)\.env\b'
_SENSITIVE_WRITE_TARGET = r'(?:/etc/|/dev/sd|{ssh}|{hermes_env})'
```

## 4. 审批工作流 (Approval Workflow)

### 4.1 三种审批模式

| 模式 | 说明 |
|------|------|
| `manual` | 用户手动审批（默认） |
| `smart` | 辅助 LLM 自动评估风险 |
| `off` | 跳过所有审批检查 |

### 4.2 审批决策选项

```
[o]nce    — 仅本次允许
[s]ession — 本会话允许（session-scoped）
[a]lways  — 永久加入白名单（persistent to config.yaml）
[d]eny    — 拒绝执行
```

### 4.3 Smart Approve (智能审批)

使用**辅助 LLM (auxiliary model)** 作为安全评审员：

```python
prompt = """You are a security reviewer for an AI coding agent...
Rules:
- APPROVE if clearly safe (benign script, dev tools, git ops...)
- DENY if genuinely dangerous (recursive delete, fork bombs...)
- ESCALATE if uncertain
Respond with exactly one word: APPROVE, DENY, or ESCALATE"""
```

**三元决策 (Ternary Decision):** APPROVE → auto-allow, DENY → auto-block, ESCALATE → 回退手动审批

### 4.4 会话状态管理 (Session State Management)

**线程安全设计:**
- `_lock = threading.Lock()` — 全局互斥锁
- `_pending: dict[str, dict]` — 待审批队列
- `_session_approved: dict[str, set]` — 每会话审批集
- `_session_yolo: set[str]` — YOLO 模式会话集
- `_permanent_approved: set` — 永久白名单

**Gateway 审批桥接:**
- `_ApprovalEntry` — 使用 `threading.Event` 实现同步阻塞
- FIFO 队列支持多个并发 subagent 阻塞等待
- `/approve all` 批量解锁所有待审批

### 4.5 审批键别名 (Approval Key Aliasing)

新旧键兼容机制，允许 config.yaml 中的历史 regex-derived 键继续生效：

```python
_PATTERN_KEY_ALIASES: dict[str, set[str]] = {}
# 同时接受 canonical (description) 和 legacy (regex-derived) keys
```

### 4.6 沙箱环境旁路 (Sandbox Bypass)

容器化环境自动跳过审批：
```python
if env_type in ("docker", "singularity", "modal", "daytona"):
    return {"approved": True, "message": None}
```

## 5. 路径遍历防护 (Path Traversal Defense)

**文件:** `tools/path_security.py` (43 lines) — 极简但关键

### 5.1 双层检查架构

```python
def validate_within_dir(path: Path, root: Path) -> Optional[str]:
    """确保 path 解析后位于 root 目录内"""
    resolved = path.resolve()          # 跟踪 symlink、规范化 ..
    root_resolved = root.resolve()
    resolved.relative_to(root_resolved)  # 不在内部会抛 ValueError

def has_traversal_component(path_str: str) -> bool:
    """快速检查是否包含 .. 遍历组件"""
    parts = Path(path_str).parts
    return ".." in parts
```

**两阶段验证:**
1. **快速检查 (Quick check):** `has_traversal_component()` — O(1) 拒绝明显攻击
2. **完整解析 (Full resolution):** `validate_within_dir()` — 使用 `Path.resolve()` 跟踪符号链接

### 5.2 使用场景

被以下模块引用：
- `skill_manager_tool.py` — 防止 skill 安装逃逸
- `skills_tool.py` — 防止 skill 执行路径逃逸
- `cronjob_tools.py` — 防止 cron job 路径逃逸
- `credential_files.py` — 防止凭证文件挂载逃逸

### 5.3 Credential Files 路径安全

```python
# credential_files.py — 拒绝绝对路径
if os.path.isabs(relative_path):
    return False

# 使用 path_security 验证
containment_error = validate_within_dir(host_path, hermes_home)
if containment_error:
    return False  # 阻止 ../../.ssh/id_rsa 类攻击
```

## 6. SSRF 防护 (Server-Side Request Forgery Prevention)

**文件:** `tools/url_safety.py` (97 lines)

### 6.1 Pre-Flight URL 验证

```python
def is_safe_url(url: str) -> bool:
    """DNS 解析后检查 IP 是否为私有/内部地址"""
    # 1. 解析 hostname
    # 2. 检查已知内部主机名 (metadata.google.internal)
    # 3. DNS 解析 → 检查每个返回 IP
    # 4. 任何失败 → fail-closed
```

**IP 阻止规则 (Blocked IP Categories):**
```python
def _is_blocked_ip(ip) -> bool:
    if ip.is_private: return True      # RFC 1918
    if ip.is_loopback: return True     # 127.0.0.0/8
    if ip.is_link_local: return True   # 169.254.0.0/16
    if ip.is_reserved: return True     # IANA 保留
    if ip.is_multicast: return True    # 224.0.0.0/4
    if ip.is_unspecified: return True  # 0.0.0.0
    if ip in _CGNAT_NETWORK: return True  # 100.64.0.0/10 (RFC 6598)
```

**特殊处理:**
- `_CGNAT_NETWORK = 100.64.0.0/10` — Python 的 `ipaddress.is_private` 不覆盖此范围
- `_BLOCKED_HOSTNAMES = {"metadata.google.internal", "metadata.goog"}` — 云元数据端点

### 6.2 Post-Redirect SSRF Guard

Pre-flight 检查可被 302 重定向绕过。解决方案：**httpx event hook 在每次重定向时重新验证**。

```python
async def _ssrf_redirect_guard(response):
    """Re-validate each redirect target"""
    if response.is_redirect and response.next_request:
        redirect_url = str(response.next_request.url)
        if not is_safe_url(redirect_url):
            raise ValueError(
                f"Blocked redirect to private/internal address: {redirect_url}"
            )
```

**部署位置:**
- `gateway/platforms/base.py` — 平台媒体下载
- `gateway/platforms/slack.py` — Slack 文件下载
- `tools/vision_tools.py` — 图片下载
- `tools/web_tools.py` — 使用 pre-flight 过滤（第三方 SDK 处理重定向）

### 6.3 已知限制 (Documented Limitations)

代码中**明确记录 (explicitly documented)** 的限制：

```python
# DNS rebinding (TOCTOU): attacker DNS server with TTL=0
# can return public IP for check, then private IP for connection.
# Fixing requires connection-level validation (e.g. Champion library
# or egress proxy like Stripe's Smokescreen).
```

### 6.4 Web Tools SSRF 流程

```python
# web_tools.py — 批量 URL 预过滤
safe_urls = []
ssrf_blocked = []
for url in urls:
    if not is_safe_url(url):
        ssrf_blocked.append({
            "url": url,
            "error": "Blocked: URL targets a private or internal network address"
        })
    else:
        safe_urls.append(url)
# 仅安全 URL 发送到后端
```

## 7. Prompt Injection 扫描 (Prompt Injection Scanning)

### 7.1 MCP Tool Description Scanner

**文件:** `tools/mcp_tool.py` — `_MCP_INJECTION_PATTERNS`

扫描 MCP 服务器返回的**工具描述文本 (tool descriptions)**，检测注入尝试：

```python
_MCP_INJECTION_PATTERNS = [
    (r"ignore\s+(all\s+)?previous\s+instructions", "prompt override attempt"),
    (r"you\s+are\s+now\s+a", "identity override attempt"),
    (r"your\s+new\s+(task|role|instructions?)\s+(is|are)", "task override attempt"),
    (r"system\s*:\s*", "system prompt injection attempt"),
    (r"<\s*(system|human|assistant)\s*>", "role tag injection attempt"),
    (r"do\s+not\s+(tell|inform|mention|reveal)", "concealment instruction"),
    (r"(curl|wget|fetch)\s+https?://", "network command in description"),
    (r"base64\.(b64decode|decodebytes)", "base64 decode reference"),
    (r"exec\s*\(|eval\s*\(", "code execution reference"),
    (r"import\s+(subprocess|os|shutil|socket)", "dangerous import reference"),
]
```

**设计决策:** WARNING-level only — 不阻止，仅记录警告。原因：误报会破坏合法 MCP 服务器。

### 7.2 Skills Guard 注入检测

**文件:** `tools/skills_guard.py` — THREAT_PATTERNS 中的 injection 类别

Skills Guard 对外部 skill 执行**更严格的静态分析**，injection 类模式包括：

| Pattern | Severity | 描述 |
|---------|----------|------|
| `ignore previous instructions` | critical | 指令覆盖 |
| `you are now` | high | 身份劫持 |
| `do not tell the user` | critical | 欺骗隐藏 |
| `system prompt override` | critical | 系统提示覆盖 |
| `DAN mode` | critical | 越狱尝试 |
| `developer mode enabled` | critical | 开发者模式越狱 |
| `act as if... no restrictions` | critical | 绕过限制 |
| `respond without filters` | critical | 移除安全过滤 |
| `hidden HTML div` | high | 不可见指令 |
| `HTML comment injection` | high | 注释中隐藏指令 |

### 7.3 不可见 Unicode 字符检测

```python
INVISIBLE_CHARS = {
    '\u200b',  # zero-width space
    '\u200c',  # zero-width non-joiner
    '\u200d',  # zero-width joiner
    '\u2060',  # word joiner
    '\ufeff',  # BOM
    '\u202e',  # RTL override (视觉文本方向欺骗)
    # ... 17 种不可见字符
}
```

## 8. Skills Guard — 技能安全扫描器 (Skills Security Scanner)

**文件:** `tools/skills_guard.py` (928 lines)

### 8.1 信任模型 (Trust Model)

```python
TRUSTED_REPOS = {"openai/skills", "anthropics/skills"}

INSTALL_POLICY = {
    #                  safe      caution    dangerous
    "builtin":       ("allow",  "allow",   "allow"),    # 内置技能，始终信任
    "trusted":       ("allow",  "allow",   "block"),    # 受信仓库
    "community":     ("allow",  "block",   "block"),    # 社区技能，严格限制
    "agent-created": ("allow",  "allow",   "ask"),      # Agent 创建，危险时询问
}
```

### 8.2 威胁模式分类

**484+ 个威胁模式**，按类别组织：

| 类别 (Category) | 示例 | 严重度 |
|-----------------|------|--------|
| **exfiltration** | curl + $API_KEY, base64 + env, SSH dir access, DNS exfil | critical/high |
| **injection** | ignore instructions, role hijack, DAN mode, hidden HTML | critical/high |
| **destructive** | rm -rf /, chmod 777, mkfs, dd, shutil.rmtree | critical |
| **persistence** | crontab, .bashrc, authorized_keys, systemd service | medium/critical |
| **network** | reverse shell (nc -l), ngrok, webhook.site | critical/high |
| **obfuscation** | base64 decode pipe, eval(), chr() building, string reversal | high/medium |
| **execution** | subprocess.run, os.system, child_process.exec | medium/high |
| **traversal** | ../../.., /etc/passwd, /proc/self | high/critical |
| **supply_chain** | curl pipe sh, unpinned pip install, git clone | critical/medium |
| **privilege_escalation** | sudo, setuid, NOPASSWD, allowed-tools field | high/critical |
| **credential_exposure** | hardcoded API keys, embedded private keys | critical |
| **mining** | xmrig, stratum+tcp, cryptonight | critical |

### 8.3 结构检查 (Structural Checks)

```python
MAX_FILE_COUNT = 50        # 技能不应有 50+ 文件
MAX_TOTAL_SIZE_KB = 1024   # 1MB 总大小上限
MAX_SINGLE_FILE_KB = 256   # 单个文件 256KB 上限
```

额外检查：
- **符号链接逃逸:** symlink 指向 skill 目录外部 → critical
- **二进制文件检测:** .exe, .dll, .so, .dylib 等 → critical
- **异常可执行权限:** 非脚本文件有 +x → medium

### 8.4 扫描结果与判决 (Scan Result & Verdict)

```python
def _determine_verdict(findings):
    if not findings: return "safe"
    if any(f.severity == "critical"): return "dangerous"
    if any(f.severity == "high"): return "caution"
    return "caution"
```

### 8.5 内容完整性 (Content Integrity)

```python
def content_hash(skill_path: Path) -> str:
    """SHA-256 hash of all files — integrity tracking"""
    h = hashlib.sha256()
    for f in sorted(skill_path.rglob("*")):
        if f.is_file():
            h.update(f.read_bytes())
    return f"sha256:{h.hexdigest()[:16]}"
```

## 9. 秘密脱敏 (Secret Redaction)

**文件:** `agent/redact.py` (181 lines)

### 9.1 多层脱敏策略

| 层级 | 模式 | 示例 |
|------|------|------|
| **API Key 前缀** | 30+ 已知前缀 | sk-*, ghp_*, AIza*, xox* |
| **环境变量赋值** | KEY=value | OPENAI_API_KEY=sk-xxx |
| **JSON 字段** | "apiKey": "value" | {"token": "abc123"} |
| **Authorization Header** | Bearer token | Authorization: Bearer sk-xxx |
| **Telegram Bot Token** | bot<digits>:<token> | bot123456789:AAF... |
| **Private Key Block** | PEM 格式 | -----BEGIN PRIVATE KEY----- |
| **数据库连接串** | protocol://user:pass@host | postgres://admin:secret@db |
| **E.164 电话号码** | +国际格式 | +1234****5678 |

### 9.2 Masking 策略

```python
def _mask_token(token: str) -> str:
    if len(token) < 18:
        return "***"               # 短 token 完全遮蔽
    return f"{token[:6]}...{token[-4:]}"  # 长 token 保留首尾（调试用）
```

### 9.3 防篡改设计

```python
# 导入时快照，运行时 env 修改无法禁用脱敏
_REDACT_ENABLED = os.getenv("HERMES_REDACT_SECRETS", "").lower() not in ("0", "false", "no", "off")
```

### 9.4 日志集成

```python
class RedactingFormatter(logging.Formatter):
    """自动脱敏所有日志消息"""
    def format(self, record: logging.LogRecord) -> str:
        original = super().format(record)
        return redact_sensitive_text(original)
```

## 10. MCP 安全上下文 (Security in MCP Context)

### 10.1 环境变量过滤

MCP stdio subprocess 只接收**最小安全环境**：

```python
_SAFE_ENV_KEYS = frozenset({
    "PATH", "HOME", "USER", "LANG", "LC_ALL", "TERM", "SHELL", "TMPDIR",
})

def _build_safe_env(user_env):
    env = {}
    for key, value in os.environ.items():
        if key in _SAFE_ENV_KEYS or key.startswith("XDG_"):
            env[key] = value
    if user_env:
        env.update(user_env)   # 用户显式配置的变量
    return env
```

### 10.2 错误消息凭证剥离

```python
_CREDENTIAL_PATTERN = re.compile(
    r"(?:ghp_...|sk-...|Bearer\s+\S+|token=...|key=...|password=...|secret=...)",
    re.IGNORECASE,
)

def _sanitize_error(text: str) -> str:
    return _CREDENTIAL_PATTERN.sub("[REDACTED]", text)
```

### 10.3 Sampling 安全控制

MCP server 可请求 LLM 补全（sampling/createMessage），安全机制包括：
- **速率限制 (Rate Limiting):** 滑动窗口，默认 10 RPM
- **Token 上限 (Token Cap):** 默认 4096 max_tokens
- **Tool Loop 限制:** 默认 5 轮，防止无限工具循环
- **模型白名单:** `allowed_models` 限制可用模型

## 11. 环境变量通行 (Environment Variable Passthrough)

**文件:** `tools/env_passthrough.py` (101 lines)

**问题:** 沙箱默认剥离所有秘密环境变量，但 skill 可能需要特定 API key。

**解决方案:** Session-scoped 白名单，使用 ContextVar 防止跨会话泄露：

```python
_allowed_env_vars_var: ContextVar[set[str]] = ContextVar("_allowed_env_vars")

# 两个来源:
# 1. Skill 声明: required_environment_variables
# 2. 用户配置: terminal.env_passthrough in config.yaml
```

## 12. 设计模式总结 (Design Patterns Summary)

### 12.1 Fail-Closed by Default (默认安全)
- `url_safety.is_safe_url()` — DNS 失败 → 阻止
- `path_security.validate_within_dir()` — 解析失败 → 拒绝
- `redact.py` — 导入时锁定配置

### 12.2 Defense in Depth (纵深防御)
- SSRF: pre-flight IP check + redirect hook + website policy
- Command safety: pattern detection + Tirith scan + approval prompt + smart approve
- Skill security: trust model + structural check + pattern scan + verdict + install policy

### 12.3 Separation of Concerns (关注分离)
- `tirith_security.py` — 外部引擎包装
- `approval.py` — 审批状态与 UI
- `skills_guard.py` — 静态分析引擎
- `url_safety.py` — 网络安全独立模块
- `path_security.py` — 路径安全独立模块

### 12.4 ContextVar for Session Isolation (会话隔离)
- `approval.py` — `_approval_session_key` ContextVar
- `env_passthrough.py` — `_allowed_env_vars_var` ContextVar
- `credential_files.py` — `_registered_files_var` ContextVar

### 12.5 Transparent Security (透明安全)
- 所有限制都有**人类可读的描述 (human-readable descriptions)**
- Tirith findings 包含结构化 JSON 详情
- Skills Guard 输出格式化报告 (`format_scan_report`)

### 12.6 Graceful Degradation (优雅降级)
- Tirith 不可用 → fail-open (可配置)
- Smart approve LLM 失败 → escalate to manual
- MCP SDK 不可用 → 禁用 MCP（no-op）
- Cosign 不可用 → 仅 SHA-256 验证

## 13. 安全边界图 (Security Boundary Diagram)

```
User Input
    │
    ▼
┌──────────────┐
│ detect_      │ ← DANGEROUS_PATTERNS (38+ regex)
│ dangerous_   │ ← Unicode normalization + ANSI strip
│ command()    │
└──────┬───────┘
       │ is_dangerous?
       ▼
┌──────────────┐
│ check_       │ ← Tirith binary (exit code verdict)
│ command_     │ ← Content-level threats
│ security()   │ ← Homograph URLs, terminal injection
└──────┬───────┘
       │ action = allow/warn/block
       ▼
┌──────────────┐
│ Approval     │ ← manual / smart / off / YOLO
│ Workflow     │ ← Session + Permanent allowlist
│              │ ← Gateway blocking queue
└──────┬───────┘
       │ approved?
       ▼
┌──────────────┐
│ URL Safety   │ ← Pre-flight: DNS resolve → IP block check
│ + SSRF Guard │ ← Post-redirect: httpx event hooks
│              │ ← Website policy + robots.txt
└──────┬───────┘
       │ safe?
       ▼
┌──────────────┐
│ Path Safety  │ ← resolve() + relative_to()
│              │ ← has_traversal_component()
└──────┬───────┘
       │ contained?
       ▼
┌──────────────┐
│ Execution    │ ← Env var filtering
│ Sandbox      │ ← Secret redaction on output
│              │ ← Container isolation (docker/modal)
└──────────────┘
```

## 14. 关键文件速查表 (File Quick Reference)

| 文件 | 行数 | 职责 |
|------|------|------|
| `tools/tirith_security.py` | 670 | Tirith 二进制包装、自动安装、供应链验证 |
| `tools/approval.py` | 926 | 危险模式检测、审批状态、Smart Approve、Gateway 桥接 |
| `tools/skills_guard.py` | 928 | Skill 静态分析、484+ 威胁模式、信任模型、安装策略 |
| `tools/url_safety.py` | 97 | SSRF pre-flight 检查、IP 分类阻止 |
| `tools/path_security.py` | 43 | 路径遍历防护、符号链接跟踪 |
| `tools/credential_files.py` | 407 | 凭证文件挂载安全、路径约束 |
| `tools/env_passthrough.py` | 101 | 环境变量白名单、ContextVar 隔离 |
| `tools/mcp_tool.py` | 2273 | MCP 安全助手、注入扫描、env 过滤、sampling 限制 |
| `agent/redact.py` | 181 | 30+ 前缀的秘密脱敏、日志集成 |
| `tools/website_policy.py` | — | robots.txt 合规、网站访问策略 |

## 15. 待深入主题 (Topics for Further Investigation)

- Tirith 引擎内部实现（Rust 源码，独立仓库 sheeki03/tirith）
- 具体 Tirith 规则集（homograph URL 检测、terminal injection 规则）
- Website policy 策略配置与 robots.txt 集成细节
- Gateway 层面的认证与授权机制
- 沙箱后端（Docker/Modal/SSH）的具体隔离实现
- Smart Approve 的误报/漏报率分析
- TOCTOU DNS rebinding 的实际影响评估
