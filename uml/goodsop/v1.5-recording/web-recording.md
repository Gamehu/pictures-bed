# v1.5 Web 工作站多医生并发录音详细架构图

> Web 工作站多医生并发录音全链路详细图示。
> 返回总览：[overview.md](./overview.md)

---

## 1. 组件关系图

```mermaid
graph TD
    subgraph WEB["🖥️ Web 工作站页面（院长登录，N 个医生分栏）"]
        UI_A["医生A 操作区\nregId=101"]
        UI_B["医生B 操作区\nregId=202"]
        UI_C["医生C 操作区\nregId=303（空闲）"]
        POOL["前端连接池\nMap&lt;registrationId, RecordingConnection&gt;"]
        MIC["浏览器 MediaRecorder\n每个活跃连接独立采集"]
    end

    subgraph HTTP["HTTP 接口层"]
        CALL_API["POST /workstation/consultation/call\n叫号接口（含声纹校验）"]
        DONE_API["POST /workstation/consultation/complete\n完成诊疗接口"]
    end

    subgraph GW["goodsop-common-websocket（不改动）"]
        Interceptor["UserAttributeHandshakeInterceptor"]
        Handler["CustomWebSocketHandler\n路由 Text / Binary 消息"]
    end

    subgraph BIZ["goodsop-app-server-biz（最小化改动）"]
        ConsultSvc["ConsultationService\n声纹校验 + 状态流转"]
        VPSvc["VoiceprintService\n声纹库查询"]
        AFH["AudioFrameHandler\n音频帧处理（核心逻辑不改动）"]
        Resolver["HeaderDelegatedIdentityResolver\n读 doctorId Header / fallback loginUserId"]
        Coord["RecordingSessionCoordinator"]
        Store["RecordingSessionContextStore\nConcurrentHashMap\n多 wsSessionId 并发安全"]
        Timeout["AsrSessionTimeoutManager\n10min 无帧 → 自动 stopSession"]
        ASR_A["AsrWebSocketClient-A\n医生A 专属 ASR 连接"]
        ASR_B["AsrWebSocketClient-B\n医生B 专属 ASR 连接"]
    end

    subgraph STORAGE["存储层"]
        PG[("PostgreSQL\n按 archiveSessionId 完全隔离")]
        REDIS[("Redis\n会话状态持久化")]
    end

    UI_A --> POOL
    UI_B --> POOL
    POOL --> CALL_API
    POOL --> DONE_API
    ConsultSvc --> VPSvc
    CALL_API --> ConsultSvc

    POOL -- "WS连接A\ndoctorId=1001 / roleCode=DOCTOR" --> Interceptor
    POOL -- "WS连接B\ndoctorId=1002 / roleCode=DOCTOR" --> Interceptor
    Interceptor --> Handler
    Handler --> AFH
    AFH --> Resolver
    AFH --> Coord
    Coord --> Store
    Store --> Timeout
    AFH --> ASR_A
    AFH --> ASR_B
    Coord --> PG
    Store --> REDIS
```

---

## 2. 多医生并发录音时序图

