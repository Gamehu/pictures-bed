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
        CALL_API["POST /workstation/consultation/call\n叫号接口（含声纹校验）"]
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

## 2. 多医生并发录音时序图

### 关键设计要点

| 要点 | 说明 |
|------|------|
| 登录用户与录音归属分离 | 院长登录（adminId），doctorId 从 WS Header 读取 |
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
        Note over 医生A,DB: 医生A：叫号并开始录音（regId=101）
        医生A ->> WEB: 点击「叫号/开始」(regId=101, doctorId=1001)
        WEB ->> CallAPI: POST /call {registrationId:101, doctorId:1001}
        CallAPI ->> VP: existsByUserId(1001)
        VP -->> CallAPI: 声纹已录入
        CallAPI ->> DB: 挂号单→诊疗中，INSERT lu_communication_log
        CallAPI -->> WEB: {archiveSessionId:"sess-A"}

        WEB ->> GW: WS握手 Header: doctorId=1001, archiveSessionId=sess-A
        GW -->> WEB: 101 WS连接建立
        WEB ->> GW: TextMessage {controlAction:START, sessionId:UUID-A}
        GW ->> AFH: handleTextMessage()
        AFH ->> Resolver: resolveRecordingUserId(session, adminId)
        Resolver -->> AFH: doctorId=1001（从Header读取）
        AFH ->> Coord: startSession(UUID-A, sess-A, recordingUserId=1001)
        Coord ->> Store: put(UUID-A, SessionContext{RECORDING, 1001, sess-A})
        AFH -->> WEB: ACK RECORDING_STARTED
    end

    rect rgb(232, 245, 232)
        Note over 医生A,DB: 医生B：叫号并开始录音（regId=202）—— 与医生A并发
        医生B ->> WEB: 点击「叫号/开始」(regId=202, doctorId=1002)
        WEB ->> CallAPI: POST /call {registrationId:202, doctorId:1002}
        CallAPI ->> VP: existsByUserId(1002)
        VP -->> CallAPI: 声纹已录入
        CallAPI -->> WEB: {archiveSessionId:"sess-B"}

        WEB ->> GW: WS握手（第2条连接）Header: doctorId=1002, archiveSessionId=sess-B
        GW -->> WEB: 101 WS连接建立
        WEB ->> GW: TextMessage {controlAction:START, sessionId:UUID-B}
        GW ->> AFH: handleTextMessage()
        AFH ->> Resolver: resolveRecordingUserId(session, adminId)
        Resolver -->> AFH: doctorId=1002（从Header读取）
        AFH ->> Coord: startSession(UUID-B, sess-B, recordingUserId=1002)
        Coord ->> Store: put(UUID-B, SessionContext{RECORDING, 1002, sess-B})
        Note over Store: UUID-A 和 UUID-B 完全独立，互不影响
        AFH -->> WEB: ACK RECORDING_STARTED
    end

    rect rgb(255, 248, 225)
        Note over 医生A,DB: 医生A 和医生B 同时传输音频帧（互不干扰）
        par 医生A传帧
            loop 每 100ms
                WEB ->> GW: BinaryMessage（医生A音频帧，via WS连接-A）
                GW ->> AFH: handleBinaryMessage()
                AFH ->> Store: getContext(UUID-A)
                AFH ->> DB: 写入 segment(archiveSessionId=sess-A)
            end
        and 医生B传帧
            loop 每 100ms
                WEB ->> GW: BinaryMessage（医生B音频帧，via WS连接-B）
                GW ->> AFH: handleBinaryMessage()
                AFH ->> Store: getContext(UUID-B)
                AFH ->> DB: 写入 segment(archiveSessionId=sess-B)
            end
        end
    end

    rect rgb(252, 228, 236)
        Note over 医生A,DB: 医生A 暂停（不影响医生B）
        医生A ->> WEB: 点击「暂停」
        WEB ->> GW: TextMessage {controlAction:PAUSE} via WS连接-A
        AFH ->> Coord: pauseSession(UUID-A)
        Coord ->> Store: UUID-A.state=PAUSED
        Note over Store: UUID-B 状态不变，医生B继续录音
        AFH -->> WEB: ACK PAUSED
    end

    rect rgb(240, 232, 255)
        Note over 医生A,DB: 医生A 完成诊疗
        医生A ->> WEB: 点击「完成诊疗」
        WEB ->> GW: TextMessage {controlAction:STOP} via WS连接-A
        AFH ->> Coord: stopSession(UUID-A)
        Coord ->> Store: 移除 UUID-A
        Coord ->> DB: 完成 sess-A 的会话记录
        WEB ->> WEB: pool.delete(101) 关闭WS连接-A
        Note over Store: 只剩 UUID-B 在运行
    end
