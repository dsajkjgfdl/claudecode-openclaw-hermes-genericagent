# Plan: 《深入 Hermes Agent 源码》书籍写作计划

## Context

用户（luyao618）此前完成了 [Claude-Code-Source-Study](https://github.com/luyao618/Claude-Code-Source-Study)，一本 25 篇深度源码分析系列。现希望以同样的深度和风格，为 [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent)（v0.9.0 版本线）撰写一本源码解析书。

**目标读者：** 1-3 年 Python 经验的程序员
**目标效果：** 读完后既理解 Hermes 的方方面面优秀设计，又能上手开发
**输出位置：** `/Users/yao/work/code/Hermes-Source-Code-Study`（独立仓库）
**Hermes 源码位置：** `/Users/yao/work/code/hermes-agent`
**源码基线：** 以 `RELEASE_v0.9.0.md` 对应版本线为准；若本地缺失 `v0.9.0` tag，则在正式写作前补充 commit SHA 并统一校准行号
**语言：** 中文为主，技术术语保留英文原文
**内容范围：** 源码分析 + 实战教程
**产出流程：** 两阶段制 — 每个 task 先产出源码分析 research notes，再整理成正式章节
**并发模式：** 多个 Claude Code session 并发处理不同 task，通过 `TASKS.md` 文件协调状态

## 仓库结构

```
Hermes-Source-Code-Study/           # /Users/yao/work/code/Hermes-Source-Code-Study
├── README.md                       # 目录、阅读路线、Star History
├── TASKS.md                        # 并发 task 状态追踪文件（见下方详细设计）
├── claims/                         # 原子认领锁目录（空目录锁）
│   └── README.md                   # 锁目录与回收规则说明
├── docs/
│   ├── 00-目录与阅读指引.md
│   ├── 01-项目全景.md
│   ├── ...
│   └── 附录/
│       ├── A-实战：添加一个新工具.md
│       ├── B-实战：编写一个自定义技能.md
│       └── C-实战：接入一个新通信平台.md
├── research/                       # 阶段一产出（分析笔记）
│   ├── 01-project-overview-notes.md
│   ├── ...
└── LICENSE
```

## TASKS.md — 并发状态追踪文件

用于多个 Claude Code session 之间协调 task 认领和进度。

### 状态定义

| 状态 | 含义 |
|------|------|
| `TODO` | 未开始，可认领 |
| `IN_PROGRESS` | 已被某 session 认领，正在处理 |
| `DRAFT_DONE` | 已完成初稿，待复核 |
| `IN_REVIEW` | reviewer 已认领，正在复核 |
| `CHANGES_REQUESTED` | 复核未通过，需补充修改 |
| `DONE` | 已完成 |
| `BLOCKED` | 被依赖阻塞 |
| `STALE` | lease 过期，可被重新认领 |

### 并发操作规则

每个 session 开始时：
1. 读取 `TASKS.md`
2. 找到一个可认领的 task（`TODO` / `CHANGES_REQUESTED` / `STALE`）
3. 使用原子目录锁认领：`mkdir "claims/<TASK_ID>"`
4. 再次读取 `TASKS.md`，确认该 task 仍然可认领；若不满足则 `rmdir "claims/<TASK_ID>"` 并重选
5. 将其改为 `IN_PROGRESS`，填写 assignee / started / heartbeat / lease
6. 初稿完成后移除工作锁，并改为 `DRAFT_DONE`
7. reviewer 使用 `mkdir "claims/review-<TASK_ID>"` 认领复核，并改为 `IN_REVIEW`
8. 复核通过后移除 review 锁并改为 `DONE`
9. 若复核不通过，则移除 review 锁并改为 `CHANGES_REQUESTED`

**依赖规则：**
- 同一章节的 `write` task 依赖其 `research` task `DONE` 后才能开始
- 不同章节之间**无硬依赖**，可并发执行
- T25（架构模式总结）和 T-final 建议在大部分其他 task 完成后再做
- T00（仓库初始化）必须最先完成

### Session 标识规范

- 每个 session 使用短 ID：`s01`, `s02`, `s03` ...
- reviewer 必须与 assignee 不同，禁止自审
- 同一时间一个 session 只持有一个工作锁；必要时可额外持有一个 review 锁

### 原子认领协议

工作 task：
1. 读取 `TASKS.md`
2. 选择可认领 task
3. 执行 `mkdir "claims/<TASK_ID>"`
4. 若 `mkdir` 失败，说明已有其他 session 抢先认领，立刻换 task
5. 若 `mkdir` 成功，再次读取 `TASKS.md`
6. 确认状态仍可认领后，再更新 `TASKS.md`

复核 task：
1. 选择 `DRAFT_DONE` 且 assignee 不是自己的 task
2. 执行 `mkdir "claims/review-<TASK_ID>"`
3. 若成功，再将 `TASKS.md` 更新为 `IN_REVIEW`

### Lease 与回收规则

- 默认 lease：**90 分钟**
- heartbeat 频率：**至少每 20 分钟更新一次**
- 若 `lease_until` 已过期且 `heartbeat` 超过 20 分钟未更新，其他 session 可：
  1. 将 task 标记为 `STALE`
  2. 对空锁目录执行 `rmdir "claims/<TASK_ID>"`
  3. 按正常流程重新认领
- 禁止删除他人已经产出的 `research/`、`docs/` 文件

### Review 规则

- `DRAFT_DONE` 不算下游可依赖；只有 `DONE` 才算正式完成
- reviewer 检查重点：源码覆盖范围是否符合 `PLAN.md`、行号是否可回溯、章节模板是否完整、是否回答引导问题、是否包含最小运行验证
- 若 review 失败，状态改为 `CHANGES_REQUESTED`，并在 `TASKS.md` 的 `Notes` 列写明缺口
- 修改完成后，重新走 `IN_PROGRESS -> DRAFT_DONE -> IN_REVIEW -> DONE`

### TASKS.md 文件格式

```markdown
# Task Tracker

> 最后更新: YYYY-MM-DD HH:MM
> 活跃 session 数: N

## 并发规则
- 认领前先刷新此文件（git pull 或重新读取）
- 工作认领先用 `mkdir claims/<TASK_ID>` 抢原子锁，再改本表
- review 认领用 `mkdir claims/review-<TASK_ID>`
- 同一章节 write 依赖 research `DONE`
- 不同章节可并行
- 修改此文件时尽量原子操作（只改自己的行）
- 默认 lease 90 分钟，heartbeat 至少每 20 分钟更新一次
- 推荐状态流转：TODO → IN_PROGRESS → DRAFT_DONE → IN_REVIEW → DONE
- review 未通过时：DRAFT_DONE / IN_REVIEW → CHANGES_REQUESTED → IN_PROGRESS
- lease 过期可标记为 `STALE` 后回收空锁目录

## Tasks

| ID | Task | Status | Assignee | Reviewer | Started | Heartbeat | Lease Until | Completed | Notes | Output |
|----|------|--------|----------|----------|---------|-----------|-------------|-----------|-------|--------|
| T00 | 仓库初始化 + README + 目录 | TODO | - | - | - | - | - | - | - | README.md, docs/00-*.md |
| T01-R | Ch01 项目全景 Research | TODO | - | - | - | - | - | - | - | research/01-notes.md |
| T01-W | Ch01 项目全景 Write | BLOCKED(T01-R>=DONE) | - | - | - | - | - | - | - | docs/01-项目全景.md |
| T02-R | Ch02 启动链路 Research | TODO | - | - | - | - | - | - | - | research/02-notes.md |
| T02-W | Ch02 启动链路 Write | BLOCKED(T02-R>=DONE) | - | - | - | - | - | - | - | docs/02-启动链路.md |
| ... | ... | ... | ... | ... | ... | ... | ... | ... | ... | ... |
```

---

## 写作风格（沿用 Claude Code Source Study）

- 每篇开头：**一个引导问题**说明"为什么要了解这个模块"
- 每篇遵循统一骨架：**引导问题 → 先看结论 → 主链路图 → 逐文件拆解 → 关键函数精读 → 设计取舍 → 可迁移模式 → 小结**
- 正文：**逐文件、逐函数拆解**，精确到源码行号，附关键代码片段
- 架构图：使用 **Mermaid** 画调用链路、类关系、数据流
- 每篇结尾：**可迁移的设计模式**（2-3 个）
- 面向实战：对 Python 特有的技巧（如装饰器、元类、`__slots__`）做专门解释
- 每篇尽量补一个**最小运行验证**，验证关键结论不是只靠静态阅读得出

---

## 章节规划（共 25 篇 + 3 篇实战附录）

### Part 1: 全局架构（4 篇）

| Task ID | 章节 | 核心源码文件 | 产出文件 |
|---------|------|-------------|----------|
| T01-R | **01-项目全景** Research | `pyproject.toml`, `hermes_constants.py`, `AGENTS.md`, `CONTRIBUTING.md` | `research/01-notes.md` |
| T01-W | **01-项目全景** — 技术栈选型（Python + OpenAI SDK + Rich + prompt_toolkit）、模块依赖全景、文件依赖链 | 同上 | `docs/01-项目全景.md` |
| T02-R | **02-启动链路** Research | `hermes_cli/main.py`, `run_agent.py` 的 `AIAgent.__init__` / `chat()` / CLI 入口调用点 | `research/02-notes.md` |
| T02-W | **02-启动链路** — 从 `hermes` 命令到 Agent 启动：argparse 路由、profile 预处理、env/bootstrap、Agent 初始化；只覆盖入口链路，不展开 `cli.py` 内部 UI 细节 | 同上 | `docs/02-启动链路.md` |
| T03-R | **03-配置系统** Research | `hermes_cli/config.py`, `hermes_cli/main.py` 中 early config load, `gateway/run.py` 中 config→env bridge, `cli-config.yaml.example`, `hermes_constants.py`, `.env.example` | `research/03-notes.md` |
| T03-W | **03-配置系统** — 多层配置合并（硬编码默认→YAML→环境变量→CLI参数）、profile 机制、gateway 配置桥接、config migration | 同上 | `docs/03-配置系统.md` |
| T04-R | **04-状态持久化** Research | `hermes_state.py`(1238行) | `research/04-notes.md` |
| T04-W | **04-状态持久化** — SessionDB 设计：SQLite WAL 模式、FTS5 全文搜索、session 分裂与链接 | 同上 | `docs/04-状态持久化.md` |

### Part 2: AI 核心（6 篇）

| Task ID | 章节 | 核心源码文件 | 产出文件 |
|---------|------|-------------|----------|
| T05-R | **05-对话循环** Research | `run_agent.py` 的 `run_conversation()`, `chat()`, `IterationBudget`, tool dispatch 主链路 | `research/05-notes.md` |
| T05-W | **05-对话循环** — 同步 Agent 主循环：消息推进、tool_calls 处理、iteration budget、中断恢复；retry/fallback 只讲挂接点，具体策略留给第 10 章 | 同上 | `docs/05-对话循环.md` |
| T06-R | **06-System Prompt 工程** Research | `agent/prompt_builder.py`(1043行) | `research/06-notes.md` |
| T06-W | **06-System Prompt 工程** — 分段构建、context file 注入与安全扫描、skills index | 同上 | `docs/06-System-Prompt-工程.md` |
| T07-R | **07-上下文管理** Research | `agent/context_compressor.py`(1091行), `agent/context_engine.py`, `agent/context_references.py` | `research/07-notes.md` |
| T07-W | **07-上下文管理** — 自动压缩：head/tail 保护、tool output 预裁剪、iterative summary | 同上 | `docs/07-上下文管理.md` |
| T08-R | **08-多模型适配** Research | `agent/model_metadata.py`(1102行), `agent/anthropic_adapter.py`(1411行), `agent/smart_model_routing.py`, `agent/models_dev.py` | `research/08-notes.md` |
| T08-W | **08-多模型适配** — OpenAI 兼容层 + Anthropic 原生适配、metadata 注册表、smart routing | 同上 | `docs/08-多模型适配.md` |
| T09-R | **09-辅助客户端与成本控制** Research | `agent/auxiliary_client.py`(2615行), `agent/usage_pricing.py`, `agent/rate_limit_tracker.py`, `agent/credential_pool.py`(1416行) | `research/09-notes.md` |
| T09-W | **09-辅助客户端与成本控制** — 辅助 LLM 调用、凭证池与轮转、定价引擎、rate limit 追踪 | 同上 | `docs/09-辅助客户端与成本控制.md` |
| T10-R | **10-错误处理与韧性** Research | `agent/error_classifier.py`(820行), `run_agent.py` 中的 retry 逻辑 | `research/10-notes.md` |
| T10-W | **10-错误处理与韧性** — 错误分类器、retry 策略、优雅降级、ASCII 恢复、模型回退 | 同上 | `docs/10-错误处理与韧性.md` |

### Part 3: 工具系统（5 篇）

| Task ID | 章节 | 核心源码文件 | 产出文件 |
|---------|------|-------------|----------|
| T11-R | **11-工具注册表设计** Research | `tools/registry.py`, `model_tools.py`, `toolsets.py`, `toolset_distributions.py` | `research/11-notes.md` |
| T11-W | **11-工具注册表设计** — AST 自发现、ToolEntry `__slots__`、toolset 分组、import chain 设计 | 同上 | `docs/11-工具注册表设计.md` |
| T12-R | **12-终端工具深度剖析** Research | `tools/terminal_tool.py`(1749行), `tools/process_registry.py`(1172行), `tools/approval.py`(926行), `tools/tirith_security.py` 的调用接缝 | `research/12-notes.md` |
| T12-W | **12-终端工具深度剖析** — 终端工具编排层：命令执行生命周期、前置安全检查、持久会话、后台进程注册表；具体 backend 实现留给第 22 章 | 同上 | `docs/12-终端工具深度剖析.md` |
| T13-R | **13-文件与代码工具** Research | `tools/file_tools.py`, `tools/file_operations.py`(1216行), `tools/code_execution_tool.py`(1377行) | `research/13-notes.md` |
| T13-W | **13-文件与代码工具** — 文件读写/搜索/patch、代码沙箱执行 | 同上 | `docs/13-文件与代码工具.md` |
| T14-R | **14-Web 与浏览器工具** Research | `tools/web_tools.py`(2100行), `tools/browser_tool.py`(2393行), `tools/browser_camofox.py` | `research/14-notes.md` |
| T14-W | **14-Web 与浏览器工具** — Parallel + Firecrawl 搜索、Browserbase 自动化、CamoFox 反检测 | 同上 | `docs/14-Web-与浏览器工具.md` |
| T15-R | **15-MCP 协议实现** Research | `tools/mcp_tool.py`(2273行), `mcp_serve.py`, `tools/mcp_oauth.py` | `research/15-notes.md` |
| T15-W | **15-MCP 协议实现** — MCP 客户端/服务端双向实现、OAuth 认证 | 同上 | `docs/15-MCP-协议实现.md` |

### Part 4: 智能体系统（4 篇）

| Task ID | 章节 | 核心源码文件 | 产出文件 |
|---------|------|-------------|----------|
| T16-R | **16-子智能体委托** Research | `tools/delegate_tool.py`(1103行) | `research/16-notes.md` |
| T16-W | **16-子智能体委托** — context 隔离、toolset 限制、batch 并行、MAX_DEPTH 防递归 | 同上 | `docs/16-子智能体委托.md` |
| T17-R | **17-记忆系统** Research | `agent/memory_manager.py`, `agent/memory_provider.py`, `tools/memory_tool.py`, `tools/session_search_tool.py`, `plugins/memory/` | `research/17-notes.md` |
| T17-W | **17-记忆系统** — MemoryManager 编排、Provider/Plugin 扩展、MEMORY.md 持久化、FTS5 搜索 | 同上 | `docs/17-记忆系统.md` |
| T18-R | **18-技能系统** Research | `tools/skills_tool.py`(1419行), `tools/skill_manager_tool.py`, `tools/skills_hub.py`(3053行), `tools/skills_guard.py`, `agent/skill_utils.py` | `research/18-notes.md` |
| T18-W | **18-技能系统** — 技能发现/加载/执行、Skills Hub 生态、自我改进、安全沙箱 | 同上 | `docs/18-技能系统.md` |
| T19-R | **19-安全体系** Research | `tools/tirith_security.py`, `tools/skills_guard.py`, `tools/path_security.py`, `tools/approval.py`, `tools/url_safety.py` | `research/19-notes.md` |
| T19-W | **19-安全体系** — Tirith 安全引擎、path traversal 防护、prompt injection 扫描、URL 安全 | 同上 | `docs/19-安全体系.md` |

### Part 5: 平台与基础设施（4 篇）

| Task ID | 章节 | 核心源码文件 | 产出文件 |
|---------|------|-------------|----------|
| T20-R | **20-CLI 交互界面** Research | `cli.py`(10039行), `hermes_cli/commands.py`, `hermes_cli/callbacks.py`, `agent/display.py`, `hermes_cli/skin_engine.py` | `research/20-notes.md` |
| T20-W | **20-CLI 交互界面** — `cli.py` 内部 UI：Rich + prompt_toolkit TUI、slash 命令注册表、KawaiiSpinner、皮肤引擎；避免与第 02 章的启动入口重复 | 同上 | `docs/20-CLI-交互界面.md` |
| T21-R | **21-消息网关** Research | `gateway/run.py`(9654行), `gateway/session.py`, `gateway/session_context.py`, `gateway/delivery.py`, `gateway/platforms/base.py`, `gateway/platforms/telegram.py` | `research/21-notes.md` |
| T21-W | **21-消息网关** — 统一网关架构、平台适配器模式、会话上下文组装、消息投递、跨平台 slash 命令 | 同上 | `docs/21-消息网关.md` |
| T22-R | **22-终端后端** Research | `tools/environments/base.py`, `local.py`, `docker.py`, `ssh.py`, `daytona.py`, `modal.py`, `singularity.py`（只覆盖 `tools/environments/`，不混入仓库根目录 `environments/`） | `research/22-notes.md` |
| T22-W | **22-终端后端** — 多 backend 的统一抽象：local/docker/ssh/daytona/modal/singularity，及持久化/远程执行差异 | 同上 | `docs/22-终端后端.md` |
| T23-R | **23-批处理与轨迹基础设施** Research | `cron/scheduler.py`, `cron/jobs.py`, `batch_runner.py`, `trajectory_compressor.py`, 仓库根目录 `environments/` | `research/23-notes.md` |
| T23-W | **23-批处理与轨迹基础设施** — Cron 调度、batch run、trajectory 压缩，以及 agent environment/benchmark environment 的衔接 | 同上 | `docs/23-批处理与轨迹基础设施.md` |

### Part 6: 总结与实战（2 篇 + 3 篇附录）

| Task ID | 章节 | 核心内容 | 产出文件 |
|---------|------|---------|----------|
| T24-R | **24-日志、用量统计与可观测性** Research | `hermes_logging.py`, `agent/insights.py`, `agent/trajectory.py`, `agent/rate_limit_tracker.py`, `agent/usage_pricing.py` | `research/24-notes.md` |
| T24-W | **24-日志、用量统计与可观测性** — 结构化日志、usage insights、rate limit 追踪、trajectory 保存与回放 | 同上 | `docs/24-日志与可观测性.md` |
| T25-R | **25-架构模式总结** Research | 全项目回顾 | `research/25-notes.md` |
| T25-W | **25-架构模式总结** — 从 Hermes 中提炼的 7-10 个可迁移设计模式 | 同上 | `docs/25-架构模式总结.md` |
| TA-R | **附录 A** Research | `tools/registry.py`, `CONTRIBUTING.md` | `research/appendix-a-notes.md` |
| TA-W | **附录 A：实战 — 添加一个新工具** | 同上 | `docs/附录/A-实战-添加一个新工具.md` |
| TB-R | **附录 B** Research | `skills/` 目录, `tools/skills_tool.py`, `CONTRIBUTING.md` | `research/appendix-b-notes.md` |
| TB-W | **附录 B：实战 — 编写一个自定义技能** | 同上 | `docs/附录/B-实战-编写一个自定义技能.md` |
| TC-R | **附录 C** Research | `gateway/platforms/ADDING_A_PLATFORM.md`, `gateway/platforms/base.py` | `research/appendix-c-notes.md` |
| TC-W | **附录 C：实战 — 接入一个新通信平台** | 同上 | `docs/附录/C-实战-接入一个新通信平台.md` |

---

## 并发执行策略

### 依赖图

```
T00 (仓库初始化) ─── 必须最先完成
 │
 ├── T01-R ──→ T01-W
 ├── T02-R ──→ T02-W
 ├── T03-R ──→ T03-W
 ├── T04-R ──→ T04-W       ← 这些全部可以并行
 ├── T05-R ──→ T05-W
 ├── T06-R ──→ T06-W
 ├── ...
 ├── T24-R ──→ T24-W
 │
 ├── TA-R ──→ TA-W         ← 附录也可并行
 ├── TB-R ──→ TB-W
 └── TC-R ──→ TC-W

 所有 Write tasks DONE ──→ T25-R ──→ T25-W ──→ T-final
```

### 最大并发建议

- **Research tasks：可完全并行**（不同章节读不同源码文件，无冲突）
- **Write tasks：同章节的 write 需等对应 research 完成，不同章节可并行**
- **建议并发数：3-4 个 session**；在 Hermes 源码 commit SHA 未冻结前，不建议继续扩大并发
- **冲突最小化：** 每个 session 只写自己的文件（`research/XX-notes.md` 或 `docs/XX-*.md`），共享写入点只有 `TASKS.md` 和 `claims/`

### Session 工作流

每个 Claude Code session 启动时执行：

```
1. 读取 TASKS.md
2. 读取 plan 文件（本文件）了解写作风格和要求
3. 选择一个可认领 task（TODO / CHANGES_REQUESTED / STALE）
4. 执行 mkdir "claims/<TASK_ID>" 抢原子锁
5. 再次读取 TASKS.md，确认任务仍可认领
6. 更新 TASKS.md 状态为 IN_PROGRESS，并填写 assignee / heartbeat / lease
7. 执行 task：
   - Research: 读 hermes 源码 → 产出 research notes
   - Write: 读 research notes → 产出正式章节
8. 初稿完成后移除工作锁，更新为 `DRAFT_DONE`
9. 空闲 session 认领 review 锁并复核
10. review 通过后更新为 `DONE`；未通过则更新为 `CHANGES_REQUESTED`
11. 回到步骤 1，认领下一个 task
```

---

## Task 执行流程（两阶段制）

### 阶段一：Research（每章约 1 个 task）
1. 精读核心源码文件，记录关键类/函数/设计决策
2. 画出调用链路和数据流（Mermaid 草稿）
3. 标注精确行号和代码片段
4. 尽可能补一个最小运行验证，确认关键结论能被执行结果支持
5. 产出 `research/XX-notes.md`

### 阶段二：Write（每章约 1 个 task）
1. 基于 research notes 整理成正式章节
2. 遵循统一模板：引导问题 → 先看结论 → 主链路图 → 逐模块拆解 → 关键函数精读 → 设计取舍 → 可迁移模式 → 小结
3. 面向 1-3 年 Python 经验的读者：解释 Python 特有技巧
4. 明确回答本章引导问题，并讲清至少一条 end-to-end 主链路
5. 产出 `docs/XX-章节名.md`

### 初始化 Task（T00）
- 创建仓库目录结构 (`docs/`, `docs/附录/`, `research/`)
- 编写 README.md（目录 + 阅读路线）
- 编写 `docs/00-目录与阅读指引.md`
- 创建 TASKS.md（初始化所有 task 状态为 TODO）
- 创建 LICENSE（MIT）

---

## 推荐阅读路线（写入 README）

| 路线 | 篇数 | 适合人群 |
|------|------|---------|
| ⚡ 入门路线 | 7 篇 | 快速建立全局认知：01 → 02 → 05 → 11 → 16 → 17 → 25 |
| 🤖 AI 工程路线 | 10 篇 | 深入 Agent 设计：01 → 05 → 06 → 07 → 08 → 09 → 11 → 16 → 17 → 18 |
| 🔧 工具开发路线 | 8 篇 | 上手贡献代码：01 → 03 → 11 → 12 → 13 → 附录A → 附录B → 附录C |
| 📚 完整路线 | 28 篇 | 按顺序通读 |

---

## 总 Task 清单

```
T00:    仓库初始化 + README + TASKS.md + 目录文件
T01-R ~ T25-R:  25 个 research tasks
T01-W ~ T25-W:  25 个 write tasks
TA-R, TA-W:     附录 A（2 tasks）
TB-R, TB-W:     附录 B（2 tasks）
TC-R, TC-W:     附录 C（2 tasks）
T-final:        最终审校、阅读路线调整、README 完善
```

共 **57 个 task**。每个 task 产出一个独立文件，可逐步完成。

## 验证方式

- 每篇 research notes 需包含：精确文件路径 + 行号引用 + 关键代码片段 + 至少一个最小运行验证
- 每篇正式章节需包含：至少 1 个 Mermaid 图 + 至少 2 个可迁移设计模式
- 每篇正式章节必须明确回答开头的引导问题，并讲清至少一条 end-to-end 主链路
- 代码引用需可溯源到 Hermes v0.9.0 版本线源码；补齐 commit SHA 后统一校准行号
- 实战附录需包含完整的步骤，读者可照着做
