# Tether 设计 v2 — 单 binary + 浏览器 + WebTransport

**Date:** 2026-05-09
**Author:** wxk + Claude
**Status:** **Draft — P0 sections complete; light chair review 2026-05-09 done; awaiting user sign-off to replace v1 as source-of-record.**
P1/P2 sections (§10.G/H/I) remain stubs by priority (see §10.0). v1 remains source-of-record until user pulls the trigger to switch.
**Companion addenda（已写）：**
- D-05a `2026-05-09-d05a-stream-json-dual-channel.md` — cc agent 双通道
- D-17a `2026-05-09-d17a-agent-provider-multi-provider.md` — multi-provider 路径
- D-22 `2026-05-09-d22-tygo-schema-sync.md` — Go↔TS schema 同步

---

## 0. 这份文档是什么

v1（2026-04-26）锁定了 21 项 D-XX 决策、押"Go QUIC daemon + Tauri 多端 native 客户端 + 三层 E2E"路线，工程时间 14-16 周（紧），milestone 2026-08-21。

2026-05 用 cloudcli 一周 + PoC step7 / 8 / 9 实测后，**用户决定大幅简化**（见项目记忆 `project_tether_simplification_consideration.md`）：

> **用一个 Go binary + 纯浏览器前端 + WebTransport 通信，砍掉 Tauri tether-app 仓 + native 多端 build matrix + 三层 E2E。** 借 cloudcli 验证过的产品形态做 80%，剩下 20% 押 WebTransport（连接迁移 / 多 stream / datagram）这条 cloudcli 没做的真护城河。

这份 v2 把简化方向落地：

- **§1**：v2 的三个核心架构选择（**替代** v1 §1）
- **§2**：21 项 D-XX 决策的迁移表（keep / drop / replace / 链接 addendum）
- **§3-§N**：哪些 v1 章节仍 100% 适用（reference），哪些需要重写（标 TBD）
- **§Acceptance**：v0.1 简化版 milestone 的 acceptance criteria

工程时间预估：**8-10 周**（v1 是 14-16 周）。

## 1. 三个核心架构选择（v2）

### 1.1 部署形态：单 Go binary + 浏览器

```
                            ┌─ HTTP/3 → 静态 React SPA (embed.FS)
[ tether ]  ←──── 单 binary ─┤
                            └─ WebTransport → /wt/chat /wt/shell
                                                ↓
                                          spawn cc subprocess (D-05a)
```

**对比 v1**：

| 维度 | v1 | v2 |
|---|---|---|
| Native binary 数 | 5（Tauri × 4 平台 + Go daemon） | **1**（Go binary） |
| 用户安装步骤 | 装 daemon + 装 native app | **1 行**：`go install github.com/piaobeizu/tether@latest` 或下 binary |
| 客户端形态 | Tauri 多端原生 app | **浏览器**（任意支持 WebTransport 的） |
| Mobile 形态 | Tauri Android APK + iOS（v0.1.x） | **PWA**（Chrome "添加到主屏幕"） |
| iOS 支持 | spec 留口子 | Safari 26.4+（Webtransport 支持） |
| Build matrix | macOS/Linux/Win/Android × Tauri Mobile/Desktop | Linux/macOS/Win × Go 跨编译（标准 Go 流程） |

**做法**：
- Go embed.FS 把 `web/dist/` （vite build 产出）打进 binary
- HTTP/3 server 同时托管：(1) `/` 静态 SPA、(2) `/wt/*` WebTransport endpoints、(3) `/api/*` REST endpoints
- 浏览器加载页 → JS 用 `new WebTransport('https://<host>/wt/chat', ...)` 拨号 → 跟 daemon 直接 binary protocol

PoC step7 已验证：单 binary 同时跑 embed 静态 + WebTransport endpoint 走得通（5/5 测试通过）。

### 1.2 传输协议：WebTransport over HTTP/3（仍 keep）

**v1 D-02 关于 WebTransport 的部分保留**——这是 tether 跟 cloudcli 的真差异化。
**砍掉 v1 D-02 关于 Tauri 自写 transport command + web-transport-quinn shim 的部分**——浏览器原生 `WebTransport` API 直接用，无 polyfill。

**WebTransport 给的能力**（cloudcli WS 给不了）：

- **连接迁移**：手机 WiFi↔4G 不掉线
- **多 stream 不阻塞**：chat / blob / file / presence 各自独立
- **真 datagram (RFC 9297)**：presence / typing 心跳低延迟
- **HoL blocking 免疫**：弱网下 chat / 文件传输各管各的

**约束**：iOS Safari 需要 26.4+。早期 iOS 用户暂不支持。

### 1.3 cc agent 集成：stream-json + PTY 双通道（D-05a 替代 D-05）

```
浏览器
├─ Chat 标签 (主交互)
│   ├─ React 渲染 fenced-block / tool-use / text 等结构化事件
│   └─ WT bidi stream → 收 stream-json 事件
│
└─ Shell 标签 (slash commands / power-user)
    ├─ xterm.js
    └─ WT bidi stream → 双向透传 PTY bytes

tether daemon (Go)
├─ /wt/chat — 长跑 stream-json claude 子进程
└─ /wt/shell — PTY claude --resume <sid> 子进程
```

详见 `D-05a addendum`。PoC step8 已验证（3/3 PASS）。

## 2. D-XX 决策迁移表

