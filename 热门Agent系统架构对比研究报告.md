# 热门 Agent 系统架构对比研究报告

> 生成日期：2026-05-15  
> 研究对象：OpenClaw、GenericAgent、Claude Code、Hermes Agent  
> 研究范围：记忆机制、工具调用、上下文管理、执行编排，以及四类系统的相同点、差异点、优势与短板。  
> 资料依据：本报告基于本地学习文档整理，不直接等同于各项目 upstream 最新源码状态。

---

## 0. 资料来源与分析口径

本报告主要参考以下本地文档集：

| 系统 | 文档位置 | 重点参考章节 |
|---|---|---|
| OpenClaw | `D:\study\openclaw\my_doc` | `04-agent-runtime.md`、`05-session-management.md`、`openclaw-book/chapters/03-plugin-system.md`、`04-agent-engine.md`、`05-session-management.md`、`07-tool-system.md`、`10-mcp-integration.md`、`11-task-system.md`、`12-provider-model.md` |
| GenericAgent | `D:\study\GenericAgent\my_doc` | `记忆系统.md`、`记忆系统_L0-L4层级文件映射.md`、`02_Agent核心循环.md`、`04_工具系统与Handler.md`、`11_第3课_LLM接入层如何统一不同模型和工具协议.md`、`13_第5课_前端适配记忆系统与调度如何协作.md` |
| Claude Code | `.\Claude-Code-Source-Study` | `docs/05-对话循环.md`、`06-上下文管理.md`、`09-工具系统设计.md`、`12-Agent-系统.md`、`14-任务系统.md`、`15-MCP-协议实现.md`、`16-权限系统.md`、`23-Memory系统.md`、`25-架构模式总结.md` |
| Hermes Agent | `.\Hermes-Source-Code-Study` | `docs/04-状态持久化.md`、`05-对话循环.md`、`07-上下文管理.md`、`11-工具注册表设计.md`、`12-终端工具深度剖析.md`、`13-文件与代码工具.md`、`15-MCP-协议实现.md`、`16-子智能体委托.md`、`17-记忆系统.md`、`21-消息网关.md`、`23-批处理与轨迹基础设施.md` |

对比口径采用四个模块：

1. **记忆机制**：长期记忆、短期工作记忆、会话历史、记忆召回、写入与持久化。
2. **工具调用**：工具定义、schema、handler/dispatch、权限、安全、MCP/插件。
3. **上下文管理**：prompt 组装、token 预算、工具结果裁剪、压缩、缓存、消息完整性。
4. **执行编排**：Agent 主循环、任务队列、子 Agent、多平台 Gateway、异步任务、取消与恢复。

---

## 1. 一句话结论

| 系统 | 一句话定位 | 最核心能力 | 最大代价 |
|---|---|---|---|
| **OpenClaw** | 多通道 AI Gateway + 插件化 Agent runtime | Gateway、插件契约、会话/任务持久化、多 provider 和多 channel | 系统面复杂，状态分散，运维和排障成本高 |
| **GenericAgent** | 极简本地自治 Agent 框架 | L0-L4 文件记忆、少量高权限原子工具、极短主循环 | 安全边界较弱，调度/权限/持久化工程化不足 |
| **Claude Code** | 垂直整合的生产级 CLI Coding Agent | prompt cache、上下文压缩、权限系统、Agent 隔离、ToolSearch、自动记忆 | 深度绑定 Claude API 能力，内部 feature gate 和复杂度高 |
| **Hermes Agent** | 通用全栈 Agent 操作系统 | SQLite/FTS5 状态、工具自注册、MemoryProvider、多平台 Gateway、delegate_task | 核心文件偏大，全局状态多，并发边界维护复杂 |

更具体地说：

- **如果看记忆设计**：Claude Code 最系统化，GenericAgent 最清晰可读，Hermes 最本地工程化，OpenClaw 更偏插件/会话/任务综合状态。
- **如果看工具系统**：Claude Code 最强调安全默认和 UI/权限/并发一体化，Hermes 最强调 Python 自注册与 toolset 组合，OpenClaw 最强调多来源工具统一描述，GenericAgent 最强调少工具高权限。
- **如果看上下文管理**：Claude Code 的层级最细，包含 microcompact、full compact、session memory compact、cache_edits、prompt cache；Hermes 的 ContextCompressor 是独立工程体系；OpenClaw 已有 context engine 和 compaction 管线；GenericAgent 主要依赖 history trim、anchor prompt 和工作记忆。
- **如果看执行编排**：OpenClaw 和 Hermes 都是“Agent + Gateway + 多平台”的全栈系统；Claude Code 是“CLI + 多 Agent + 任务系统”的垂直产品；GenericAgent 是单进程队列驱动的极简模型。

---

## 2. 系统概览

### 2.1 OpenClaw

OpenClaw 更像一个多通道 AI 网关，而不是单纯 CLI Agent。其核心结构包含 Gateway、Agent runtime、插件 runtime、channel adapter、MCP、session store、task registry 等。资料中反复出现的关键模块包括：

- `src/gateway/`：WebSocket RPC、HTTP endpoints、Control UI、session、nodes、cron、tools、models、config reload。
- `src/agents/`：CLI backend、embedded runner、model/auth/profile、工具、prompt、session history、failover。
- `src/plugins/`：manifest、metadata snapshot、runtime registry、plugin public surface。
- `src/config/sessions/`：`sessions.json`、`SessionEntry`、store writer、cache、maintenance。
- `src/tasks/`：TaskRecord、TaskFlow、SQLite registry、detached runtime。

OpenClaw 的核心思路是“核心不直接依赖具体插件”，插件通过 manifest、contracts、runtime API、SDK surface 贡献 provider、channel、tool、hook、memory、service 等能力。

### 2.2 GenericAgent

GenericAgent 是四个系统中最小、最直观的。核心文件集中在：

- `agentmain.py`：Agent 入口、任务队列、系统提示词、模型选择。
- `agent_loop.py`：核心多轮循环，负责 LLM 调用、工具分发、结果回传。
- `ga.py`：工具实现与 `GenericAgentHandler`。
- `llmcore.py`：模型接入、原生工具协议、文本工具协议、流式解析、历史裁剪。
- `memory/`：L0-L4 长期记忆、SOP、脚本、调度归档。

它的典型链路是：前端调用 `GeneraticAgent.put_task()` 入队，`run()` 消费任务，拼 system prompt，创建 handler，进入 `agent_runner_loop()`，模型返回工具调用，`BaseHandler.dispatch()` 映射到 `do_xxx()`，工具返回 `StepOutcome`，再构造下一轮 prompt。

### 2.3 Claude Code

Claude Code 是高度产品化的 CLI Coding Agent。文档显示它围绕 Claude API 特性做了深度优化，包括：

- `query.ts`：AsyncGenerator 对话循环，流式响应、错误恢复、工具执行、附件注入。
- `services/compact/`：microcompact、auto compact、session memory compact、reactive compact。
- `tools.ts` + `Tool.ts`：40+ 内置工具，`buildTool()` 安全默认，MCP 工具合并。
- `tools/AgentTool/`：子 Agent、fork agent、Agent memory、MCP 增量扩展。
- `tasks/`：后台 Shell、本地 Agent、远程 Agent、DreamTask、主会话后台化。
- `memdir/`、`services/SessionMemory/`、`services/extractMemories/`：自动记忆、会话记忆、后台提取。

