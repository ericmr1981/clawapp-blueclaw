# Architecture Draft — clawapp-blueclaw

> Status: Draft v0.1
> Stack: Node.js + Swift/SwiftUI + H5

---

## 1. 设计目标

1. **外网穿透**：手机与内网 OpenClaw Gateway 之间的可靠连接，不暴露 Gateway 端口
2. **跨平台**：iOS 原生 + 浏览器 H5 双端可用
3. **协议统一**：Gateway WS 协议层在 Proxy/iOS/H5 三端共用
4. **可自托管**：不依赖任何第三方云服务

---

## 2. 系统边界

```
┌─────────────────────────────────────────────────────────────┐
│                      手机端                                  │
│  ┌─────────────────────┐    ┌─────────────────────────┐   │
│  │   iOS App (Swift)   │    │   H5 / PWA (浏览器)      │   │
│  │   BlueClaw SwiftUI   │    │   ClawApp Web Components │   │
│  └──────────┬───────────┘    └────────────┬────────────┘   │
│             │                              │                 │
│             │  WS / SSE / HTTP             │                 │
└─────────────┼──────────────────────────────┼─────────────────┘
              │                              │
   ┌──────────▼──────────┐         ┌──────────▼──────────┐
   │    Proxy Server     │         │   Direct WS Client  │
   │    (Node.js :3210)  │         │   (仅公网 Gateway)   │
   │                      │         └─────────────────────┘
   │  SSE endpoint        │                   │
   │  HTTP POST endpoint  │                   │
   │  WS endpoint         │                   │
   │  Ed25519 sign        │                   │
   │  Session store       │                   │
   └──────────┬────────────┘                   │
              │  WS + Ed25519 signature        │
              │  (或 SSH Tunnel :2222)          │
              ▼                                ▼
┌─────────────────────────────────────────────────────────────┐
│                OpenClaw Gateway                            │
│           ws://host:18789 (Gateway WS API)                  │
│                                                             │
│  消息渠道：WhatsApp / Telegram / ...                       │
│  Agent Loop / Tools / Session Manager                      │
└─────────────────────────────────────────────────────────────┘

上游依赖：OpenClaw Gateway（已有）
下游依赖：无
```

### 边界原则
- Proxy Server **不存储**消息内容，只做透传 + 缓存
- Proxy Server **不参与** OpenClaw Agent Loop 逻辑
- 三端（iOS/H5/Proxy）共用同一套 `shared/protocol/` 类型定义

---

## 3. 模块划分

### 3.1 `proxy/` — Proxy Server（Node.js）

```
proxy/
├── src/
│   ├── index.js              # 入口：HTTP Server（SSE + HTTP POST + WS）
│   ├── gateway-ws.js         # 与 OpenClaw Gateway 的 WS 客户端
│   ├── sse-client.js         # SSE 长连接管理
│   ├── auth.js               # Token 验证
│   ├── device-key.js         # Ed25519 密钥对生成/加载/签名
│   ├── session-store.js      # 会话管理 + 事件缓存
│   ├── offline-queue.js      # 离线消息队列
│   └── ssh-tunnel.js         # SSH Reverse Tunnel（可选）
├── .env.example
└── package.json
```

**关键设计决策**：
- 单进程单线程，用 `Map<sessionId, SessionState>` 管理会话
- Ed25519 密钥持久化到 `.device-key.json`（重启后可复用 deviceId）
- SSE 用 `text/event-stream`，心跳间隔 15s
- 事件缓存用环形缓冲区（max 200 条），支持断线续传

### 3.2 `ios/` — iOS App（Swift/SwiftUI）