| D-XX | 主题 | v1 决策 | v2 决策 | 改动 |
|---|---|---|---|---|
| **D-01** | CLI / server 语言 | Go，CLI + server 两个 binary | Go，**单 binary** + 子命令 (`tether server` / `tether attach`) | **简化**：合二为一 |
| **D-02** | 传输协议 | WebTransport + Tauri 自写 transport command | WebTransport（**浏览器原生 API，无 polyfill**） | **大幅减**：transport shim 整段砍 |
| **D-03** | 整体拓扑 | 三层 Mobile App ↔ server ↔ CLI | **两层** Browser ↔ Go binary | **简化** |
| **D-04** | v0.1 范围 | 单用户单机 | 同 v1 | **保留** |
| **D-05** | cc 交互方式 | PTY + Hooks + JSONL watcher，禁用 SDK / stream-json | **替代为 D-05a**：stream-json + PTY 双通道 | **替换** → D-05a |
| **D-06** | PTY 持有 | daemon goroutine 自持 | 仍持，但仅 shell 副通道用；chat 主通道走 stream-json | **范围缩** |
| **D-07** | Daemon 死亡语义 | daemon 死 → agent 死，replay 兜底 | 同 v1 | **保留** |
| **D-08** | 切换心智模型 | agent 长存，多 client 共 attach | 同 v1（cloudcli 模式已验证多设备共挂同 session） | **保留** |
| **D-09** | Lock 强制性 | input gate 在 daemon 内存 | 同 v1 | **保留** |
| **D-10** | 终端入口 | `tether attach <sid>` raw-mode TTY | 同 v1，且 stream-json 让 attach 客户端可重建装饰层（~150 LOC） | **保留** |
| **D-11** | CLI 进程结构 | main watchdog + daemon goroutine + client goroutine | **简化**：删 client goroutine（browser 才是 client，无需 binary 内 model） | **简化** |
| **D-12** | E2E 加密 | 完整 E2E（X25519 + XChaCha20 + HKDF），server 看不到 plaintext | **v0.1 降级**：browser TLS 终结于 server（cloudcli 同等威胁模型）。**v1.0+** 评估 browser-side WebCrypto E2E（私钥存 IndexedDB） | **降级** + v1.0+ 重新设计 |
| **D-13** | App 壳子 | Tauri 2 Mobile + Tauri 2 Desktop 4 target | **整段砍**。浏览器 PWA 替代。tether-app 仓**删除** | **删除** |
| **D-14** | Wire 协议 | Envelope-based + 5 channel | 同 v1（多 stream WebTransport 直接对应） | **保留** |
| **D-15** | Lock 状态机 | 单层 lock + 60s auto release + force takeover | 同 v1 | **保留** |
| **D-16** | Auth Token | J2 简化版 access + refresh | 同 v1（HttpOnly cookie 或 localStorage 都行） | **保留** |
| **D-17** | AgentProvider seam | v0.1 仅 cc，不暴露多 provider 承诺 | 同 v1，但**新增 D-17a addendum** 写明未来扩展路径与每 provider 踩坑 | **保留** + D-17a |
| **D-18** | v0.2 forward-compat（9 项 seam） | workspaceRoot / `/hooks/` / `/blob/*` 等 | 同 v1（独立于 transport / shell） | **保留** |
| **D-19** | App surface model（三栏 + chat-first + fenced-block） | 桌面三栏 + 移动单栏 + 共享 fenced-block renderer | 同 v1，仅运行容器从 Tauri WebView → 浏览器 WebView | **保留**（核心愿景不变） |
| **D-20** | Skill = cc plugin overlay | 同左 | 同 v1（服务端模型，独立于前端） | **保留** |
| **D-21** | Desktop / Mobile 统一 transport | desktop 永远是 daemon 的 remote 客户端，无 UDS 直连 | 同 v1，新方向更对称（browser 天生 remote） | **保留** |

**新增 D-XX**：
- **D-22**: Tygo 单向 schema 同步（Go → TypeScript）
- **D-05a**: cc agent stream-json + PTY 双通道
- **D-17a**: AgentProvider 扩展路径与多 provider 踩坑表

**统计**：保留 12 / 简化 3 / 替换 1 / 降级 1 / 删除 1 / 范围缩 1 / 新增 3。**75% 决策原封不动。**

## 3. 架构总览（v2 重写 v1 §3）

```
┌──────────────── 用户设备 ────────────────┐
│                                          │
│  Chrome / Safari / Firefox（任意）       │
│   └─ React SPA（D-19 三栏 / chat-first） │
│       ├─ workspace tree   panel 1        │
│       ├─ skill output     panel 2        │
│       └─ chat / shell     panel 3        │
│            ↓ WebTransport                │
│            ↓ HTTPS/H3 (TCP+UDP fallback) │
└──────────────────│───────────────────────┘
                   │
                   │ HTTP/3 (UDP/443)
                   │
┌──────────────────│───────────────────────┐
│  GCP VM / 工作站                          │
│  ┌────────────────────────────────────┐  │
│  │  tether（单 Go binary）             │  │
│  │                                     │  │
│  │   embed.FS (web/dist/*)            │  │
│  │   ↓                                 │  │
│  │   HTTP/3 server                     │  │
│  │     ├─ /         → static SPA      │  │
│  │     ├─ /api/*    → REST endpoints  │  │
│  │     └─ /wt/*     → WebTransport    │  │
│  │            ├─ /wt/chat → stream-json claude (D-05a) │
│  │            ├─ /wt/shell → PTY claude               │
│  │            └─ /wt/events → broadcast             │
│  │   ↓                                 │  │
│  │   internal/                         │  │
│  │     ├─ agent/        # AgentProvider impl (D-17a) │
│  │     ├─ workspace/    # D-19 状态层               │
│  │     ├─ skill/        # D-20 cc plugin overlay   │
│  │     ├─ wire/         # D-22 schema source       │
│  │     └─ session/      # D-08/D-15 lock + multi-attach │
│  │                                     │  │
│  │   subprocess: claude (long-running, stream-json) │
│  │   subprocess: claude (PTY, on-demand for shell) │
│  └────────────────────────────────────┘  │
│                                          │
│  ~/.claude/projects/<sid>/<sid>.jsonl    │
│  ~/.tether/                              │
│      ├─ workspaces.toml                  │
│      └─ skills/<skill>/...               │
└──────────────────────────────────────────┘
```

**关键变化对照 v1**：
- ✗ 没有 `tether-app` Tauri 客户端
- ✗ 没有"自写 transport command + web-transport-quinn shim"
- ✗ 没有"Mobile App ↔ server ↔ CLI" 三层
- ✗ 没有 4-target Tauri build matrix
- ✗ 没有 native push（FCM / APNs）
- ✗ 没有 native keychain 集成（macOS Keychain / libsecret / Windows Credential Manager）
- ✓ 增加：Go embed.FS 静态托管 React SPA
- ✓ 增加：HTTP/3 + WebTransport 单 listener
- ✓ 增加：双通道 cc 集成（stream-json + PTY）

