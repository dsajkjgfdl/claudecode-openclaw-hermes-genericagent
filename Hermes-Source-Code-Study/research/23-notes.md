---
Task: T23-R
Status: DRAFT_DONE
Date: 2026-04-15
Chapter: Ch23 批处理与轨迹基础设施
Keywords: Cron 调度, batch run, trajectory 压缩, agent environment, Atropos, RL training
Sources:
  - cron/jobs.py (762 lines)
  - cron/scheduler.py (992 lines)
  - batch_runner.py (1287 lines)
  - agent/trajectory.py (56 lines)
  - trajectory_compressor.py (1462 lines)
  - environments/hermes_base_env.py (714 lines)
  - environments/agent_loop.py (534 lines)
  - environments/tool_context.py (474 lines)
  - toolset_distributions.py (364 lines)
  - tools/cronjob_tools.py (506 lines)
---

# Ch23 批处理与轨迹基础设施 (Batch Processing & Trajectory Infrastructure)

## 1. 章节概述 (Chapter Overview)

本章涵盖 Hermes Agent 的三大后台基础设施层：**Cron 调度系统**、**批处理运行器**和**轨迹 (Trajectory) 管道**。这三层共同构成了 agent 从"单次交互"扩展到"规模化数据生产和 RL 训练"的桥梁。

关键架构关系：
```
┌──────────────────────────────────────────────────────────┐
│                   Gateway (网关层)                        │
│                        │                                 │
│         ┌──────────────┼──────────────┐                  │
│    cron/scheduler.py   │      batch_runner.py            │
│    (定时任务调度)       │      (批量数据生成)              │
│         │              │              │                  │
│    cron/jobs.py        │      toolset_distributions.py   │
│    (任务持久化)        │      (工具集概率分布)             │
│         │              │              │                  │
│    run_agent.AIAgent   │      run_agent.AIAgent          │
│         │              │              │                  │
│         └──────┬───────┘      ┌───────┘                  │
│                │              │                          │
│         agent/trajectory.py   │                          │
│         (轨迹保存)            │                          │
│                │              │                          │
│    trajectory_compressor.py   │                          │
│    (轨迹压缩后处理)           │                          │
│                               │                          │
│    environments/hermes_base_env.py  ← Atropos 集成       │
│    environments/agent_loop.py       ← RL Agent Loop      │
│    environments/tool_context.py     ← 奖励函数工具访问    │
└──────────────────────────────────────────────────────────┘
```

---

## 2. Cron 调度架构 (Cron Scheduling Architecture)

### 2.1 文件级职责划分

| 文件 | 职责 | 核心类/函数 |
|------|------|------------|
| `cron/jobs.py` | Job CRUD + 持久化 (persistence) | `create_job()`, `load_jobs()`, `save_jobs()`, `mark_job_run()` |
| `cron/scheduler.py` | 定时执行 + 投递 (delivery) | `tick()`, `run_job()`, `_deliver_result()` |
| `tools/cronjob_tools.py` | Agent 工具接口 (tool API) | `cronjob()` — 统一 action 入口 |

### 2.2 Job 持久化 (Job Persistence)

存储路径 (Storage Layout)：
```
~/.hermes/cron/
├── jobs.json                    # 所有 job 的 JSON 数组
└── output/
    └── {job_id}/
        └── {timestamp}.md      # 每次运行的输出文档
```

**关键设计**：
- **JSON 文件存储** — 非数据库方案，HERMES_HOME (`~/.hermes/`) 下的 `cron/jobs.json`
- **`ensure_dirs()`** — 启动时自动创建目录结构
- **`_apply_skill_fields()`** — 向后兼容：将旧版单 `skill` 字段迁移为 `skills` 数组