```
ios/
├── BlueClaw/
│   ├── App/
│   │   └── BlueClawApp.swift
│   ├── Protocol/                    # ← 直接复用 BlueClaw 源码
│   │   ├── Frame.swift
│   │   ├── ConnectParams.swift
│   │   ├── Methods.swift
│   │   ├── ChatTypes.swift
│   │   ├── SessionTypes.swift
│   │   └── AnyCodable.swift
│   ├── Services/
│   │   ├── GatewayClient.swift     # 核心 WS 客户端
│   │   ├── WebSocketService.swift  # URLSessionWebSocketTask 封装
│   │   ├── SSHTunnelService.swift  # 保留 BlueClaw 原生
│   │   ├── ProxySSEClient.swift   # ★ 新增：连接 Proxy SSE
│   │   ├── BiometricAuth.swift
│   │   └── KeychainHelper.swift
│   ├── ViewModels/
│   │   ├── AppState.swift
│   │   ├── ChatViewModel.swift
│   │   ├── ConnectionViewModel.swift  # 三模式切换逻辑
│   │   └── VoiceViewModel.swift
│   ├── Views/
│   │   ├── Chat/
│   │   │   ├── ChatView.swift
│   │   │   ├── MessageBubbleView.swift
│   │   │   ├── MessageInputView.swift
│   │   │   └── MarkdownTextView.swift
│   │   ├── Connection/
│   │   │   └── ProxyConnectView.swift  # ★ 新增
│   │   ├── Sessions/
│   │   ├── Settings/
│   │   └── Voice/
│   └── Theme/
├── project.yml                 # XcodeGen
└── Podfile
```

### 3.3 `h5/` — H5 / PWA（复用 ClawApp）

```
h5/
├── src/
│   ├── ws-client.js           # WS 直连客户端（已有）
│   ├── proxy-client.js        # ★ 新增：SSE + HTTP POST
│   ├── chat-ui.js
│   ├── markdown.js
│   ├── message-db.js          # IndexedDB 离线缓存
│   ├── offline-queue.js
│   ├── i18n.js
│   ├── theme.js
│   └── main.js
├── public/
│   ├── manifest.json          # PWA
│   └── sw.js                  # Service Worker
├── index.html
└── vite.config.js
```

### 3.4 `shared/protocol/` — 跨端协议类型（TypeScript）

```
shared/
└── protocol/
    ├── methods.ts             # GatewayMethod enum
    ├── frame.ts               # RawFrame / RequestFrame
    └── chat-types.ts          # ChatMessage / Session
```

---

## 4. 数据流

### 4.1 Proxy SSE 模式（手机 → Proxy → Gateway）

```
1. 手机  GET /api/events?token=xxx&sessionId=s1
   ─────────────────────────────────────────────► Proxy Server
                                                      ├─ 查找/创建 session s1
                                                      ├─ 尚未连接 Gateway
                                                      └─ 保持 SSE 连接 open

2. 手机  POST /api/send { message: "hello" }
   ─────────────────────────────────────────────► Proxy Server
                                                      ├─ 查找 session s1
                                                      ├─ 首次：建立 WS 到 Gateway
                                                      │     ├─ 收到 challenge (nonce)
                                                      │     ├─ Ed25519 签名
                                                      │     └─ 发送 connect 帧
                                                      ├─ 收到 hello-ok，保存 sessionKey
                                                      ├─ 发送 chat.send 到 Gateway WS
                                                      └─ 返回 { ok: true, runId }

3. Gateway  WS → event:agent { stream:"assistant", delta:"He" }
   Proxy Server
   ├─ 收到 Gateway WS 消息
   ├─ 解析 frame，转为 SSE event
   └─ sseWrite(session.sseRes, "agent", frame)
   ─────────────────────────────────────────────► 手机 SSE stream
   "event: agent\ndata: {...}\n\n"

4. Gateway  WS → res (chat.send response)
   Proxy Server
   └─ sseWrite(session.sseRes, "response", frame)
   ─────────────────────────────────────────────► 手机 SSE

5. 手机  POST /api/disconnect
   ─────────────────────────────────────────────► Proxy Server
                                                      ├─ 关闭 Gateway WS
                                                      ├─ 保留 session 120s（linger）
                                                      └─ 关闭 SSE
```

### 4.2 设备 Ed25519 签名握手时序

```
手机 ──► Proxy Server ──► OpenClaw Gateway

1. Proxy → Gateway:  WS connect（含 deviceId + 无签名）
2. Gateway → Proxy:  challenge { nonce: "abc123", ... }
3. Proxy 签名: sig = Ed25519.sign(
     "v2|{deviceId}|backend|operator|scopes|{timestamp}|{token}|{nonce}"
   )
4. Proxy → Gateway:  WS connect（含 signature）
5. Gateway 验证签名 → hello-ok
```

