# D-05b: Permission UI 通过 cc PreToolUse hook callback 实施

**Date:** 2026-05-09
**Author:** wxk + Claude
**Status:** Proposed — 解决 D-05a §3 的 critical gap（spec patch checklist C1）；与 D-05a 配套，等用户拍板新方向后正式纳入主 spec
**Companion docs:**
- D-05a `2026-05-09-d05a-stream-json-dual-channel.md` (§3 待据本文档重写)
- v2 主 spec §10.K.3 (acceptance criteria 待据本文档调整)
- spec-patch-checklist `2026-05-09-spec-patch-checklist.md` C1
- PoC step11 `poc/go-quic-wt/step11_permission.go` (验证 stream-json **无** permission 事件)
- PoC step12 `poc/go-quic-wt/step12_permission_hook.go` (验证 hook 回调全流程)

---

## 0. 这份文档解决什么

D-05a §3 原假设：cc stream-json 模式会 emit `permission_required` 事件，daemon 订阅之 → broadcast 给 UI → 用户决定 → daemon 写回 stdin。

**PoC step11 实测推翻该假设**：

- `--permission-mode default` 与 `--permission-mode bypassPermissions` 输出**完全相同**的 13 个事件
- 无任何含 "perm" / "approval" / "request" 的事件类型
- cc 在 `--print` 模式下**自动批准并执行所有 tool 调用**（result.permission_denials = []）
- cloudcli 的 permission UI 是 **SDK `canUseTool` callback 专属**，跟 stream-json 协议**无关**

stream-json 协议**没有** permission 钩子。tether 必须通过 **cc 内置 PreToolUse hook 机制** 自建一层。

D-05b 详细 spec 这层。**PoC step12 已端到端验证**（DENY + ALLOW 两 pass 都通）。

## 1. 决策

**v0.1 ClaudeCodeProvider 通过 cc PreToolUse hook 实施 permission UI**：

- daemon 启动时**生成** + **注入** 一个 PreToolUse hook 到 cc settings.json
- Hook 实体：tether 自带的 Go 编译 binary（跨平台），位置 `~/.tether/bin/tether-permission-hook`
- Hook 行为：从 cc 读 stdin tool 调用 JSON → POST 给 daemon HTTP endpoint 阻塞 → 收响应按 exit code 0(allow) / 2(deny) 返回 cc
- daemon HTTP endpoint：`POST /api/v1/agent/permission/request`（hook 调，daemon 阻塞返回）+ `POST /api/v1/agent/permission/<id>/decide`（UI 调，让前者解锁）

cc 内置规则（PoC 已验证）：

- Hook exit code 0 → allow，cc 继续执行 tool
- Hook exit code 非 0 → deny，cc 在 `result.permission_denials[]` 记录，UI 看到 "blocked by PreToolUse hook" 文字

## 2. 架构图

```
[user 在 UI 输入 prompt]
       ↓ /wt/chat (stream-json)
   tether daemon
       ↓ stdin JSON
   cc subprocess (--print --output-format stream-json)
       ↓ 决定调 Bash/Edit/Read/...
   cc 触发 PreToolUse hook (matcher: "*")
       ↓ exec tether-permission-hook 子进程
       ↓ tool 调用 JSON 走 stdin
   tether-permission-hook
       ↓ POST {tool_name, tool_input, request_id} → daemon HTTP
       ↓ 阻塞等响应（client.Timeout = 60s）
   daemon /api/v1/agent/permission/request handler
       ↓ 生成 request_id，存 sync.Map[id] = chan
       ↓ broadcast permission_request envelope → /wt/events
   browser UI
       ↓ 渲染 PermissionBlock {tool, input, allow/deny 按钮}
       ↓ 用户点 Allow / Deny
       ↓ POST /api/v1/agent/permission/<id>/decide {allow, message}
   daemon /api/v1/agent/permission/<id>/decide handler
       ↓ sync.Map[id] <- decision
   daemon /perm-request handler 解锁
       ↓ JSON response {allow, message} → hook
   tether-permission-hook
       ↓ exit 0 (allow) 或 exit 2 (deny)
   cc
       ↓ allow: 继续执行 tool；deny: 跳过，结果记入 permission_denials[]
       ↓ 继续 stream-json 输出
[结果回流到 UI]
```