Job 数据结构 (Job Data Structure)：
```python
{
    "id": "uuid",
    "name": "描述性名称",
    "prompt": "用户指令文本",
    "skills": ["skill_name_1", "skill_name_2"],  # 可选
    "schedule": {"type": "cron|interval|once", ...},
    "schedule_display": "every 30m",
    "enabled": True,
    "state": "scheduled|paused|completed",
    "repeat": {"times": 5, "completed": 0},  # None = forever
    "deliver": "origin|local",
    "origin": {"platform": "telegram", "chat_id": "123", ...},
    "model": "claude-opus-4-20250514",          # 可选，覆盖默认
    "provider": "openrouter",                   # 可选
    "base_url": "https://...",                  # 可选
    "script": "my_script.py",                   # 可选，数据收集脚本
    "next_run_at": "2026-04-15T12:00:00",
    "last_run_at": "2026-04-15T11:30:00",
    "last_status": "ok|error",
    "last_delivery_error": null,
}
```

### 2.3 Schedule 解析 (Schedule Parsing)

`parse_schedule()` 支持三种格式：

1. **Cron 表达式** — `"30 9 * * *"` (通过可选的 `croniter` 库)
2. **Interval 表达式** — `"every 30m"`, `"every 2h"`, `"every 1d"` (自定义 `parse_duration()`)
3. **One-shot 表达式** — `"in 5m"`, `"in 1h"` (一次性执行)

**优雅降级 (Graceful Degradation)**：`HAS_CRONITER` 标志控制 cron 表达式是否可用。如未安装 croniter，仅支持 interval/once。

### 2.4 Scheduler Tick 机制

```python
# scheduler.py: tick() 核心流程
def tick(verbose=True, adapters=None, loop=None) -> int:
    # 1. 文件锁 — 跨进程互斥
    lock_fd = open(_LOCK_FILE, "w")
    fcntl.flock(lock_fd, LOCK_EX | LOCK_NB)  # 非阻塞

    # 2. 获取到期任务
    due_jobs = get_due_jobs()

    # 3. 逐个执行
    for job in due_jobs:
        advance_next_run(job["id"])      # 先推进，防崩溃重复触发
        success, output, response, error = run_job(job)
        save_job_output(job["id"], output)
        _deliver_result(job, response)   # 投递到 Telegram/Discord/...
        mark_job_run(job["id"], success, error)
```

**关键设计模式**：

- **File-based Lock** (文件锁) — `~/.hermes/cron/.tick.lock`，使用 `fcntl.flock()` (Unix) / `msvcrt.locking()` (Windows) 实现跨进程互斥
- **Pre-advance** — `advance_next_run()` 在执行前就推进 `next_run_at`，崩溃恢复时不会重复触发
- **ONESHOT_GRACE_SECONDS** = 120 — 一次性任务的宽限窗口
- **Tick 间隔** — 60 秒，由 Gateway 的后台线程调用

### 2.5 Job 执行 (Job Execution)

`run_job()` 核心流程：
1. **Prompt 构建** — `_build_job_prompt()` 注入 cron 执行提示、skill 内容、脚本输出
2. **AIAgent 实例化** — 使用 `run_agent.AIAgent`，传入 model/provider/reasoning_config
3. **Inactivity Timeout** — 基于 agent 活动追踪器 (activity tracker)，默认 600s 无活动即超时
4. **投递 (Delivery)** — `_deliver_result()` 路由到 20+ 平台 (telegram, discord, slack, wecom, feishu, etc.)
5. **SILENT 标记** — Agent 回复 `[SILENT]` 时跳过投递

```python
# Prompt 构建注入的 cron 执行提示
cron_hint = (
    "[SYSTEM: You are running as a scheduled cron job. "
    "DELIVERY: Your final response will be automatically delivered... "
    "SILENT: If nothing new to report, respond with exactly \"[SILENT]\"...]"
)
```

**安全特性**：
- 禁用的 toolsets: `["cronjob", "messaging", "clarify"]`
- `skip_context_files=True` — 不注入 SOUL.md/AGENTS.md
- `skip_memory=True` — 不加载用户记忆 (防止 cron 系统提示污染)

### 2.6 Cron 工具 API (cronjob_tools.py)

`cronjob()` 函数是 Agent 的统一入口：