Claude Code 的特点是“垂直整合”：它不追求模型无关，而是尽量利用 Claude API 的 prompt cache、tool_reference、cache_edits、thinking blocks、cache-safe fork 等能力。

### 2.4 Hermes Agent

Hermes 是 Python 全栈 Agent 框架。它不像 GenericAgent 那样极简，也不像 Claude Code 那样绑定单一模型生态，而是试图覆盖 CLI、Gateway、MCP、Cron、Batch、Memory、Browser、Terminal、安全审批等一整套 Agent 操作系统能力。

关键模块包括：

- `run_agent.py`：`AIAgent.run_conversation()` 主循环。
- `hermes_state.py`：`SessionDB`，SQLite WAL + FTS5。
- `tools/registry.py`：中央工具注册表。
- `toolsets.py`：工具集组合。
- `model_tools.py`：工具 schema 输出与 dispatch。
- `agent/context_compressor.py`：上下文压缩。
- `tools/delegate_tool.py`：子智能体委托。
- `agent/memory_manager.py`、`tools/memory_tool.py`、`plugins/memory/`：记忆系统。
- `gateway/run.py`、`gateway/platforms/`：消息平台网关。

---

## 3. 记忆机制对比

### 3.1 共同点

四个系统虽然实现差异很大，但在记忆设计上有几个共识：

1. **记忆最终都要进入 prompt 或消息上下文**  
   无论是 OpenClaw 的 memory prompt section、GenericAgent 的 L1 注入、Claude Code 的 Relevant Memories attachment，还是 Hermes 的 frozen memory snapshot 和 `<memory-context>`，本质都是把外部状态转成模型可读文本。

2. **索引与实体分离**  
   GenericAgent 明确提出“上层只存指针，下层存实体”；Claude Code 的 `MEMORY.md` 和 topic files 也是索引/内容分离；Hermes 通过 `MEMORY.md/USER.md` 和 `SessionDB/FTS5` 分离精炼知识与完整历史；OpenClaw 的 session/transcript/memory plugin 也将会话状态、转录本、memory search 分开。

3. **文件系统/Markdown 是主流载体**  
   GenericAgent 用 `memory/*.md`、`global_mem*.txt`；Claude Code 用 `CLAUDE.md`、memdir topic files、agent-memory；Hermes 用 `MEMORY.md/USER.md`；OpenClaw 也兼容 Claude Code memory 格式并通过 memory 插件扩展。

4. **当前任务的工作记忆与跨会话长期记忆分离**  
   GenericAgent 的 `handler.working`，Claude Code 的 Session Memory / tool context，Hermes 的 tool loop prefetch cache，OpenClaw 的 session/transcript/runtime state 都体现了这点。

5. **记忆系统必须防 prompt 注入**  
   Hermes 明确使用 `<memory-context>` fence 和注入扫描；Claude Code 区分 CLAUDE.md、Auto Memory、Relevant Memories，并通过附件与去重控制注入；OpenClaw 使用输入溯源和 inter-session prefix；GenericAgent 通过 L0 SOP 约束写入规则，但安全隔离相对轻。

### 3.2 OpenClaw 的记忆与状态

OpenClaw 的“记忆”需要拆成三类看：

**第一类：会话状态记忆。**  
`SessionEntry` 是最核心的持久化对象，包含会话路由、模型覆盖、token/成本、compaction checkpoint、pending final delivery、父子会话、ACP metadata、skills snapshot 等。它说明 OpenClaw 把“Agent 记住什么”很大一部分建模为 session 级运行状态。

**第二类：transcript/history。**  
OpenClaw 的 Gateway/session 模块维护 transcript、history、reset、cleanup、compaction。Agent run 后会持久化 transcript，并在需要时触发 compaction 生命周期。

**第三类：插件化 memory。**  
资料中的 source index 指向 `src/memory/`，并提到 `memory-core`、`memory-lancedb`、`memory-wiki`、`active-memory` 等扩展。插件 SDK 可注册 memory embedding provider 或 memory runtime。这说明 OpenClaw 不把 memory 固定成一种文件结构，而是把它作为插件能力纳入 runtime。

OpenClaw 的记忆优势是与多通道会话、任务、交付和插件系统打通；缺点是概念分散，理解“记忆”时要同时看 session store、transcript、memory plugin、skills snapshot、compaction checkpoint。

### 3.3 GenericAgent 的 L0-L4 记忆

GenericAgent 的记忆系统最像一套人类可读的个人知识库。核心原则是：

> 上层只存指针，下层存实体。

分层如下：

| 层级 | 定位 | 载体 | 运行时作用 |
|---|---|---|---|
| L0 | 元规则，定义如何操作记忆 | `memory/memory_management_sop.md`、`memory_cleanup_sop.md` | 写入 L1/L2/L3 前先读 L0 校验 |
| L1 | 极简索引，存在性编码 | `memory/global_mem_insight.txt` | 每次任务启动注入 system prompt |
| L2 | 稳定环境事实 | `memory/global_mem.txt` | 按需 `file_read` |
| L3 | SOP、脚本、技能 | `memory/*.md`、`memory/*.py` | 按需读入，指导复杂操作 |
| L4 | 历史会话归档 | `memory/L4_raw_sessions/` | 每 12 小时从 `temp/model_responses/` 压缩归档 |

GenericAgent 有一个重要特点：记忆写入不是后台数据库操作，而是普通 Agent 工具流程。`start_long_term_update` 会生成记忆提炼任务，模型根据 L0 决策树，用 `file_read/file_patch/file_write` 自己更新 L1/L2/L3。

短期工作记忆则保存在 `GenericAgentHandler.working` 中，字段包括：

- `key_info`
- `related_sop`
- `passed_sessions`

每轮工具后 `_get_anchor_prompt()` 把近期摘要、当前轮次、key_info、related_sop 注入下一轮 `### [WORKING MEMORY]`。

优势：

- 设计极其清晰，人类可审查、可手工修正。
- L1 极简索引节省 token，L2/L3 按需读取。
- SOP 既是记忆，也是“技能”与操作流程。

短板：

- L1 如果维护不及时，会导致实体记忆不可发现。
- 写入依赖模型遵守 L0，缺少强制事务和并发控制。
- 高权限文件工具使记忆写入和普通文件修改共享同一安全边界。

### 3.4 Claude Code 的五层记忆

Claude Code 的记忆系统最丰富，文档中明确列出五层：

| 层级 | 名称 | 生命周期 | 作用 |
|---|---|---|---|
| 1 | `CLAUDE.md` 指令文件 | 永久 | 静态项目/用户/本地/managed 指令 |
| 2 | Auto Memory / memdir | 跨会话 | AI 自动提取的持久知识 |
| 3 | Session Memory | 单次会话 | 当前会话结构化笔记，可参与 compact |
| 4 | Agent Memory | 跨会话 | 每个自定义 Agent 的专属记忆 |
| 5 | Relevant Memories | 每用户 turn 按需注入 | side query 选择相关 topic files |