## 3. 协议详细

### 3.1 Hook → daemon 请求

`POST http://<daemon_host>:<daemon_port>/api/v1/agent/permission/request`

**Body**（cc 提供给 hook 的 stdin JSON 直接 forward）：

```json
{
  "session_id": "e2970d8f-...",
  "transcript_path": "/path/to/...jsonl",
  "tool_name": "Bash",
  "tool_input": {"command": "echo hello", "description": "..."},
  "hook_event_name": "PreToolUse"
}
```

cc 实际 stdin 字段会更多（hook_event / event_id / signature 等），daemon 全 forward 给 UI。

**Response**：

```json
{
  "allow": false,
  "message": "denied by user — tool 'Bash' with command 'echo hello'"
}
```

- `allow: true` → hook 退出 0
- `allow: false` → hook 退出 2，message 写 stderr 给 cc 作 reason

### 3.2 UI → daemon 决策

`POST http://<daemon_host>:<daemon_port>/api/v1/agent/permission/<request_id>/decide`

**Body**：

```json
{
  "allow": true,
  "remember": false,
  "message": ""
}
```

- `remember: true` 是 UX 优化（"以后这个 tool 都允许"）—— v0.1 不实现，留 stub
- `message` 可选，仅 deny 路径有意义

**Response**：

```json
{ "ok": true }
```

或 404 if `request_id` unknown / 过期。

### 3.3 daemon 内部状态

```go
type PermRequest struct {
    ID         string
    Tool       string
    Input      json.RawMessage
    SessionID  string
    Decision   chan PermDecision  // buffered 1
    CreatedAt  time.Time
}

type PermDecision struct {
    Allow   bool
    Message string
}

var perms sync.Map  // request_id (string) -> *PermRequest
```

超时清理：goroutine 每 30s 扫描 perms，丢 `time.Since(CreatedAt) > 5min` 的 entries。

### 3.4 超时策略

- hook → daemon HTTP client.Timeout: 60s（超时 → exit 2 deny）
- UI → daemon decide：无超时（用户随时点）
- daemon 端 default-deny：如果 request 等待 > 60s 没收到 decision，返回 `{allow: false, message: "timeout"}`

60s 是 cloudcli 同款数字（claude-sdk.js TOOL_APPROVAL_TIMEOUT_MS=55000）。

## 4. Hook binary 实现

### 4.1 源码（PoC step12 已验证）

```go
// internal/agent/permhook/main.go
// 编译为 ~/.tether/bin/tether-permission-hook，跨平台 Go 单 binary
package main

import (
    "bytes"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "os"
    "time"
)

func main() {
    body, _ := io.ReadAll(os.Stdin)
    endpoint := os.Getenv("TETHER_DAEMON_PERM_ENDPOINT")
    if endpoint == "" {
        fmt.Fprintln(os.Stderr, "[hook] TETHER_DAEMON_PERM_ENDPOINT unset")
        os.Exit(2)
    }
    client := &http.Client{Timeout: 60 * time.Second}
    resp, err := client.Post(endpoint, "application/json", bytes.NewReader(body))
    if err != nil {
        fmt.Fprintf(os.Stderr, "[hook] daemon unreachable: %v\n", err)
        os.Exit(2)
    }
    defer resp.Body.Close()
    var dec struct {
        Allow   bool   `json:"allow"`
        Message string `json:"message,omitempty"`
    }
    if err := json.NewDecoder(resp.Body).Decode(&dec); err != nil {
        fmt.Fprintf(os.Stderr, "[hook] decode: %v\n", err)
        os.Exit(2)
    }
    if dec.Allow {
        os.Exit(0)
    }
    fmt.Fprintf(os.Stderr, "[hook] denied: %s\n", dec.Message)
    os.Exit(2)
}
```