```python
cronjob(action="create|list|get|update|delete|pause|resume|trigger|history",
        job_id=..., prompt=..., schedule=..., ...)
```

**安全扫描 (Security Scanning)**：
- `_scan_cron_prompt()` — 扫描 10 种威胁模式 (prompt injection, exfiltration, destructive commands)
- `_CRON_INVISIBLE_CHARS` — 检测隐形 Unicode 字符 (U+200B, U+FEFF, etc.)
- `_validate_cron_script_path()` — 脚本路径必须相对于 `~/.hermes/scripts/`，禁止绝对路径和 `..` 遍历

---

## 3. 批处理运行器 (Batch Runner)

### 3.1 整体架构

`batch_runner.py` 实现大规模数据生成：从 JSONL 数据集加载 prompt，并行运行 agent，保存 ShareGPT 格式的轨迹。

```
JSONL Dataset  →  BatchRunner  →  AIAgent × N workers  →  batch_*.jsonl  →  trajectories.jsonl
                       ↕
              checkpoint.json (断点续传)
```

### 3.2 BatchRunner 类

```python
class BatchRunner:
    def __init__(self,
        dataset_file: str,        # JSONL 输入
        batch_size: int,          # 每批大小
        run_name: str,            # 运行名称 (用于输出目录)
        distribution: str = "default",  # 工具集分布
        max_iterations: int = 10,       # 每个 prompt 最大工具调用轮次
        model: str = "claude-opus-4-20250514",
        num_workers: int = 4,           # 并行进程数
        reasoning_config: dict = None,  # 推理配置
        prefill_messages: list = None,  # 预填充消息
        max_samples: int = None,        # 限制处理数量
        ...
    )
```

### 3.3 并行执行模型

```python
# 多进程 batch 处理
with ProcessPoolExecutor(max_workers=num_workers) as executor:
    futures = {
        executor.submit(process_single_prompt, prompt, ...): prompt_index
        for prompt_index, prompt in prompts_to_process
    }
    for future in as_completed(futures):
        result = future.result()
        # 收集工具统计、推理统计
```

**设计特点**：
- **ProcessPoolExecutor** — 使用多进程而非多线程，绕过 GIL
- **Batch-level 并行** — 每个 batch 内部并行，batch 间串行
- **Per-prompt checkpoint** — 完成的 prompt index 写入 `checkpoint.json`

### 3.4 断点续传 (Checkpointing & Resume)

```python
# checkpoint.json 结构
{
    "completed_prompts": [0, 1, 2, 5, 8, ...],  # 已完成的 prompt 索引
    "run_name": "my_run",
    "distribution": "default",
    ...
}
```

恢复时，`prompts_to_process` 过滤掉已完成的索引，只处理剩余。

### 3.5 轨迹格式 (Trajectory Format)

输出使用 **ShareGPT 格式** — `from`/`value` 对：

```json
{
    "conversations": [
        {"from": "system", "value": "You are Hermes..."},
        {"from": "human", "value": "Search for..."},
        {"from": "gpt", "value": "<think>Let me search...</think>\n<tool_call>..."},
        {"from": "tool", "value": "{\"results\": [...]}"},
        {"from": "gpt", "value": "Based on my research..."}
    ],
    "tool_stats": {"web_search": {"count": 3, "success": 3, "failure": 0}},
    "reasoning_stats": {"total_assistant_turns": 4, "turns_with_reasoning": 3},
    "model": "claude-opus-4-20250514",
    "timestamp": "2026-04-15T23:30:00"
}
```

### 3.6 工具集分布 (Toolset Distributions)

`toolset_distributions.py` 定义概率化工具组合：

```python
DISTRIBUTIONS = {
    "default": {"web": 100, "vision": 100, "terminal": 100, "file": 100, ...},
    "development": {"terminal": 80, "file": 80, "moa": 60, "web": 30, ...},
    "research": {"web": 90, "browser": 70, "vision": 50, ...},
    "terminal_tasks": {"terminal": 97, "file": 97, ...},
    "browser_tasks": {"browser": 97, "vision": 12, ...},
    "creative": {"image_gen": 90, "vision": 90, "web": 30},
    ...
}
```

