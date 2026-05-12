# tether — upgrade path

> Authoritative source for "what does each version contain and why."
> Updated 2026-05-12. Spec reference: `wiki/specs/2026-05-09-tether-simplified-design.md`.

---

## 最终产物是什么

**tether** 是一个 AI workspace 平台（D-19），运行形态是单 Go binary：

```
单 Go binary (tether server)
  │
  ├─ HTTP/3 + WebTransport (单端口)
  │     ├─ React SPA (浏览器，无 native 客户端)
  │     │     ├─ 桌面三栏 layout (workspace | chat | skill/shell)
  │     │     ├─ 移动单栏 chat-first layout
  │     │     └─ fenced-block renderer (dag / form / candidates / media / permission)
  │     │
  │     ├─ /wt/chat  → stream-json 双向通道 (Claude Code session)
  │     ├─ /wt/shell → PTY shell (xterm.js)
  │     └─ /wt/events → broadcast (multi-attach)
  │
  ├─ MCP Host (外部工具 via MCP 协议)
  │     ├─ /mcp loopback (127.0.0.1:8899, for CC subprocess)
  │     ├─ /mcp HTTPS  (外部客户端: Cursor / Goose)
  │     └─ OAuth 2.1 PKCE (自动鉴权，无需手动 token)
  │
  ├─ Skill = cc plugin overlay (D-20)
  │     └─ ~/.tether/skills/<id>/ → <workspace>/.claude/plugins/<id>/ symlink farm
  │
  └─ Workspace management (D-19 state layer)
        └─ ~/.tether/workspaces.toml + REST API
```

**差异化**：WebTransport 实时双向通道（cloudcli 没有） + 本地运行（数据不上云）+ MCP 生态接入。

**v1.0 追加**（spec §10.I P2）：WebCrypto E2E（浏览器端 SubtleCrypto，私钥 IndexedDB），多用户 session 隔离，`tether pair` QR 配对，Gemini / Cursor providers。

---

## 版本体系规则

- **版本号** = 功能边界的发布快照，不等于 spec 阶段完成标志
- **spec 阶段** (s1–s9) = 原始实施路线图，用于追踪 §10.K acceptance criteria 的完成度
- **每个版本的 scope** 在本文件里定义，不临时拍板

---

## 已发布版本

| 版本 | Spec 阶段 | 核心内容 | 状态 |
|---|---|---|---|
| **v0.1.0** | s1–s2 | HTTP/3 + TCP 单端口 + embed SPA + WT smoke | ✅ |
| **v0.2.0** | s3–s6 | D-19 SPA + fenced-block renderer + stream-json chat + PTY shell + auth + multi-attach | ✅ |
| **v0.3.0** | s5 (permission 迁移) | MCP Host Core + permission API 重构 | ✅ |
| **v0.3.1** | — | /mcp loopback + builtin workspace tools + CC settings inject | ✅ |
| **v0.3.2** | — | HTTPS /mcp + 手动 API token + 动态工具列表 | ✅ |
| **v0.3.3** | — | OAuth 2.1 PKCE（Cursor/Goose 自动鉴权）| ✅ |

> v0.3.x 整条线是 MCP 集成轨道，原始 spec s1–s9 计划之外追加。

---

## K.1–K.10 当前完成度（v0.3.3 之后）

| Criterion | 内容 | 状态 |
|---|---|---|
| K.1 | 安装 / `tether doctor` / `tether server` / graceful shutdown | ✅ |
| K.2 | Browser SPA + D-19 三栏 + 移动 layout + PWA manifest | ✅ |
| K.3 | Chat channel: stream-json + fenced-block renderer + permission block | ✅ |
| K.4 | Shell channel: PTY + xterm.js + slash commands | ✅ |
| K.5 | Multi-attach: broadcast + D-15 lock | ✅ |
| K.6 | Workspace + skill overlay REST API + SPA 左栏展示 | ✅ |
| K.7 | `make codegen` drift gate + wire-contract tests (D-22 §6 6条) + CI | ✅ |
| K.8 | README troubleshooting (VPN / 浏览器矩阵 / Chrome dev flag) | ⚠️ 需确认 |
| K.9 | 性能基线 (cold start ≤5s, warm ≤1s, multi-stream 不阻塞) | ⚠️ 未正式 benchmark |
| K.10 | Dogfood ≥1 周 (polyforge 日常 + 跨设备 multi-attach) | ❌ 未做 |

