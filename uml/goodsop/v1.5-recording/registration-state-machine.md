# 挂号单逻辑状态迁移图

> 覆盖从挂号到诊疗结束的完整业务状态流转，包含录音子状态。
> 返回系列索引：[README.md](./README.md)

---

## 1. 挂号单完整状态机

```mermaid
stateDiagram-v2
    direction TB

    [*] --> 候诊中 : 现场挂号 / 预约到店签到\n前台 PAD 创建挂号单

    候诊中 --> 诊疗中 : 医生叫号\n① 创建 lu_communication_log（archiveSessionId）\n② 建立 WebSocket 连接\n③ 开始录音（sequenceNumber = 1）

    候诊中 --> 已取消 : 取消挂号

    state 诊疗中 {
        direction LR
        [*] --> 录音中
        录音中 --> 录音已暂停 : 点击「暂停」\nWS 保持 · ASR 停止计费
        录音已暂停 --> 录音中 : 点击「继续」\nASR 恢复
    }

    诊疗中 --> 已完成 : 点击「完成诊疗」\n① 发送 STOP 帧\n② WS 关闭\n③ 触发 AI 档案生成异步任务

    已完成 --> 诊疗中 : 继续诊疗\n① sequenceNumber++\n② 建立新 WS 连接\n③ 重新开始录音\narcheSessionId 不变

    已完成 --> 已结束 : 当日诊疗彻底结束\n（无需继续诊疗）

    已取消 --> [*]
    已结束 --> [*]
```

---

## 2. 关键节点说明

| 状态 | 对应数据变化 | 触发端 |
|------|------------|--------|
| **候诊中** | `tb_registration.status = WAITING` | 前台 PAD 挂号 / 预约到店 |
| **诊疗中（录音中）** | 创建 `lu_communication_log`，`lu_communication_log_session`（sequenceNumber=N），WS 连接建立 | Web 工作站医生叫号 |
| **诊疗中（录音已暂停）** | `SessionContext.state = PAUSED`，同步写 Redis | Web 工作站点击暂停 |
| **已完成** | `tb_registration.status = COMPLETED`，`SessionContext` 进入 STOPPED，触发档案生成 | Web 工作站完成诊疗 |
| **继续诊疗** | 新增 `lu_communication_log_session`（sequenceNumber+1），archiveSessionId **不变** | Web 工作站继续诊疗 |
| **已结束** | `tb_registration.status = CLOSED` | 当日结束 / 手动关闭 |
| **已取消** | `tb_registration.status = CANCELLED` | 前台 PAD 或系统端取消 |

---

## 3. 继续诊疗与暂停录音的区别

```mermaid
graph LR
    subgraph 暂停录音["暂停录音（同一诊疗轮次内）"]
        A1["录音中"] -- "点击「暂停」" --> A2["录音已暂停"]
        A2 -- "点击「继续」" --> A1
    end

    subgraph 继续诊疗["继续诊疗（跨诊疗轮次）"]
        B1["已完成\n(sequenceNumber=1)"] -- "点击「继续诊疗」" --> B2["诊疗中\n(sequenceNumber=2)"]
        B2 -- "点击「完成诊疗」" --> B3["已完成\n(sequenceNumber=2)"]
    end
```

| 维度 | 暂停录音 | 继续诊疗 |
|------|---------|---------|
| WS 连接 | 保持不断 | 关闭旧连接，建立新连接 |
| archiveSessionId | 不变 | 不变 |
| sequenceNumber | 不变 | +1 |
| ASR 连接 | 保持（停止计费） | 关闭后重新建立 |
| 挂号单状态 | 诊疗中（不变） | 已完成 → 诊疗中 |
| Redis active key | 不变 | 旧 key 删除，写入新 wsSessionId |