## 4. 仍 100% 适用的 v1 章节

以下 v1 章节内容**不动**，v2 直接 reference：

| v1 章节 | 主题 | 状态 |
|---|---|---|
| §3.3 | Wire envelope 5 channel 协议 | **保留**（D-14） |
| §4 | CLI 进程模型 | **大改**：参 v2 §[Y]TBD（删 client goroutine） |
| §5 | Agent 交互模型（PTY + Hooks + JSONL） | **替换**：参 D-05a |
| §6 | Daemon 死亡 / 恢复契约 | **保留**（D-07） |
| §7 | 切换心智模型（agent 长存，multi-attach） | **保留**（D-08） |
| §8 | Lock 状态机 | **保留**（D-15） |
| §9 | Server 角色 | **范围缩**：v2 server 跟 daemon 合并（D-01） |
| §10 | PoC 优先级 | **大改**：v2 §[Z]TBD 重写（PoC 1-6 对应 v2 step7-9 已 PASS） |
| §11.X / D-18 | v0.2 forward-compat seam | **保留** |
| §11.Y / D-19 | App surface model | **保留**（容器 Tauri → 浏览器） |
| §11.Z / D-20 | Skill = cc plugin overlay | **保留** |

## 5. 待重写章节（TBD 清单）

以下章节需要根据 v2 方向重写。**优先级标注**：[P0] 必须 v0.1 前完成，[P1] 实施期可逐步细化，[P2] v0.2+ 再说。

| 章节 | 原 v1 章节 | 重写要点 | 优先级 |
|---|---|---|---|
| §[A]TBD: 单 binary 启动流程 | v1 §3 部分 | `tether server` 子命令、env 配置、TLS cert 管理（mkcert / Let's Encrypt / serverCertificateHashes pinning） | [P0] |
| §[B]TBD: HTTP/3 server 设计 | （v1 没对应） | h3.Server 配置、ConfigureHTTP3Server() 必要、static + WT mux | [P0] |
| §[C]TBD: 浏览器 SPA 项目结构 | v1 §11.Y.3（部分） | `web/` 目录、Vite 配置、tygo 集成、SPA 路由（D-19 三栏 + 移动 chat-first） | [P0] |
| §[D]TBD: cc agent 双通道实施细节 | v1 §5 重写 | D-05a 草案 → 完整 spec，含 stream-json 事件 schema、PTY shell endpoint、共享 sessionId 协议 | [P0] |
| §[E]TBD: AgentProvider Go interface | v1 §11.W 重写 | D-17a 草案 → Go interface 定义文件路径 + 各 provider stub 文件结构 | [P0] |
| §[F]TBD: tygo schema 工作流 | （v1 没对应） | D-22 草案 → CI 集成、wire-contract test 写法 | [P0] |
| §[G]TBD: PWA + 浏览器分发 | v1 §11.G 重写 | manifest.json、service worker、添加到主屏幕 UX、Chrome flags 处理 HTTP 源 | [P1] |
| §[H]TBD: TLS / 证书策略 | v1 §11.G 部分 | dev：mkcert / IS_SANDBOX；prod：Let's Encrypt / Cloudflare Tunnel | [P1] |
| §[I]TBD: WebCrypto E2E（v1.0+） | v1 §11.C 重写 | 浏览器 SubtleCrypto X25519 + age-style 信封；私钥 IndexedDB 持久化；多设备 pair 协议（无 native client） | [P2] |
| §[J]TBD: PoC 路线 | v1 §10 重写 | step7 ✓ / step8 ✓ / step9 ✓；step10 = browser dual-channel 端到端 | [P0] |
| §[K]TBD: v0.1 acceptance criteria | v1 缺 | 实际可验收条件清单，跟 v0.1 milestone 2026-08-21 对齐 | [P0] |

## 6. 工程时间重估

| Phase | 周数估算 | 主要工作 |
|---|---|---|
| Phase 0：spec 重写完成 | 1-1.5 | TBD §[A]-§[F] + §[J]-§[K] 全部填完 |
| Phase 1：单 binary 骨架 + cc 双通道 | 2-2.5 | §[B] + §[D]，AgentProvider impl，PoC step10 通过 |
| Phase 2：tether-web React SPA + tygo 集成 | 2.5-3 | §[C] + §[F]，D-19 三栏 layout + fenced-block renderer |
| Phase 3：workspace / skill / lock | 1.5-2 | v1 §6/§7/§8 实现（unchanged） |
| Phase 4：v0.1 acceptance + dogfood | 1 | §[K] checklist 全过；用户日常用 tether 跑活儿 |
| **总计** | **8-10 周** | （v1 估 14-16 周） |

**关键 milestone**：v0.1 ship date 仍可保 **2026-08-21** —— 现在距离 14 周，按 v2 估算有 4-6 周缓冲。

## 7. v0.1 简化版 acceptance criteria（draft）

- [ ] `go install github.com/piaobeizu/tether@latest` → 可装单 binary
- [ ] `tether server --port 8898` → 起 HTTP/3 + WebTransport
- [ ] Chrome / Edge / Firefox / Safari 26.4+ 浏览器开 `https://<host>:8898/` → React SPA 加载
- [ ] 浏览器 chat 通道：发送 prompt → 实时收 stream-json 事件 → fenced-block renderer 正确显示文本 / tool_use / result
- [ ] 浏览器 shell 通道：xterm.js 跑 `claude --resume`，slash command（`/help` 至少）工作
- [ ] D-19 三栏桌面 layout（左 workspace / 中 skill / 右 chat）正确渲染
- [ ] D-19 移动单栏 layout 正确渲染（≤ 600px 视口）
- [ ] 多设备 attach：两个浏览器 tab 登同 session 都看到实时事件
- [ ] PWA：Chrome "添加到主屏幕" 装上后图标可启动，全屏体验
- [ ] cc plugin overlay (D-20): `~/.tether/skills/<id>/` 状态写入，`<workspace>/.claude/plugins/<id>` symlink farm 正确
- [ ] tygo CI: `make codegen` + `git diff --exit-code` clean
- [ ] wire-contract test: ≥ 5 项 wire 约定有 contract test 覆盖

