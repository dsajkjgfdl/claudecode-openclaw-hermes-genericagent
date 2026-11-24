---
Task: T24-R
Chapter: 24
Title: "日志、用量统计与可观测性"
Status: DRAFT_DONE
Date: 2026-04-15
Keywords:
  - 结构化日志 (structured logging)
  - usage insights (用量洞察)
  - rate limit 追踪 (rate limit tracking)
  - trajectory 保存与回放 (trajectory save & replay)
  - token 用量统计 (token usage accounting)
  - 成本估算 (cost estimation)
  - 密钥脱敏 (secret redaction)
Sources:
  - hermes_logging.py (390 lines)
  - agent/redact.py (181 lines)
  - agent/usage_pricing.py (613 lines)
  - agent/rate_limit_tracker.py (246 lines)
  - agent/insights.py (789 lines)
  - agent/trajectory.py (56 lines)
  - trajectory_compressor.py (1462 lines)
  - run_agent.py (lines 780–3050)
---

# Ch24 日志、用量统计与可观测性

## 1. 总体架构 (Overall Architecture)

Hermes Agent 的可观测性子系统由六个独立但协作的模块构成，覆盖了从底层结构化日志到上层分析洞察的完整链路：

```
┌───────────────┐  ┌───────────────┐  ┌───────────────┐
│ hermes_logging│  │ agent/redact  │  │ agent/        │
│ .py           │  │ .py           │  │ usage_pricing │
│ (日志基础设施) │  │ (密钥脱敏)     │  │ .py (定价引擎) │
└───────┬───────┘  └───────┬───────┘  └───────┬───────┘
        │                  │                  │
        └──────────┬───────┘                  │
                   │                          │
        ┌──────────▼──────────┐    ┌──────────▼──────────┐
        │ run_agent.py        │    │ agent/insights.py   │
        │ (运行时集成/累计)    │    │ (分析报告引擎)       │
        └──────────┬──────────┘    └─────────────────────┘
                   │
        ┌──────────▼──────────┐    ┌──────────────────────┐
        │ agent/trajectory.py │    │ trajectory_compressor │
        │ (轨迹保存)          │    │ .py (轨迹压缩)        │
        └─────────────────────┘    └──────────────────────┘
                                   ┌──────────────────────┐
                                   │ agent/rate_limit_     │
                                   │ tracker.py            │
                                   │ (速率限制追踪)         │
                                   └──────────────────────┘
```

### 设计原则 (Design Principles)

1. **安全优先 (Safety First)**: 所有日志经过 `RedactingFormatter` 脱敏，密钥永远不会落盘
2. **永不阻断 (Never Block the Loop)**: 可观测性代码全部用 `try/except` 保护，失败静默降级
3. **幂等初始化 (Idempotent Setup)**: 日志和追踪器初始化可多次调用，无副作用
4. **关注点分离 (Separation of Concerns)**: 每个模块独立可测，通过 `run_agent.py` 集成

---

## 2. 结构化日志系统 (Structured Logging System)

**核心文件**: `hermes_logging.py` (390 行)

### 2.1 日志文件布局 (Log File Layout)

所有日志文件存储在 `~/.hermes/logs/` 目录下（profile-aware），按职责分为三个文件：

| 文件名 | 级别 | 用途 |
|--------|------|------|
| `agent.log` | INFO+ | 主活动日志（catch-all），记录所有 agent/tool/session 活动 |
| `errors.log` | WARNING+ | 仅错误和警告，便于快速排查 (quick triage) |
| `gateway.log` | INFO+ | 仅 gateway 模式生效，仅接收 `gateway.*` 命名空间的日志 |

### 2.2 日志格式 (Log Format)

```python
# 标准格式
_LOG_FORMAT = "%(asctime)s %(levelname)s%(session_tag)s %(name)s: %(message)s"
# Verbose 格式
_LOG_FORMAT_VERBOSE = "%(asctime)s - %(name)s - %(levelname)s%(session_tag)s - %(message)s"
```