每个 toolset 的值是**包含概率 (%)**，而非权重。`sample_toolsets_from_distribution()` 对每个 toolset 独立进行 Bernoulli 采样。

**设计意义**：数据生成时随机化工具可用性，让模型学会在不同工具组合下解决问题。

### 3.7 合并与过滤 (Merge & Filter)

`run()` 完成后：
1. 遍历所有 `batch_*.jsonl` 文件
2. **过滤损坏条目** — 检测无效工具名 (模型幻觉) 和 JSON 解码错误
3. 合并为 `trajectories.jsonl`
4. 统计工具使用率、推理覆盖率

```python
# 过滤检测
VALID_TOOLS = ALL_POSSIBLE_TOOLS  # 来自 model_tools.TOOL_TO_TOOLSET_MAP
invalid_tools = [k for k in tool_stats if k not in VALID_TOOLS]
if invalid_tools:
    # 跳过该条目 — 模型幻觉了不存在的工具名
    filtered_entries += 1
```

---

## 4. 轨迹管道 (Trajectory Pipeline)

### 4.1 trajectory.py — 基础保存

```python
def save_trajectory(trajectory, model, completed, filename=None):
    """追加写入 JSONL"""
    entry = {
        "conversations": trajectory,      # ShareGPT 格式
        "timestamp": datetime.now().isoformat(),
        "model": model,
        "completed": completed,
    }
    with open(filename, "a") as f:
        f.write(json.dumps(entry) + "\n")
```

**辅助函数**：
- `convert_scratchpad_to_think()` — `<REASONING_SCRATCHPAD>` → `<think>` 标签转换
- `has_incomplete_scratchpad()` — 检测未闭合的推理标签 (truncation 指标)

### 4.2 trajectory_compressor.py — LLM 驱动的轨迹压缩

#### 4.2.1 压缩配置 (CompressionConfig)

```python
@dataclass
class CompressionConfig:
    tokenizer_name: str = "moonshotai/Kimi-K2-Thinking"
    target_max_tokens: int = 15250          # 目标 token 上限
    summary_target_tokens: int = 750        # 摘要目标 token
    summarization_model: str = "google/gemini-3-flash-preview"  # 摘要 LLM
    base_url: str = OPENROUTER_BASE_URL     # OpenRouter API
    protect_last_n_turns: int = 4           # 尾部保护轮数
    max_concurrent_requests: int = 50       # 最大并发 API 调用
    per_trajectory_timeout: int = 300       # 每条轨迹超时 (5 min)
```

#### 4.2.2 压缩算法 (Compression Algorithm)

```
原始轨迹 (e.g., 30000 tokens):
┌─────────────────────────────────────────────────────┐
│ [Protected Head]       [Compressible Region]  [Protected Tail] │
│ system → human →       tool → gpt → tool →    last 4 turns     │
│ first gpt → first tool ...many middle turns... gpt → tool → gpt│
└─────────────────────────────────────────────────────┘

压缩后 (e.g., 15000 tokens):
┌────────────────────────────────────────────────────┐
│ [Protected Head]  [CONTEXT SUMMARY]  [Protected Tail]          │
│ system → human →  "[CONTEXT SUMMARY]: The     last 4 turns     │
│ first gpt → tool  assistant searched..."       gpt → tool → gpt│
└────────────────────────────────────────────────────┘
```

核心步骤：
1. **Token 计数** — 使用 `moonshotai/Kimi-K2-Thinking` tokenizer
2. **判定是否需要压缩** — 总 token > `target_max_tokens` (15250)
3. **标识受保护区域** — Head: system+human+first gpt+first tool; Tail: last 4 turns
4. **计算压缩量** — `tokens_to_save = total - target + summary_target`
5. **累积中间轮次** — 从 compress_start 向后累积，直到节省量足够
6. **LLM 生成摘要** — 调用 Gemini Flash 生成 `[CONTEXT SUMMARY]` 前缀的摘要
7. **构建压缩轨迹** — head + summary + (剩余未压缩的中间轮次) + tail