```mermaid
sequenceDiagram
    actor 医生A
    actor 医生B
    participant WEB as Web 工作站页面
    participant CallAPI as /consultation/call
    participant VP as VoiceprintService
    participant GW as CustomWebSocketHandler
    participant AFH as AudioFrameHandler
    participant Resolver as HeaderDelegatedIdentityResolver
    participant Store as ContextStore
    participant Coord as Coordinator
    participant ASR_A as AsrClient-A
    participant ASR_B as AsrClient-B
    participant DB as PostgreSQL

    rect rgb(232, 244, 253)
        Note over 医生A,DB: 医生A：叫号并开始录音（regId=101）
        医生A ->> WEB: 点击「叫号 / 开始」(regId=101, doctorId=1001)
        WEB ->> CallAPI: POST /call {registrationId:101, doctorId:1001}
        CallAPI ->> VP: existsByUserId(doctorId=1001)
        VP -->> CallAPI: ✅ 声纹已录入
        CallAPI ->> DB: 挂号单状态 → 诊疗中，INSERT lu_communication_log
        CallAPI -->> WEB: {archiveSessionId:"sess-A"}
        WEB ->> WEB: 生成 wsSessionId=UUID-A，连接池 pool.set(101, conn-A)
        WEB ->> GW: WS 握手 Header: doctorId=1001, roleCode=DOCTOR, archiveSessionId=sess-A
        GW -->> WEB: 101 WS 连接建立
        WEB ->> GW: TextMessage {controlAction:"START", sessionId:"UUID-A"}
        GW ->> AFH: handleTextMessage()
        AFH ->> Resolver: resolveRecordingUserId(session, adminId)
        Resolver ->> Resolver: 读取 Header doctorId=1001
        Resolver -->> AFH: recordingUserId=1001（医生A）
        AFH ->> Coord: startSession("UUID-A", "sess-A", recordingUserId=1001)
        Coord ->> Store: put("UUID-A", SessionContext{state=RECORDING, archiveSessionId=sess-A, recordingUserId=1001})
        Coord ->> ASR_A: 建立 ASR 连接
        AFH -->> WEB: ACK RECORDING_STARTED
    end

    rect rgb(232, 245, 232)
        Note over 医生A,DB: 医生B：叫号并开始录音（regId=202）—— 与医生A 并发
        医生B ->> WEB: 点击「叫号 / 开始」(regId=202, doctorId=1002)
        WEB ->> CallAPI: POST /call {registrationId:202, doctorId:1002}
        CallAPI ->> VP: existsByUserId(doctorId=1002)
        VP -->> CallAPI: ✅ 声纹已录入
        CallAPI -->> WEB: {archiveSessionId:"sess-B"}
        WEB ->> WEB: 生成 wsSessionId=UUID-B，连接池 pool.set(202, conn-B)
        WEB ->> GW: WS 握手（第2条连接）Header: doctorId=1002, roleCode=DOCTOR, archiveSessionId=sess-B
        GW -->> WEB: 101 WS 连接建立
        WEB ->> GW: TextMessage {controlAction:"START", sessionId:"UUID-B"}
        GW ->> AFH: handleTextMessage()
        AFH ->> Resolver: resolveRecordingUserId(session, adminId)
        Resolver -->> AFH: recordingUserId=1002（医生B）
        AFH ->> Coord: startSession("UUID-B", "sess-B", recordingUserId=1002)
        Coord ->> Store: put("UUID-B", SessionContext{state=RECORDING, archiveSessionId=sess-B, recordingUserId=1002})
        Note over Store: ConcurrentHashMap 中 UUID-A 和 UUID-B 完全独立
        Coord ->> ASR_B: 建立 ASR 连接
        AFH -->> WEB: ACK RECORDING_STARTED
    end

    rect rgb(255, 248, 225)
        Note over 医生A,DB: 医生A 和医生B 同时传输音频帧（互不干扰）
        par 医生A 传帧
            loop 每 100ms
                WEB ->> GW: BinaryMessage（医生A 音频帧）via WS连接-A
                GW ->> AFH: handleBinaryMessage()，按 wsSessionId=UUID-A 路由
                AFH ->> Store: getContext("UUID-A")
                AFH ->> ASR_A: 转发音频
                ASR_A -->> AFH: 转写结果
                AFH ->> DB: 写入 segment(archiveSessionId=sess-A)
            end
        and 医生B 传帧
            loop 每 100ms
                WEB ->> GW: BinaryMessage（医生B 音频帧）via WS连接-B
                GW ->> AFH: handleBinaryMessage()，按 wsSessionId=UUID-B 路由
                AFH ->> Store: getContext("UUID-B")
                AFH ->> ASR_B: 转发音频
                ASR_B -->> AFH: 转写结果
                AFH ->> DB: 写入 segment(archiveSessionId=sess-B)
            end
        end
    end

    rect rgb(252, 228, 236)
        Note over 医生A,DB: 医生A 暂停（不影响医生B）
        医生A ->> WEB: 点击「暂停」
        WEB ->> GW: TextMessage {controlAction:"PAUSE"} via WS连接-A
        GW ->> AFH: handleTextMessage()
        AFH ->> Coord: pauseSession("UUID-A")
        Coord ->> Store: UUID-A.state = PAUSED
        Note over Store: UUID-B 状态不变，医生B 继续录音
        AFH -->> WEB: ACK PAUSED
    end

    rect rgb(240, 232, 255)
        Note over 医生A,DB: 医生A 完成诊疗
        医生A ->> WEB: 点击「完成诊疗」
        WEB ->> GW: TextMessage {controlAction:"STOP"} via WS连接-A
        GW ->> AFH: handleTextMessage()
        AFH ->> Coord: stopSession("UUID-A")
        Coord ->> Store: 移除 UUID-A
        Coord ->> ASR_A: 关闭 ASR 连接
        Coord ->> DB: 完成 sess-A 的会话记录
        WEB ->> WEB: pool.delete(101)，关闭 WS连接-A
        Note over Store: 只剩 UUID-B 在运行
    end
```

---

## 3. 声纹校验失败流程

```mermaid
sequenceDiagram
    actor 医生C
    participant WEB as Web 工作站
    participant CallAPI as /consultation/call
    participant VP as VoiceprintService

    医生C ->> WEB: 点击「叫号 / 开始」(doctorId=1003)
    WEB ->> CallAPI: POST /call {doctorId:1003}
    CallAPI ->> VP: existsByUserId(doctorId=1003)
    VP -->> CallAPI: ❌ 未录入声纹
    CallAPI -->> WEB: 400 {code:"VOICEPRINT_NOT_REGISTERED", msg:"医生尚未录入声纹，请先完成声纹注册"}
    WEB ->> WEB: 前端提示错误，不建立 WS 连接
    Note over WEB: 录音链路从未开启，不影响其他医生
```