关键设计点：`%(session_tag)s` 字段是通过 **LogRecord 工厂替换** 注入的，而不是通过 Filter。这保证了所有日志记录（包括第三方库的）都可以安全地使用此格式字段。

### 2.3 会话上下文关联 (Session Context Correlation)

```python
# 线程本地存储
_session_context = threading.local()

def set_session_context(session_id: str) -> None:
    _session_context.session_id = session_id  # 所有后续日志包含 [session_id]

def clear_session_context() -> None:
    _session_context.session_id = None
```

**Record Factory 注入机制**：

```python
def _install_session_record_factory() -> None:
    current_factory = logging.getLogRecordFactory()
    if getattr(current_factory, "_hermes_session_injector", False):
        return  # 幂等保护
    
    def _session_record_factory(*args, **kwargs):
        record = current_factory(*args, **kwargs)
        sid = getattr(_session_context, "session_id", None)
        record.session_tag = f" [{sid}]" if sid else ""
        return record
    
    _session_record_factory._hermes_session_injector = True
    logging.setLogRecordFactory(_session_record_factory)

# 在 import 时立即安装！
_install_session_record_factory()
```

**设计决策**：使用 LogRecord factory 而非 Handler-level Filter 的原因是：factory 运行在每个 LogRecord 上，包括从子 logger 传播上来的和第三方 handler 处理的。这避免了 `%(session_tag)s` 因缺失而导致的 KeyError。

### 2.4 日志轮转 (Log Rotation)

使用自定义 `_ManagedRotatingFileHandler`，继承自 `RotatingFileHandler`：

- **默认配置**: 5MB 文件大小上限，保留 3 个备份 (configurable via `config.yaml`)
- **NixOS 兼容**: 在 managed 模式下（NixOS setgid 目录），自动 chmod 0o660 确保 gateway 用户和交互用户共享日志文件
- **配置读取**: 从 `config.yaml` 的 `logging.level`、`logging.max_size_mb`、`logging.backup_count` 读取

### 2.5 组件过滤 (Component Filtering)

```python
COMPONENT_PREFIXES = {
    "gateway": ("gateway",),
    "agent": ("agent", "run_agent", "model_tools", "batch_runner"),
    "tools": ("tools",),
    "cli": ("hermes_cli", "cli"),
    "cron": ("cron",),
}
```

`_ComponentFilter` 使用 logger 名称前缀匹配，将 gateway 日志路由到专用文件而不污染 agent.log。

### 2.6 Verbose 模式 (Verbose Mode)

```python
def setup_verbose_logging() -> None:
    # 添加 StreamHandler 到 root logger
    # 降低到 DEBUG 级别
    # 但保持第三方库在 WARNING 级别抑制噪音
```

在 `AIAgent.__init__` 中通过 `self.verbose_logging` 标志触发。第三方噪音源（openai, httpx, httpcore, asyncio, hpack, grpc, modal, urllib3 等 13 个）始终被抑制到 WARNING 级别。

### 2.7 Quiet 模式与控制台抑制

```python
# run_agent.py:830-840
if self.quiet_mode:
    for quiet_logger in [
        'tools', 'run_agent', 'trajectory_compressor',
        # ... 更多
    ]:
        # 抑制控制台噪音，但文件日志 (agent.log, errors.log) 仍然捕获
```

---

## 3. 密钥脱敏 (Secret Redaction)

**核心文件**: `agent/redact.py` (181 行)

### 3.1 启用机制与安全快照

```python
# 导入时快照环境变量，防止运行时 LLM 生成 export 命令来禁用脱敏
_REDACT_ENABLED = os.getenv("HERMES_REDACT_SECRETS", "").lower() not in ("0", "false", "no", "off")
```

**关键安全设计**：默认开启，只有显式 opt-out 才能关闭。且在 import 时冻结，运行时无法通过环境变量修改来绕过。

### 3.2 识别模式分类 (Detection Pattern Categories)