#### 4.2.3 异步批量处理

```python
async def _process_directory_async(self, input_dir, output_dir):
    semaphore = asyncio.Semaphore(self.config.max_concurrent_requests)  # 50 并发
    tasks = [process_single(...) for entry in all_entries]
    await asyncio.gather(*tasks)
```

- **Semaphore 限流** — 最多 50 个并发 API 调用
- **Per-trajectory timeout** — 单条轨迹最长 300s
- **Fallback 摘要** — API 调用失败时生成固定文本 `[Summary generation failed ...]`

#### 4.2.4 聚合指标 (TrajectoryMetrics)

每条轨迹追踪：
- `original_tokens` / `compressed_tokens` / `compression_ratio`
- `turns_removed` / `tokens_saved`
- `summarization_api_calls` / `summarization_errors`
- `still_over_limit` — 压缩后仍超标的标志

---

## 5. Agent 环境与 Atropos 集成 (Agent Environment & Atropos Integration)

### 5.1 HermesAgentBaseEnv — RL 训练环境基类

`environments/hermes_base_env.py` 是所有 Hermes RL 环境的抽象基类，集成 **Atropos** (RL 训练框架)。

```python
class HermesAgentBaseEnv(BaseEnv):
    """
    Two-phase operation:
    - Phase 1 (OpenAI server): 直接使用 server.chat_completion()，
      服务端原生解析 tool_calls。用于 SFT 数据生成、评估。
    - Phase 2 (VLLM server):  使用 ManagedServer + /generate endpoint，
      客户端 ToolCallTranslator 解析。获取精确 token IDs + logprobs。
      用于 RL 训练。
    """
```

子类必须实现的四个方法：
1. `setup()` — 加载数据集
2. `get_next_item()` — 返回下一个 dataset item
3. `format_prompt()` — 将 dataset item 转为 user message
4. `compute_reward()` — 使用 ToolContext 打分

### 5.2 HermesAgentEnvConfig — 环境配置

```python
class HermesAgentEnvConfig(BaseEnvConfig):
    enabled_toolsets: Optional[List[str]]   # 明确启用的工具集
    disabled_toolsets: Optional[List[str]]  # 禁用的工具集
    distribution: Optional[str]             # 工具集分布名 (与 enabled_toolsets 互斥)
    max_agent_turns: int = 30               # 每次 rollout 最大 LLM 调用数
    terminal_backend: str = "local"         # local/docker/modal/daytona/ssh/singularity
    terminal_timeout: int = 120             # 每命令超时
    terminal_lifetime: int = 3600           # 沙箱不活动生命期
    tool_call_parser: str = "hermes"        # Phase 2 的 tool call 解析器
    tool_pool_size: int = 128               # 工具执行线程池大小
    extra_body: Optional[Dict]              # OpenRouter provider 偏好
```

### 5.3 collect_trajectory() — 单次 Rollout 流程

```python
async def collect_trajectory(self, item):
    # 1. 获取 group-level 工具集 (resolve_tools_for_group)
    tools, valid_names = self._current_group_tools

    # 2. 构建初始消息
    messages = [system_prompt, user_prompt]

    # 3. 根据 Phase 选择执行模式
    if self._use_managed_server():  # Phase 2
        async with self.server.managed_server(...) as managed:
            agent = HermesAgentLoop(server=managed, ...)
            result = await agent.run(messages)
    else:  # Phase 1
        agent = HermesAgentLoop(server=self.server, ...)
        result = await agent.run(messages)

    # 4. 奖励计算 (使用 ToolContext)
    ctx = ToolContext(task_id)
    reward = await self.compute_reward(item, result, ctx)
    ctx.cleanup()

    # 5. 构建 ScoredDataItem
    return {"tokens": ..., "masks": ..., "scores": reward, "messages": result.messages}
```