几个设计值得重点关注：

**CLAUDE.md 多源加载。**  
Managed、User、Project、Local 以反优先级顺序加载，越高优先级越靠后，利用模型 recency bias。项目目录向上遍历，可包含 `.claude/rules/*.md` 和 `@include`。

**Auto Memory 分类法。**  
Auto Memory 使用 `user/feedback/project/reference` 四类闭合分类法，并明确“什么不该保存”：可由代码、git、当前状态派生的信息不应写入长期记忆。

**后台 extract memories。**  
每轮结束后 fire-and-forget 提取记忆。它和主 Agent 写记忆互斥：如果主 Agent 已经写了 memory 文件，后台提取跳过并推进游标。

**Session Memory 与 compact 协同。**  
Session Memory 使用固定结构模板，达到 token / tool call 阈值后增量提取。Auto compact 时，系统可以直接用 Session Memory 作为压缩摘要，减少一次完整 compact API 调用。

**Relevant Memories 双阶段召回。**  
先扫描 memory topic 文件的 frontmatter 和描述，再用轻量 side query 选择最多 5 个相关文件，注入为 attachment。已注入过的记忆会在 session 内去重，compact 后由于旧 attachment 被删，可重新召回。

优势：

- 记忆质量控制强，有分类法、排除规则、后台提取、主后台互斥。
- 注入方式细，支持常驻索引和按需召回两种路径。
- 与 prompt cache、compact、Agent isolation 深度协同。

短板：

- 工程复杂度高，对 Claude API、feature gate、prompt cache 机制依赖很强。
- 多条记忆路径同时存在，排障需要理解 CLAUDE.md、memdir、Session Memory、Agent Memory、attachments。

### 3.5 Hermes 的四层记忆

Hermes 的记忆系统介于 GenericAgent 的文件知识库和 Claude Code 的产品级记忆系统之间，更强调本地持久化和可扩展 provider。

四层可以概括为：

| 层 | 机制 | 载体 |
|---|---|---|
| 精炼记忆 | Agent 经验和用户画像 | `~/.hermes/memories/MEMORY.md`、`USER.md` |
| 会话历史 | 完整消息流与跨会话搜索 | `~/.hermes/state.db`，SQLite + FTS5 |
| 外部记忆 Provider | 插件式高级记忆后端 | `MemoryProvider` ABC，`plugins/memory/` |
| 上下文注入 | prefetch recall 临时注入 | `<memory-context>` fence |

Hermes 的关键设计是 **Frozen Snapshot**：

- 会话初始化时从 `MEMORY.md/USER.md` 读取并渲染 system prompt snapshot。
- 之后 `memory` 工具写入文件，但不改变当前会话的 system prompt。
- 当前会话内，模型通过工具响应看到最新状态。
- 这样保护 prefix cache 稳定性，也避免 system prompt 在会话中漂移。

Hermes 同时把完整对话历史存入 SQLite：

- `sessions` 表保存元数据、模型、token、费用、parent_session_id。
- `messages` 表遵循 OpenAI 消息格式，便于恢复。
- `messages_fts` 是 FTS5 content-sync 虚拟表，可全文搜索。

记忆召回方面，Hermes 的 `session_search` 会先用 FTS5 搜索，再按 session 分组、截断相关上下文，并用便宜模型做摘要，避免把原始历史整段塞回上下文。

优势：

- 本地优先，SQLite/FTS5 零额外服务依赖。
- Frozen Snapshot + write-through 兼顾 cache 稳定与持久更新。
- MemoryProvider 插件体系扩展性好，且最多一个外部 provider 避免冲突。

短板：

- `MEMORY.md/USER.md` 字符限制较小，需要模型主动整理。
- 单 SQLite 文件适合个人/小团队，不是分布式后端。
- provider、FTS、fence、snapshot 多机制并存，学习成本高于 GenericAgent。

### 3.6 记忆机制横向表

| 维度 | OpenClaw | GenericAgent | Claude Code | Hermes |
|---|---|---|---|---|
| 核心载体 | session store、transcript、memory plugins | L0-L4 文件层级 | CLAUDE.md、memdir、session memory、agent memory | MEMORY.md/USER.md、SQLite FTS5、provider |
| 常驻注入 | session/skills/system prompt contribution | L1 索引 + L0 引用 | CLAUDE.md 或 memory prompt section | Frozen snapshot |
| 按需召回 | memory runtime/search 插件 | L1 指针后 file_read | sideQuery 选择 Relevant Memories | prefetch + FTS/session_search/provider |
| 自动写入 | 依赖插件/runtime | `start_long_term_update` 触发模型写文件 | background extract + main agent direct write + dream | memory 工具 + provider sync |
| 会话历史 | transcript + sessions.json | backend history + L4 归档 | session transcript / messages + compact | SQLite `messages` |
| 防注入 | InputProvenance、inter-session prefix | L0 SOP 约束为主 | attachment 去重、memory 分类、权限沙箱 | `<memory-context>` fence + threat scan |
| 最大优势 | 与 Gateway/多通道状态打通 | 简洁、可读、可改 | 最完整、最 cache-aware | 本地持久化强、FTS 搜索实用 |
| 最大风险 | 状态分散 | 索引维护和安全弱 | 复杂且绑定 Claude 能力 | 全局状态和单库并发边界 |

---

## 4. 工具调用机制对比

### 4.1 共同点

四个系统都遵循类似的工具调用主链路：

```text
工具 schema/description 暴露给模型
  -> 模型返回 tool_call/tool_use
  -> Agent 解析参数
  -> 权限/可用性/安全检查
  -> 调用 handler/executor
  -> 标准化 tool_result
  -> 追加到消息历史并进入下一轮
```

共同工程问题包括：

- 工具名幻觉或不存在。
- 参数 JSON 类型不稳定。
- tool_call 和 tool_result 必须配对。
- 写入类工具和只读工具并发安全不同。
- 外部工具和插件可能扩大攻击面。
- 工具结果可能过大，需要裁剪。

### 4.2 OpenClaw：统一描述符 + owner/executor 分离

OpenClaw 工具系统的核心是 `ToolDescriptor`，字段包括：

- `name`
- `description`
- `inputSchema`
- `outputSchema`
- `owner`
- `executor`
- `availability`
- `annotations`
- `sortKey`

最值得注意的是 **owner 与 executor 分离**：

- `owner` 表示工具是谁声明的，可以是 core、plugin、channel、mcp。
- `executor` 表示工具实际路由到哪里执行，可以是 core executor、plugin tool、channel action、MCP server tool。

这让 OpenClaw 可以统一管理 core / plugin / channel / MCP 四类工具，而不是为每类工具写一套执行路径。

工具规划由 `buildToolPlan()` 完成：

1. 排序。
2. 检查名称唯一性。
3. 根据 availability 表达式求值。
4. 可用工具进入 `visible`，不可用工具进入 `hidden` 并携带诊断。
5. 转成面向模型的 `ToolProtocolDescriptor`，只保留 `name/description/inputSchema`。

OpenClaw 的工具系统和插件系统高度耦合。插件 manifest 负责静态 metadata 和 contracts，runtime 执行 `register(api)` 后才能注册能力。MCP 也被分成 Channel MCP、OpenClaw Tools MCP、Plugin Tools MCP 等层，协议适配和内部调用桥接分离。