## 8. 写作进度

| 部分 | 状态 |
|---|---|
| §0 文档说明 | ✅ |
| §1 三个核心架构选择 | ✅ |
| §2 D-XX 迁移表 | ✅ |
| §3 架构总览 | ✅ |
| §4 reference v1 章节 | ✅ |
| §5 TBD 清单 | ✅ |
| §6 工程时间 | ✅ |
| §7 v0.1 acceptance | ✅ |
| §10 详细设计（原 §[A]-§[K]） | ✅ P0 实质内容；P1/P2 留位 |

PoC step10 验证后（2026-05-09），P0 §10.A/B/C/D/E/F/J/K 已填。P1/P2 (§10.G/H/I) 仅占位 + 关键事实。

---

# §10 详细设计

## §10.0 优先级定义（P0 / P1 / P2）

本节后续每个 subsection 标注优先级，含义如下：

| 标签 | 含义 | 不做会怎样 |
|---|---|---|
| **P0** | v0.1 ship (milestone 2026-08-21) 前**必须写完 + 实现** | 软件跑不起来 / 核心功能缺失 |
| **P1** | v0.1 实施期**可以先粗后细**，ship 前最终敲定 | 软件能跑，但用户体验有 friction / 文档有空白 |
| **P2** | v0.5 / v1.0+ 再做 | v0.1 范围内**完全不做**，仅留 stub / 占位 |

**判断准则**："没它东西不工作" → P0；"能跑后才会暴露" → P1；"D-12 那种已经明确 v1.0+ 的" → P2。

## §10.A 单 binary 启动流程 [P0]

### A.1 命令结构

```
tether <subcommand> [options]

子命令：
  server          起 HTTP/3 + WebTransport 服务
  attach <sid>    raw-mode TTY 客户端进入活会话（D-10）
  pair            生成 device pair token（v1.0+ 配合 WebCrypto E2E §10.I）
  doctor          诊断：cc binary 路径 / cert / port reachability / dep 版本
  version
```

`server` 是 99% 用户的入口。其他子命令是 power-user / debug / 未来。

### A.2 server 子命令配置

env 变量优先（容器友好）+ flag 后置覆盖：

| env / flag | 默认 | 说明 |
|---|---|---|
| `TETHER_HOST` / `--host` | `0.0.0.0` | bind address |
| `TETHER_PORT` / `--port` | `8898` | HTTP/3 + WT 端口（UDP）。同时 bind TCP 同端口（HTTP/2 alt-svc）— 见 §10.B |
| `TETHER_CC_PATH` / `--cc-path` | auto-discover | cc binary 位置；空则 `exec.LookPath("claude")` + 5 个备选路径回退 |
| `TETHER_CERT_FILE` / `--cert-file` | _(none)_ | TLS cert PEM 路径；为空则用自签名 ECDSA P-256（见 §10.H） |
| `TETHER_KEY_FILE` / `--key-file` | _(none)_ | 同上 |
| `TETHER_DATA_DIR` / `--data-dir` | `~/.tether/` | workspace / skill / config 持久化根 |
| `TETHER_LOG_LEVEL` / `--log-level` | `info` | trace / debug / info / warn / error |
| `IS_SANDBOX` (cc 透传) | _(uid==0 时自动加)_ | server 跑在 root 时给 cc 子进程加这个 env，绕开 cc 的 root-bypass-permissions 检查（v1 已踩坑实测） |

### A.3 自动找 cc binary

PoC step10 实测过——服务端启动时调 `resolveClaudePath()`：

```
1. TETHER_CC_PATH env override（绝对路径）
2. exec.LookPath("claude")
3. fallback: ~/.local/bin/claude, ~/.claude/local/bin/claude,
            ~/.npm-global/bin/claude, /usr/local/bin/claude,
            /opt/homebrew/bin/claude
4. 默认 "claude"（PATH lookup，让 exec 在执行时报原始错误）
```

启动日志第二行打印 `claude binary: <resolved path>`，避免"找不到 cc"这种最常见 issue 摸黑诊断。

### A.4 启动顺序

```
1. parse flags / env
2. resolveClaudePath()   → 失败也不退，记到 server state
3. ensure ~/.tether/ + ~/.tether/skills/, ~/.tether/users/ 存在
4. 加载或生成 TLS cert（§10.H）
5. 加载或生成 JWT secret（首次写 ~/.tether/jwt-secret）
6. start static HTTP listener（embed.FS）  → :PORT TCP
7. start HTTP/3 + WebTransport listener   → :PORT UDP
8. start session synchronizer goroutine   → ~/.claude/projects watcher
   ⚠ 角色限定（per D-05a §1）：仅 catch-up + cross-session sync 副路径，**不**作实时事件主源；
     可 lazy-start（首次 client attach 才起，省冷启动开销）
9. log "✓ tether server up on https://<host>:<port>/"
10. block on signal (SIGINT/SIGTERM) → graceful shutdown
```

**graceful shutdown 顺序**：
1. 拒新 WT 连接
2. 给所有活 cc subprocess 发 SIGTERM
3. 5 秒后还活的 cc 子进程发 SIGKILL
4. close TCP / UDP listener
5. 等所有 goroutine 汇合（带 ctx + 5s 超时）

## §10.A.5 `tether doctor` 子命令详细 spec [P0]

诊断子命令，给用户一行命令快速发现"为啥 tether 不工作"。

### A.5.1 行为

```bash
tether doctor [--verbose] [--json]
```

**Exit codes**:
- `0` — all green
- `1` — warnings only（譬如某个非关键检查失败，但 tether 仍可启动）
- `2` — error（至少一个检查失败到 tether 起不来）

**输出格式**（默认人类可读）：