~30 LOC。

### 4.2 编译时机 + 安装位置

daemon 启动时（`tether server` step 4 in §10.A.4）做：

1. 检查 `~/.tether/bin/tether-permission-hook` 是否存在 + 版本一致（compare git SHA / build hash）
2. 如不存在或版本不同 → daemon 内嵌 hook source 编译写入
3. chmod 755

替代方案：tether release tarball 同时打包 hook binary 一起分发。**单 binary 叙事**仍然成立——主 binary 启动期"释放" hook helper，类似 `kubectl` 释放 `oc` plugins。

## 5. settings.json 注入策略

### 5.1 daemon 在 cc settings 注入 hook（careful merge）

cc 读 `~/.config/claude/settings.json`（or platform 等价路径）。tether daemon **不动用户既有 hook**，只**追加** PreToolUse 条目。

伪代码：

```go
// 启动期：
existingPath := claudeSettingsPath()  // ~/.config/claude/settings.json
existing := loadJSON(existingPath)    // { hooks: { ... user 设置的 ... } }

tetherEntry := map[string]interface{}{
    "matcher": "*",
    "hooks": []map[string]interface{}{{
        "type":    "command",
        "command": filepath.Join(home, ".tether/bin/tether-permission-hook"),
    }},
    "_tether_managed": true,  // 标记位，方便 daemon 关闭时清理
}

// PreToolUse 数组里如果已经有 _tether_managed=true 的条目就跳过；否则 append
appendIfNotPresent(existing.Hooks.PreToolUse, tetherEntry)

writeJSON(existingPath, existing)
```

### 5.2 daemon 关闭时清理（best-effort）

daemon 收到 SIGINT/SIGTERM 时，从 settings.json 移除 `_tether_managed=true` 的条目。

但**不强保证**：crash / kill -9 → settings.json 留 stale 条目。下次启动 step 5.1 会去重，no-op；用户**不会**因此卡住，hook script 找不到 daemon 就 exit 2 deny（保守 fail-safe）。

### 5.3 用户 opt-out

env: `TETHER_NO_PERMISSION_HOOK=1` → daemon 不注入 hook → cc 在 --print 模式自动允许所有 tool（cloudcli "Auto Mode = ON" 等价）

或 `tether server --no-perm-hook` flag。

## 6. UI 端协议

### 6.1 前端收到的 envelope（经 /wt/events broadcast）

```json
{
  "kind": "permission_request",
  "id": "perm-2026-05-09-001",
  "session_id": "e2970d8f-...",
  "tool_name": "Bash",
  "tool_input": {"command": "echo hello", "description": "..."},
  "created_at": "2026-05-09T18:30:00Z"
}
```

### 6.2 UI 渲染（D-19 PermissionBlock）

参考 cloudcli 的 permission UI：tool 名 + input JSON 高亮 + Allow / Deny 按钮 + 可选 "Always allow this tool" 复选（remember，v0.1 stub）。

### 6.3 UI → daemon 决策

```json
POST /api/v1/agent/permission/perm-2026-05-09-001/decide
{ "allow": true, "remember": false, "message": "" }
```

## 7. 跟其他模式的衔接

### 7.1 `permission_mode=bypassPermissions`（"Auto Mode"）

cc 加 `--permission-mode bypassPermissions` 时跳过 PreToolUse hook 吗？**未 PoC 验证**（猜测：会跳过，因为 hook 是 permission 路径的一部分）。

**操作**：v0.1 实施期 PoC 一下，确认 bypass 模式确实跳 hook。如果不跳，daemon 需要根据 user 的 mode 选择是否注入。

待验证后写进 D-05b §3 第 6 条 verified fact。

### 7.2 cc 自带 hook 跟 tether hook 共存

