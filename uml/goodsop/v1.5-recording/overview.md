# v1.5 录音架构总览图

> PAD 与 Web 工作站共用后端录音架构的整体结构。
> 详细流程图见：[PAD 前台录音](./pad-recording.md) · [Web 工作站录音](./web-recording.md)

---

## 1. 整体架构分层图

```mermaid
graph TB
    subgraph Clients["接入端（两端共用同一套后端）"]
        PAD["📱 前台 PAD\nroleCode = RECEPTIONIST\n单连接 · 固定用户"]
        WEB["🖥️ Web 工作站\nroleCode = DOCTOR\n连接池 · 多医生并发"]
    end

    subgraph WSInfra["goodsop-common-websocket（基础设施层）"]
        Interceptor["UserAttributeHandshakeInterceptor\n验证 JWT · 提取用户属性"]
        Router["CustomWebSocketHandler\n按 type 路由消息"]
        SessionHolder["WebSocketSessionHolder\nConcurrentHashMap&lt;sessionKey, WsSession&gt;"]
    end

    subgraph BizLayer["goodsop-app-server-biz（业务层）"]
        AFH["AudioFrameHandler\n音频帧处理 · 状态机控制"]

        subgraph Strategy["身份解析策略（Strategy 模式）"]
            direction LR
            Resolver{"RecordingIdentityResolver"}
            Default["DefaultIdentityResolver\nPAD：loginUser = recordingUser"]
            HeaderDel["HeaderDelegatedIdentityResolver\nWeb：读 doctorId Header"]
            Resolver --> Default
            Resolver --> HeaderDel
        end

        Coordinator["RecordingSessionCoordinator\nSTART / PAUSE / RESUME / STOP"]
        Store["RecordingSessionContextStore\nConcurrentHashMap&lt;wsSessionId, SessionContext&gt;"]
        Timeout["AsrSessionTimeoutManager\n静默超时检测（10min）"]
    end

    subgraph Downstream["下游服务"]
        ASR["AsrWebSocketClient\n第三方 ASR 转写服务"]
        ConsultSvc["ConsultationService\n叫号 · 声纹校验 · 状态流转"]
    end

    subgraph Storage["存储层"]
        DB[("PostgreSQL\nlu_communication_segment\nlu_communication_audio\nlu_communication_log_session")]
        Redis[("Redis\nArchiveSessionRedisStore\n会话状态持久化")]
    end

    PAD -- "WS Header\nroleCode=RECEPTIONIST" --> Interceptor
    WEB -- "WS Header\nroleCode=DOCTOR\ndoctorId={id}" --> Interceptor
    Interceptor --> Router
    Router --> SessionHolder
    Router --> AFH
    AFH --> Resolver
    AFH --> Coordinator
    Coordinator --> Store
    Store --> Timeout
    AFH --> ASR
    ASR -- "转写结果" --> AFH
    ConsultSvc -- "archiveSessionId" --> AFH
    Coordinator --> DB
    Store --> Redis
```

---

## 2. Session ID 双层设计

```mermaid
graph LR
    subgraph Frontend["前端"]
        REG["registrationId\n挂号单 ID\n业务层唯一键"]
        WS_ID["wsSessionId\nUUID（前端生成）\n每次 WS 连接独立"]
    end

    subgraph Backend["后端内存"]
        CTX_STORE["RecordingSessionContextStore\nkey = wsSessionId"]
    end

    subgraph DB["数据库"]
        ARCHIVE["archiveSessionId\n= lu_communication_log.id\n后端叫号接口返回"]
    end

    REG -- "1. 叫号接口" --> ARCHIVE
    ARCHIVE -- "2. 返回给前端" --> WS_ID
    WS_ID -- "3. WS 握手 + START 帧" --> CTX_STORE
    ARCHIVE -- "4. Header 携带" --> CTX_STORE

    style REG fill:#e8f4fd,stroke:#2196F3
    style WS_ID fill:#fff3e0,stroke:#FF9800
    style ARCHIVE fill:#e8f5e9,stroke:#4CAF50
    style CTX_STORE fill:#fce4ec,stroke:#E91E63
```

| ID | 生命周期 | 职责 |
|----|---------|------|
| `registrationId` | 挂号单全程 | 前端连接池的索引键 |
| `wsSessionId` | 单次 WS 连接 | 后端内存路由键，隔离并发会话 |
| `archiveSessionId` | 诊疗全程（跨多次连接） | 数据库业务锚点，保证录音数据连续性 |

---

## 3. 身份解析策略类图

```mermaid
classDiagram
    class RecordingIdentityResolver {
        <<interface>>
        +resolveRecordingUserId(session, loginUserId) Long
    }

    class DefaultIdentityResolver {
        +resolveRecordingUserId(session, loginUserId) Long
    }
    note for DefaultIdentityResolver "直接返回 loginUserId\nPAD 场景默认实现"

    class HeaderDelegatedIdentityResolver {
        +resolveRecordingUserId(session, loginUserId) Long
    }
    note for HeaderDelegatedIdentityResolver "读取 doctorId Header\n失败则 fallback loginUserId"

    class AudioFrameHandler {
        -identityResolver RecordingIdentityResolver
        +initializeSession(session, attrs) void
        +handleBinaryMessage(session, message) void
        +handleControlAction(session, dto) void
    }

    class SessionContext {
        +Long recordingUserId
        +RecordingState state
        +String archiveSessionId
        +String roleCode
    }
    note for SessionContext "recordingUserId 永远非 null\n由 Resolver 在初始化时赋值"

    RecordingIdentityResolver <|.. DefaultIdentityResolver
    RecordingIdentityResolver <|.. HeaderDelegatedIdentityResolver
    AudioFrameHandler --> RecordingIdentityResolver : 组合
    AudioFrameHandler --> SessionContext : 创建/操作
```

---

## 4. PAD vs Web 核心差异对比

```mermaid
graph TB
    subgraph PAD_FLOW["PAD 前台录音（单会话）"]
        P1["登录用户 = 录音归属人\nDefaultIdentityResolver"]
        P2["单条 WS 连接\n始终保持"]
        P3["roleCode = RECEPTIONIST"]
        P4["声纹校验：弱（可选）"]
        P5["静默超时：可配置"]
    end

    subgraph WEB_FLOW["Web 工作站录音（多会话并发）"]
        W1["登录用户 ≠ 录音归属人\nHeaderDelegatedIdentityResolver"]
        W2["连接池\nMap&lt;registrationId, WsConn&gt;"]
        W3["roleCode = DOCTOR"]
        W4["声纹校验：强（叫号前必须通过）"]
        W5["静默超时：10min 无帧自动收尾"]
    end

    subgraph SHARED["共用部分（不改动）"]
        S1["AudioFrameHandler 核心逻辑"]
        S2["RecordingSessionCoordinator"]
        S3["RecordingSessionContextStore\nConcurrentHashMap 天然并发安全"]
        S4["AsrWebSocketClient"]
        S5["归档 / 分段 / Segment 处理"]
    end

    PAD_FLOW --> SHARED
    WEB_FLOW --> SHARED
```
