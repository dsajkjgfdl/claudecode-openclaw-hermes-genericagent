# 热门 Agent 系统架构对比与源码研读

> 对当前主流 AI Agent 系统的横向架构对比，以及对其中两个系统的逐文件、逐函数级源码研读。

## 这个仓库是什么

本仓库由三部分组成：

| 目录 | 内容 | 篇幅 |
|------|------|------|
| [`热门Agent系统架构对比研究报告.md`](./热门Agent系统架构对比研究报告.md) | OpenClaw / GenericAgent / Claude Code / Hermes 四大系统的横向架构对比 | 单篇长报告 |
| [`Claude-Code-Source-Study/`](./Claude-Code-Source-Study/) | Claude Code 源码深度解析系列 | 26 篇 |
| [`Hermes-Source-Code-Study/`](./Hermes-Source-Code-Study/) | Hermes Agent 源码深度解析系列 | 26 篇 + 研究笔记 |

> **说明：** 对比报告覆盖四个系统，但 OpenClaw 与 GenericAgent 目前仅有报告中的对比章节，尚无独立源码研读目录。

---

## 一、横向对比研究报告

[`热门Agent系统架构对比研究报告.md`](./热门Agent系统架构对比研究报告.md) 从四个维度对四套系统做横向对比，每节均附横向对照表：

- **记忆机制** — OpenClaw 的记忆与状态、GenericAgent 的 L0–L4 记忆、Claude Code 的五层记忆、Hermes 的四层记忆
- **工具调用机制** — 统一描述符 + owner/executor 分离、少量高权限原子工具、`Tool` 接口 + `buildTool` 安全默认、Python 中央注册表 + AST 自发现
- **上下文管理**
- **资料来源与分析口径**（含"一句话结论"）

---

## 二、Claude Code 源码研读（26 篇）

> 从 Anthropic 的 AI 编程助手源码中，学习构建 AI Agent 应用的全栈技术。
> 覆盖约 1900 个文件，每篇精确到源码行号，附关键代码片段。

📂 全部文章在 [`docs/`](./Claude-Code-Source-Study/docs/) 下，完整目录见
[`00-目录与阅读指引.md`](./Claude-Code-Source-Study/docs/00-目录与阅读指引.md)

| # | 章节 |
|---|------|
| 01 | [项目全景](./Claude-Code-Source-Study/docs/01-项目全景.md) |
| 02 | [启动优化](./Claude-Code-Source-Study/docs/02-启动优化.md) |
| 03 | [状态管理](./Claude-Code-Source-Study/docs/03-状态管理.md) |
| 04 | [System Prompt 工程](./Claude-Code-Source-Study/docs/04-System-Prompt-工程.md) |
| 05 | [对话循环](./Claude-Code-Source-Study/docs/05-对话循环.md) |
| 06 | [上下文管理](./Claude-Code-Source-Study/docs/06-上下文管理.md) |
| 07 | [Prompt Cache](./Claude-Code-Source-Study/docs/07-Prompt-Cache.md) |
| 08 | [Thinking 与推理控制](./Claude-Code-Source-Study/docs/08-Thinking-与推理控制.md) |
| 09 | [工具系统设计](./Claude-Code-Source-Study/docs/09-工具系统设计.md) |
| 10 | [BashTool 深度剖析](./Claude-Code-Source-Study/docs/10-BashTool-深度剖析.md) |
| 11 | [命令系统](./Claude-Code-Source-Study/docs/11-命令系统.md) |
| 12 | [Agent 系统](./Claude-Code-Source-Study/docs/12-Agent-系统.md) |
| 13 | [内置 Agent 设计模式](./Claude-Code-Source-Study/docs/13-内置Agent设计模式.md) |
| 14 | [任务系统](./Claude-Code-Source-Study/docs/14-任务系统.md) |
| 15 | [MCP 协议实现](./Claude-Code-Source-Study/docs/15-MCP-协议实现.md) |
| 16 | [权限系统](./Claude-Code-Source-Study/docs/16-权限系统.md) |
| 17 | [Settings 系统](./Claude-Code-Source-Study/docs/17-Settings-系统.md) |
| 18 | [Hooks 系统](./Claude-Code-Source-Study/docs/18-Hooks系统.md) |
| 19 | [Feature Flag 与编译期优化](./Claude-Code-Source-Study/docs/19-Feature-Flag与编译期优化.md) |
| 20 | [API 调用与错误恢复](./Claude-Code-Source-Study/docs/20-API调用与错误恢复.md) |
| 21 | [Ink 框架深度定制](./Claude-Code-Source-Study/docs/21-Ink框架深度定制.md) |
| 22 | [设计系统](./Claude-Code-Source-Study/docs/22-设计系统.md) |
| 23 | [Memory 系统](./Claude-Code-Source-Study/docs/23-Memory系统.md) |
| 24 | [Skill / Plugin 开发实战](./Claude-Code-Source-Study/docs/24-Skill-Plugin开发实战.md) |
| 25 | [架构模式总结](./Claude-Code-Source-Study/docs/25-架构模式总结.md) |