### 5.4 HermesAgentLoop — Agent 循环引擎

`environments/agent_loop.py` 实现通用的多轮 tool-calling 循环：

```python
class HermesAgentLoop:
    async def run(self, messages) -> AgentResult:
        for turn in range(self.max_turns):
            # 1. 调用 LLM API
            response = await self.server.chat_completion(messages=messages, tools=..., ...)

            # 2. 提取推理内容 (reasoning_content, reasoning, reasoning_details)
            reasoning = _extract_reasoning_from_message(assistant_msg)

            # 3. 如果有 tool_calls
            if assistant_msg.tool_calls:
                # 3a. Fallback: 从原始内容解析 <tool_call> 标签
                # 3b. 验证工具名 → 执行 → 收集结果
                for tc in tool_calls:
                    tool_result = await loop.run_in_executor(
                        _tool_executor,  # ThreadPoolExecutor(128)
                        lambda: handle_function_call(name, args, task_id=...)
                    )
                    # 3c. 大结果持久化 (maybe_persist_tool_result)
                    # 3d. 追加 tool message

                # 3e. 轮次预算执行 (enforce_turn_budget)

            else:
                # 无 tool_calls → 模型自然结束
                return AgentResult(finished_naturally=True, ...)

        # 达到 max_turns
        return AgentResult(finished_naturally=False, ...)
```

**关键设计**：
- **ThreadPoolExecutor** — 128 worker 线程，解决 Modal/Docker 后端的 `asyncio.run()` 死锁
- **`resize_tool_pool()`** — 运行时调整线程池大小
- **ToolError 追踪** — 记录每个工具错误 (turn, name, args, error)，用于 wandb 日志
- **Tool Result Budget** — `maybe_persist_tool_result()` + `enforce_turn_budget()` 控制上下文膨胀

### 5.5 ToolContext — 奖励函数的工具访问

```python
class ToolContext:
    """奖励函数 (compute_reward) 可使用的全工具访问层"""

    def __init__(self, task_id: str):
        self.task_id = task_id  # 复用 rollout 的沙箱

    def terminal(self, command, timeout=180) -> Dict:
        """在 rollout 的终端会话中执行命令"""
        return _run_tool_in_thread("terminal", {"command": command, ...}, self.task_id)

    def read_file(self, path) -> Dict: ...
    def write_file(self, path, content) -> Dict: ...
    def search_files(self, pattern, path=".") -> Dict: ...
    def web_search(self, query) -> Dict: ...
    # ... 所有 hermes 工具均可用

    def cleanup(self):
        """清理 rollout 的终端/浏览器沙箱"""
        cleanup_vm(self.task_id)
        cleanup_browser(self.task_id)
```

**核心理念**：奖励函数与 Agent 共享同一个 task_id 对应的沙箱。Agent 创建的文件、运行的进程、打开的浏览器标签——奖励函数都能直接访问来验证结果。

---

## 6. 设计模式与架构洞察 (Design Patterns & Insights)

### 6.1 Fire-and-Forget vs. Activity-Based Timeout

Cron scheduler 使用**活动追踪器 (Activity Tracker)** 而非固定超时：
```python
# 基于不活动的超时 — 不是 "最多运行 10 分钟"
# 而是 "如果 10 分钟没有任何活动 (API 调用、工具调用、流式 token)，则超时"
while True:
    done, _ = concurrent.futures.wait({future}, timeout=POLL_INTERVAL)
    if done: break
    idle_secs = agent.get_activity_summary()["seconds_since_activity"]
    if idle_secs >= inactivity_limit:
        raise TimeoutError(...)
```

### 6.2 Pre-Advance Pattern (预推进模式)

Cron 任务执行前先推进 `next_run_at`：
- 防止崩溃恢复时重复执行
- 一次性任务 (one-shot) 除外，允许重试

### 6.3 Protected-Region Compression (受保护区域压缩)