```
[OK]   cc binary           /Users/wxk/.local/bin/claude  (v2.1.138)
[OK]   cert state          self-signed ECDSA P-256, valid 13d (auto-rotate enabled)
[OK]   port :8898          TCP + UDP bindable on 0.0.0.0
[OK]   data dir            ~/.tether/  (writable, schema v1)
[WARN] cc settings hooks   PreToolUse already has 2 entries (rtk-rewrite, custom);
                            tether will append `_tether_managed` (no conflict expected)
[OK]   ✓ ready to start
```

`--json` 模式输出 jsonl，每行一个 check：

```json
{"check":"cc-binary","status":"ok","detail":"/Users/wxk/.local/bin/claude","metadata":{"version":"2.1.138"}}
```

### A.5.2 必查的 5 项

| 检查 | 通过条件 | 失败影响 |
|---|---|---|
| **cc-binary** | `resolveClaudePath()` 返回的路径存在且 `claude --version` 成功 | error — tether 没法 spawn cc，启动后所有 chat 失败 |
| **cert-state** | TLS cert 加载成功且过期时间 > 24 小时 | error if 加载失败；warn if 24h-7d；ok if > 7d |
| **port-bindable** | 配置的 TCP + UDP 都能短期 listen + close（不冲突） | error — 启动时会 bind 失败 |
| **data-dir** | `~/.tether/` 可创建可写，schema version match | error if 不可写；warn if schema 落后（升级提示） |
| **cc-settings-hooks** | `~/.config/claude/settings.json` 存在且解析成功；统计 PreToolUse 条目数 | warn if 用户已有 hook（说明 tether 注入会跟它共存——non-blocking） |

### A.5.3 进阶检查（`--verbose` 才跑）

| 检查 | 通过条件 |
|---|---|
| **cc-version-compat** | cc 版本在 tether 测过的范围（譬如 ≥ 2.1.0） |
| **stream-json-supported** | `claude --output-format stream-json --print` 不报 unknown flag |
| **hook-binary** | `~/.tether/bin/tether-permission-hook` 存在且可执行 |
| **wt-availability** | spawn 一个 ephemeral HTTP/3 server 5s，验证 quic-go 这个机器 work |
| **vpn-detected** | （warn-only）检测是否在 VPN（路由 / DNS 异常），提示 WT 可能不工作 |

### A.5.4 实施位置

`internal/cmd/doctor/doctor.go` (~150 LOC)。

### A.5.5 acceptance

- [ ] `tether doctor` 退出码 0/1/2 跟 §A.5.1 表对齐
- [ ] 默认输出含 `[OK]/[WARN]/[ERR]` 前缀
- [ ] `--json` 输出 jsonl 一行一个 check
- [ ] 5 项基础检查都触达
- [ ] 缺 cc binary 时显示恢复提示（"install: curl ... | bash"）

## §10.B HTTP/3 server 设计 [P0]

### B.1 单端口 双协议

> **PoC 状态**：step10 验证的是 TCP :8082 + UDP :4433 **双端口**布局；step13 验证的是 TCP+UDP **同端口** 单 mux 布局（即下面的代码骨架）。Phase 1 实施直接走单端口路线。

```go
//go:embed all:web/dist/*
var webFS embed.FS

func main() {
    cert := loadOrGenCert()
    mux := http.NewServeMux()
    mux.Handle("/", http.FileServer(http.FS(webFSStripped)))
    mux.HandleFunc("/api/v1/...", apiRouter)
    mux.HandleFunc("/wt/chat", wtUpgrade(handleChat))
    mux.HandleFunc("/wt/shell", wtUpgrade(handleShell))
    mux.HandleFunc("/wt/events", wtUpgrade(handleEvents))   // broadcast (D-08)

    // --- TCP :port — HTTP/2 over TLS（SPA 首次加载 + Alt-Svc 升级提示） ---
    tcpServer := &http.Server{
        Addr:    ":" + port,
        Handler: altSvcMiddleware(mux),               // 注入 Alt-Svc: h3=":<port>"; ma=86400
        TLSConfig: &tls.Config{
            Certificates: []tls.Certificate{cert},
            NextProtos:   []string{"h2", "http/1.1"},
        },
    }
    go tcpServer.ListenAndServeTLS("", "")

    // --- UDP :port — HTTP/3 + WebTransport ---
    h3 := &http3.Server{
        Addr:            ":" + port,
        TLSConfig:       &tls.Config{Certificates: []tls.Certificate{cert}, NextProtos: []string{"h3"}},
        Handler:         mux,
        EnableDatagrams: true,
        QUICConfig:      &quic.Config{EnableDatagrams: true, EnableStreamResetPartialDelivery: true},
    }
    webtransport.ConfigureHTTP3Server(h3)  // ← step10 实测过这行漏了浏览器 abort 握手

    wt := &webtransport.Server{H3: h3, CheckOrigin: allowAll}
    wt.ListenAndServe()
}

func altSvcMiddleware(h http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Alt-Svc", `h3=":`+port+`"; ma=86400`)
        h.ServeHTTP(w, r)
    })
}
```

### B.2 关键约束（PoC 已暴露）

1. **`webtransport.ConfigureHTTP3Server(h3)` 必须调用** — 设置 H3 SETTINGS 帧的 `SETTINGS_ENABLE_WEBTRANSPORT=1`。step10 第一次跑漏了这行，浏览器握手 abort。
2. **TCP listener 跟 UDP listener 各起一遍** — Go `http3.Server.ListenAndServe()` **只**绑 UDP；想要同端口的 HTTP/2 over TCP fallback 必须**额外**起一个 `http.Server{TLSConfig: ...}` 跑 `ListenAndServeTLS`。两个 socket 不冲突（不同协议族），共享同一张 cert。step13 PoC 实测通过（见 §10.J）。
3. **TCP 响应必须打 `Alt-Svc: h3=":<port>"; ma=86400` 头** — 浏览器初次走 TCP HTTP/2 拿到 SPA + Alt-Svc 后，**后续**请求才会尝试 HTTP/3。step13 中间件 `altSvcMiddleware` 已示范。
4. **EnableDatagrams 双重设置** — `http3.Server.EnableDatagrams=true` AND `quic.Config.EnableDatagrams=true`，少一个 datagram 通道不工作。
5. **Self-signed cert 必须 ECDSA P-256 + ≤14 天** — 浏览器 `serverCertificateHashes` 强制规则。
6. **WT 始终走 UDP HTTP/3** — `serverCertificateHashes` API 仅对 WT 生效；HTML 页面初次加载（TCP HTTPS）需要走正常 CA 信任链或 dev flag。dev / 自签场景下：用户首跑加 Chrome `--ignore-certificate-errors-spki-list=<spki>` 或 `chrome://flags/#allow-insecure-localhost`（仅 loopback）；prod 用 Let's Encrypt（见 §10.H）。

