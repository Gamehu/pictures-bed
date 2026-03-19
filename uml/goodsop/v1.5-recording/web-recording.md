# v1.5 Web 工作站多医生并发录音详细架构图

> 本文档为 `v1.5-详细设计.md` 第 9 章配套图示 —— Web 工作站多医生并发录音全链路。
> 返回总览：[v1.5-架构图-总览.md](./v1.5-架构图-总览.md)

---

## 1. 组件关系图

```mermaid
graph TD
    subgraph WEB["Web 工作站页面（院长登录，N个医生分栏）"]
        UI_A["医生A操作区 regId=101"]
        UI_B["医生B操作区 regId=202"]
        UI_C["医生C操作区 regId=303 空闲"]
        POOL["前端连接池\nMap(registrationId, RecordingConnection)"]
        MIC["浏览器 MediaRecorder\n每个活跃连接独立采集"]
    end

    subgraph HTTP["HTTP 接口层"]
        CALL_API["POST /workstation/consultation/call\n开始诊疗接口（含声纹校验）"]
        DONE_API["POST /workstation/consultation/complete\n完成诊疗接口"]
    end

    subgraph GW["goodsop-common-websocket（不改动）"]
        Interceptor["UserAttributeHandshakeInterceptor"]
        Handler["CustomWebSocketHandler"]
    end

    subgraph BIZ["goodsop-app-server-biz（最小化改动）"]
        ConsultSvc["ConsultationService\n声纹校验 + 状态流转"]
        VPSvc["VoiceprintService\n声纹库查询"]
        AFH["AudioFrameHandler（核心逻辑不改动）"]
        Resolver["HeaderDelegatedIdentityResolver\n读doctorId Header"]
        Coord["RecordingSessionCoordinator"]
        Store["RecordingSessionContextStore\nConcurrentHashMap 并发安全"]
        Timeout["AsrSessionTimeoutManager\n10min无帧自动stopSession"]
    end

    subgraph STORAGE["存储层"]
        PG[("PostgreSQL\n按archiveSessionId完全隔离")]
        REDIS[("Redis\n会话状态持久化")]
    end

    UI_A --> POOL
    UI_B --> POOL
    POOL --> CALL_API
    POOL --> DONE_API
    CALL_API --> ConsultSvc
    ConsultSvc --> VPSvc

    POOL -- "WS连接A doctorId=1001" --> Interceptor
    POOL -- "WS连接B doctorId=1002" --> Interceptor
    Interceptor --> Handler
    Handler --> AFH
    AFH --> Resolver
    AFH --> Coord
    Coord --> Store
    Store --> Timeout
    Coord --> PG
    Store --> REDIS
```

---

## 2. 关键接口入参说明

### 2.1 开始诊疗接口 POST /workstation/consultation/call

**Request Body：**

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `registrationId` | Long | ✅ | 挂号单 ID |
| `doctorId` | Long | ✅ | 实际接诊医生 ID（≠ 登录院长 ID） |

**Response Body（成功）：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `archiveSessionId` | String | 本次诊疗会话 ID，用于后续 WS 连接和数据关联 |
| `registrationId` | Long | 挂号单 ID（回显） |

**业务前置校验（接口内部）：**
1. 校验 `doctorId` 是否已录入声纹（`VoiceprintService.existsByUserId`）
2. 未录入 → 返回 400 `VOICEPRINT_NOT_REGISTERED`
3. 校验通过 → 更新挂号单状态为「诊疗中」，INSERT `lu_communication_log`，返回 `archiveSessionId`

---

### 2.2 WS 握手 Headers（Web 工作站端）

| Header 字段 | 类型 | 必填 | 说明 |
|------------|------|------|------|
| `Authorization` | String | ✅ | `Bearer {JWT}`（院长 token） |
| `roleCode` | String | ✅ | 固定值 `DOCTOR` |
| `doctorId` | Long | ✅ | 实际接诊医生 ID（与 JWT userId 不同） |
| `archiveSessionId` | String | ✅ | 开始诊疗接口返回的诊疗会话 ID |
| `tenantId` | Long | ✅ | 租户 ID（Interceptor 从 JWT 提取） |
| `shopId` | Long | ✅ | 门店 ID（Interceptor 从 JWT 提取） |

