# Task Tracker

> 最后更新: 2026-04-16 01:22
> 活跃 session 数: 1

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
| T00 | 仓库初始化 + README + 目录 | DONE | session-init | - | 2026-04-15 19:00 | 2026-04-15 19:00 | - | 2026-04-15 19:00 | - | README.md, docs/00-*.md |
| T01-R | Ch01 项目全景 Research | DRAFT_DONE | s-claude-01 | - | 2026-04-15 19:15 | 2026-04-15 20:48 | 2026-04-15 20:45 | 2026-04-15 20:48 | - | research/01-notes.md |
| T01-W | Ch01 项目全景 Write | DRAFT_DONE(T01-R>=DONE) | - | - | - | - | - | - | - | docs/01-项目全景.md |
| T02-R | Ch02 启动链路 Research | DRAFT_DONE | s-opus-02 | - | 2026-04-15 20:56 | 2026-04-15 20:56 | 2026-04-15 22:26 | - | - | research/02-notes.md |
| T02-W | Ch02 启动链路 Write | DRAFT_DONE(T02-R>=DONE) | - | - | - | - | - | - | - | docs/02-启动链路.md |
| T03-R | Ch03 配置系统 Research | DRAFT_DONE | s-opus-03 | - | - | - | - | - | - | research/03-notes.md |
| T03-W | Ch03 配置系统 Write | DRAFT_DONE(T03-R>=DONE) | - | - | - | - | - | - | - | docs/03-配置系统.md |
| T04-R | Ch04 状态持久化 Research | DRAFT_DONE | s-opus-04 | - | 2026-04-15 20:56 | 2026-04-15 20:56 | 2026-04-15 22:26 | 2026-04-15 21:10 | - | research/04-notes.md |
| T04-W | Ch04 状态持久化 Write | DRAFT_DONE(T04-R>=DONE) | - | - | - | - | - | - | - | docs/04-状态持久化.md |
| T05-R | Ch05 对话循环 Research | DRAFT_DONE | - | - | - | - | - | - | - | research/05-notes.md |
| T05-W | Ch05 对话循环 Write | DRAFT_DONE(T05-R>=DONE) | - | - | - | - | - | - | - | docs/05-对话循环.md |
| T06-R | Ch06 System Prompt 工程 Research | DRAFT_DONE | - | - | - | - | - | - | - | research/06-notes.md |
| T06-W | Ch06 System Prompt 工程 Write | DRAFT_DONE(T06-R>=DONE) | - | - | - | - | - | - | - | docs/06-System-Prompt-工程.md |
| T07-R | Ch07 上下文管理 Research | DRAFT_DONE | - | - | - | - | - | - | - | research/07-notes.md |
| T07-W | Ch07 上下文管理 Write | DRAFT_DONE(T07-R>=DONE) | - | - | - | - | - | - | - | docs/07-上下文管理.md |
| T08-R | Ch08 多模型适配 Research | DRAFT_DONE | - | - | - | - | - | - | - | research/08-notes.md |
| T08-W | Ch08 多模型适配 Write | DRAFT_DONE(T08-R>=DONE) | - | - | - | - | - | - | - | docs/08-多模型适配.md |
| T09-R | Ch09 辅助客户端与成本控制 Research | DRAFT_DONE | - | - | - | - | - | - | - | research/09-notes.md |
| T09-W | Ch09 辅助客户端与成本控制 Write | DRAFT_DONE(T09-R>=DONE) | - | - | - | - | - | - | - | docs/09-辅助客户端与成本控制.md |
| T10-R | Ch10 错误处理与韧性 Research | DRAFT_DONE | - | - | - | - | - | - | - | research/10-notes.md |
| T10-W | Ch10 错误处理与韧性 Write | DRAFT_DONE(T10-R>=DONE) | - | - | - | - | - | - | - | docs/10-错误处理与韧性.md |
| T11-R | Ch11 工具注册表设计 Research | DRAFT_DONE | - | - | - | - | - | - | - | research/11-notes.md |
| T11-W | Ch11 工具注册表设计 Write | DRAFT_DONE(T11-R>=DONE) | - | - | - | - | - | - | - | docs/11-工具注册表设计.md |
| T12-R | Ch12 终端工具深度剖析 Research | DRAFT_DONE | - | - | - | - | - | - | - | research/12-notes.md |
| T12-W | Ch12 终端工具深度剖析 Write | DRAFT_DONE(T12-R>=DONE) | - | - | - | - | - | - | - | docs/12-终端工具深度剖析.md |
| T13-R | Ch13 文件与代码工具 Research | DRAFT_DONE | - | - | - | - | - | - | - | research/13-notes.md |
| T13-W | Ch13 文件与代码工具 Write | DRAFT_DONE(T13-R>=DONE) | - | - | - | - | - | - | - | docs/13-文件与代码工具.md |
| T14-R | Ch14 Web 与浏览器工具 Research | DRAFT_DONE | - | - | - | - | - | - | - | research/14-notes.md |
| T14-W | Ch14 Web 与浏览器工具 Write | DRAFT_DONE(T14-R>=DONE) | - | - | - | - | - | - | - | docs/14-Web-与浏览器工具.md |
| T15-R | Ch15 MCP 协议实现 Research | DRAFT_DONE | - | - | - | - | - | - | - | research/15-notes.md |
| T15-W | Ch15 MCP 协议实现 Write | DRAFT_DONE(T15-R>=DONE) | - | - | - | - | - | - | - | docs/15-MCP-协议实现.md |
| T16-R | Ch16 子智能体委托 Research | DRAFT_DONE | - | - | - | - | - | - | - | research/16-notes.md |
| T16-W | Ch16 子智能体委托 Write | DRAFT_DONE(T16-R>=DONE) | - | - | - | - | - | - | - | docs/16-子智能体委托.md |
| T17-R | Ch17 记忆系统 Research | DRAFT_DONE | - | - | - | - | - | - | - | research/17-notes.md |
| T17-W | Ch17 记忆系统 Write | DRAFT_DONE(T17-R>=DONE) | - | - | - | - | - | - | - | docs/17-记忆系统.md |
| T18-R | Ch18 技能系统 Research | DRAFT_DONE | - | - | - | - | - | - | - | research/18-notes.md |
| T18-W | Ch18 技能系统 Write | DRAFT_DONE(T18-R>=DONE) | - | - | - | - | - | - | - | docs/18-技能系统.md |
| T19-R | Ch19 安全体系 Research | DRAFT_DONE | - | - | - | - | - | - | - | research/19-notes.md |
| T19-W | Ch19 安全体系 Write | DRAFT_DONE(T19-R>=DONE) | - | - | - | - | - | - | - | docs/19-安全体系.md |
| T20-R | Ch20 CLI 交互界面 Research | DRAFT_DONE | - | - | - | - | - | - | - | research/20-notes.md |
| T20-W | Ch20 CLI 交互界面 Write | DRAFT_DONE(T20-R>=DONE) | - | - | - | - | - | - | - | docs/20-CLI-交互界面.md |
| T21-R | Ch21 消息网关 Research | DRAFT_DONE | - | - | - | - | - | - | - | research/21-notes.md |
| T21-W | Ch21 消息网关 Write | DRAFT_DONE(T21-R>=DONE) | - | - | - | - | - | - | - | docs/21-消息网关.md |
| T22-R | Ch22 终端后端 Research | DRAFT_DONE | - | - | - | - | - | - | - | research/22-notes.md |
| T22-W | Ch22 终端后端 Write | DRAFT_DONE | - | - | - | - | - | - | - | docs/22-终端后端.md |
| T23-R | Ch23 批处理与轨迹基础设施 Research | DRAFT_DONE | - | - | - | - | - | - | - | research/23-notes.md |
| T23-W | Ch23 批处理与轨迹基础设施 Write | DRAFT_DONE | - | - | - | - | - | - | - | docs/23-批处理与轨迹基础设施.md |
| T24-R | Ch24 日志、用量统计与可观测性 Research | DRAFT_DONE | - | - | - | - | - | - | - | research/24-notes.md |
| T24-W | Ch24 日志、用量统计与可观测性 Write | DRAFT_DONE | - | - | - | - | - | - | - | docs/24-日志与可观测性.md |
| T25-R | Ch25 架构模式总结 Research | DRAFT_DONE | - | - | - | - | - | - | - | research/25-notes.md |
| T25-W | Ch25 架构模式总结 Write | DRAFT_DONE | - | - | - | - | - | - | - | docs/25-架构模式总结.md |
| TA-R | 附录 A Research | DRAFT_DONE | - | - | - | - | - | - | - | research/appendix-a-notes.md |
| TA-W | 附录 A Write | DRAFT_DONE | - | - | - | - | - | - | - | docs/附录/A-实战-添加一个新工具.md |
| TB-R | 附录 B Research | DRAFT_DONE | - | - | - | - | - | - | - | research/appendix-b-notes.md |
| TB-W | 附录 B Write | DRAFT_DONE | - | - | - | - | - | - | - | docs/附录/B-实战-编写一个自定义技能.md |
| TC-R | 附录 C Research | DRAFT_DONE | - | - | - | - | - | - | - | research/appendix-c-notes.md |
| TC-W | 附录 C Write | DRAFT_DONE | - | - | - | - | - | - | - | docs/附录/C-实战-接入一个新通信平台.md |
| T-final | 最终审校 + README 完善 | DONE | - | - | - | - | - | - | - | README.md (final) |
