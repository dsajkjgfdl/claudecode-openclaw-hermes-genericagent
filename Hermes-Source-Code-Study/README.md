# 深入 Hermes Agent 源码

> 基于 [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent) v0.9.0 版本线的深度源码分析系列。

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

## 这本书是什么

这是一本面向 **1-3 年 Python 经验程序员** 的 Hermes Agent 源码解析书。

读完后你将：
- 理解 Hermes 的整体架构与每个模块的设计思想
- 掌握 Agent 系统中的核心设计模式
- 具备上手开发、贡献代码的能力

**写作风格：** 每篇以一个引导问题开头，逐文件逐函数拆解源码（精确到行号），配以 Mermaid 架构图，结尾提炼可迁移的设计模式。中文为主，技术术语保留英文原文。

---

## 目录

### Part 1: 全局架构（4 篇）

| # | 章节 | 关键词 |
|---|------|--------|
| [01](docs/01-项目全景.md) | **项目全景** | 技术栈选型、模块依赖全景、文件依赖链 |
| [02](docs/02-启动链路.md) | **启动链路** | argparse 路由、profile 预处理、env/bootstrap、Agent 初始化 |
| [03](docs/03-配置系统.md) | **配置系统** | 多层配置合并、profile 机制、gateway 配置桥接、config migration |
| [04](docs/04-状态持久化.md) | **状态持久化** | SessionDB、SQLite WAL、FTS5 全文搜索、session 分裂与链接 |

### Part 2: AI 核心（6 篇）

| # | 章节 | 关键词 |
|---|------|--------|
| [05](docs/05-对话循环.md) | **对话循环** | 消息推进、tool_calls 处理、iteration budget、中断恢复 |
| [06](docs/06-System-Prompt-工程.md) | **System Prompt 工程** | 分段构建、context file 注入与安全扫描、skills index |
| [07](docs/07-上下文管理.md) | **上下文管理** | head/tail 保护、tool output 预裁剪、iterative summary |
| [08](docs/08-多模型适配.md) | **多模型适配** | OpenAI 兼容层、Anthropic 原生适配、metadata 注册表、smart routing |
| [09](docs/09-辅助客户端与成本控制.md) | **辅助客户端与成本控制** | 辅助 LLM 调用、凭证池与轮转、定价引擎、rate limit 追踪 |
| [10](docs/10-错误处理与韧性.md) | **错误处理与韧性** | 错误分类器、retry 策略、优雅降级、模型回退 |

### Part 3: 工具系统（5 篇）

| # | 章节 | 关键词 |
|---|------|--------|
| [11](docs/11-工具注册表设计.md) | **工具注册表设计** | AST 自发现、ToolEntry `__slots__`、toolset 分组、import chain |
| [12](docs/12-终端工具深度剖析.md) | **终端工具深度剖析** | 命令执行生命周期、前置安全检查、持久会话、后台进程注册表 |
| [13](docs/13-文件与代码工具.md) | **文件与代码工具** | 文件读写/搜索/patch、代码沙箱执行 |
| [14](docs/14-Web-与浏览器工具.md) | **Web 与浏览器工具** | Parallel + Firecrawl 搜索、Browserbase 自动化、CamoFox 反检测 |
| [15](docs/15-MCP-协议实现.md) | **MCP 协议实现** | MCP 客户端/服务端双向实现、OAuth 认证 |

### Part 4: 智能体系统（4 篇）

| # | 章节 | 关键词 |
|---|------|--------|
| [16](docs/16-子智能体委托.md) | **子智能体委托** | context 隔离、toolset 限制、batch 并行、MAX_DEPTH 防递归 |
| [17](docs/17-记忆系统.md) | **记忆系统** | MemoryManager 编排、Provider/Plugin 扩展、MEMORY.md 持久化、FTS5 搜索 |
| [18](docs/18-技能系统.md) | **技能系统** | 技能发现/加载/执行、Skills Hub 生态、自我改进、安全沙箱 |
| [19](docs/19-安全体系.md) | **安全体系** | Tirith 安全引擎、path traversal 防护、prompt injection 扫描、URL 安全 |

### Part 5: 平台与基础设施（4 篇）

| # | 章节 | 关键词 |
|---|------|--------|
| [20](docs/20-CLI-交互界面.md) | **CLI 交互界面** | Rich + prompt_toolkit TUI、slash 命令注册表、KawaiiSpinner、皮肤引擎 |
| [21](docs/21-消息网关.md) | **消息网关** | 统一网关架构、会话上下文组装、消息投递、跨平台 slash 命令 |
| [22](docs/22-终端后端.md) | **终端后端** | local/docker/ssh/daytona/modal/singularity 的统一抽象 |
| [23](docs/23-批处理与轨迹基础设施.md) | **批处理与轨迹基础设施** | Cron 调度、batch run、trajectory 压缩、agent environment 衔接 |

### Part 6: 总结与实战（2 篇 + 3 篇附录）

| # | 章节 | 关键词 |
|---|------|--------|
| [24](docs/24-日志与可观测性.md) | **日志、用量统计与可观测性** | 结构化日志、usage insights、rate limit 追踪、trajectory 保存与回放 |
| [25](docs/25-架构模式总结.md) | **架构模式总结** | 从 Hermes 中提炼的 7-10 个可迁移设计模式 |
| [附录 A](docs/附录/A-实战-添加一个新工具.md) | **实战：添加一个新工具** | 完整开发流程 |
| [附录 B](docs/附录/B-实战-编写一个自定义技能.md) | **实战：编写一个自定义技能** | 技能开发与发布 |
| [附录 C](docs/附录/C-实战-接入一个新通信平台.md) | **实战：接入一个新通信平台** | 平台适配器开发 |

---

## 推荐阅读路线

| 路线 | 篇数 | 适合人群 | 章节序列 |
|------|------|---------|----------|
| ⚡ **入门路线** | 7 篇 | 快速建立全局认知 | 01 → 02 → 05 → 11 → 16 → 17 → 25 |
| 🤖 **AI 工程路线** | 10 篇 | 深入 Agent 设计 | 01 → 05 → 06 → 07 → 08 → 09 → 11 → 16 → 17 → 18 |
| 🔧 **工具开发路线** | 8 篇 | 上手贡献代码 | 01 → 03 → 11 → 12 → 13 → 附录A → 附录B → 附录C |
| 📚 **完整路线** | 28 篇 | 按顺序通读全部章节 | 01 → 02 → ... → 25 → 附录A → 附录B → 附录C |

---

## 关于

本书基于 Hermes Agent **v0.9.0 版本线**源码撰写。Hermes 是一个由 [NousResearch](https://github.com/NousResearch) 开发的开源 AI Agent 框架，基于 Python 构建，集成了丰富的工具系统、多模型适配、记忆管理、技能系统等特性。

**前作：** [Claude-Code-Source-Study](https://github.com/luyao618/Claude-Code-Source-Study) — 25 篇 Claude Code 深度源码分析系列

---

## License

[MIT](LICENSE)