轨迹压缩器不触碰"头"和"尾"：
- **Head**: system + human + first assistant + first tool — 保留任务定义和初始策略
- **Tail**: last 4 turns — 保留最终行动和结论
- **Middle**: 可压缩 — 通常是重复的搜索/文件操作

### 6.4 Bernoulli Sampling for Toolsets (工具集伯努利采样)

不同于传统的"选一个 toolset"，Hermes 对每个 toolset 独立采样：
- `"web": 90` = 90% 概率包含 web 工具
- 结果是**不同数量、不同组合**的工具集
- 增加训练数据多样性

### 6.5 Two-Phase Server Architecture (双阶段服务器)

| Phase | Server Type | 用途 | Token 处理 |
|-------|------------|------|-----------|
| Phase 1 | OpenAI Server | SFT 数据生成、评估 | 占位符 tokens |
| Phase 2 | VLLM/SGLang + ManagedServer | RL 训练 | 精确 token IDs + logprobs |

### 6.6 Thread Pool Isolation (线程池隔离)

Agent loop 使用独立的 `ThreadPoolExecutor(128)` 执行工具调用：
- 解决 Modal/Docker/Daytona 后端 `asyncio.run()` 在 Atropos 事件循环内死锁的问题
- 每个工具调用在独立线程中获得干净的事件循环
- `resize_tool_pool()` 允许运行时调整 (e.g., 89 个并行 TB2 评估任务)

---

## 7. 数据流总结 (Data Flow Summary)

### 7.1 Cron 数据流
```
User → cronjob(action="create") → jobs.json
Gateway 每 60s → tick() → get_due_jobs() → run_job()
  → AIAgent.run_conversation(prompt)
  → save_job_output() → ~/.hermes/cron/output/{id}/{ts}.md
  → _deliver_result() → Telegram/Discord/Slack/...
  → mark_job_run() → jobs.json
```

### 7.2 Batch 数据流
```
JSONL dataset → BatchRunner.run()
  → ProcessPoolExecutor(4) × process_single_prompt()
    → AIAgent → ShareGPT trajectory
  → batch_*.jsonl → 过滤 → trajectories.jsonl
  → TrajectoryCompressor → compressed_trajectories.jsonl
```

### 7.3 RL 训练数据流
```
HermesAgentBaseEnv.collect_trajectories(item)
  → _resolve_tools_for_group()  # 采样工具集
  → collect_trajectory(item)    # N 次并行
    → HermesAgentLoop.run(messages)  # 多轮 tool-calling
      → server.chat_completion() → tool execution → repeat
    → ToolContext → compute_reward()  # 验证 + 打分
    → ScoredDataItem(tokens, masks, scores, messages)
  → Atropos trainer → model update
```

---

## 8. 关键配置与环境变量 (Key Config & Environment Variables)

| 变量 | 默认值 | 用途 |
|------|--------|------|
| `HERMES_HOME` | `~/.hermes/` | 全局 home 目录 |
| `HERMES_CRON_TIMEOUT` | `600` | Cron 不活动超时 (秒) |
| `TERMINAL_ENV` | `local` | 终端后端 (local/docker/modal/...) |
| `TERMINAL_TIMEOUT` | `120` | 命令执行超时 |
| `TERMINAL_LIFETIME_SECONDS` | `3600` | 沙箱生命期 |
| `OPENROUTER_API_KEY` | — | 轨迹压缩摘要 API 密钥 |

---

## 9. 待探索方向 (Further Investigation)

- `environments/agentic_opd_env.py` — Open Problem Dataset 环境
- `environments/hermes_swe_env/` — SWE-bench 集成
- `environments/tool_call_parsers/` — 11+ 模型的 tool call 解析器 (hermes, qwen, mistral, deepseek, kimi_k2, ...)
- `environments/patches.py` — SwerexModalEnvironment 猴子补丁
- `batch_runner.py` 中的 `_normalize_tool_stats()` / `_normalize_tool_error_counts()` — 工具统计规范化
- `agent/smart_model_routing.py` — Cron 的智能模型路由
- `agent/credential_pool.py` — Provider 凭证池
