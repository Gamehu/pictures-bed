# v1.5 PAD 前台录音详细架构图

> 本文档为 `v1.5-详细设计.md` 第 9 章配套图示 —— PAD 端录音全链路。
> 返回总览：[v1.5-架构图-总览.md](./v1.5-架构图-总览.md)

---

## 1. 组件关系图

```mermaid
graph TD
    subgraph PAD["前台 PAD（单用户设备）"]
        MIC["麦克风 MediaRecorder API"]
        UI_PAD["录音控制 UI"]
        WS_PAD["WebSocket 单连接"]
    end

    subgraph GW["goodsop-common-websocket"]
        Interceptor["UserAttributeHandshakeInterceptor\n验证JWT 提取userId/tenantId/shopId"]
        Handler["CustomWebSocketHandler\nText→路由 Binary→AudioFrameHandler"]
        Holder["WebSocketSessionHolder"]
    end

    subgraph BIZ["goodsop-app-server-biz"]
        AFH["AudioFrameHandler"]
        Resolver["DefaultIdentityResolver\nloginUserId = recordingUserId"]
        Coord["RecordingSessionCoordinator\nIDLE→RECORDING→PAUSED→STOPPED"]
        Store["RecordingSessionContextStore\nConcurrentHashMap"]
        ASR_CLI["AsrWebSocketClient\n转发音频 接收转写结果"]
        Timeout["AsrSessionTimeoutManager\n无帧超时检测"]
    end

    subgraph STORAGE["存储"]
        PG[("PostgreSQL")]
        REDIS[("Redis")]
    end

    MIC --> WS_PAD
    UI_PAD --> WS_PAD
    WS_PAD -- "Header: roleCode=RECEPTIONIST" --> Interceptor
    Interceptor --> Handler
    Handler --> Holder
    Handler --> AFH
    AFH --> Resolver
    AFH --> Coord
    Coord --> Store
    AFH --> ASR_CLI
    Store --> Timeout
    Coord --> PG
    Store --> REDIS
    ASR_CLI -- "ASR转写回调" --> AFH
```

---

## 2. 完整录音时序图

### 阶段说明

| 阶段 | 触发动作 | 核心操作 |
|------|---------|---------|
| ① 握手 | 点击「叫号/开始」 | HTTP Upgrade → WS 连接建立 |
| ② START | 发送 START 控制帧 | 创建 SessionContext，建立 ASR 连接 |
| ③ 传帧 | 持续采集音频 | 每 100ms 一帧，ASR 实时转写 |
| ④ PAUSE | 点击「暂停」 | state → PAUSED，WS/ASR 保持 |
| ⑤ RESUME | 点击「继续」 | state → RECORDING |
| ⑥ STOP | 点击「完成」 | 关闭 ASR，归档，关闭 WS |

