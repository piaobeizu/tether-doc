# Spec Patch Checklist — light chair review (2026-05-09)

**Source review**: `polyforge-v2:methodology-review depth=light`，14 项 findings。
**PoC follow-up**: step11 PoC-PERM 验证 critical (C1)，**改变了 fix path**。

## 状态图例

| 标 | 含义 |
|---|---|
| ✅ | 已修 |
| ⏳ | 待 Phase 1 启动前修 |
| 📋 | 待 ship 前修 |
| 🔮 | v0.5+ / v1.0+ 再说 |

---

## Critical (1 项)

### C1. cc stream-json 没有原生 permission 协议 — PoC 结果颠覆

**原 finding**: D-05a §3 标 `permission_required` "待 PoC 验证"，K.3 acceptance 要求 PermissionBlock 端到端工作。

**PoC step11 实测结果**:
- `--permission-mode default` 跟 `--permission-mode bypassPermissions` 输出**完全相同的 13 个事件**
- 无任何含 "perm" / "approval" / "request" 的事件类型
- cc 在 `--print` 模式下**自动执行所有 tool 调用**（`permission_denials: []`）
- cc 的 permission UI 流程是 **SDK 专属**（cloudcli 用 `canUseTool` callback，不是 stream-json 事件）

**结论**：D-05a §3 写错了。stream-json 模式根本**没** permission_required 事件可订阅。

**Fix path**（颠覆原 D-05a 假设）:

走 **cc PreToolUse hook callback**，回到 v1 D-05 当年的设计（hook callback HTTP endpoint），但**只用 hook 这一层**，不要 v1 D-05 的 4 路（PTY + Hooks + JSONL）大全套：

```
[user 操作触发 tool 调用]
  ↓
cc 跑 PreToolUse hook script
  ↓ (hook script POST 到 daemon /api/v1/agent/permission/<request_id>)
daemon 阻塞 hook script，broadcast permission_request envelope 给 /wt/events
  ↓
UI 渲染 PermissionBlock，用户点 Allow / Deny
  ↓
daemon 收 permission_decision envelope，let hook script 返回 0 (allow) 或 1 (deny)
  ↓
cc 看 hook 退出码决定继续还是 abort
```

**操作清单**:

- [ ] ⏳ **重写 D-05a §3 整段**: 移除"待 PoC 验证"措辞，写"cc stream-json 模式无 permission_required 事件；permission UI 流程必须走 PreToolUse hook callback"
- [ ] ⏳ **写新的 D-05b addendum**: hook callback 协议详细 spec
  - daemon 在 settings.json 自动注入 tether-permission PreToolUse hook（matcher: `*`）
  - hook script: 读 stdin tool 调用 JSON，POST 给 daemon 阻塞，等响应，按响应退出
  - daemon endpoint: POST /api/v1/agent/permission/request（hook → daemon），POST /api/v1/agent/permission/<id>/decide（UI → daemon）
  - 超时策略：默认 60s 没决定 → deny
- [ ] ⏳ **更新 v2 §10.K.3 acceptance**: 把"PermissionBlock 端到端"改成"PreToolUse hook 触发 → /wt/events 推送 → UI 渲染 → 决策回写 hook 退出码"
- [ ] ⏳ **PoC step12**: spike hook callback 流程，~半天工作

---

## High (4 项)

### H1. ✅ env var `CC_PATH` → `TETHER_CC_PATH`

**Fix**: ✅ done. step10_browser_dual.go 已改为 TETHER_CC_PATH。规范统一到 TETHER_* 前缀。Build 通过。

### H2. ✅ IS_SANDBOX 注入策略统一为 root-only

**Fix**: ✅ done. D-05a §2 mitigation 段、§4 Spawn 代码示例、§8 acceptance criteria 三处统一改为 `os.Geteuid() == 0` 条件注入。

### H3. ✅ step10 双端口 vs spec 单端口设计 — step13 PoC 验证通过

step10 跑的是 TCP :8082 + UDP :4433 双端口。v2 §10.B 描述的是单 HTTP/3 listener。两套不一样。

**Fix**: ✅ done. PoC step13 (`step13_singleport.go`) 验证 TCP :4433 + UDP :4433 同端口共存：

- ✓ TCP `http.Server{TLSConfig}` 跟 UDP `http3.Server` 各起一遍，同一张 ECDSA cert，无 socket 冲突
- ✓ TCP HTTP/2 响应头 `Alt-Svc: h3=":4433"; ma=86400` 通过中间件注入
- ✓ TCP /cert-hash 跟 UDP /cert-hash 返回同一 64-char hex
- ✓ WT 握手走 UDP HTTP/3 + serverCertificateHashes pinned verifier
- ✓ 自动测试 3/3 PASS（TCP /cert-hash + Alt-Svc 头 + UDP WT echo）

**Spec 落地**:
- [x] ✅ **PoC step13** 跑过 3/3
- [x] ✅ **§10.J PoC 路线表加 step13 一行**
- [x] ✅ **§10.B.1 代码骨架改写**：双 listener pattern（`http.Server` TCP + `http3.Server` UDP + `altSvcMiddleware`）
- [x] ✅ **§10.B.2 关键约束**：从 4 条扩到 6 条，新增 #2 (双 listener)、#3 (Alt-Svc 头)、#6 (WT 仅走 UDP)

### H4. Event/EventKind 类型放哪 — D-17a 跟 D-22 矛盾