---

## 三、Hermes Agent 源码研读（26 篇）

> 基于 [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent) v0.9.0 版本线。
> 每篇以一个引导问题开头，逐文件逐函数拆解（精确到行号），配 Mermaid 架构图。

📂 全部文章在 [`docs/`](./Hermes-Source-Code-Study/docs/) 下，完整目录见
[`00-目录与阅读指引.md`](./Hermes-Source-Code-Study/docs/00-目录与阅读指引.md)

| # | 章节 |
|---|------|
| 01 | [项目全景](./Hermes-Source-Code-Study/docs/01-项目全景.md) |
| 02 | [启动链路](./Hermes-Source-Code-Study/docs/02-启动链路.md) |
| 03 | [配置系统](./Hermes-Source-Code-Study/docs/03-配置系统.md) |
| 04 | [状态持久化](./Hermes-Source-Code-Study/docs/04-状态持久化.md) |
| 05 | [对话循环](./Hermes-Source-Code-Study/docs/05-对话循环.md) |
| 06 | [System Prompt 工程](./Hermes-Source-Code-Study/docs/06-System-Prompt-工程.md) |
| 07 | [上下文管理](./Hermes-Source-Code-Study/docs/07-上下文管理.md) |
| 08 | [多模型适配](./Hermes-Source-Code-Study/docs/08-多模型适配.md) |
| 09 | [辅助客户端与成本控制](./Hermes-Source-Code-Study/docs/09-辅助客户端与成本控制.md) |
| 10 | [错误处理与韧性](./Hermes-Source-Code-Study/docs/10-错误处理与韧性.md) |
| 11 | [工具注册表设计](./Hermes-Source-Code-Study/docs/11-工具注册表设计.md) |
| 12 | [终端工具深度剖析](./Hermes-Source-Code-Study/docs/12-终端工具深度剖析.md) |
| 13 | [文件与代码工具](./Hermes-Source-Code-Study/docs/13-文件与代码工具.md) |
| 14 | [Web 与浏览器工具](./Hermes-Source-Code-Study/docs/14-Web-与浏览器工具.md) |
| 15 | [MCP 协议实现](./Hermes-Source-Code-Study/docs/15-MCP-协议实现.md) |
| 16 | [子智能体委托](./Hermes-Source-Code-Study/docs/16-子智能体委托.md) |
| 17 | [记忆系统](./Hermes-Source-Code-Study/docs/17-记忆系统.md) |
| 18 | [技能系统](./Hermes-Source-Code-Study/docs/18-技能系统.md) |
| 19 | [安全体系](./Hermes-Source-Code-Study/docs/19-安全体系.md) |
| 20 | [CLI 交互界面](./Hermes-Source-Code-Study/docs/20-CLI-交互界面.md) |
| 21 | [消息网关](./Hermes-Source-Code-Study/docs/21-消息网关.md) |
| 22 | [终端后端](./Hermes-Source-Code-Study/docs/22-终端后端.md) |
| 23 | [批处理与轨迹基础设施](./Hermes-Source-Code-Study/docs/23-批处理与轨迹基础设施.md) |
| 24 | [日志与可观测性](./Hermes-Source-Code-Study/docs/24-日志与可观测性.md) |
| 25 | [架构模式总结](./Hermes-Source-Code-Study/docs/25-架构模式总结.md) |

### 附录：实战练习

| # | 章节 |
|---|------|
| A | [实战 - 添加一个新工具](./Hermes-Source-Code-Study/docs/附录/A-实战-添加一个新工具.md) |
| B | [实战 - 编写一个自定义技能](./Hermes-Source-Code-Study/docs/附录/B-实战-编写一个自定义技能.md) |
| C | [实战 - 接入一个新通信平台](./Hermes-Source-Code-Study/docs/附录/C-实战-接入一个新通信平台.md) |

其他材料：

- [`research/`](./Hermes-Source-Code-Study/research/) — 29 篇研究笔记
- [`claims/`](./Hermes-Source-Code-Study/claims/) — 事实主张与出处核对
- [`PLAN.md`](./Hermes-Source-Code-Study/PLAN.md) / [`TASKS.md`](./Hermes-Source-Code-Study/TASKS.md)

---

## 阅读建议

- **想先要结论** → 直接读[横向对比研究报告](./热门Agent系统架构对比研究报告.md)的第 1 节"一句话结论"
- **想系统学 Agent 工程** → 任选一个源码研读系列，从 `00-目录与阅读指引.md` 开始顺序读
- **只关心某个机制** → 用对比报告定位差异，再跳到对应深读章节

## License

各源码研读系列目录下各自的 `LICENSE` 为准。