```mermaid
sequenceDiagram
    actor 前台接待员
    participant PAD as 前台 PAD
    participant Interceptor as 握手拦截器
    participant Handler as WsHandler
    participant AFH as AudioFrameHandler
    participant Resolver as DefaultIdentityResolver
    participant Coord as Coordinator
    participant Store as ContextStore
    participant ASR as AsrWebSocketClient
    participant DB as PostgreSQL

    rect rgb(232, 244, 253)
        Note over 前台接待员,DB: 阶段① WebSocket 握手
        前台接待员 ->> PAD: 点击「叫号/开始」
        PAD ->> Interceptor: HTTP Upgrade (roleCode=RECEPTIONIST, JWT)
        Interceptor ->> Interceptor: 验证JWT，提取userId/tenantId
        Interceptor -->> PAD: 101 Switching Protocols
        Handler ->> Handler: onConnectionEstablished 注册到Holder
    end

    rect rgb(232, 245, 232)
        Note over 前台接待员,DB: 阶段② START 开始录音
        PAD ->> Handler: TextMessage {controlAction:START, sessionId:wsUUID}
        Handler ->> AFH: handleTextMessage()
        AFH ->> Resolver: resolveRecordingUserId(session, loginUserId)
        Resolver -->> AFH: loginUserId（PAD：登录人即录音人）
        AFH ->> Coord: startSession(wsUUID, archiveSessionId, recordingUserId)
        Coord ->> Store: createContext(wsUUID, state=RECORDING)
        Coord ->> ASR: 建立ASR WebSocket连接
        Coord ->> DB: INSERT lu_communication_log_session (seq=1)
        AFH -->> PAD: ACK RECORDING_STARTED
    end

    rect rgb(255, 248, 225)
        Note over 前台接待员,DB: 阶段③ 音频帧持续传输
        loop 每 100ms 一帧
            PAD ->> Handler: BinaryMessage（PCM音频帧）
            Handler ->> AFH: handleBinaryMessage()
            AFH ->> Store: getContext(wsUUID) 验证state=RECORDING
            AFH ->> ASR: 转发音频帧
            ASR -->> AFH: ASR转写结果（异步回调）
            AFH ->> DB: INSERT audio帧 + segment转写文本
        end
    end

    rect rgb(252, 228, 236)
        Note over 前台接待员,DB: 阶段④ 暂停录音
        前台接待员 ->> PAD: 点击「暂停」
        PAD ->> Handler: TextMessage {controlAction:PAUSE}
        AFH ->> Coord: pauseSession(wsUUID)
        Coord ->> Store: state=PAUSED，记录pauseTime
        AFH -->> PAD: ACK RECORDING_PAUSED
        Note over PAD,ASR: WS连接保持，ASR连接保持，停止传帧
    end

    rect rgb(232, 245, 232)
        Note over 前台接待员,DB: 阶段⑤ 继续录音
        前台接待员 ->> PAD: 点击「继续」
        PAD ->> Handler: TextMessage {controlAction:RESUME}
        AFH ->> Coord: resumeSession(wsUUID)
        Coord ->> Store: state=RECORDING，计算pauseDuration
        AFH -->> PAD: ACK RECORDING_RESUMED
    end

    rect rgb(240, 232, 255)
        Note over 前台接待员,DB: 阶段⑥ 停止录音
        前台接待员 ->> PAD: 点击「完成」
        PAD ->> Handler: TextMessage {controlAction:STOP}
        AFH ->> Coord: stopSession(wsUUID)
        Coord ->> Store: state=STOPPED
        Coord ->> ASR: 关闭ASR连接
        Coord ->> DB: UPDATE lu_communication_log_session (endTime, duration)
        Coord ->> Store: removeContext(wsUUID)
        AFH -->> PAD: ACK RECORDING_STOPPED
        PAD ->> PAD: 关闭WebSocket连接
    end
```

---

## 3. 异常场景：WS 意外断开

```mermaid
sequenceDiagram
    participant PAD as 前台 PAD
    participant Handler as WsHandler
    participant Coord as Coordinator
    participant Store as ContextStore
    participant DB as PostgreSQL

    Note over PAD,DB: 场景：网络断开 / 浏览器崩溃
    PAD -x Handler: TCP连接中断
    Handler ->> Handler: afterConnectionClosed()
    Handler ->> Coord: stopSession(wsUUID) 强制收尾
    Coord ->> Store: state=STOPPED
    Coord ->> DB: UPDATE 会话记录 aborted=true
    Coord ->> Store: removeContext(wsUUID)
    Note over PAD,DB: 音频数据已持久化，下次重新叫号可「继续诊疗」
```

---

## 4. SessionContext 状态机

```mermaid
stateDiagram-v2
    [*] --> IDLE : createContext

    IDLE --> RECORDING : START帧

    RECORDING --> PAUSED : PAUSE帧

    PAUSED --> RECORDING : RESUME帧

    RECORDING --> STOPPED : STOP帧/WS断开/超时

    PAUSED --> STOPPED : WS断开

    STOPPED --> [*] : removeContext
```

| 状态转换 | 触发条件 | 后续动作 |
|---------|---------|---------|
| IDLE → RECORDING | START 控制帧 | 创建 ASR 连接，写 DB session |
| RECORDING → PAUSED | PAUSE 控制帧 | 停止传帧，WS/ASR 保持 |
| PAUSED → RECORDING | RESUME 控制帧 | 继续传帧，累计 pauseDuration |
| RECORDING → STOPPED | STOP / WS 断开 / 超时 | 关闭 ASR，归档 DB，清除 Context |
| PAUSED → STOPPED | WS 断开 | 同上 |
