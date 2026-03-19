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

## 2. 关键接口入参说明

### 2.1 WS 握手 Headers（PAD 端）

| Header 字段 | 类型 | 必填 | 说明 |
|------------|------|------|------|
| `Authorization` | String | ✅ | `Bearer {JWT}` 登录 token |
| `roleCode` | String | ✅ | 固定值 `RECEPTIONIST` |
| `archiveSessionId` | String | ✅ | 来自挂号/诊疗管理接口，关联诊疗全程数据 |
| `tenantId` | Long | ✅ | 租户 ID（Interceptor 从 JWT 提取） |
| `shopId` | Long | ✅ | 门店 ID（Interceptor 从 JWT 提取） |

> **注意**：PAD 前台**没有叫号动作**。`archiveSessionId` 由前台通过查询当前诊疗状态或接收事件通知获得，在「开始录音」前必须已拿到该值。

### 2.2 控制帧 TextMessage（前端 → 后端）

```json
{
  "controlAction": "START | PAUSE | RESUME | STOP",
  "sessionId": "uuid-v4（前端每次连接时生成，作为 wsSessionId）",
  "archiveSessionId": "来自握手 Header，冗余携带用于校验"
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `controlAction` | Enum | ✅ | `START` / `PAUSE` / `RESUME` / `STOP` |
| `sessionId` | String(UUID) | ✅ | wsSessionId，后端 ContextStore 的路由 key |
| `archiveSessionId` | String | ✅（START） | 仅 START 帧必填，其他帧可省略 |

### 2.3 音频帧 BinaryMessage（前端 → 后端）

| 内容 | 说明 |
|------|------|
| 消息体 | 原始 PCM 音频字节流（16kHz 16bit 单声道） |
| 路由方式 | 后端通过 WS Session 对象查找对应 wsSessionId，无需额外字段 |
| 发送频率 | 每 100ms 一帧 |

### 2.4 后端核心方法签名

| 方法 | 入参 | 说明 |
|------|------|------|
| `startSession(wsSessionId, archiveSessionId, recordingUserId, roleCode)` | String, String, Long, String | 创建 SessionContext，建立 ASR 连接 |
| `pauseSession(wsSessionId)` | String | state → PAUSED，记录 pauseTime |
| `resumeSession(wsSessionId)` | String | state → RECORDING，累计 pauseDuration |
| `stopSession(wsSessionId)` | String | 归档，关闭 ASR，清除 Context |
| `createContext(wsSessionId, ...)` | SessionContext 入参 | 写入 ConcurrentHashMap + Redis |

---

## 3. 完整录音时序图

### 阶段说明

| 阶段 | 触发动作 | 核心操作 |
|------|---------|---------|
| ① 握手 | 点击「开始录音」 | HTTP Upgrade → WS 连接建立 |
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
        前台接待员 ->> PAD: 点击「开始录音」
        PAD ->> Interceptor: HTTP Upgrade
        Note over Interceptor: Headers: Authorization Bearer JWT
        Note over Interceptor: roleCode=RECEPTIONIST, archiveSessionId={id}
        Interceptor ->> Interceptor: 验证JWT，提取userId/tenantId/shopId
        Interceptor -->> PAD: 101 Switching Protocols
        Handler ->> Handler: onConnectionEstablished 注册到Holder
    end

    rect rgb(232, 245, 232)
        Note over 前台接待员,DB: 阶段② START 开始录音
        PAD ->> Handler: TextMessage
        Note over Handler: {controlAction:START, sessionId:wsUUID, archiveSessionId:sess-X}
        Handler ->> AFH: handleTextMessage(session, dto)
        AFH ->> Resolver: resolveRecordingUserId(session, loginUserId)
        Resolver -->> AFH: loginUserId（PAD：登录人即录音人）
        AFH ->> Coord: startSession(wsUUID, sess-X, recordingUserId, RECEPTIONIST)
        Coord ->> Store: createContext(wsUUID, state=RECORDING, recordingUserId, archiveSessionId)
        Coord ->> ASR: 建立ASR WebSocket连接
        Coord ->> DB: INSERT lu_communication_log_session(archiveSessionId, seq=1, startTime)
        AFH -->> PAD: ACK {action:RECORDING_STARTED, sessionId:wsUUID}
    end

    rect rgb(255, 248, 225)
        Note over 前台接待员,DB: 阶段③ 音频帧持续传输
        loop 每 100ms 一帧
            PAD ->> Handler: BinaryMessage（PCM字节流，16kHz 16bit）
            Handler ->> AFH: handleBinaryMessage(session, payload)
            AFH ->> Store: getContext(wsUUID) 验证state=RECORDING
            AFH ->> ASR: 转发音频帧
            ASR -->> AFH: ASR转写结果（异步回调）
            AFH ->> DB: INSERT lu_communication_audio + lu_communication_segment
        end
    end

    rect rgb(252, 228, 236)
        Note over 前台接待员,DB: 阶段④ 暂停录音
        前台接待员 ->> PAD: 点击「暂停」
        PAD ->> Handler: TextMessage {controlAction:PAUSE, sessionId:wsUUID}
        AFH ->> Coord: pauseSession(wsUUID)
        Coord ->> Store: state=PAUSED，pauseTime=now()
        AFH -->> PAD: ACK {action:RECORDING_PAUSED}
        Note over PAD,ASR: WS连接保持，ASR连接保持，停止传帧
    end

    rect rgb(232, 245, 232)
        Note over 前台接待员,DB: 阶段⑤ 继续录音
        前台接待员 ->> PAD: 点击「继续」
        PAD ->> Handler: TextMessage {controlAction:RESUME, sessionId:wsUUID}
        AFH ->> Coord: resumeSession(wsUUID)
        Coord ->> Store: state=RECORDING，pauseDuration+=now()-pauseTime
        AFH -->> PAD: ACK {action:RECORDING_RESUMED}
    end

    rect rgb(240, 232, 255)
        Note over 前台接待员,DB: 阶段⑥ 停止录音
        前台接待员 ->> PAD: 点击「完成」
        PAD ->> Handler: TextMessage {controlAction:STOP, sessionId:wsUUID}
        AFH ->> Coord: stopSession(wsUUID)
        Coord ->> Store: state=STOPPED
        Coord ->> ASR: 关闭ASR连接，等待最后转写结果
        Coord ->> DB: UPDATE lu_communication_log_session(endTime, totalDuration, pauseDuration)
        Coord ->> Store: removeContext(wsUUID)
        AFH -->> PAD: ACK {action:RECORDING_STOPPED}
        PAD ->> PAD: 关闭WebSocket连接
    end
```

---

## 4. 异常场景：WS 意外断开

```mermaid
sequenceDiagram
    participant PAD as 前台 PAD
    participant Handler as WsHandler
    participant Coord as Coordinator
    participant Store as ContextStore
    participant DB as PostgreSQL

    Note over PAD,DB: 场景：网络断开 / 设备重启
    PAD -x Handler: TCP连接中断
    Handler ->> Handler: afterConnectionClosed(session, status)
    Handler ->> Coord: stopSession(wsUUID) 强制收尾
    Coord ->> Store: state=STOPPED
    Coord ->> DB: UPDATE 会话记录 aborted=true, endTime=now()
    Coord ->> Store: removeContext(wsUUID)
    Note over PAD,DB: 已录制音频完整保存，可通过「继续诊疗」重新开始
```

---

## 5. SessionContext 状态机

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