| 类别 | 示例模式 | 数量 |
|------|----------|------|
| **API Key 前缀** | `sk-*`, `ghp_*`, `gho_*`, `AIza*`, `AKIA*` | 30+ 种 |
| **环境变量赋值** | `OPENAI_API_KEY=xxx` | 通用正则 |
| **JSON 字段** | `"apiKey": "xxx"`, `"token": "xxx"` | 12+ 字段名 |
| **Authorization Header** | `Authorization: Bearer xxx` | 1 |
| **Telegram Bot Token** | `bot123456:ABCdef...` | 1 |
| **私钥块** | `-----BEGIN PRIVATE KEY-----` | 1 |
| **数据库连接串** | `postgres://user:PASSWORD@host` | 6 种协议 |
| **E.164 电话号码** | `+1234567890` | 1 |

### 3.3 脱敏策略 (Masking Strategy)

```python
def _mask_token(token: str) -> str:
    if len(token) < 18:
        return "***"                    # 短 token 完全隐藏
    return f"{token[:6]}...{token[-4:]}"  # 长 token 保留首6末4，便于调试
```

### 3.4 RedactingFormatter — 日志层集成

```python
class RedactingFormatter(logging.Formatter):
    def format(self, record: logging.LogRecord) -> str:
        original = super().format(record)
        return redact_sensitive_text(original)  # 每条日志自动脱敏
```

所有文件 handler 都使用 `RedactingFormatter`，确保密钥永不写入磁盘。

---

## 4. Token 用量追踪 (Token Usage Tracking)

**核心文件**: `agent/usage_pricing.py` + `run_agent.py`

### 4.1 CanonicalUsage — 统一用量表示

```python
@dataclass(frozen=True)
class CanonicalUsage:
    input_tokens: int = 0
    output_tokens: int = 0
    cache_read_tokens: int = 0
    cache_write_tokens: int = 0
    reasoning_tokens: int = 0
    request_count: int = 1
    raw_usage: Optional[dict[str, Any]] = None

    @property
    def prompt_tokens(self) -> int:
        return self.input_tokens + self.cache_read_tokens + self.cache_write_tokens

    @property
    def total_tokens(self) -> int:
        return self.prompt_tokens + self.output_tokens
```

### 4.2 normalize_usage — 多 API 格式归一化

`normalize_usage()` 处理三种不同的 API 响应格式：

| API 模式 | 输入字段 | 输出字段 | 缓存字段 |
|----------|----------|----------|----------|
| **Anthropic** | `input_tokens` | `output_tokens` | `cache_read_input_tokens`, `cache_creation_input_tokens` |
| **Codex Responses** | `input_tokens`(含缓存) | `output_tokens` | `input_tokens_details.cached_tokens` |
| **OpenAI Chat** | `prompt_tokens`(含缓存) | `completion_tokens` | `prompt_tokens_details.cached_tokens` |

关键实现细节：在 Codex 和 OpenAI 模式中，`input_tokens`/`prompt_tokens` **包含**缓存 token，需要减去 `cache_read_tokens` 和 `cache_write_tokens` 得到纯输入 token 数：

```python
input_tokens = max(0, input_total - cache_read_tokens - cache_write_tokens)
```

Reasoning tokens 通过 `output_tokens_details.reasoning_tokens` 独立提取，适用于 o1/o3 等推理模型。

### 4.3 Session 级别累计 (Session-Level Accumulation)

在 `run_agent.py` 的 `run_conversation()` 方法中初始化，每次 API 调用后累加：

```python
# run_agent.py:1445-1457
self.session_prompt_tokens = 0
self.session_completion_tokens = 0
self.session_total_tokens = 0
self.session_api_calls = 0
self.session_input_tokens = 0
self.session_output_tokens = 0
self.session_cache_read_tokens = 0
self.session_cache_write_tokens = 0
self.session_reasoning_tokens = 0
self.session_estimated_cost_usd = 0.0
self.session_cost_status = "unknown"
self.session_cost_source = "none"
```

### 4.4 _usage_summary_for_api_request_hook — 插件接口