> **身份分离设计**：`JWT.userId` = 院长 ID（登录人），`doctorId` Header = 实际录音归属医生 ID。`HeaderDelegatedIdentityResolver` 读取 `doctorId` 作为 `recordingUserId`。

---

### 2.3 控制帧 TextMessage（前端 → 后端）

```json
{
  "controlAction": "START | PAUSE | RESUME | STOP",
  "sessionId": "uuid-v4（前端连接时生成，作为 wsSessionId）",
  "archiveSessionId": "开始诊疗接口返回值（START帧必填）"
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `controlAction` | Enum | ✅ | `START` / `PAUSE` / `RESUME` / `STOP` |
| `sessionId` | String(UUID) | ✅ | 前端为本次 WS 连接生成的 UUID，即 wsSessionId |
| `archiveSessionId` | String | ✅（START） | 仅 START 帧必填，其他帧可省略 |

---

### 2.4 音频帧 BinaryMessage（前端 → 后端）

| 内容 | 说明 |
|------|------|
| 消息体 | 原始 PCM 音频字节流（16kHz 16bit 单声道） |
| 路由方式 | 后端通过 WS Session 对象找到对应 wsSessionId，各医生帧互不干扰 |
| 发送频率 | 每 100ms 一帧 |

---

### 2.5 后端核心方法签名

| 方法 | 入参 | 说明 |
|------|------|------|
| `resolveRecordingUserId(session, loginUserId)` | WsSession, Long | 从 Header 读 `doctorId`，失败则 fallback `loginUserId` |
| `startSession(wsSessionId, archiveSessionId, recordingUserId, roleCode)` | String, String, Long, String | 创建 SessionContext，建立 ASR 连接 |
| `pauseSession(wsSessionId)` | String | 仅操作该 wsSessionId 对应 Context，不影响其他医生 |
| `resumeSession(wsSessionId)` | String | 同上 |
| `stopSession(wsSessionId)` | String | 归档、关闭 ASR、清除 Context |

---

## 3. 多医生并发录音时序图

### 关键设计要点

| 要点 | 说明 |
|------|------|
| 登录用户与录音归属分离 | 院长 JWT（adminId），doctorId 从 WS Header 读取 |
| 每医生独立 WS 连接 | 连接池 `pool.set(registrationId, conn)` |
| ConcurrentHashMap 天然并发安全 | key=wsSessionId，无需额外加锁 |
| ASR 连接独立 | 每个 SessionContext 持有独立的 AsrWebSocketClient |

```mermaid
sequenceDiagram
    actor 医生A
    actor 医生B
    participant WEB as Web工作站
    participant CallAPI as /consultation/call
    participant VP as VoiceprintService
    participant GW as WsHandler
    participant AFH as AudioFrameHandler
    participant Resolver as HeaderDelegatedResolver
    participant Store as ContextStore
    participant Coord as Coordinator
    participant DB as PostgreSQL

    rect rgb(232, 244, 253)
        Note over 医生A,DB: 医生A：开始诊疗并录音（regId=101）
        医生A ->> WEB: 点击「开始诊疗」(regId=101, doctorId=1001)
        WEB ->> CallAPI: POST /call {registrationId:101, doctorId:1001}
        CallAPI ->> VP: existsByUserId(doctorId=1001)
        VP -->> CallAPI: 声纹已录入
        CallAPI ->> DB: 挂号单→诊疗中，INSERT lu_communication_log
        CallAPI -->> WEB: {archiveSessionId:"sess-A", registrationId:101}

        WEB ->> GW: HTTP Upgrade（WS握手）
        Note over GW: Headers: Authorization Bearer JWT(adminId)
        Note over GW: roleCode=DOCTOR, doctorId=1001, archiveSessionId=sess-A
        GW -->> WEB: 101 WS连接建立
        WEB ->> GW: TextMessage
        Note over GW: {controlAction:START, sessionId:UUID-A, archiveSessionId:sess-A}
        GW ->> AFH: handleTextMessage(session, dto)
        AFH ->> Resolver: resolveRecordingUserId(session, adminId)
        Resolver -->> AFH: doctorId=1001（从Header doctorId读取）
        AFH ->> Coord: startSession(UUID-A, sess-A, 1001, DOCTOR)
        Coord ->> Store: put(UUID-A, {state:RECORDING, recordingUserId:1001, archiveSessionId:sess-A})
        AFH -->> WEB: ACK {action:RECORDING_STARTED, sessionId:UUID-A}
    end

    rect rgb(232, 245, 232)
        Note over 医生A,DB: 医生B：开始诊疗并录音（regId=202）—— 与医生A并发
        医生B ->> WEB: 点击「开始诊疗」(regId=202, doctorId=1002)
        WEB ->> CallAPI: POST /call {registrationId:202, doctorId:1002}
        CallAPI ->> VP: existsByUserId(doctorId=1002)
        VP -->> CallAPI: 声纹已录入
        CallAPI -->> WEB: {archiveSessionId:"sess-B", registrationId:202}

        WEB ->> GW: HTTP Upgrade（第2条WS连接）
        Note over GW: Headers: roleCode=DOCTOR, doctorId=1002, archiveSessionId=sess-B
        GW -->> WEB: 101 WS连接建立
        WEB ->> GW: TextMessage
        Note over GW: {controlAction:START, sessionId:UUID-B, archiveSessionId:sess-B}
        AFH ->> Resolver: resolveRecordingUserId(session, adminId)
        Resolver -->> AFH: doctorId=1002（从Header doctorId读取）
        AFH ->> Coord: startSession(UUID-B, sess-B, 1002, DOCTOR)
        Coord ->> Store: put(UUID-B, {state:RECORDING, recordingUserId:1002, archiveSessionId:sess-B})
        Note over Store: UUID-A 和 UUID-B 完全独立，互不影响
        AFH -->> WEB: ACK {action:RECORDING_STARTED, sessionId:UUID-B}
    end

    rect rgb(255, 248, 225)
        Note over 医生A,DB: 医生A 和医生B 同时传输音频帧（互不干扰）
        par 医生A传帧（via WS连接-A）
            loop 每 100ms
                WEB ->> GW: BinaryMessage（医生A PCM帧）
                GW ->> AFH: handleBinaryMessage(session-A, payload)
                AFH ->> Store: getContext(UUID-A) state=RECORDING
                AFH ->> DB: INSERT segment(archiveSessionId=sess-A, recordingUserId=1001)
            end
        and 医生B传帧（via WS连接-B）
            loop 每 100ms
                WEB ->> GW: BinaryMessage（医生B PCM帧）
                GW ->> AFH: handleBinaryMessage(session-B, payload)
                AFH ->> Store: getContext(UUID-B) state=RECORDING
                AFH ->> DB: INSERT segment(archiveSessionId=sess-B, recordingUserId=1002)
            end
        end
    end

    rect rgb(252, 228, 236)
        Note over 医生A,DB: 医生A 暂停（不影响医生B）
        医生A ->> WEB: 点击「暂停」
        WEB ->> GW: TextMessage {controlAction:PAUSE, sessionId:UUID-A}
        AFH ->> Coord: pauseSession(UUID-A)
        Coord ->> Store: UUID-A.state=PAUSED，pauseTime=now()
        Note over Store: UUID-B 状态不变，医生B继续录音
        AFH -->> WEB: ACK {action:RECORDING_PAUSED}
    end

    rect rgb(240, 232, 255)
        Note over 医生A,DB: 医生A 完成诊疗
        医生A ->> WEB: 点击「完成诊疗」
        WEB ->> GW: TextMessage {controlAction:STOP, sessionId:UUID-A}
        AFH ->> Coord: stopSession(UUID-A)
        Coord ->> Store: 移除 UUID-A
        Coord ->> DB: UPDATE lu_communication_log_session(endTime, totalDuration)
        WEB ->> WEB: pool.delete(101) 关闭WS连接-A
        Note over Store: 只剩 UUID-B 在运行
    end
