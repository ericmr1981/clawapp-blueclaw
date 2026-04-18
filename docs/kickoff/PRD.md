# PRD — clawapp-blueclaw

> Kickoff: 2026-04-18
> Stack: Node.js (Proxy) + Swift/SwiftUI (iOS) + H5/PWA (Cross-platform)
> Status: Draft v0.1

---

## 1. 背景与目标

### 1.1 Why Now?
OpenClaw Gateway 已有完整的 WebSocket 控制面协议，但官方移动端（iOS/Android App）尚未公开分发或功能有限。目前缺乏一个**开源、可自部署、跨平台（iOS/Android/浏览器）、支持外网穿透**的移动客户端。

ClawApp（JS/H5，388⭐）提供了完整的代理穿透方案和功能集，但技术债在于：
- 纯 Web 技术，无原生 iOS 体验
- SwiftUI 代码不可用

BlueClaw（Swift/SwiftUI，12⭐）提供了高质量原生 iOS 架构和协议层，但：
- 缺少中继代理（仅支持直连/SSH Tunnel）
- 功能集较薄

两者互补，融合价值高。

### 1.2 业务目标
- 让用户在**任何网络环境**下（内网/外网/NAT后）都能通过手机与 OpenClaw Agent 对话
- 提供**三种连接模式**：直连 WS、SSH Tunnel、Proxy SSE 中继
- 一套代码，支持 iOS 原生 App + H5 浏览器双端
- 完全自托管，不依赖第三方云服务

### 1.3 非业务目标
- 不是另一个 OpenClaw Gateway（不承担消息渠道接入职责）
- 不做 Agent 能力扩展（工具/记忆等由 OpenClaw 自身管理）

---

## 2. 用户与场景

### 2.1 目标用户
| 用户类型 | 场景 |
|---------|------|
| 技术用户 / 开发者 | 已有 OpenClaw Gateway，需要移动端管理 |
| 自托管用户 | Gateway 在家庭 NAS/树莓派/内网服务器，无法暴露公网端口 |
| 企业用户 | 通过手机随时向 AI Agent 提问，不开电脑 |

### 2.2 核心场景（Top 3）

**场景 1：外网穿透访问家庭 NAS 上的 OpenClaw**
> 用户在家内网 NAS 上运行 OpenClaw Gateway，手机通过 Proxy Server（可部署在云服务器）SSE 隧道连接，不暴露 NAS 端口。

**场景 2：出差途中使用 AI 助手**
> 用户通过 iOS 原生 App（BlueClaw SwiftUI）直连公司服务器的 OpenClaw，支持打字机效果流式回复。

**场景 3：临时借用他人手机快速对话**
> 无需安装 App，通过手机浏览器（H5/PWA）访问 Proxy Server 的 Web 页面，输入 Token 即可对话。

---

## 3. 需求范围

### 3.1 In Scope

#### Proxy Server（Node.js）
| 功能 | 优先级 | 说明 |
|------|--------|------|
| SSE 长连接（手机 → Proxy → Gateway） | P0 | 穿透防火墙，支持 100+ 并发 |
| HTTP POST → Gateway WS 转发 | P0 | SSE 不可用时的降级方案 |
| WS 直通模式（Proxy → Gateway） | P0 | H5 WS-Compat 客户端 |
| Ed25519 设备签名握手 | P0 | 兼容 OpenClaw 2.15+ 认证 |
| Token 密码认证 | P0 | 首次连接时设置 |
| 消息事件流缓存 + 断线续传 | P1 | 200 条事件环形缓冲区 |
| 离线队列（手机离线时消息暂存） | P2 | 重连后自动发送 |
| 部署脚本（Linux/macOS/Windows） | P0 | 一键启动 |

#### iOS App（SwiftUI）
| 功能 | 优先级 | 说明 |
|------|--------|------|
| Gateway WS 协议实现 | P0 | 移植 BlueClaw Protocol 层 |
| Proxy SSE Client | P0 | 连接 Proxy Server |
| 直连 WS Client | P0 | 保留 BlueClaw 原有能力 |
| SSH Tunnel Client | P1 | 保留 BlueClaw 原有能力 |
| 聊天界面（消息气泡/Markdown） | P0 | 移植 BlueClaw ChatView |
| 流式回复（打字机效果） | P0 | BlueClaw 已有 |
| 多会话管理 | P0 | BlueClaw SessionListView |
| 生物识别解锁（Face ID/Touch ID） | P1 | BlueClaw 已有 |
| 图片发送 | P1 | BlueClaw AttachmentPreviewView |
| 连接模式切换 UI | P0 | 新增 Proxy/WS/SSH 三选项卡 |
| Ed25519 设备密钥存储（Keychain） | P0 | 移植 device-key 逻辑 |

#### H5 / PWA
| 功能 | 优先级 | 说明 |
|------|--------|------|
| Proxy SSE Client | P0 | 移植 ClawApp ws-client.js |
| HTTP POST fallback | P0 | ClawApp 已有 |
| IndexedDB 离线缓存 | P0 | ClawApp message-db.js |
| Markdown 渲染 | P0 | ClawApp markdown.js |
| PWA 安装（manifest + sw.js） | P0 | ClawApp 已有 |
| 暗色/亮色主题 | P0 | ClawApp theme.js |