```python
def _usage_summary_for_api_request_hook(self, response) -> Optional[Dict[str, Any]]:
    """Token buckets for post_api_request plugins (no raw response object)."""
    raw_usage = getattr(response, "usage", None)
    cu = normalize_usage(raw_usage, provider=self.provider, api_mode=self.api_mode)
    summary = asdict(cu)
    summary.pop("raw_usage", None)
    summary["prompt_tokens"] = cu.prompt_tokens
    summary["total_tokens"] = cu.total_tokens
    return summary
```

---

## 5. 成本估算引擎 (Cost Estimation Engine)

### 5.1 计费路由 (Billing Route Resolution)

`resolve_billing_route()` 将 model + provider + base_url 映射到计费模式：

| 模式 | 描述 | 示例 |
|------|------|------|
| `subscription_included` | 订阅包含，成本为 0 | OpenAI Codex |
| `official_models_api` | 从 API 获取定价 | OpenRouter |
| `official_docs_snapshot` | 本地硬编码定价 | Anthropic, OpenAI 直连 |
| `unknown` | 无法确定 | Custom/local |

### 5.2 定价数据源层级 (Pricing Source Hierarchy)

1. **Official Docs Snapshot** — 硬编码在 `_OFFICIAL_DOCS_PRICING` dict 中，覆盖 Claude (Opus/Sonnet/Haiku)、GPT-4o/4.1/mini、DeepSeek、Gemini 等主流模型
2. **Provider Models API** — 从 OpenRouter `/models` API 动态获取
3. **Endpoint Model Metadata** — 从自定义 endpoint 查询

### 5.3 成本计算 (Cost Calculation)

```python
def estimate_usage_cost(model_name, usage, *, provider, base_url, api_key) -> CostResult:
    # 使用 Decimal 精确计算，避免浮点误差
    amount += Decimal(usage.input_tokens) * entry.input_cost_per_million / _ONE_MILLION
    amount += Decimal(usage.output_tokens) * entry.output_cost_per_million / _ONE_MILLION
    amount += Decimal(usage.cache_read_tokens) * entry.cache_read_cost_per_million / _ONE_MILLION
    amount += Decimal(usage.cache_write_tokens) * entry.cache_write_cost_per_million / _ONE_MILLION
    amount += Decimal(usage.request_count) * entry.request_cost  # 有些模型按请求计费
```

**CostResult** 包含：
- `amount_usd: Optional[Decimal]` — 美元金额
- `status: CostStatus` — "actual" | "estimated" | "included" | "unknown"
- `source: CostSource` — 定价来源
- `label: str` — 格式化显示，如 "~$0.05" 或 "included"
- `notes: tuple[str, ...]` — 附加说明

### 5.4 格式化工具 (Formatting Utilities)

```python
def format_token_count_compact(value: int) -> str:
    # 1234 -> "1.2K", 1234567 -> "1.2M", 1234567890 -> "1.2B"

def format_duration_compact(seconds: float) -> str:
    # 45 -> "45s", 150 -> "2m", 7260 -> "2h 1m", 86400 -> "1.0d"
```

---

## 6. 速率限制追踪 (Rate Limit Tracking)

**核心文件**: `agent/rate_limit_tracker.py` (246 行)

### 6.1 数据模型

```python
@dataclass
class RateLimitBucket:
    limit: int = 0         # 上限
    remaining: int = 0     # 剩余
    reset_seconds: float   # 重置剩余秒数
    captured_at: float     # 捕获时间戳

    @property
    def used(self) -> int:         # limit - remaining
    @property
    def usage_pct(self) -> float:  # (used / limit) * 100
    @property
    def remaining_seconds_now(self) -> float:  # 动态计算实际剩余时间

@dataclass
class RateLimitState:
    requests_min: RateLimitBucket   # RPM
    requests_hour: RateLimitBucket  # RPH
    tokens_min: RateLimitBucket     # TPM
    tokens_hour: RateLimitBucket    # TPH
    captured_at: float
    provider: str
```