---

## 5. 部署形态

### 5.1 Proxy Server（轻量，Node.js 单进程）

```bash
# 最简部署（一台有公网 IP 的 VM）
ssh my-vps
curl -fsSL https://raw.githubusercontent.com/.../install.sh | bash
# 安装脚本：
#   1. 安装 Node.js 18+
#   2. npm install
#   3. 生成 .env（随机 Token）
#   4. systemctl start clawapp-proxy
#   5. 输出：Proxy Server running on :3210
```

**Docker Compose 部署（推荐）**：
```yaml
services:
  proxy:
    image: clawapp-proxy:latest
    ports:
      - "3210:3210"      # SSE/HTTP/SSE
      - "3211:3211"      # WS 直通
    environment:
      - PROXY_TOKEN=<set-on-first-run>
      - OPENCLAW_GATEWAY_URL=ws://host:18789
      - OPENCLAW_GATEWAY_TOKEN=<gateway-token>
    restart: unless-stopped
```

### 5.2 iOS App（Xcode + TestFlight / App Store）

- 用 XcodeGen 生成 `.xcodeproj`
- `openclaw-mobile.mobileprovision` 签名
- 上传 TestFlight

### 5.3 H5 / PWA（静态托管）

- `h5/dist/` 目录直接 nginx 托管
- 或部署在 Proxy Server 同一域名（`proxy:3210` → `/app/` 路由）

---

## 6. 关键接口（草案）

### 6.1 Proxy Server HTTP API

| Method | Path | 说明 |
|--------|------|------|
| `GET` | `/api/events?token=&sessionId=` | SSE 事件流 |
| `POST` | `/api/send` | 发送消息 + 建立连接 |
| `POST` | `/api/disconnect` | 断开会话 |
| `POST` | `/api/connect` | 显式建立 Gateway WS（可选） |
| `GET` | `/health` | 健康检查 |

### 6.2 OpenClaw Gateway WS 协议（对 Proxy 暴露的方法）

| Method | 说明 |
|--------|------|
| `connect` | 首帧握手 |
| `chat.send` | 发送消息 |
| `chat.history` | 获取历史 |
| `chat.abort` | 中止运行 |
| `sessions.list` | 列出会话 |
| `agents.list` | 列出 Agent |

### 6.3 iOS → Proxy SSE Client API（Swift）

```swift
actor ProxySSEClient {
    func connect(proxyUrl: String, token: String, sessionId: String) async throws -> AsyncStream<SSEEvent>
    func send(message: String) async throws
    func disconnect() async
}

struct SSEEvent {
    let event: String      // "agent" | "response" | "error" | "ping"
    let frame: RawFrame    // Gateway RawFrame
}
```

---

## 7. 观测性与运维

| 维度 | 方案 |
|------|------|
| 日志 | Proxy: stdout JSON structured log → `journalctl` |
| 指标 | Proxy: Node.js `prom-client`（连接数/QPS/延迟） |
| 健康检查 | `GET /health` → `200 OK` |
| 告警 | 连接数 > 阈值 / Proxy → Gateway WS 断开 |
| iOS Debug | Xcode Network Instrument + NSLog |

---

## 8. Open Questions

| ID | 问题 | 决策 |
|----|------|------|
| Q1 | Proxy Server 是否需要 Redis 做分布式会话？ | **否**，单进程 `Map` 足够；后续多实例加 `Redis` |
| Q2 | H5 离线缓存用 IndexedDB 上限多少？ | 1000 条消息 / 10MB 超限淘汰旧消息 |
| Q3 | iOS 是否支持 Dark Mode？ | **是**，跟随系统 `ColorScheme`，BlueClaw 已有 |
| Q4 | Proxy Server 崩溃后手机如何处理？ | 自动重连 + 离线队列，SSE 断开时 3s 重试 |
| Q5 | SSH Tunnel 模式是否还需要？ | **保留**，适合企业内网无公网端口场景 |