PoC step12 实测：用户 settings.json 里的 rtk-rewrite hook 跟 tether hook 同时跑，rtk 先 fire（matcher 更精确）→ tether 后 fire（matcher: "*"）。两个都 exit 0 才 allow。

如果用户 hook 自己就 deny → tether daemon 永远等不到 hook callback POST（hook 在用户 hook 阻断时根本没机会跑）。daemon 端 60s 超时后 cleanup request。**对 user 体验**：UI 永远不见 PermissionBlock，cc 直接 deny。

## 8. PoC 路线（已完成）

- ✅ **step11**：验证 stream-json 没有 permission_required 事件
- ✅ **step12**：完整 hook → daemon HTTP → decision → exit code 流程，DENY + ALLOW 两 pass 均通

## 9. 跟 D-05a 的衔接

D-05a §3 需要重写：

- 删除"待 PoC 验证"措辞
- 加事实：stream-json 模式无 permission_required 事件（step11 实测）
- 引用 D-05b 作为 permission UI 实施方法

D-05a §8 acceptance criteria 加：

- [ ] tether-permission-hook binary 在 daemon 启动时安装到 `~/.tether/bin/`
- [ ] daemon 启动时注入 PreToolUse hook 到 cc settings.json，关闭时清理
- [ ] 6 个端到端 case：default mode allow / default mode deny / bypass mode (skip hook?) / hook timeout → deny / daemon down → deny / user hook 共存

## 10. 风险与 mitigation

| 风险 | 影响 | Mitigation |
|---|---|---|
| daemon 修改 settings.json 跟用户编辑器冲突 | 用户改 settings 时 daemon 又改 | settings.json 有 file lock；daemon 改时 acquire；冲突回退 |
| daemon crash 留 stale hook 条目 | 用户重启 cc 时 hook script 找不到 daemon → deny 一切 | 启动期 step 5.1 自带去重；用户用 `--no-perm-hook` 临时禁用 |
| hook binary 跨平台编译失败 | Windows / 老 macOS 跑不起来 | release tarball 同时打包 darwin/linux/windows 三套；启动期检测 GOOS 选合适的 |
| cc 改 hook 协议（新版本）| daemon 注入的 hook script 失效 | hook script 容错：未知字段忽略；release notes 监控 cc 版本兼容性 |
| 用户已有 PreToolUse hook 跟 tether hook 顺序问题 | 用户 hook 阻断 tether hook 永远不 fire | spec §7.2 已明示两个共存语义；用户可以用 `--no-perm-hook` opt-out |
| 60s 超时太长 / 太短 | UX 过慢或过短 | 实施期可配（`TETHER_PERM_TIMEOUT=Ns`），默认 60s 跟 cloudcli SDK 对齐 |

## 11. Acceptance criteria

- [ ] `tether-permission-hook` Go binary 跨平台编译通过（linux/macOS/windows）
- [ ] daemon 启动时安装 hook binary 到 `~/.tether/bin/`，权限 755
- [ ] daemon 启动时注入 PreToolUse hook 条目到 cc settings.json（matcher: "*", `_tether_managed: true`）
- [ ] daemon 关闭时移除 `_tether_managed: true` 的条目（best-effort）
- [ ] cc 触发 tool → hook → daemon → broadcast 到 /wt/events
- [ ] UI POST decide → daemon 解锁 hook → cc 按 exit code 执行/拒绝
- [ ] 60s timeout fail-safe deny
- [ ] daemon 进程死 → hook script POST 失败 → 自动 deny（fail-safe）
- [ ] `TETHER_NO_PERMISSION_HOOK=1` env 跳过整套机制
- [ ] cc bypass 模式行为已 PoC 验证（hook 是否跳过、daemon 是否处理）
- [ ] 用户既有 PreToolUse hook 不被破坏（merge 逻辑保留）
- [ ] `permission_denials[]` 字段在 deny 路径正确填充（PoC step12 已实测）