### 6.2 HTTP Header Schema (12 个头部)

解析 `x-ratelimit-*` 系列 header（Nous Portal / OpenRouter / OpenAI 通用格式）：

```
x-ratelimit-limit-requests          RPM cap
x-ratelimit-limit-requests-1h       RPH cap
x-ratelimit-limit-tokens            TPM cap
x-ratelimit-limit-tokens-1h         TPH cap
x-ratelimit-remaining-requests      requests left (minute)
x-ratelimit-remaining-requests-1h   requests left (hour)
x-ratelimit-remaining-tokens        tokens left (minute)
x-ratelimit-remaining-tokens-1h     tokens left (hour)
x-ratelimit-reset-requests          seconds until reset (minute)
x-ratelimit-reset-requests-1h       seconds until reset (hour)
x-ratelimit-reset-tokens            seconds until reset (minute)
x-ratelimit-reset-tokens-1h         seconds until reset (hour)
```

### 6.3 防御性集成 (Defensive Integration)

```python
# run_agent.py:2975-2992
def _capture_rate_limits(self, http_response: Any) -> None:
    if http_response is None:
        return
    headers = getattr(http_response, "headers", None)
    if not headers:
        return
    try:
        from agent.rate_limit_tracker import parse_rate_limit_headers
        state = parse_rate_limit_headers(headers, provider=self.provider)
        if state is not None:
            self._rate_limit_state = state
    except Exception:
        pass  # 永远不让 header 解析阻断 agent 循环
```

### 6.4 显示格式 (Display Formats)

两种格式化输出：
- **详细显示** (`format_rate_limit_display`): 带 ASCII 进度条和警告
  ```
  Provider Rate Limits (captured 5s ago):
    Requests/min  [████████████░░░░░░░░] 60.0%  600/1000 used  (400 left, resets in 32s)
    ⚠ tokens/hr at 85% — resets in 58m 57s
  ```
- **紧凑显示** (`format_rate_limit_compact`): 状态栏单行
  ```
  RPM: 400/1000 | RPH: 8.0K/10.0K (resets 58m) | TPM: 33.6K/50.0K
  ```

---

## 7. 活动诊断 (Activity Diagnostics)

### 7.1 Activity Tracking

```python
# run_agent.py:808-815
self._last_activity_ts: float = time.time()
self._last_activity_desc: str = "initializing"
self._current_tool: str | None = None
self._api_call_count: int = 0
```

`_touch_activity(desc)` 在每次 API 调用、工具执行、流式块接收时更新。

### 7.2 Activity Summary (Gateway 超时报告)

```python
def get_activity_summary(self) -> dict:
    return {
        "last_activity_ts": self._last_activity_ts,
        "last_activity_desc": self._last_activity_desc,
        "seconds_since_activity": round(elapsed, 1),
        "current_tool": self._current_tool,
        "api_call_count": self._api_call_count,
        "max_iterations": self.max_iterations,
        "budget_used": self.iteration_budget.used,
        "budget_max": self.iteration_budget.max_total,
    }
```

用于 gateway 超时 handler 报告 agent 被 kill 时的状态，以及 "still working" 周期性通知。

### 7.3 API 请求调试转储 (Debug Dump)

`_dump_api_request_debug()` 在 4xx 等不可重试错误时：
1. 序列化完整请求体 (model, messages, tools, etc.)
2. 记录 error 详情 (status_code, body, response_text)
3. 脱敏 API key (`_mask_api_key_for_logs`)
4. 写入 JSON 文件 `~/.hermes/logs/request_dump_{session_id}_{timestamp}.json`
5. 可选 stdout 输出 (`HERMES_DUMP_REQUEST_STDOUT` 环境变量)

---

## 8. Insights 分析引擎 (Insights Analytics Engine)

**核心文件**: `agent/insights.py` (789 行)

### 8.1 数据源

直接查询 SQLite 状态数据库 (`SessionDB`)，主要表：
- `sessions` — 会话元数据（model, source, tokens, cost, timestamps）
- `messages` — 消息记录（role, tool_name, tool_calls JSON）