优势：

- 多来源工具统一抽象清晰。
- hidden diagnostics 有利于控制面解释“为什么不可用”。
- owner/executor 分离适合 Gateway/Plugin/MCP 混合架构。

短板：

- 概念多，工具执行链可能跨 Gateway、plugin runtime、channel bridge、MCP server。
- 安全边界依赖 contracts、scope、approval、SecretRef、policy 多层协作，排查复杂。

### 4.3 GenericAgent：少量高权限原子工具 + Handler 约定映射

GenericAgent 默认工具数量少，但能力很大：

| 工具 | 作用 |
|---|---|
| `code_run` | 执行 Python/PowerShell/bash，可能安装依赖、操作系统 |
| `file_read` | 读取文件，支持行号、关键词、截断 |
| `file_write` | 覆盖/追加/前置写入 |
| `file_patch` | 唯一文本块替换 |
| `web_scan` | 扫描真实浏览器页面 |
| `web_execute_js` | 在浏览器执行 JS |
| `update_working_checkpoint` | 更新短期工作记忆 |
| `ask_user` | 向用户提问并中断 |
| `start_long_term_update` | 触发长期记忆提炼 |

工具链路是：

```text
assets/tools_schema.json
  -> LLM response.tool_calls
  -> agent_runner_loop
  -> BaseHandler.dispatch()
  -> GenericAgentHandler.do_<tool_name>()
  -> StepOutcome
```

`StepOutcome` 是核心协议：

- `data`：工具结果，序列化后作为 tool_result 回传。
- `next_prompt`：下一轮 user message 主体。
- `should_exit`：是否退出当前任务。

特别设计是 `no_tool`：模型没有工具调用时，系统不会直接结束，而是生成内部伪工具 `no_tool`，交给 `do_no_tool()` 判断是否接受最终回复、要求重试、处理假完成或空回复。

优势：

- 极简，新增工具路径清楚：schema + `do_xxx()`。
- 工具权限大，单个工具可以完成复杂环境探索。
- 同时支持 native tool 和文本 `<tool_use>` 协议。

短板：

- 安全默认偏弱，`code_run`、`file_write`、`web_execute_js` 都是高风险能力。
- 缺少 Claude Code / Hermes 那样细粒度权限管线。
- 工具少意味着模型需要更多自主组合，错误操作的影响也更集中。

### 4.4 Claude Code：Tool 接口 + buildTool 安全默认 + ToolSearch

Claude Code 的工具系统是四个系统中最产品化的。`Tool` 接口不仅包含执行逻辑，还包含：

- 输入 schema。
- 输出 schema。
- 权限检查。
- 只读/破坏性/并发安全标注。
- UI render 方法。
- 结果序列化。
- max result size。
- MCP info。
- deferred/alwaysLoad 等加载策略。

`buildTool()` 提供安全默认：

- `isConcurrencySafe()` 默认 `false`。
- `isReadOnly()` 默认 `false`。
- `isDestructive()` 默认 `false`。
- `checkPermissions()` 默认 allow，但外层权限管线兜底。

工具注册通过 `tools.ts` 的 `getAllBaseTools()`，再经过：

1. 编译期 `feature()` + 条件 `require()`。
2. 模块加载期 `process.env` 判断。
3. 运行时 `isEnabled()`。
4. deny rules。
5. MCP 工具合并。

工具执行时，`partitionToolCalls()` 根据每个工具的 `isConcurrencySafe()` 分批：

- 连续只读安全工具并发执行。
- 写入或不安全工具串行执行。
- 默认不安全，显式 opt-in 才并发。

Claude Code 的独特设计是 **ToolSearch / deferred tools**。当 MCP 工具太多时，不把全部 schema 放入上下文，只告诉模型有哪些 deferred tool name。模型需要时先调用 ToolSearch，返回 `tool_reference`，API 再展开完整工具定义。这显著降低大规模 MCP 工具的 token 开销。

优势：

- 安全默认、权限、并发、UI、schema、结果映射高度统一。
- ToolSearch 解决工具规模膨胀问题。
- 与 prompt cache 深度协同，例如工具排序稳定、built-in 与 MCP 分段。

短板：

- 对 Claude API beta 能力依赖较深，如 `tool_reference`、cache behavior。
- Tool 接口很大，新工具开发需要理解 UI/权限/上下文多个面。

### 4.5 Hermes：Python 中央注册表 + AST 自发现 + toolset 组合

Hermes 的工具系统核心是 `ToolRegistry` 和 `ToolEntry`。

`ToolEntry` 使用 `__slots__`，字段包括：

- `name`
- `toolset`
- `schema`
- `handler`
- `check_fn`
- `requires_env`
- `is_async`
- `description`
- `emoji`
- `max_result_size_chars`

工具文件在模块顶层调用 `registry.register()` 自注册。`discover_builtin_tools()` 不盲目 import 所有 `tools/*.py`，而是先用 AST 判断模块顶层是否存在 `registry.register()`，再 import 触发注册。这减少副作用，也保持启动可控。

`toolsets.py` 负责组合能力：

- 基础叶子 toolset，如 `web`、`file`、`terminal`。
- 复合 toolset，如 `debugging`、`safe`。
- 平台 toolset，如 `hermes-cli`、`hermes-telegram`。
- 动态 MCP toolset，如 `mcp-github`。

`model_tools.py:handle_function_call()` 是上层调度入口，包含：

1. `coerce_tool_args()` 参数类型强制转换。
2. `_AGENT_LOOP_TOOLS` 拦截。
3. plugin pre_tool_call hook。
4. registry dispatch。
5. plugin post_tool_call hook。

Hermes 的安全重点体现在工具实现层：

- Terminal：危险命令正则、Tirith、Smart Approval、人工审批。
- 文件工具：realpath/normpath、敏感文件、设备路径、二进制检测、staleness。
- MCP：环境变量过滤、错误脱敏、工具描述注入扫描、OSV 检查。
- `execute_code`：通过 RPC 间接调用 Hermes 工具，隔离 API key 和 Agent 内部状态。

优势：

- Python 插件式开发简单，新增工具成本低。
- AST 预筛选是很实用的工程优化。
- toolset 组合适合 CLI/Gateway/平台差异。

短板：

- 运行时全局 registry 和 `_last_resolved_tool_names` 等状态需要锁和 save/restore。
- 单个工具文件或核心调度文件偏大，长期维护难度高。

### 4.6 工具系统横向表