### B.3 静态 SPA 托管（embed.FS）

```go
//go:embed all:web/dist/*
var webFS embed.FS

// strip the "web/dist" prefix so /xterm.js maps to web/dist/xterm.js
sub, _ := fs.Sub(webFS, "web/dist")
mux.Handle("/", http.FileServer(http.FS(sub)))
```

dev 模式可走文件系统读盘（参见 §10.C dev 工作流）：

```go
if devMode {
    mux.Handle("/", http.FileServer(http.Dir("web/dist")))
}
```

### B.4 路由表

| Path | Method | 处理 | 说明 |
|---|---|---|---|
| `/` | GET | embed.FS 静态 | SPA 入口 |
| `/assets/*` | GET | embed.FS 静态 | Vite hash-named chunks |
| `/sw.js` `/manifest.json` `/icons/*` | GET | embed.FS 静态 | PWA |
| `/api/v1/auth/*` | POST | REST | login / refresh / logout（D-16 J2 简化版） |
| `/api/v1/sessions/*` | GET / POST / DELETE | REST | session 列表 / 元数据修改 |
| `/api/v1/skills/*` | GET / POST | REST | skill 安装 / 配置 |
| `/api/v1/workspaces/*` | GET / POST | REST | workspace 状态 |
| `/wt/chat` | WT bidi | stream-json claude（per-session） | D-05a 主通道 |
| `/wt/shell` | WT bidi | PTY claude --resume | D-05a 副通道 |
| `/wt/events` | WT broadcast | server → 多 client 广播状态变更 | D-08 multi-attach |
| `/wt/blob/*` | WT | blob 上传/下载 | D-18 #3-#4 forward-compat |

## §10.C 浏览器 SPA 项目结构 [P0]

### C.1 顶层布局

```
tether/
├── cmd/
│   └── tether/main.go         # 入口
├── internal/
│   ├── server/                # HTTP/3 + WT (§10.B)
│   ├── workspace/             # D-19 状态层
│   ├── skill/                 # D-20 plugin overlay
│   ├── session/               # D-08/D-15 lock + multi-attach
│   ├── agent/                 # D-17a / §10.E
│   │   ├── provider.go        # AgentProvider interface
│   │   ├── claude_provider.go # v0.1 唯一实现 (D-05a)
│   │   ├── opencode_provider.go.stub
│   │   ├── cursor_provider.go.stub
│   │   ├── gemini_provider.go.stub
│   │   └── codex_provider.go.stub
│   ├── wire/                  # D-22 Go schema source
│   │   ├── doc.go
│   │   ├── types.go           # Envelope / Event / FencedBlock 等
│   │   └── format.go          # type alias for format conventions (HashHex64, ...)
│   ├── crypto/                # v1.0+ §10.I（v0.1 stub）
│   └── pair/                  # v1.0+ §10.I（v0.1 stub）
├── web/
│   ├── src/
│   │   ├── main.tsx
│   │   ├── App.tsx            # SPA root + 三栏 / 单栏 layout 切换 (D-19)
│   │   ├── lib/
│   │   │   ├── wire.gen.ts    # ⚠ 自动生成 (tygo) — 不手改
│   │   │   ├── wt.ts          # WebTransport 抽象
│   │   │   ├── auth.ts        # JWT + refresh
│   │   │   └── store.ts       # zustand state
│   │   ├── panes/
│   │   │   ├── workspace/     # 左栏：workspace tree (D-19)
│   │   │   ├── skill/         # 中栏：skill 产物 + actions
│   │   │   ├── chat/          # 右栏：chat input + fenced-block renderer
│   │   │   └── shell/         # shell tab：xterm.js
│   │   ├── fenced-blocks/
│   │   │   ├── DagBlock.tsx
│   │   │   ├── FormBlock.tsx
│   │   │   ├── CandidatesBlock.tsx
│   │   │   ├── MediaBlock.tsx
│   │   │   └── PermissionBlock.tsx  # cloudcli's `canUseTool` 模式 (D-05a)
│   │   └── pwa/
│   │       └── manifest.json
│   ├── public/
│   │   └── icons/             # PWA icon set
│   ├── dist/                  # vite build 输出（被 Go embed.FS 吃）
│   ├── index.html
│   ├── package.json           # react 19 + vite 7 + tailwind + xterm + ...
│   └── vite.config.ts
├── scripts/
│   ├── codegen.sh             # tygo generate
│   └── build.sh               # web build → go build
├── tygo.yaml                  # D-22 配置
├── Makefile
├── go.mod
└── README.md
```

### C.2 dev / prod 工作流

**Dev mode**（hot reload）：

```bash
# Terminal 1 — vite dev server (HMR)
cd web && pnpm install && pnpm dev   # → http://localhost:1420

# Terminal 2 — Go server in dev mode
TETHER_DEV_MODE=1 \
TETHER_DEV_FRONTEND_URL=http://localhost:1420 \
go run ./cmd/tether server
```

dev mode 时 Go server `/` handler 反向代理到 vite dev server（让浏览器加载 vite-served HMR 资源），但 `/wt/*` 跟 `/api/*` 仍由 Go 处理。

**Prod build**：

```bash
make codegen    # tygo 生成 wire.gen.ts
make web-build  # vite build → web/dist/
make go-build   # go build with embed.FS → bin/tether
```

或合并：

```bash
make build  # 三步走完
```

### C.3 frontend 关键依赖（package.json 草案）

