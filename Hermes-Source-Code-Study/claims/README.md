# Claims Protocol

`claims/` 用于多 session 并发时的原子认领。

规则：
- 工作 task 认领：`mkdir "claims/<TASK_ID>"`
- review 认领：`mkdir "claims/review-<TASK_ID>"`
- `mkdir` 成功才算抢到锁；失败说明已有其他 session 先认领
- 锁目录必须保持为空，方便 lease 过期后使用 `rmdir` 回收
- 只允许删除空锁目录，禁止删除他人的 `research/`、`docs/` 文件

推荐流程：
1. 读取 `TASKS.md`
2. 选择可认领 task
3. `mkdir "claims/<TASK_ID>"`
4. 再读一次 `TASKS.md`
5. 更新该 task 的 `Status/Assignee/Heartbeat/Lease Until`

回收流程：
1. 确认 `lease_until` 已过期，且 `heartbeat` 超过 20 分钟未更新
2. 将 `TASKS.md` 状态改为 `STALE`
3. `rmdir "claims/<TASK_ID>"`
4. 按正常流程重新认领