| 维度 | OpenClaw | GenericAgent | Claude Code | Hermes |
|---|---|---|---|---|
| 工具定义 | `ToolDescriptor` | JSON schema + `do_xxx()` | `Tool` interface + `buildTool()` | `ToolEntry` + register |
| 工具来源 | core/plugin/channel/MCP | 内置 9 个原子工具 | 40+ 内置 + MCP +插件/Agent | 内置工具 + MCP +插件 |
| 路由方式 | owner/executor ref | Handler 方法名约定 | tool.call + orchestration | registry.dispatch |
| 可用性 | availability expression | 较少形式化 | `isEnabled` + deny rules | `check_fn` + toolset |
| 并发策略 | runtime lane/tool policy | 多工具逐个执行为主 | `isConcurrencySafe` 分区 | safe/path/never 分类 |
| 权限体系 | approval/scope/plugin contracts | 较轻 | 多步权限管线 | Smart Approval/Tirith/path security |
| MCP | 一等集成，多 server 类型 | 文档未突出 | 深度集成，ToolSearch | 动态注册到 registry |
| 最大优势 | 多来源统一 | 简单高效 | 安全和规模最成熟 | 自注册和 toolset 很灵活 |
| 最大风险 | 链路复杂 | 高权限工具风险 | 接口复杂/API 绑定 | 全局状态和复杂工具实现 |

---

## 5. 上下文管理对比

### 5.1 共同点

四个系统都意识到上下文窗口不是“消息数组无限追加”，都采用了某种压缩或裁剪：

- 工具结果过大必须截断。
- 历史消息需要裁剪或摘要。
- tool_call/tool_result 配对不能破坏。
- system prompt 应尽量稳定，保护缓存。
- 当前任务关键状态需要重复注入或在 compact 后恢复。

差异在于实现深度：Claude Code 最精细，Hermes 最独立工程化，OpenClaw 偏 runtime/agent/session 一体化，GenericAgent 偏 prompt 层锚点和简单裁剪。

### 5.2 OpenClaw 的上下文管理

OpenClaw 的上下文管理散落在 Agent runtime 和 session 系统中。关键能力包括：

- `ensureContextEnginesInitialized()`。
- `resolveContextEngine()`。
- `createSystemPromptOverride()`。
- `contextLimits` Agent 配置。
- `compact.*` 与 `runPostCompactionSideEffects()`。
- `post-compaction loop guard`。
- `tool-result-truncation.ts`。
- session store 中的 `compactionCount` 和 `compactionCheckpoints`。

OpenClaw 的 context 是围绕会话和 runtime 执行计划运转的：输入进入 `runEmbeddedPiAgent` 后，会构建 runtime/auth/model plan，加载插件，进入 session/global lane，再处理模型流、工具、retry、compaction、failover 和持久化。

它还特别强调消息来源安全，`InputProvenance` 区分：

- `external_user`
- `inter_session`
- `internal_system`

跨会话消息会加上前缀说明，不让模型把它误解成直接用户指令。

### 5.3 GenericAgent 的上下文管理

GenericAgent 的上下文管理轻量但有效：

**System prompt 拼接：**

```text
assets/sys_prompt*.txt
  + 当前日期
  + get_global_memory() 读取 L1/L0 引用
```

**每轮消息：**

`agent_runner_loop()` 每轮只构造新的 user message，包含工具结果和 `next_prompt`。完整对话历史主要存在 backend session，例如 native tool 模式下的 `llmclient.backend.history`。

**历史裁剪：**

`trim_messages_history()` 根据 `context_win` 裁剪历史，并先压缩旧的 `<thinking>`、`<tool_use>`、`<tool_result>` 标签。

**工作记忆锚点：**

多数工具执行后，`_get_anchor_prompt()` 注入：

- 最近摘要。
- 当前轮次。
- `key_info`。
- `related_sop`。
- passed sessions。

**工具 schema 刷新：**

文本工具协议下，`ToolClient` 会缓存工具说明以省 token；每 10 轮清空 `client.last_tools`，强制重新注入工具协议，避免模型遗忘。

优点是简单；缺点是缺少 Claude/Hermes 那样的多级 compact、缓存精细控制和工具结果预算模型。

### 5.4 Claude Code 的上下文管理

Claude Code 的上下文管理是四个系统里最复杂的，主要包含：

**Token 预算：**

- `getEffectiveContextWindowSize()`：context window 减去输出预留。
- `getAutoCompactThreshold()`：有效窗口减 13K buffer。
- `calculateTokenWarningState()`：warning/error/autoCompact/blocking。

**Microcompact：**

- time-based microcompact：cache 冷却后直接替换旧工具结果。
- cached microcompact：使用 `cache_edits`，本地消息不变，在 API 层删除旧 tool_result，保护 prompt cache。
- API-level context management：向 Claude API 声明清理旧 thinking/tool results 的策略。

**Full Compact：**

- Auto compact 触发后优先尝试 Session Memory Compact。
- 如果不可用，则 `compactConversation()` 调模型生成结构化总结。
- compact prompt 要求模型输出 `<summary>`，并剥离 `<analysis>`。
- compact 后恢复最近文件、plan、skill、deferred tools、agent/MCP 指令等附件。

**FileStateCache：**

它不是简单缓存，而是追踪模型看过哪些文件、是否为 partial view、是否可安全编辑。Compact 后从 pre-compact readFileState 恢复最多 5 个关键文件，每个有 token 限制。

**Post-compact cleanup：**

清理 microcompact state、system prompt section cache、classifier approvals、session messages cache 等。子 Agent compact 不清主线程缓存，避免共享模块状态被误伤。

优势：

- 对 token 成本、prompt cache、工具结果、文件恢复的控制最细。
- 对 Claude API 特性利用充分。
- withhold-recover 机制可暂扣可恢复错误，成功恢复后用户无感。

短板：

- 高度复杂，需要理解 feature gate、cache breakpoint、cache_edits、attachments、file state、session memory。
- 很多优化依赖 Claude API 专有能力。

### 5.5 Hermes 的上下文管理

Hermes 的上下文管理是一个独立的 `ContextEngine/ContextCompressor` 体系，有三道防线：

1. **工具输出预裁剪。**
   - Terminal 50K chars，40/60 head-tail。
   - 文件读取 100K chars。
   - Web extract 5K chars。
   - Browser snapshot 8K chars，行边界截断。
   - Process rolling buffer 200K。

2. **API 调用前 preflight。**
   - `estimate_request_tokens_rough()` 粗略估算，包含 tool schema。
   - 超阈值时最多预压缩 3 轮。

3. **API 返回后压缩。**
   - 使用真实 `usage.prompt_tokens` 和 completion tokens。
   - 若缺 usage，回退到粗估。

压缩算法采用 head/middle/tail 三段式：

- head：默认保护前 3 条，通常是 system prompt 和初始任务。
- tail：按 token budget 从后往前保护，而不是固定消息数。
- middle：由 LLM 生成结构化 summary。

重要防护：

- `_align_boundary_forward/backward` 不切断 tool_call/tool_result。
- `_sanitize_tool_pairs()` 删除孤立 result 或补 stub result。
- summary role 避免连续相同 role，如果冲突就合并进 tail。
- Anti-thrashing：连续低效压缩时跳过，避免浪费。
- summary 失败时插入静态 fallback marker，而不是崩溃。

Hermes 的优势是模型无关、工程独立、容灾清楚；短板是相比 Claude Code 缺少专有 prompt cache / cache edits 层面的精细优化。

### 5.6 上下文管理横向表