`internal/wire/` 单一来源（共享）。Event/EventKind 是 daemon 内部翻译边界类型，不需要前端拿到。

- [ ] ⏳ **D-17a §3 调整**: AgentProvider Event 类型放 `internal/agent/event.go`（不进 wire/，不被 tygo 生成）
- [ ] ⏳ **D-17a §8 删去** `internal/agent/wire/` 提及
- [ ] ⏳ **D-22 §2.2 加注**: "wire/ 仅含跨 daemon ↔ browser 类型"

---

## Medium (5 项)

### M1. tether doctor 实施细节缺失

K.1 列了 4 项 doctor 检查，没 spec exit code / 输出格式 / 检查具体逻辑。

- [ ] ⏳ **新 §10.A.5**: doctor 完整 spec
  - exit codes: 0=all green, 1=warn, 2=error
  - 输出 line-prefix: `[OK] / [WARN] / [ERR] check_name: detail`
  - 5 项检查: cc binary / cert state / port bindable / ~/.tether/ 权限 / settings.json 兼容性

### M2. K.4 /plugin update → chat 通道协同协议未定

- [ ] 📋 **PoC step13**: chat process 长跑 → shell 里装 plugin → chat 下条 prompt 看 system/init 的 plugins 字段
- [ ] 📋 **D-05a §6 重写**: 实测后写明确协议
- [ ] 📋 **K.4 加 acceptance dependency**: 可能 ship 时降级（用户点 reload 按钮重启 chat process）

### M3. K.7 5 条 wire-contract test 跟 D-22 §6 不一致

- [ ] ⏳ **D-22 §6 重写**: 改成 6 条覆盖（合并两边）
- [ ] ⏳ **v2 §10.K.7 改成 reference** D-22 §6

### M4. D-08 multi-attach broadcast 未 PoC

- [ ] 📋 **PoC step14**: /wt/events endpoint + 两浏览器 tab + server 推送 → 两 tab 都收
- [ ] 📋 **§10.K.5 wait on PoC**

### M5. PoC-PROD (Let's Encrypt) 优先级未定

- [ ] 📋 **§10.J PoC-PROD 标 P0**: ship 前必须验证一条 prod cert path
- [ ] 📋 **§10.H §10.G 衔接**: Let's Encrypt + Cloudflare Tunnel fallback

---

## Low (4 项)

### L1. ✅ v2 spec Status 已更新

**Fix**: ✅ done.

### L2. D-22 §2.7 关于 step10 用 wire types 的描述不准

- [ ] 📋 **D-22 §2.7 改写**: 明确 step7 是初始 dogfood；step10 因 build-tag 隔离用 local helper

### L3. ✅ §10.A.4 step 8 watcher 角色已限定

**Fix**: ✅ done.

### L4. K.6 cc plugin overlay (D-20) 在 stream-json 下未 PoC

- [ ] 📋 **PoC step13 同时验**: 启动 cc workspace 含 plugin → system/init 的 plugins/tools 字段
- [ ] 📋 **D-20 补 dogfood 注释**

---

## 执行汇总

### 已完成 (✅) — 这一轮
- H1 env var rename
- H2 IS_SANDBOX 三处统一
- L1 v2 Status header
- L3 watcher 角色限定
- C1 D-05a §3 重写 + D-05b hook callback spec + step11/step12 PoC（推翻原假设 + 验证替代路线）
- H3 step13 single-port mux PoC（TCP+UDP 同端口，Alt-Svc，cert 共享，WT 握手）+ §10.B 双 listener 代码骨架 + §10.J 表更新
- H4 D-17a §3/§5/§8 + D-22 §2.2 — Event/EventKind 类型位置统一到 `internal/agent/event.go`，wire/ 仅含跨前后端类型
- M1 §10.A.5 — `tether doctor` 完整 spec（exit codes 0/1/2 + 5 项基础检查 + --verbose 5 项进阶检查 + 实施位置）
- M3 D-22 §6 — 6 条权威 wire-contract（cert-hash 格式、WT bidi pure echo、Envelope shape、SessionID format、FencedBlockKind 枚举、tool_use input shape）+ v2 §10.K.7 改成 reference 形式

### Phase 1 启动前必修 (⏳)
*（无剩余项 — 上述 5 条 ⏳ 全部清完）*

### Ship 前必修 (📋)
- M2 PoC step13 + D-05a §6 重写 plugin reload 协议
- M4 PoC step14 multi-attach broadcast
- M5 PoC-PROD 标 P0 + Let's Encrypt 路径
- L2 D-22 §2.7 准确描述 dogfood 范围
- L4 plugin overlay PoC 验证

### 未来 (🔮)
- 自部署 PKI / 内网 CA (v0.5+)

---

## 关键洞察 — C1 PoC-PERM 颠覆性

PoC step11 让我们**避免了一个 v0.1 ship 后才会暴露的灾难**：

> 假设 cc stream-json 模式有 permission_required 事件 → 实施 PermissionBlock UI → ship → 用户点 "Auto Mode = OFF" → **完全没用，cc 仍然自动执行所有工具**。

这种"ship 后发现 spec 跟 cc 实际行为不符"的 bug 修复成本极高。**Light chair review + PoC-PERM 把这个问题在 spec 阶段杀掉**，重新设计了 hook callback 路线。

这一条**单项就 justify 了 review 这次活动的全部成本**。