```

---

## 4. 声纹校验失败流程

```mermaid
sequenceDiagram
    actor 医生C
    participant WEB as Web工作站
    participant CallAPI as /consultation/call
    participant VP as VoiceprintService

    医生C ->> WEB: 点击「开始诊疗」(doctorId=1003)
    WEB ->> CallAPI: POST /call {registrationId:303, doctorId:1003}
    CallAPI ->> VP: existsByUserId(1003)
    VP -->> CallAPI: 未录入声纹
    CallAPI -->> WEB: 400 {code:VOICEPRINT_NOT_REGISTERED}
    WEB ->> WEB: 前端提示错误，不建立WS连接
    Note over WEB: 录音链路从未开启，不影响其他医生
```

---

## 5. 静默超时自动收尾流程

```mermaid
sequenceDiagram
    participant WEB as Web工作站
    participant GW as WebSocket
    participant Timeout as TimeoutManager
    participant Coord as Coordinator
    participant Store as ContextStore
    participant DB as PostgreSQL

    Note over WEB,DB: 场景：医生B开始录音后离开，10分钟内无音频帧
    loop 定时检查（每分钟）
        Timeout ->> Store: 遍历所有RECORDING状态Context
        Timeout ->> Timeout: lastFrameTime与now()差值 > 10min？
    end

    Timeout ->> Timeout: UUID-B 超过10分钟无帧
    Timeout ->> Coord: stopSession(UUID-B)
    Coord ->> Store: state=STOPPED，removeContext(UUID-B)
    Coord ->> DB: UPDATE lu_communication_log_session(endTime, timeoutStopped=true)
    Coord ->> GW: 关闭WS连接-B
    GW -->> WEB: WS连接关闭通知
    WEB ->> WEB: pool.delete(202)
    Note over WEB,DB: 已录制音频完整保存，可通过「继续诊疗」重新开始