| 维度 | OpenClaw | GenericAgent | Claude Code | Hermes |
|---|---|---|---|---|
| 触发基础 | runtime context engine + session compaction | `context_win` + loop anchor | token warning/auto compact/API errors | preflight + post-response usage |
| 轻量清理 | tool-result truncation | tag compression | microcompact/cache_edits | tool output pruning |
| 重量压缩 | compact lifecycle | L4 归档，不是在线 compact 主机制 | full compact / session memory compact | LLM structured summary |
| 缓存意识 | session/prompt/runtime | 较弱 | 最强，prompt cache 深度设计 | frozen snapshot/prompt caching |
| 结构完整性 | session/history utils | 依赖 backend | tool pair、file state、attachments | tool pair sanitization、role conflict |
| 当前任务锚点 | session state + skills snapshot | `_get_anchor_prompt()` | file attachments/session memory/todos | tail token budget + summary |
| 最大优势 | 与 Gateway/session 结合 | 简洁可控 | 最省 token、最产品化 | 独立通用、容灾好 |
| 最大风险 | 多模块分散 | 长任务压缩不够强 | 复杂且 API 绑定 | summary 质量依赖辅助模型 |

---

## 6. 执行编排对比

### 6.1 OpenClaw：Gateway + lane + TaskFlow

OpenClaw 的执行编排是 Gateway 中心化的：

```text
用户/通道/CLI
  -> Gateway RPC / ingress
  -> agentCommand / agentCommandFromIngress
  -> prepareAgentCommandExecution
  -> ACP 或 Embedded Pi 路径
  -> model fallback / tools / compaction
  -> session store / transcript / pending delivery
  -> channel delivery
```

关键点：

- 本地可信入口 `agentCommand()` 和网络入口 `agentCommandFromIngress()` 分离，网络边界必须显式声明 trust。
- 默认走 Embedded Pi；也支持 ACP，把 turn 转给 Claude Code、Codex 等外部 harness。
- 模型选择支持 allowlist、session override、fallback chain、Auth Profile cooldown。
- 并发通过 global/session lane 控制，例如 `enqueueCommandInLane()`、`resolveGlobalLane()`、`resolveSessionLane()`。
- active run 状态可 queue/abort/wait。
- 最终回复先写入 `pendingFinalDelivery`，投递成功后再清除，支持进程重启后恢复交付。

任务系统支持：

- `TaskRecord`：`queued/running/succeeded/failed/timed_out/cancelled/lost`。
- `TaskFlowRecord`：`queued/running/waiting/blocked/succeeded/failed/cancelled/lost`。
- `DetachedTaskLifecycleRuntime`：ACP/subagent 等外部 runtime 可注册自己的生命周期处理。
- SQLite task registry，支持重启恢复。

OpenClaw 适合长生命周期、多平台、多 Agent、多 provider 的系统；代价是 orchestration 层复杂。

### 6.2 GenericAgent：单进程 FIFO 队列 + StepOutcome 循环

GenericAgent 编排最简单：

```text
frontends / reflect
  -> GeneraticAgent.put_task()
  -> task_queue
  -> GeneraticAgent.run()
  -> agent_runner_loop()
  -> tools
  -> display_queue
```

`put_task()` 为每个任务创建私有 `display_queue`，前端只要读取 `next/done`。核心循环执行：

1. 构造 messages。
2. 调 `client.chat(messages, tools_schema)`。
3. 解析 tool calls。
4. 逐个 dispatch。
5. 收集 tool results 和 next prompts。
6. 调 `turn_end_callback()`。

reflect/scheduler 也不是真正独立执行器，它只是返回 prompt，再交给同一个 Agent 队列执行。

优势：

- 极易理解和二次开发。
- 前端多样但核心复用一个 `put_task()`。
- 工具、记忆、调度都回到同一 Agent loop，概念少。

短板：

- 默认 FIFO，无优先级和多租户隔离。
- 子 Agent/任务系统不如 Claude/OpenClaw/Hermes 成熟。
- 高并发、多平台场景需要额外工程。

### 6.3 Claude Code：AsyncGenerator query + Agent + Task system

Claude Code 的核心对话循环是 `query()` / `queryLoop()`，返回 AsyncGenerator：

- 可以流式 yield token、tool progress、system message、attachment。
- 调用方可以 `.return()` 中断。
- 自然支持背压。

主循环使用显式 `State`，包含：

- messages。
- toolUseContext。
- autoCompactTracking。
- max output token recovery。
- pendingToolUseSummary。
- stopHookActive。
- turnCount。
- transition。

错误恢复和 continue 点包括：

- context collapse drain。
- reactive compact retry。
- max output tokens escalate。
- max output tokens recovery。
- stop hook blocking。
- token budget continuation。
- next turn。

子 Agent 方面，Claude Code 有：

- `AgentDefinition`：built-in/custom/plugin agent。
- `runAgent()`：完整生命周期，包括模型解析、MCP 初始化、context clone、query loop、finally cleanup。
- `createSubagentContext()`：默认隔离，显式共享。
- Fork Subagent：继承父上下文，最大化 prompt cache 命中。
- Async Agent lifecycle：后台运行，状态先完成再丰富化通知。

任务系统统一管理：

- local bash。
- local agent。
- remote agent。
- in-process teammate。
- dream task。
- main session background task。

优势：

- 任务、UI、权限、Agent、工具、缓存协同很深。
- 子 Agent 隔离语义成熟：readFileState、memory triggers、skill discovery、denial tracking、AppState 穿透都细分。
- Fork 模式特别适合高 cache 命中的并行探索。

短板：

- 编排层很复杂，feature gate 多。
- Agent/task/context/UI 状态耦合较深。

### 6.4 Hermes：AIAgent 主循环 + IterationBudget + delegate_task + Gateway

Hermes 的 `AIAgent.run_conversation()` 是经典 ReAct：

```text
LLM call
  -> tool dispatch
  -> append result
  -> repeat
```

但它叠加了很多工程机制：

- `IterationBudget`：线程安全迭代预算。
- Grace call：预算耗尽后允许一次无工具 API 调用，让模型总结。
- 工具并发：`_PARALLEL_SAFE_TOOLS`、`_NEVER_PARALLEL_TOOLS`、`_PATH_SCOPED_TOOLS`。
- 自修复：工具名 hallucination 自动修复或返回 available tools。
- Cooperative interruption：用 flag，不用异常，确保每个 tool_call 都有 result。
- Streaming 优先：即使无消费者也用 streaming，以获得 stale stream 健康检查。

`delegate_task` 是 Hermes 的多 Agent 核心：

- 子 Agent 拥有全新 conversation history。
- `skip_context_files=True`，`skip_memory=True`，不继承父上下文。
- 工具集是父工具集交集再过滤黑名单。
- `ThreadPoolExecutor` 并行多个子任务。
- `MAX_DEPTH=2`，黑名单、toolset 过滤、深度检查三重防递归。
- 子 Agent 返回 summary、tokens、duration、tool_trace。

Gateway 方面：

- 多平台 adapter 统一成 `MessageEvent`。
- `build_session_key()` 决定会话隔离。
- `_active_sessions`、`_pending_messages` 处理运行中新消息、中断和排队。
- `PENDING_SENTINEL` 防止异步创建重复 Agent。
- `GatewayStreamConsumer` 用 `queue.Queue` 桥接同步 Agent 线程和异步平台投递。

优势：