```json
{
  "dependencies": {
    "react": "^19.0.0",
    "react-dom": "^19.0.0",
    "zustand": "^5.0.0",
    "@xterm/xterm": "^5.5.0",
    "@xterm/addon-fit": "^0.10.0",
    "@xterm/addon-web-links": "^0.11.0",
    "@uiw/react-codemirror": "^4.x",
    "react-markdown": "^9.x",
    "remark-gfm": "^4.x"
  },
  "devDependencies": {
    "vite": "^7.0.0",
    "@vitejs/plugin-react": "^4.x",
    "typescript": "^5.9.0",
    "tailwindcss": "^3.x"
  }
}
```

xterm.js 用 `@xterm/*` 命名空间（5.5+ 改了 npm scope；step10 实测）。

## §10.D cc agent 双通道实施细节 [P0]

参考 **D-05a addendum**（`2026-05-09-d05a-stream-json-dual-channel.md`），完整覆盖：

- Chat 主通道：long-running stream-json claude subprocess
- Shell 副通道：PTY claude --resume 同 sessionId
- IS_SANDBOX=1 root-only 注入（避开 cc 的 dangerous-skip-permissions 拒绝）
- system/init 每 turn 触发但 sessionId 跨 turn 稳定（PoC step8 事实）
- 两通道 jsonl 协调由 cc 内部处理

step10 已端到端验证浏览器 → 双 WT 通道 → cc subprocess pipeline。

## §10.E AgentProvider Go interface [P0]

参考 **D-17a addendum**（`2026-05-09-d17a-agent-provider-multi-provider.md`）。

v0.1 仅 ClaudeCodeProvider 一个实现。其他 provider stub 文件保留命名 + godoc 指向 §5.X 各自踩坑表，等 v0.2 / v0.5 / v1.0+ 实施。

interface 定义在 `internal/agent/provider.go`（D-17a §3 草案）。

## §10.F tygo schema 工作流 [P0]

参考 **D-22 addendum**（`2026-05-09-d22-tygo-schema-sync.md`）。

CI 强制：

```yaml
# .github/workflows/ci.yml （草案）
- name: codegen
  run: make codegen
- name: ensure no schema drift
  run: git diff --exit-code   # 如果 tygo 输出跟 commit 的 wire.gen.ts 不同则失败
```

PoC `poc/go-quic-wt/wire/types.go` + `tygo.yaml` 已 dogfood 验证（D-22 §2.7）。

## §10.G PWA + 浏览器分发 [P1，留位]

待 v0.1 主路径稳定后细化。需覆盖：

- `manifest.json` schema（name / icons / display=standalone / start_url / theme_color）
- service worker 策略（v0.1 不做离线，仅做 SPA shell 缓存）
- "添加到主屏幕"UX（Chrome / Edge / Safari 26.4+）
- HTTP 公网 IP 用户引导：Chrome flag `unsafely-treat-insecure-origin-as-secure` 或 cloudflared tunnel
- iOS 限制：Safari 26.4+ 必需

## §10.H TLS / 证书策略 [P1，留位]

待细化。已知约束：

- **Dev**：自签名 ECDSA P-256，14 天有效期，浏览器 `serverCertificateHashes` 钉哈希（PoC step3-step10 已用）
- **Prod**：Let's Encrypt（需要域名指向）或 Cloudflare Tunnel 分发 HTTPS
- **Public IP, no domain**：受限——浏览器 WT 必须 secure context（HTTPS 或 localhost）。不走 cloudflared 就要 Chrome flag

**已知客户端环境陷阱**：

- **VPN / 代理软件破坏 HTTP/3**：很多 VPN 只代理 TCP，丢弃或 rewrite UDP 流量，QUIC 整个不工作。step10 实测过：用户开 VPN 时 WT 握手失败（"Opening handshake failed"），关 VPN 立刻通。**README 必须有这条 troubleshooting**。
- **企业 firewall 拦 UDP**：常见 IT 策略只放 TCP。tether 在企业环境需要 fallback 文档（HTTP/2 + WS 退化模式 v1.0+ 评估）。
- **Cloudflare WARP / 双栈 IPv4/IPv6**：可能改路由，QUIC 兼容性参差。

## §10.I WebCrypto E2E (v1.0+) [P2，留位]

D-12 v1.0+ 重新设计。简化路线：

- 浏览器 SubtleCrypto X25519 keypair，私钥存 IndexedDB
- pair 协议走带外（QR 码 / pair token）
- envelope payload 用 age-style ECDH + ChaCha20-Poly1305 框（保留 D-12 cipher choice）
- server 看到的 envelope payload 永远是 ciphertext

工程量评估：~3000 LOC（前端 crypto + key 管理 + UI；后端 envelope 框架）。**v0.1 不做。**

## §10.J PoC 路线（重写 v1 §10）[P0]

v1 §10 列了 PoC 1-6。v2 已经做完同等验证：

| v2 step | 验证 | 状态 |
|---|---|---|
| step7 | 单 binary embed.FS + WebTransport | ✅ 5/5 通过 |
| step8 | chat (stream-json) + shell (PTY) 双通道 | ✅ 3/3 通过 |
| step9 | step8 模式跨 provider（opencode 验证） | ✅ 3/3 通过 |
| step10 | 浏览器 + 双 WT 通道端到端（TCP :8082 + UDP :4433 双端口布局，公网 IP） | ✅ visual ✓，含 cc 真 TUI 渲染 |
| step11 | cc stream-json **没有** permission_required 事件（颠覆 D-05a §3 假设） | ✅ critical 验证 |
| step12 | cc PreToolUse hook → daemon HTTP 回调（D-05b 协议） | ✅ DENY+ALLOW 全跑 |
| step13 | TCP :4433 + UDP :4433 **同端口**单 mux + Alt-Svc + cert 共享 + WT 握手 | ✅ 3/3 通过（TCP /cert-hash + Alt-Svc 头 + UDP WT echo） |

**path 全亮**，**含 single-port mux 已验证**。Phase 1 实施期不再需要 PoC，直接进 production code。

剩下的真正 PoC 需求只在某些 v1.0+ 边角：

