# ADR-0001: Tech Stack Decision — clawapp-blueclaw

- Status: **accepted**
- Date: 2026-04-18

## Context

Build a cross-platform mobile client (iOS native + H5 browser) that connects to OpenClaw Gateway behind NAT/firewall via a self-hosted relay proxy. Must support streaming responses, offline resilience, and three connection modes.

## Decision Drivers

| Driver | Weight |
|--------|--------|
| OpenClaw Gateway protocol compatibility | Critical |
| 可自托管，不依赖第三方 | Critical |
| iOS 原生体验 | High |
| 跨平台（H5 浏览器）| High |
| 防火墙穿透能力 | Critical |
| 团队技术栈熟悉度 | Medium |
| 部署简便性 | High |

## Options Considered

### Option A: ClawApp only（H5 + Node.js proxy）
- **Decision**: Rejected
- **Reason**: 无 iOS 原生 App，纯 Web 体验在 iOS 上受 Safari 限制，无 Keychain 安全存储，无 SwiftUI

### Option B: BlueClaw only（SwiftUI only）
- **Decision**: Partially rejected
- **Reason**: 仅 iOS，不支持 Android/无 App 用户；缺少 Proxy 中继模式（不支持内网 Gateway）；功能集薄

### Option C: ClawApp（H5 + Proxy）+ BlueClaw（iOS SwiftUI）并行维护
- **Decision**: Rejected
- **Reason**: 两套代码库，协议层重复维护，H5/iOS 行为不一致

### Option D: 融合架构 — Proxy 来自 ClawApp + iOS 来自 BlueClaw + 统一 H5
- **Decision**: **Accepted** ✅

## Decision

**融合方案**：
- **Proxy Server**: Node.js（ClawApp server 源码为基础，增强 Ed25519 签名）
- **iOS App**: Swift/SwiftUI（BlueClaw 源码为基础，新增 ProxySSEClient）
- **H5 Client**: 复用 ClawApp H5，新增 Proxy SSE Client
- **共享层**: `shared/protocol/` TypeScript 类型定义（跨端共用）

### Why Node.js for Proxy
| 优点 | 说明 |
|------|------|
| WebSocket 支持成熟 | `ws` npm 包，稳定 |
| Ed25519 内置 | `crypto` 模块原生支持 |
| SSE 实现简单 | `express` + `text/event-stream` |
| 部署广泛 | 任何 Node 18+ 环境 |
| ClawApp 已有成熟实现 | 减少重复开发 |

### Why SwiftUI for iOS
| 优点 | 说明 |
|------|------|
| BlueClaw 已有高质量实现 | 移植成本低 |
| Swift 并发原生支持 | `actor` / `AsyncStream` |
| SwiftUI MVVM 架构清晰 | ViewModels 分层明确 |
| Keychain 集成 | `Security.framework` |

### Why not React Native / Flutter
| 原因 | 说明 |
|------|------|
| SwiftUI 更贴近 BlueClaw 现有代码 | 移植路径最短 |
| Flutter 需要 Dart 重写 | 工作量大 |
| React Native 包体积大 | 不适合轻量 App |

## Consequences

### Positive
- 复用两个开源项目的成熟实现，开发成本最小化
- iOS 有原生体验，H5 有跨平台覆盖
- Proxy Server 部署在任意公网 VM 即可，成本 $0
- 三种连接模式覆盖所有网络场景

### Negative
- 两套前端代码（SwiftUI + H5 JS）需要保持同步
- 需要维护 Node.js 服务端
- ClawApp H5 和 BlueClaw Swift 功能集不完全一致

### Risks & Mitigations
| 风险 | 缓解 |
|------|------|
| OpenClaw WS 协议变更 | 协议层独立版本探测 |
| BlueClaw 停更 | 已有完整源码，可独立维护 |
| SSE 被企业防火墙屏蔽 | 降级 HTTP POST long-polling |

## Validation Plan

### Spike Checklist
- [ ] Proxy Server 在内网 Gateway + 公网 Proxy 环境下跑通
- [ ] iOS App 通过 SSE 模式收发消息
- [ ] H5 页面在 Safari/Chrome 离线缓存正常
- [ ] Ed25519 设备签名握手成功（Gateway 2.15+）
- [ ] 全新 VM 一键部署脚本 5 分钟内完成

### Baseline Benchmarks
| 指标 | 目标 |
|------|------|
| Proxy SSE 延迟（手机→手机响应） | < 800ms |
| iOS App 冷启动 | < 3s |
| H5 Lighthouse Performance | ≥ 80 |
| 部署脚本成功率 | ≥ 95% |