- 通用性强，Python 技术栈下覆盖 CLI/Gateway/Cron/Batch。
- IterationBudget、cooperative interrupt、tool pair 完整性都很务实。
- 子 Agent 安全隔离强。

短板：

- 子 Agent fresh context 不继承父历史，安全但可能重复探索。
- `run_agent.py`、`gateway/run.py` 等核心文件偏大。
- 全局状态需要 save/restore，说明架构还有可拆分空间。

### 6.5 执行编排横向表

| 维度 | OpenClaw | GenericAgent | Claude Code | Hermes |
|---|---|---|---|---|
| 主循环形态 | Gateway + Embedded/ACP runtime | 单进程 generator loop | AsyncGenerator query loop | `run_conversation()` ReAct loop |
| 队列/并发 | global/session lane | FIFO task_queue | Task system + Async agents | IterationBudget + ThreadPool |
| 子 Agent | ACP/subagent/task flow | 文档有 subagent SOP，但核心不突出 | AgentTool/Fork/Async/Teammate | `delegate_task` fresh child |
| Gateway | 核心能力 | 多前端适配但非 daemon | CLI/SDK 为主 | 19 平台 Gateway |
| 取消/中断 | abort active embedded run/task cancel | stop_sig/code_stop_signal | AbortController/task kill | cooperative flag + child interrupt |
| 持久恢复 | sessions.json、transcript、task SQLite | model response logs/L4 | session/tasks/transcripts | SQLite SessionDB/checkpoint |
| 最大优势 | 多通道长期运行 | 极简清楚 | 编排与产品体验最成熟 | 通用全栈、状态可搜索 |
| 最大风险 | 状态和 runtime 复杂 | 缺少强多租户调度 | 内部状态复杂 | 全局状态和大文件维护 |

---

## 7. 相同点总结

### 7.1 都是 ReAct 变体

四个系统都基于“模型决定行动，工具反馈进入下一轮”的模式：

```text
user message
  -> model
  -> tool call
  -> tool result
  -> model
  -> final response
```

差异只是工程化程度：

- GenericAgent 是最薄的一层。
- Hermes 加入预算、中断、并行和压缩。
- Claude Code 加入流式、缓存、附件、权限、任务和子 Agent。
- OpenClaw 把这个循环嵌进 Gateway、plugin runtime、session/task/provider 系统。

### 7.2 都把工具 schema 视为模型与真实世界的契约

工具 schema 不只是类型定义，也是行为指导：

- GenericAgent 把工具说明写进 `assets/tools_schema.json` 或 native tools。
- Claude Code 的 Tool prompt、description、searchHint、ToolSearch 都服务于模型选择工具。
- Hermes 会动态重写 schema，避免描述不可用工具导致模型幻觉。
- OpenClaw 将完整 descriptor 转成模型可见 protocol descriptor。

### 7.3 都需要压缩与摘要

长任务中，上下文无限增长是必然问题：

- GenericAgent 用 history trim + anchor prompt。
- Hermes 用 ContextCompressor。
- Claude Code 用 microcompact/full compact/session memory compact。
- OpenClaw 用 context engine + compaction lifecycle。

### 7.4 都在做“隔离”

隔离对象不同：

- 工具权限隔离。
- 子 Agent 上下文隔离。
- 会话隔离。
- 记忆注入隔离。
- plugin/runtime 边界隔离。

隔离程度最高的是 Claude Code 和 Hermes；OpenClaw 更偏系统边界；GenericAgent 更偏 prompt 和 SOP 约束。

---

## 8. 关键差异总结

### 8.1 架构哲学不同

| 系统 | 架构哲学 |
|---|---|
| OpenClaw | 微内核 + 插件 + Gateway，核心只定义边界和编排 |
| GenericAgent | 少即是多，少量高权限工具 + 文件记忆驱动自治 |
| Claude Code | 生产级 CLI 产品，围绕 Claude API 做深度纵向优化 |
| Hermes | 通用 Agent OS，本地持久化 + 可扩展工具/记忆/Gateway |

### 8.2 记忆写入责任不同

| 系统 | 谁写记忆 |
|---|---|
| OpenClaw | runtime/plugin/memory service/session side effects |
| GenericAgent | 主 Agent 通过普通文件工具写 |
| Claude Code | 主 Agent、后台 extract Agent、Session Memory、Dream、Agent Memory |
| Hermes | memory 工具、MemoryManager、provider sync/session_search |

### 8.3 工具安全默认不同

| 系统 | 安全风格 |
|---|---|
| OpenClaw | policy/contracts/scope/approval 多层系统化 |
| GenericAgent | 信任本地 Agent，高权限工具直接给模型 |
| Claude Code | fail-closed 工具元数据 + 多步权限管线 |
| Hermes | 工具实现内嵌安全检查 + approval + registry/toolset 限制 |

### 8.4 子 Agent 上下文策略不同

| 系统 | 子 Agent 策略 |
|---|---|
| OpenClaw | ACP/subagent/task runtime，可接外部 harness |
| GenericAgent | 核心文档不以子 Agent 为主，更多靠 SOP/工具 |
| Claude Code | fresh/fork/async/in-process 多模式，支持 cache-sharing fork |
| Hermes | fresh child，默认不继承 memory/context files，返回 summary/tool_trace |

### 8.5 状态持久化后端不同

| 系统 | 状态后端 |
|---|---|
| OpenClaw | `sessions.json` + transcript + task SQLite + config/auth/profile |
| GenericAgent | 文件系统 + backend history + temp logs + L4 zip/txt |
| Claude Code | `.claude` 目录、memdir、session memory、task output、transcript |
| Hermes | SQLite WAL/FTS5 + Markdown memory + jobs/checkpoint |

---

## 9. 各系统优势与短板

### 9.1 OpenClaw

**优势：**

- 多通道/多平台能力强，Gateway 是一等核心。
- 插件 manifest 和 contracts 使控制面可在不加载 runtime 的情况下读取 metadata。
- owner/executor 分离适合 core/plugin/channel/MCP 混合工具。
- session store 和 task flow 支持长生命周期运行、取消、恢复、投递重试。
- provider 模型抽象完整，支持 fallback、auth profile、cooldown、tool schema compat。

**短板：**

- 学习曲线最陡之一，需要同时理解 Gateway、PluginRuntime、Channel、MCP、ACP、TaskFlow、SessionEntry。
- 状态分散在多个存储与 runtime cache 中，排障链路长。
- Stateful daemon 带来端口、鉴权、热配置、drain、重启恢复等运维问题。

### 9.2 GenericAgent

**优势：**

- 极简，核心路径短，适合学习和快速二开。
- L0-L4 记忆结构非常清楚，人类可读可维护。
- 少量原子工具给模型很强环境操作能力。
- 同时支持 native tool 与文本 `<tool_use>`，模型兼容性好。
- 工作记忆 anchor 对长任务很实用。

**短板：**

- 高权限工具风险集中，缺少细粒度安全管线。
- FIFO 队列和调度能力有限。
- 上下文压缩相对简单，长时间复杂任务可能不如 Claude/Hermes 稳定。
- 记忆索引依赖模型维护，缺少强一致事务。

### 9.3 Claude Code

**优势：**