---

## 下一步版本计划

### v0.3.4 — `tether doctor` MCP health reporting（小 patch）

**Scope：** `internal/doctor/doctor.go` 追加 MCP 相关检查项：
- MCP loopback 状态（127.0.0.1:8899 可连）
- `~/.tether/api-tokens.json` 是否存在、是否有 token
- CC settings injection 状态（tether 是否在 `~/.claude/settings.json` 的 mcpServers 里）

**K criteria touch：** K.1（`tether doctor` 完整度）

---

### v0.4 — Per-task MCP lifecycle

**Scope：** polyforge task start/pause 时 spawn/kill MCP servers：
- task start → spawn 配置的 MCP servers（supervisor per-task 实例）
- task pause/wrap → kill 对应 MCP servers
- `workspace_run_shell` — PTY wrapping，支持 shell 命令作为 MCP tool
- `*mcp.Server` 从 singleton 架构迁移到 per-task 实例

**端到端目标：** CC → tether → polyforge-coding-mcp 完整通路

**K criteria touch：** 不直接对应 K criteria（功能扩展）

---

### v0.5 — spec-complete + ship gate

**Scope：**
- K.8：README troubleshooting 补全（VPN 关闭 / 浏览器矩阵 / Chrome dev flag）
- K.9：性能基线正式 benchmark 并记录（达标即可，不是优化）
- K.10：dogfood ≥1 周（polyforge 日常工作 + 跨设备 multi-attach ≥30 min）
- 发布 `v0.5.0` tag，`CHANGELOG.md` v0.5 entry，`tether-doc` release notes

**这是 spec §10.K 意义上的 acceptance-complete 版本。**

---

### v1.0 — E2E + multi-user + providers（长期）

**Scope（spec §10.I P2 + D-17a）：**
- WebCrypto E2E（浏览器端 SubtleCrypto X25519 + age-style 信封，私钥 IndexedDB）
- Multi-user session 隔离（user identity > 单用户 daemon 模式）
- `tether pair` QR 配对流程
- Gemini / Cursor providers（D-17a 追加路径）
- Mobile PWA offline（§10.G P1 补全）

**前置：** v0.5 dogfood 通过，无 ship-blocker

---

## 版本 vs spec 阶段对照

```
spec s1–s9 (原始 v0.1 计划)     实际版本轨道
─────────────────────────       ─────────────────────────
s1 (骨架+cert+tygo)      ─→    v0.1.0
s2 (HTTP/3+SPA mux)      ─→    v0.1.0
s3 (React SPA)           ─→    v0.2.0
s4 (CC stream-json)      ─→    v0.2.0
s5 (permission UI)       ─→    v0.2.0 / v0.3.0
s6 (PTY+lock+multi)      ─→    v0.2.0
s5.5 (doctor)            ─→    v0.3.4 (pending)
                                v0.3.x = MCP 轨道 (spec外)
                                v0.4   = per-task MCP (spec外)
s7 (workspace+skill)     ─→    ✅ (已完成，未单独版本)
s8 (CI+contract tests)   ─→    ✅ (已完成，未单独版本)
s9 (dogfood+ship gate)   ─→    v0.5 (K.8/K.9/K.10)
                                v1.0 = E2E+multi-user
```

---

## 关于 v0.3.x 系列的位置

v0.3.x 是 MCP 集成轨道，是 spec 原计划外追加的功能轨道，理由是：

1. 外部 MCP 客户端（Cursor / Goose）支持是产品差异化的核心
2. tether 的定位从"CC wrapper"升级为"AI workspace MCP hub"
3. 这部分功能让 K.10 dogfood 更有价值（可以在 tether 上跑更复杂的工作流）

这不是技术债，是产品方向演进。v1.0 的 E2E 等 spec 原计划功能仍在路线图上。