---

## 4. 静默超时自动收尾流程

```mermaid
sequenceDiagram
    participant WEB as Web 工作站
    participant GW as WebSocket
    participant Timeout as AsrSessionTimeoutManager
    participant Coord as Coordinator
    participant Store as ContextStore
    participant DB as PostgreSQL

    Note over WEB,DB: 场景：医生B 开始录音后离开，10 分钟内无音频帧
    loop 定时检查（每分钟）
        Timeout ->> Store: 遍历所有 RECORDING 状态的 Context
        Timeout ->> Timeout: 计算距最后一帧的时间
    end

    Timeout ->> Timeout: UUID-B 超过 10 分钟无音频帧
    Timeout ->> Coord: stopSession("UUID-B")（主动触发）
    Coord ->> Store: state = STOPPED，removeContext("UUID-B")
    Coord ->> DB: 更新会话记录（标记 timeout_stopped=true）
    Coord ->> GW: 关闭 WS 连接-B
    GW -->> WEB: WS 连接关闭通知
    WEB ->> WEB: pool.delete(202)
    Note over WEB,DB: 已录制的音频数据完整保存，医生B 可通过「继续诊疗」重新开始
```

---

## 5. 页面刷新 / 断线重连流程

```mermaid
sequenceDiagram
    participant 医生A
    participant WEB as Web 工作站
    participant GW as WebSocket
    participant Coord as Coordinator
    participant Store as ContextStore
    participant DB as PostgreSQL

    Note over 医生A,DB: 场景：页面刷新（录音进行中）

    rect rgb(252, 228, 236)
        Note over 医生A,DB: 断开阶段
        医生A ->> WEB: 刷新页面（F5）
        WEB -x GW: WS 连接-A 断开
        GW ->> Coord: onConnectionClosed(wsSessionId=UUID-A)
        Coord ->> Coord: stopSession("UUID-A")（防止资源泄露）
        Coord ->> Store: removeContext("UUID-A")
        Coord ->> DB: 会话记录打上 interrupted=true
    end

    rect rgb(232, 245, 232)
        Note over 医生A,DB: 重连阶段（继续诊疗）
        WEB ->> WEB: 页面重新加载，archiveSessionId="sess-A" 仍有效
        WEB ->> WEB: 检查连接池（为空，需重建），生成新的 wsSessionId=UUID-A2
        WEB ->> GW: 重新握手 Header: doctorId=1001, archiveSessionId=sess-A
        GW -->> WEB: 101 WS 连接建立
        WEB ->> GW: TextMessage {controlAction:"START"}
        GW ->> Coord: startSession("UUID-A2", "sess-A", ...)
        Coord ->> DB: INSERT lu_communication_log_session (sequenceNumber=2)，接续 archiveSessionId=sess-A
        Note over WEB,DB: archiveSessionId 不变，数据连续性通过 sequenceNumber 保证
    end
```

---

## 6. 内存状态快照（并发中间态）

```mermaid
graph LR
    subgraph Pool["前端连接池 ConnectionPool"]
        P1["regId=101\nwsSessionId: UUID-A\narchiveSessionId: sess-A\ndoctorId: 1001\nstate: PAUSED"]
        P2["regId=202\nwsSessionId: UUID-B\narchiveSessionId: sess-B\ndoctorId: 1002\nstate: RECORDING"]
    end

    subgraph CtxStore["后端 ContextStore (ConcurrentHashMap)"]
        C1["UUID-A → SessionContext\nstate: PAUSED\nrecordingUserId: 1001\narchiveSessionId: sess-A\nroleCode: DOCTOR"]
        C2["UUID-B → SessionContext\nstate: RECORDING\nrecordingUserId: 1002\narchiveSessionId: sess-B\nroleCode: DOCTOR"]
    end

    subgraph DB_LAYER["数据库（完全隔离）"]
        D1["sess-A\nlu_communication_log_session\nsegment(recordingUserId=1001)"]
        D2["sess-B\nlu_communication_log_session\nsegment(recordingUserId=1002)"]
    end

    P1 -- "wsSessionId 路由" --> C1
    P2 -- "wsSessionId 路由" --> C2
    C1 -- "archiveSessionId 锚定" --> D1
    C2 -- "archiveSessionId 锚定" --> D2

    style P1 fill:#fff3e0,stroke:#FF9800
    style P2 fill:#e8f5e9,stroke:#4CAF50
    style C1 fill:#fff3e0,stroke:#FF9800
    style C2 fill:#e8f5e9,stroke:#4CAF50
    style D1 fill:#fff3e0,stroke:#FF9800
    style D2 fill:#e8f5e9,stroke:#4CAF50
```