- 上下文经济性最好：prompt cache、microcompact、cache_edits、full compact、session memory compact。
- 工具系统成熟：`buildTool()`、安全默认、权限管线、并发分区、UI render。
- Agent 系统成熟：built-in/custom/plugin、fork/fresh/async、多层 context isolation。
- 任务系统与 UI/通知/后台输出结合紧密。
- Memory 机制最完整，覆盖静态指令、自动记忆、会话记忆、Agent 记忆、按需召回。

**短板：**

- 高度绑定 Claude API 能力和内部 feature gate。
- 工程复杂，外部复刻成本高。
- Tool/Agent/Task/AppState/Prompt Cache 交织，局部改动容易影响缓存或 UI。

### 9.4 Hermes

**优势：**

- 本地优先状态工程扎实：SQLite WAL、FTS5、随机抖动写重试、session lineage。
- 工具注册表和 toolset 组合灵活，适合 Python 插件生态。
- 记忆系统兼顾 Markdown 精炼记忆、FTS 历史搜索和 provider 扩展。
- Terminal/file/MCP/browser 安全边界较丰富。
- `delegate_task` 简洁清楚，fresh context 隔离强，适合并行研究。
- Gateway、Cron、Batch、Trajectory 让它更像通用 Agent runtime。

**短板：**

- 核心文件偏大，职责重。
- 全局状态较多，需要锁、save/restore、sentinel 等防护。
- 子 Agent 不继承父上下文，安全但可能重复探索。
- SQLite 单文件不是分布式状态后端。
- 部分安全策略为可用性选择 fail-open，高安全环境需加固。

---

## 10. 可迁移设计模式

### 10.1 索引-实体分离

GenericAgent 的 L1/L2-L3、Claude Code 的 MEMORY.md/topic files、Hermes 的精炼记忆/FTS 历史都证明：不要把所有记忆常驻上下文。更好的结构是：

```text
小索引常驻
  -> 按需读取实体
  -> 注入当前最相关内容
```

适用于任何长期运行 Agent。

### 10.2 工具安全默认

Claude Code 的 `buildTool()` 给出一个强模式：

- 默认不并发。
- 默认非只读。
- 默认不认为安全。
- 需要显式 opt-in。

Hermes 的工具分类和 OpenClaw 的 availability 也可以吸收该思想。

### 10.3 tool_call/result 完整性保护

Hermes 和 Claude Code 都强调 tool_call 必须有 tool_result。压缩、中断、fallback、取消时都要补齐 stub result。否则模型 API 或下一轮上下文会出错。

### 10.4 Frozen Snapshot

Hermes 的 Frozen Snapshot 和 Claude Code 的 system prompt section cache 都体现了同一个原则：

> 会话内 system prompt 应尽量稳定，动态信息通过工具响应或 attachment 注入。

这对降低成本和提高可复现性很重要。

### 10.5 默认隔离，显式共享

Claude Code 的 `createSubagentContext()` 是最佳范例。子 Agent 默认隔离：

- 文件读取状态。
- memory triggers。
- skill discovery。
- UI callbacks。
- permission decisions。

但任务注册、abort、必要指标可以显式共享。Hermes 的 `delegate_task` 也采用 fresh context 强隔离。

### 10.6 渐进式上下文压力响应

Claude Code 和 Hermes 都不是等爆窗才处理，而是分层：

1. 工具源头截断。
2. 本地轻量清理。
3. API 前预检。
4. LLM summary。
5. compact 后恢复。
6. 失败熔断或 anti-thrashing。

### 10.7 Gateway 与 Agent loop 解耦

OpenClaw 和 Hermes 的 Gateway 都说明，多平台 Agent 不应让每个平台理解 LLM 和工具细节。平台层只负责：

- 消息归一化。
- session key。
- 投递。
- 中断/队列。

Agent loop 负责模型、工具、上下文。

### 10.8 Dynamic Schema Rewriting

Hermes 根据实际可用工具动态改写 `execute_code` 和 `browser_navigate` schema，减少模型幻觉。这一模式很值得迁移：

> schema 应描述当前真实能力，而不是静态理想能力。

### 10.9 后台记忆提取与主 Agent 互斥

Claude Code 的 extract memories 使用后台 Agent，但如果主 Agent 已写 memory，则跳过后台写入。这避免了重复写入和竞态。任何自动记忆系统都应有类似互斥。

---

## 11. 选型建议

| 场景 | 更适合的设计 |
|---|---|
| 想学习 Agent 最小闭环 | GenericAgent |
| 想快速做本地自动化 Agent | GenericAgent 或 Hermes |
| 想做多平台聊天/消息 Gateway | OpenClaw 或 Hermes |
| 想做生产级 Coding CLI | Claude Code 的架构模式最值得参考 |
| 想做插件/工具生态 | OpenClaw 或 Hermes |
| 想做强上下文压缩和缓存优化 | Claude Code |
| 想做本地持久化和历史搜索 | Hermes |
| 想做多模型/provider fallback | OpenClaw 或 Hermes |
| 想做子 Agent 并行研究 | Claude Code fork/async 或 Hermes delegate_task |

如果要设计一个新 Agent 系统，可以组合如下：

1. 采用 **GenericAgent 的极简主循环** 作为第一版，不要一开始引入过度框架。
2. 采用 **Hermes 的 ToolRegistry/toolset** 组织 Python 工具。
3. 采用 **Claude Code 的 buildTool 安全默认和上下文压缩梯度**。
4. 采用 **GenericAgent 的 L0-L4 或 Claude Code 的分类法** 做长期记忆。
5. 采用 **Hermes 的 SQLite/FTS5** 做本地会话历史。
6. 如果需要多平台，再采用 **OpenClaw/Hermes Gateway 的 session key + adapter** 模式。
7. 如果需要多 Agent，再采用 **Claude Code 默认隔离显式共享** 或 **Hermes fresh child + tool_trace summary**。

---

## 12. 总结

四个系统代表了 Agent 工程的四种路线：

- **GenericAgent** 证明了一个 Agent 可以很小：核心 loop、handler、少数高权限工具、文件记忆，就能形成完整自治能力。
- **Claude Code** 证明了生产级 CLI Agent 的复杂性主要不在“调用模型”，而在上下文经济、权限安全、工具规模、任务生命周期和记忆质量。
- **Hermes** 证明了通用 Agent runtime 需要本地持久化、搜索、工具注册、Gateway、批处理、子 Agent 和安全审批等系统工程。
- **OpenClaw** 证明了当 Agent 进入多通道、多 provider、多插件、多会话场景后，Gateway 和 runtime 边界会成为主架构问题。

最终，热门 Agent 系统的共同方向是：

1. **从单轮聊天走向长生命周期任务。**
2. **从工具列表走向工具操作系统。**
3. **从简单 history 走向分层记忆和按需召回。**
4. **从追加上下文走向压缩、缓存和恢复。**
5. **从单 Agent 走向隔离子 Agent 和任务编排。**
6. **从 demo 权限走向可审计、安全默认和多层防护。**

这些方向比具体框架更重要。真正可迁移的不是某个文件或类名，而是背后的工程约束：上下文有限、工具危险、记忆会污染、并发会泄漏状态、用户需要可恢复的长任务。四个系统分别在这些约束上给出了不同答案。

