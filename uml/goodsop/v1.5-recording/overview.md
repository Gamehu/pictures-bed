# v1.5 录音架构总览图

> 本文档为 `v1.5-详细设计.md` 第 9 章配套图示，呈现 PAD 与 Web 工作站共用后端录音架构的整体结构。
>
> 详细流程图见：
> - [v1.5-架构图-PAD录音.md](./v1.5-架构图-PAD录音.md)
> - [v1.5-架构图-Web录音.md](./v1.5-架构图-Web录音.md)

---

## 1. 整体架构分层图

```mermaid
graph TB
    subgraph Clients["接入端（两端共用同一套后端）"]
        PAD["PAD 前台\nroleCode=RECEPTIONIST 单连接"]
        WEB["Web 工作站\nroleCode=DOCTOR 连接池 多医生"]
    end

    subgraph WSInfra["goodsop-common-websocket（基础设施层）"]
        Interceptor["UserAttributeHandshakeInterceptor\n验证JWT 提取用户属性"]
        Router["CustomWebSocketHandler\n按type路由消息"]
        SessionHolder["WebSocketSessionHolder"]
    end

    subgraph BizLayer["goodsop-app-server-biz（业务层）"]
        AFH["AudioFrameHandler\n音频帧处理 状态机控制"]
        Resolver["RecordingIdentityResolver\nStrategy模式"]
        Coordinator["RecordingSessionCoordinator\nSTART PAUSE RESUME STOP"]
        Store["RecordingSessionContextStore\nConcurrentHashMap"]
        Timeout["AsrSessionTimeoutManager\n静默超时 10min"]
    end

    subgraph Downstream["下游服务"]
        ASR["AsrWebSocketClient\n第三方ASR转写"]
        ConsultSvc["ConsultationService\n开始诊疗 声纹校验 状态流转"]
    end

    subgraph Storage["存储层"]
        DB[("PostgreSQL")]
        Redis[("Redis\n会话状态持久化")]
    end

    PAD --> Interceptor
    WEB --> Interceptor
    Interceptor --> Router
    Router --> SessionHolder
    Router --> AFH
    AFH --> Resolver
    AFH --> Coordinator
    Coordinator --> Store
    Store --> Timeout
    AFH --> ASR
    ConsultSvc --> AFH
    Coordinator --> DB
    Store --> Redis
```

---

## 2. Session ID 双层设计

```mermaid
graph LR
    subgraph Frontend["前端"]
        REG["registrationId\n挂号单ID 业务唯一键"]
        WS_ID["wsSessionId\nUUID 每次WS连接独立"]
    end

    subgraph Backend["后端内存"]
        CTX_STORE["RecordingSessionContextStore\nkey=wsSessionId"]
    end

    subgraph DB["数据库"]
        ARCHIVE["archiveSessionId\n=lu_communication_log.id\n开始诊疗接口返回"]
    end

    REG -- "1.开始诊疗接口" --> ARCHIVE
    ARCHIVE -- "2.返回给前端" --> WS_ID
    WS_ID -- "3.WS握手+START帧" --> CTX_STORE
    ARCHIVE -- "4.Header携带" --> CTX_STORE

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

    class HeaderDelegatedIdentityResolver {
        +resolveRecordingUserId(session, loginUserId) Long
    }

    class AudioFrameHandler {
        -identityResolver RecordingIdentityResolver
        +initializeSession(session, attrs) void
        +handleBinaryMessage(session, message) void
        +handleControlAction(session, dto) void
    }

    class SessionContext {
        +recordingUserId Long
        +state RecordingState
        +archiveSessionId String
        +roleCode String
    }

    RecordingIdentityResolver <|.. DefaultIdentityResolver : PAD场景 loginUser=recordingUser
    RecordingIdentityResolver <|.. HeaderDelegatedIdentityResolver : Web场景 读doctorId Header
    AudioFrameHandler --> RecordingIdentityResolver : 组合
    AudioFrameHandler --> SessionContext : 创建/操作
```

---

## 4. PAD vs Web 核心差异对比

| 维度 | PAD 前台录音 | Web 工作站录音 |
|------|-------------|--------------|
| 登录用户与录音归属 | 登录用户 = 录音归属人 | 登录用户（院长）≠ 录音归属人（医生） |
| 连接数 | 单条 WS 始终保持 | 连接池，多医生并发各一条 |
| roleCode | RECEPTIONIST | DOCTOR |
| 声纹校验 | 弱（可选） | 强（开始诊疗前必须通过） |
| 静默超时 | 可配置 | 10min 无帧自动收尾 |
| 身份解析器 | DefaultIdentityResolver | HeaderDelegatedIdentityResolver |
| 后端改动 | 无 | 仅新增 Resolver 实现 |