- **PoC-WC1**：WebCrypto X25519 浏览器端 keypair 生成 + pair 协议（v1.0+）
- **PoC-IOS**：iOS Safari 26.4+ 实机 WT 行为（v0.1.x，等用户有 iPhone 时跑）
- **PoC-PROD**：Let's Encrypt + Caddy 反代 / 直接 Go HTTP/3（v0.1 ship 前确认 prod cert 工作）

## §10.K v0.1 acceptance criteria（替代 §7 draft）[P0]

### K.1 安装与启动

- [ ] `go install github.com/piaobeizu/tether/cmd/tether@latest` 装上单 binary
- [ ] `tether version` 输出 v0.1 版本号
- [ ] `tether doctor` 检查依赖：cc binary 找到、cert 状态、port 可绑、~/.tether/ 已建
- [ ] `tether server` 在 `0.0.0.0:8898` 起 HTTP/3 + WT 服务，stdout 第二行打印 `claude binary: <resolved path>`
- [ ] graceful shutdown：SIGINT 在 5 秒内退出，cc subprocess 全部干净结束

### K.2 Browser 端基本功能

- [ ] Chrome / Edge / Firefox / Safari 26.4+ 浏览器开 `https://<host>:<port>/` → React SPA 加载
- [ ] D-19 桌面三栏 layout（左 workspace / 中 skill / 右 chat）正确渲染（视口 ≥ 768px）
- [ ] D-19 移动单栏 layout 正确渲染（视口 ≤ 600px）
- [ ] PWA：Chrome "添加到主屏幕" 装上后图标可启动，全屏体验

### K.3 Chat 通道

- [ ] 输入 prompt → 回车 → 通过 `/wt/chat` 发送
- [ ] 实时收 stream-json 事件 → fenced-block renderer 正确显示 text / tool_use / result
- [ ] 单 cc 进程跨多 prompt 长跑（PoC step8 事实 → production）
- [ ] tool 权限请求显示 PermissionBlock，用户决策回写 stdin
- [ ] session_id 在 chat panel 顶部展示

### K.4 Shell 通道

- [ ] `/wt/shell` 启动 PTY claude --resume <chat_sid>
- [ ] xterm.js 渲染 cc 完整 TUI（Welcome banner、box-drawing、prompt redraw）
- [ ] slash command（至少 `/help`、`/plugin update`、`/reload`、`/clear`）工作
- [ ] shell 标签页跑 `/plugin update` 后，chat 通道下条 prompt 用更新后的 plugin

### K.5 Multi-attach（D-08）

- [ ] 两个浏览器 tab 登同一 user → 同 session 实时事件 broadcast
- [ ] D-15 lock：第二个 tab 想发 prompt 看到 "lock held by another client"，可 force takeover

### K.6 Skill / Workspace（D-19/D-20）

- [ ] `~/.tether/workspaces.toml` 列出 workspace；UI 左栏显示
- [ ] Skill 在 `~/.tether/skills/<id>/` 状态写入；workspace 启用后 `<workspace>/.claude/plugins/<id>` symlink farm 正确
- [ ] cc plugin overlay 工作：cc 看到 polyforge 16 个 skill 一行不改

### K.7 Schema / Wire 健康度（D-22）

- [ ] `make codegen` 跑后 `git diff --exit-code` clean
- [ ] `web/test/wire-contract.spec.ts` 覆盖 **D-22 §6 列出的 6 条权威 wire 约定**（cert-hash format、WT bidi pure echo、Envelope shape、SessionID format、FencedBlockKind 枚举、tool_use input shape）。本节不重复列；以 D-22 §6 为唯一来源

### K.8 已知客户端 friction 文档

- [ ] README 含 troubleshooting："WT 握手失败？关 VPN / 代理 / 防火墙白名单 UDP <port>"
- [ ] README 含浏览器要求清单：Chrome 96+ / Edge 96+ / Firefox 114+ / Safari 26.4+
- [ ] README 含 dev 自签名 cert 用法（Chrome `unsafely-treat-insecure-origin-as-secure` flag）

### K.9 性能（最低基线，不严格）

- [ ] cc subprocess 启动到首次 `system/init` 事件 ≤ 5 秒（cold）
- [ ] 用户输入 → server → cc → 第一个 assistant text 事件 ≤ 1 秒（warm，本机）
- [ ] HTTP/3 / WT 多 stream 不阻塞（chat 流式输出时同时跑 file 上传不卡）

### K.10 Dogfood

- [ ] 用户（wxk）自己用 tether v0.1 跑 polyforge 项目工作 ≥ 1 周，能完成日常 task
- [ ] 至少 3 个 polyforge skill（`brainstorming` / `writing-plans` / `executing-plans`）通过 fenced-block renderer 正确展示
- [ ] 1 次跨设备 multi-attach（笔记本 + 平板同时挂同 session）实测 ≥ 30 分钟无 corruption

---

## 附录 A：addenda 索引

- D-22 `2026-05-09-d22-tygo-schema-sync.md`
- D-17a `2026-05-09-d17a-agent-provider-multi-provider.md`
- D-05a `2026-05-09-d05a-stream-json-dual-channel.md`
- 项目记忆 `project_tether_simplification_consideration.md`

## 附录 B：v1 → v2 替代关系

| v1 ID | v2 替代 |
|---|---|
| D-05 | D-05a |
| D-13 | 整段砍（无替代） |
| D-12 | v0.1 降级 / v1.0+ §[I]TBD |
| D-02 | §1.2（保留 WT 部分，砍 Tauri shim 部分） |
| D-01 / D-03 / D-11 | §1.1（合并简化） |
| 其余 | 不变 |

## 附录 C：变更引用

PoC 实证（路径 `poc/go-quic-wt/`）：
- step7 单 binary + WebTransport (5/5 ✓)
- step8 chat (stream-json) + shell (PTY) 双通道 (3/3 ✓)
- step9 opencode 集成验证 (3/3 ✓)

cloudcli 来源参考：
- npm 包 `@cloudcli-ai/cloudcli`（见 `/usr/lib/node_modules/@cloudcli-ai/cloudcli/`）
- 4 provider 集成模式参考 D-17a §1
