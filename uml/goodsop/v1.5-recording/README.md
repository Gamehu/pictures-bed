# GoodSOP v1.5 录音架构系列

> PAD 前台录音 与 Web 工作站多医生并发录音的完整架构图集。
> 对应设计文档：`discuss/v1.5-详细设计.md` 第 9 章。

| 文件 | 图数 | 内容 |
|------|------|------|
| [overview.md](./overview.md) | 4 张 | 整体分层图 · Session ID 双层设计 · Strategy 类图 · PAD vs Web 对比 |
| [pad-recording.md](./pad-recording.md) | 4 张 | 组件关系图 · 完整录音时序 · WS 断开异常 · SessionContext 状态机 |
| [web-recording.md](./web-recording.md) | 6 张 | 组件关系图 · 多医生并发时序 · 声纹校验失败 · 超时收尾 · 断线重连 · 内存状态快照 |