### 3.2 Out of Scope
- 语音输入 / TTS（Future Phase）
- Canvas / Camera / Screen Recording Node 能力（Future Phase）
- 消息渠道接入（WhatsApp/Telegram 等，由 OpenClaw Gateway 负责）
- 用户注册 / 多租户管理（Future Phase）
- 端到端加密（E2EE）

---

## 4. 验收标准（Acceptance Criteria）

| ID | 标准 | 验证方法 |
|----|------|----------|
| AC-1 | iOS App 通过 Proxy SSE 模式成功发送消息并接收流式回复 | 内网 Gateway + 云端 Proxy Server，测试往返 |
| AC-2 | H5 页面在 Safari/Chrome 打开，输入 Token 后可对话 | 浏览器 Network 面板确认 SSE 连接 |
| AC-3 | 断开网络 30s 后重连，离线消息不丢失 | 飞行模式测试 |
| AC-4 | 直连 WS 模式（Gateway 在公网）仍然可用 | SSH Tunnel 模式回归测试 |
| AC-5 | Ed25519 设备签名握手成功（OpenClaw 2.15+） | Gateway 日志无签名验证错误 |
| AC-6 | 部署脚本在全新 Ubuntu 22.04 上 5 分钟内跑通 | 全新 VM 测试 |
| AC-7 | iOS App 通过 TestFlight 分发，续航无明显异常 | Xcode Organizer 能量分析 |

---

## 5. 约束与非功能需求

### 5.1 性能
| 指标 | 目标 |
|------|------|
| 消息延迟（手机 → Proxy → Gateway → 响应） | < 500ms（网络因素除外） |
| SSE 重连时间 | < 2s |
| iOS App 冷启动时间 | < 3s |
| Proxy Server 并发连接数 | ≥ 100 |

### 5.2 成本
- 目标：$0（完全自托管）
- Proxy Server 最小规格：1核 512MB VM（可选部署在已有云服务器）

### 5.3 安全
- Token 认证：Bearer Token（首次连接时设置）
- 设备签名：Ed25519（兼容 OpenClaw 2.15+）
- iOS Keychain：密钥不落磁盘明文
- HTTPS：Proxy Server 强制 TLS（生产部署）
- 不存储消息内容（只做透传）

### 5.4 部署环境
- Proxy Server：Linux（主流发行版）/ macOS / Docker
- iOS App：iOS 16.0+
- 浏览器：Safari / Chrome 最新版

---

## 6. 风险与未知

| ID | 风险 | 缓解方案 |
|----|------|----------|
| R-1 | OpenClaw Gateway WS 协议版本变更导致握手失败 | 协议层版本探测，失败时提示升级 |
| R-2 | Proxy Server 成为单点故障 | 支持直连 WS 降级；Proxy 可水平扩展 |
| R-3 | Ed25519 密钥在 iOS Keychain 同步到新设备 | 首次连接重新生成密钥，不做跨设备同步 |
| R-4 | SSE 在某些企业防火墙下被阻断 | 自动降级到 HTTP POST long-polling |
| R-5 | ClawApp H5 代码与 BlueClaw Swift 实现产生行为不一致 | 统一共享协议类型定义（shared/protocol/） |

---

## 7. 里程碑

```
M1 ✅ Kickoff docs confirmed          ← 当前
M2 🔨 Proxy Server 核心功能完成       ← 先做这个（Phase 1）
M3 🔨 H5 Client 跑通                 ← 复刻 ClawApp 功能
M4 🔨 iOS App SwiftUI + Proxy Client ← 融合 BlueClaw
M5 🔨 三端联调 + AC 验证
M6 🔨 正式发布 v1.0
```

### M2 交付物
- [ ] `server/index.js` SSE + WS + HTTP POST 三模式
- [ ] Ed25519 设备签名（兼容 OpenClaw 2.15）
- [ ] Token 认证
- [ ] 消息事件缓存（断线续传）
- [ ] `install.sh` 一键部署脚本
- [ ] Docker Compose 部署配置

### M3 交付物
- [ ] `h5/` 全功能 H5 聊天页面
- [ ] IndexedDB 离线缓存
- [ ] PWA manifest + service worker
- [ ] 暗色/亮色主题

### M4 交付物
- [ ] `ios/BlueClaw/` SwiftUI App（BlueClaw 源码为模板）
- [ ] ProxySSEClient Swift 实现
- [ ] 连接模式切换 UI（Proxy / WS 直连 / SSH Tunnel）
- [ ] Keychain Ed25519 密钥管理

---

## 8. 成功度量

| 指标 | 目标值 |
|------|--------|
| 三种连接模式全部可用 | 100% |
| iOS App Store 审核通过 | ✅ |
| PWA Lighthouse Performance Score | ≥ 80 |
| 核心用户（技术爱好者/自托管用户）首次配置成功率 | ≥ 90% |
