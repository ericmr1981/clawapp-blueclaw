# ClawApp × BlueClaw

> OpenClaw 移动客户端：SwiftUI iOS 原生 + H5/PWA 跨平台 + Node.js Proxy 中继

[![stars](https://img.shields.io/github/stars/ericmr1981/clawapp-blueclaw)](https://github.com/ericmr1981/clawapp-blueclaw/stargazers)
[![license](https://img.shields.io/github/license/ericmr1981/clawapp-blueclaw)](LICENSE)

## 特性

- 📱 **iOS 原生 App**（SwiftUI）— 移植自 [BlueClaw](https://github.com/brandon-dacrib/blueclaw)
- 🌐 **H5 / PWA**（浏览器直连）— 移植自 [ClawApp](https://github.com/qingchencloud/clawapp)
- 🔒 **Ed25519 设备签名** — 兼容 OpenClaw 2.15+ 认证
- 🌪️ **三连接模式** — Proxy SSE 中继 / WS 直连 / SSH Tunnel
- 📦 **一键部署** — Linux / macOS / Docker
- 💾 **离线缓存** — 断线重连，消息不丢失

## 架构

```
┌──────────────────┐    ┌──────────────────┐
│   iOS App        │    │   H5 / PWA       │
│  (SwiftUI)       │    │  (浏览器)         │
└────────┬─────────┘    └────────┬─────────┘
         │ SSE / WS              │ SSE / WS
         ▼                       ▼
┌──────────────────────────────────────────┐
│         Proxy Server (Node.js :3210)     │
│  SSE 长连接 · Ed25519 签名 · 会话缓存     │
└────────────────────┬─────────────────────┘
                     │ WS + Ed25519 签名
                     ▼
┌──────────────────────────────────────────┐
│         OpenClaw Gateway (:18789)         │
└──────────────────────────────────────────┘
```

## 三种连接模式

| 模式 | 适用场景 | 优点 |
|------|----------|------|
| **Proxy SSE 中继** | Gateway 在内网/NAT后 | 不暴露端口，跨网段 |
| **WS 直连** | Gateway 有公网IP | 最简单，延迟最低 |
| **SSH Tunnel** | 企业内网，无公网 | 加密隧道，跳板访问 |

## 快速开始

### Proxy Server 部署

```bash
# Linux / macOS
curl -fsSL https://raw.githubusercontent.com/ericmr1981/clawapp-blueclaw/main/install.sh | bash

# Docker
docker compose up -d
```

### iOS App（编译中）

待 Phase 4 完成，XcodeGen 生成后构建。

### H5（直接使用）

部署 Proxy Server 后，手机浏览器打开 `https://your-proxy:3210` 即可。

## 项目结构

```
clawapp-blueclaw/
├── docs/
│   ├── kickoff/
│   │   ├── PRD.md              ← 需求文档
│   │   └── ARCHITECTURE_DRAFT.md  ← 架构设计
│   └── decisions/
│       └── ADR-0001-tech-stack.md  ← 技术选型
├── features.json               ← 功能清单（22 项）
├── proxy/                      ← Proxy Server（Phase 1）
├── ios/                        ← SwiftUI App（Phase 4）
├── h5/                         ← H5/PWA（Phase 3）
└── shared/protocol/            ← 跨端协议类型
```

## 里程碑

```
M1 ✅ Kickoff 完成
M2 🔨 Proxy Server 核心
M3 🔨 H5 Client
M4 🔨 iOS App（融合 BlueClaw）
M5 🔨 三端联调
M6 🔨 v1.0 发布
```

## 技术栈

| 组件 | 技术 |
|------|------|
| Proxy Server | Node.js 18+ · Express · `ws` · `crypto` (Ed25519) |
| iOS App | Swift 5.9+ · SwiftUI · XcodeGen |
| H5 | 原生 JS · IndexedDB · Service Worker |
| 协议 | OpenClaw Gateway WebSocket Protocol |

## 参考项目

- [ClawApp](https://github.com/qingchencloud/clawapp) — JS/H5 实现，代理穿透方案来源
- [BlueClaw](https://github.com/brandon-dacrib/blueclaw) — SwiftUI iOS 原生实现，协议层来源
- [OpenClaw](https://github.com/openclaw/openclaw) — 核心平台

## License

MIT