### 8.2 分析维度

```python
class InsightsEngine:
    def generate(self, days: int = 30, source: str = None) -> Dict:
        overview = self._compute_overview(sessions, message_stats)
        models = self._compute_model_breakdown(sessions)
        platforms = self._compute_platform_breakdown(sessions)
        tools = self._compute_tool_breakdown(tool_usage)
        activity = self._compute_activity_patterns(sessions)
        top_sessions = self._compute_top_sessions(sessions)
```

| 分析维度 | 内容 |
|----------|------|
| **Overview** | 总会话数、消息数、Token 总量、预估成本、平均时长 |
| **Model Breakdown** | 按模型分类的 sessions/tokens/cost |
| **Platform Breakdown** | 按平台 (cli/discord/slack/telegram) 分类 |
| **Tool Breakdown** | 工具调用排行榜，带百分比 |
| **Activity Patterns** | 按星期/小时分布、活跃天数、最长连续使用天数 (streak) |
| **Top Sessions** | 最长/最多消息/最多 token/最多工具调用 的会话 |

### 8.3 工具使用统计的双源合并

Tool usage 从两个来源获取并智能合并：
1. **Source 1**: `messages.tool_name` 列（gateway 设置）
2. **Source 2**: `messages.tool_calls` JSON（CLI 会话，tool_name 为 NULL）

合并策略：取每个工具的两个来源中较大值，避免重复计数。

### 8.4 成本估算集成

Insights 对每个 session 调用 `_estimate_cost(session)` → `estimate_usage_cost()`:
- 区分 `has_pricing` vs `no_pricing` 的模型
- 标记 `unknown_cost_sessions` 和 `included_cost_sessions`
- 聚合时仅累计有定价数据的模型

### 8.5 输出格式

两种格式化器：
- **Terminal** (`format_terminal`): 带 Unicode 框线、ASCII 柱状图、表格对齐
- **Gateway** (`format_gateway`): Markdown 格式，适合 Discord/Slack/Telegram

---

## 9. Trajectory 保存与回放 (Trajectory Save & Replay)

### 9.1 基础保存 — trajectory.py

```python
def save_trajectory(trajectory, model, completed, filename=None):
    """Append a trajectory entry to a JSONL file."""
    if filename is None:
        filename = "trajectory_samples.jsonl" if completed else "failed_trajectories.jsonl"
    
    entry = {
        "conversations": trajectory,  # ShareGPT 格式
        "timestamp": datetime.now().isoformat(),
        "model": model,
        "completed": completed,
    }
    with open(filename, "a", encoding="utf-8") as f:
        f.write(json.dumps(entry, ensure_ascii=False) + "\n")
```

**格式**: JSONL (JSON Lines)，每行一个完整轨迹。使用 **ShareGPT 格式** (`from`/`value` 字段)。

### 9.2 轨迹转换 — _convert_to_trajectory_format()

`run_agent.py:2430` 中的方法将内部消息格式转换为训练数据格式：

1. **System Message**: 注入工具定义 (`<tools>` XML)
2. **Human Message**: 原始用户查询
3. **GPT Message**: 合并推理 (`<think>`) + 内容 + 工具调用 (`<tool_call>` XML)
4. **Tool Response**: `<tool_response>` XML 包裹

```python
# 推理标签转换
convert_scratchpad_to_think(content)  # <REASONING_SCRATCHPAD> → <think>
has_incomplete_scratchpad(content)     # 检测未闭合标签
```

### 9.3 轨迹压缩 — trajectory_compressor.py

**用途**: 后处理完成的轨迹，压缩中间部分以适应训练 token 预算。

**压缩策略**:
1. 保护首轮 (system + human + first gpt + first tool)
2. 保护末尾 N 轮 (默认 4 轮)
3. 仅压缩中间轮次，从第 2 个 tool response 开始
4. 按需压缩（仅当超过目标 token 数时）
5. 将压缩区域替换为单条 human summary 消息
6. 保留剩余工具调用完整性