```

---

## 3. 声纹校验失败流程

```mermaid
sequenceDiagram
    actor 医生C
    participant WEB as Web工作站
    participant CallAPI as /consultation/call
    participant VP as VoiceprintService

    医生C ->> WEB: 点击「叫号/开始」(doctorId=1003)
    WEB ->> CallAPI: POST /call {doctorId:1003}
    CallAPI ->> VP: existsByUserId(1003)
    VP -->> CallAPI: 未录入声纹
    CallAPI -->> WEB: 400 VOICEPRINT_NOT_REGISTERED
    WEB ->> WEB: 前端提示错误，不建立WS连接
    Note over WEB: 录音链路从未开启，不影响其他医生
```

---

## 4. 静默超时自动收尾流程

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
        Timeout ->> Timeout: 计算距最后一帧时间
    end

    Timeout ->> Timeout: UUID-B 超过10分钟无帧
    Timeout ->> Coord: stopSession(UUID-B) 主动触发
    Coord ->> Store: state=STOPPED，removeContext(UUID-B)
    Coord ->> DB: UPDATE 会话记录 timeout_stopped=true
    Coord ->> GW: 关闭WS连接-B
    GW -->> WEB: WS连接关闭通知
    WEB ->> WEB: pool.delete(202)
    Note over WEB,DB: 已录制音频完整保存，可通过「继续诊疗」重新开始
```

---

## 5. 页面刷新 / 断线重连流程

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
        WEB -x GW: WS连接-A断开
        GW ->> Coord: onConnectionClosed(wsSessionId=UUID-A)
        Coord ->> Store: removeContext(UUID-A)
        Coord ->> DB: 会话记录标记 interrupted=true
    end

    rect rgb(232, 245, 232)
        Note over 医生A,DB: 重连阶段（继续诊疗）
        WEB ->> WEB: 页面重新加载，archiveSessionId=sess-A仍有效
        WEB ->> WEB: 连接池为空，生成新wsSessionId=UUID-A2
        WEB ->> GW: 重新握手 Header: doctorId=1001, archiveSessionId=sess-A
        GW -->> WEB: 101 WS连接建立
        WEB ->> GW: TextMessage {controlAction:START}
        GW ->> Coord: startSession(UUID-A2, sess-A, ...)
        Coord ->> DB: INSERT lu_communication_log_session (seq=2)
        Note over WEB,DB: archiveSessionId不变，数据连续性通过sequenceNumber保证
    end
```

---

## 6. 内存状态快照（并发中间态）

```mermaid
graph LR
    subgraph Pool["前端连接池"]
        P1["regId=101\nwsSessionId: UUID-A\nstate: PAUSED"]
        P2["regId=202\nwsSessionId: UUID-B\nstate: RECORDING"]
    end

    subgraph CtxStore["后端 ContextStore"]
        C1["UUID-A\nstate: PAUSED\nrecordingUserId: 1001\narchiveSessionId: sess-A"]
        C2["UUID-B\nstate: RECORDING\nrecordingUserId: 1002\narchiveSessionId: sess-B"]
    end

    subgraph DB_LAYER["数据库（完全隔离）"]
        D1["sess-A\nsegment(recordingUserId=1001)"]
        D2["sess-B\nsegment(recordingUserId=1002)"]
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