```

---

## 6. 页面刷新 / 断线重连流程

```mermaid
sequenceDiagram
    participant 医生A
    participant WEB as Web工作站
    participant GW as WebSocket
    participant Coord as Coordinator
    participant Store as ContextStore
    participant DB as PostgreSQL

    Note over 医生A,DB: 场景：页面刷新（录音进行中）

    rect rgb(252, 228, 236)
        Note over 医生A,DB: 断开阶段
        医生A ->> WEB: 刷新页面（F5）
        WEB -x GW: WS连接-A断开（TCP中断）
        GW ->> Coord: afterConnectionClosed(session-A)
        Coord ->> Store: removeContext(UUID-A)
        Coord ->> DB: UPDATE 会话记录 aborted=true
    end

    rect rgb(232, 245, 232)
        Note over 医生A,DB: 重连阶段（继续诊疗，archiveSessionId不变）
        WEB ->> WEB: 页面加载，连接池为空，生成新wsSessionId=UUID-A2
        WEB ->> GW: HTTP Upgrade（重新握手）
        Note over GW: Headers: doctorId=1001, archiveSessionId=sess-A（沿用原值）
        GW -->> WEB: 101 WS连接建立
        WEB ->> GW: TextMessage {controlAction:START, sessionId:UUID-A2, archiveSessionId:sess-A}
        GW ->> Coord: startSession(UUID-A2, sess-A, 1001, DOCTOR)
        Coord ->> DB: INSERT lu_communication_log_session(archiveSessionId=sess-A, seq=2)
        Note over WEB,DB: archiveSessionId不变，sequenceNumber=2表示第2段录音
    end
```

---

## 7. 内存状态快照（并发中间态）

```mermaid
graph LR
    subgraph Pool["前端连接池"]
        P1["regId=101 UUID-A state:PAUSED"]
        P2["regId=202 UUID-B state:RECORDING"]
    end

    subgraph CtxStore["后端 ContextStore"]
        C1["UUID-A state:PAUSED userId:1001 sess-A"]
        C2["UUID-B state:RECORDING userId:1002 sess-B"]
    end

    subgraph DB_LAYER["数据库（按archiveSessionId隔离）"]
        D1["sess-A recordingUserId=1001"]
        D2["sess-B recordingUserId=1002"]
    end

    P1 -- "wsSessionId路由" --> C1
    P2 -- "wsSessionId路由" --> C2
    C1 -- "archiveSessionId锚定" --> D1
    C2 -- "archiveSessionId锚定" --> D2

    style P1 fill:#fff3e0,stroke:#FF9800
    style P2 fill:#e8f5e9,stroke:#4CAF50
    style C1 fill:#fff3e0,stroke:#FF9800
    style C2 fill:#e8f5e9,stroke:#4CAF50
    style D1 fill:#fff3e0,stroke:#FF9800
    style D2 fill:#e8f5e9,stroke:#4CAF50
```