**配置**:

```python
@dataclass
class CompressionConfig:
    tokenizer_name: str = "moonshotai/Kimi-K2-Thinking"
    target_max_tokens: int = 15250
    summary_target_tokens: int = 750
    protect_last_n_turns: int = 4
    summarization_model: str = "google/gemini-3-flash-preview"
    base_url: str = OPENROUTER_BASE_URL
    num_workers: int = 4
    max_concurrent_requests: int = 50
    per_trajectory_timeout: int = 300  # 5 分钟
```

**运行模式**:
- 批量处理 JSONL 文件或目录
- 支持采样 (`--sample_percent`)
- 异步并发 API 调用 (asyncio + semaphore)
- Rich progress bar 显示
- 保存超限轨迹到单独文件 (`save_over_limit`)

---

## 10. 设计模式总结 (Design Patterns Summary)

### 10.1 分层可观测性 (Layered Observability)

```
Layer 1: 原始日志 (hermes_logging.py + redact.py)
    ↓ 结构化、脱敏、轮转
Layer 2: 运行时度量 (run_agent.py session_* 字段)
    ↓ 每次 API 调用累加
Layer 3: 分析洞察 (insights.py)
    ↓ 跨会话历史聚合
Layer 4: 训练数据 (trajectory.py + trajectory_compressor.py)
    ↓ 轨迹保存和压缩
```

### 10.2 关键设计决策

| 决策 | 理由 |
|------|------|
| Record Factory vs Filter | 保证所有 handler 都能访问 session_tag |
| Import-time 快照 REDACT_ENABLED | 防止 LLM 生成的命令禁用脱敏 |
| Decimal 精度计算成本 | 避免浮点误差在聚合时放大 |
| 双源工具统计合并 | CLI 和 Gateway 使用不同字段记录工具调用 |
| 防御性 rate limit 解析 | try/except 包裹，永不阻断 agent loop |
| JSONL 追加模式 | 并发安全，无需文件锁 |

### 10.3 可扩展点 (Extension Points)

1. **post_api_request 插件 hook**: 通过 `_usage_summary_for_api_request_hook` 传递标准化用量
2. **InsightsEngine.generate(source=...)**: 按平台过滤分析
3. **CompressionConfig 配置化**: 所有参数可通过 YAML 或命令行覆盖
4. **COMPONENT_PREFIXES**: 新增组件只需添加前缀映射

---

## 11. 数据流图 (Data Flow)

```
API Response
    │
    ├─→ normalize_usage()          → CanonicalUsage
    │       │
    │       ├─→ session_*_tokens   (run_agent 累加)
    │       ├─→ estimate_usage_cost() → CostResult
    │       └─→ post_api_request hook (插件)
    │
    ├─→ _capture_rate_limits()     → RateLimitState
    │       └─→ /usage command display
    │
    └─→ response headers
            └─→ parse_rate_limit_headers()

Session End
    │
    ├─→ SessionDB.update()         → SQLite (sessions + messages)
    │       └─→ InsightsEngine.generate() → insights report
    │
    └─→ save_trajectory()          → JSONL file
            └─→ trajectory_compressor.py → compressed JSONL
```

---

## 12. 关键文件清单 (Key Files)

| 文件 | 行数 | 职责 |
|------|------|------|
| `hermes_logging.py` | 390 | 结构化日志基础设施 |
| `agent/redact.py` | 181 | 密钥脱敏 (30+ 模式) |
| `agent/usage_pricing.py` | 613 | Token 归一化 + 定价引擎 |
| `agent/rate_limit_tracker.py` | 246 | 速率限制状态追踪 |
| `agent/insights.py` | 789 | 跨会话分析报告 |
| `agent/trajectory.py` | 56 | 基础轨迹保存 (JSONL) |
| `trajectory_compressor.py` | 1462 | 批量轨迹压缩 (LLM 摘要) |
| `run_agent.py` (相关段) | ~300 | 运行时集成和累加 |
