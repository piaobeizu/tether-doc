# D-05a: cc agent integration — stream-json + 双通道架构

**Date:** 2026-05-09
**Author:** wxk + Claude
**Status:** Proposed — 替代主 spec D-05（in-flight，跟 D-22 / D-17a 一同等用户拍板新方向后正式纳入主 spec）
**Companion docs:**
- 主 spec D-05 `2026-04-26-tether-go-quic-design.md` §3.5 (待替代)
- D-17a `2026-05-09-d17a-agent-provider-multi-provider.md`
- D-22 `2026-05-09-d22-tygo-schema-sync.md`
- PoC step8 `poc/go-quic-wt/step8_dualchannel.go` (3/3 ✓)

---

## 0. 这份文档解决什么

主 spec D-05 锁的是：

> v0.1 ClaudeCodeProvider 走 **PTY (mode ①) + Hooks (mode ③) + JSONL watcher**。**禁用** SDK 与 stream-json 模式。

这个决策是 2026-04 拍的，当时 stream-json 在 cc 1.x 不稳。**2026-05 这俩月信息变化**：

1. cc 已升到 2.1.138，stream-json 模式（`--print --output-format stream-json --input-format stream-json --verbose`）成熟可用
2. cloudcli 用 Anthropic Agent SDK（内部就是 stream-json）端到端验证产品可行（10K+ stars）
3. tether PoC step8 用 Go 直接驱动 stream-json，3 个假设全部通过（长跑、session 稳定、可跟 PTY 并存）

**D-05a 推翻原 D-05，转向**：
- **禁用 PTY + Hooks 复杂协调**
- **采纳 stream-json 长跑模式作 chat 主通道**
- **保留 PTY 作 shell 副通道**（cloudcli 同款双 tab，用于 slash command 类场景）

## 1. 决策

**v0.1 ClaudeCodeProvider 实施为双通道**：

### Chat 主通道（stream-json，长跑）

```
Go daemon
  └─ exec.Cmd("claude --print --verbose
                       --output-format stream-json
                       --input-format stream-json
                       --resume <sid>")
      ├─ stdin:  JSON line per user message + permission decision
      └─ stdout: JSON line per event (system/init / assistant / result / ...)
```

**特征**：
- 单进程长跑，跨多 user prompt
- 0 启动开销 per prompt（~3s 仅首次）
- 事件结构 cc 官方维护
- daemon 把这些事件 normalize 成 tether wire envelope 转发前端

### Shell 副通道（PTY，可选 attach）

```
Go daemon
  └─ pty.Spawn("claude --resume <sid>")  # 跟 chat 同 sessionId
      ├─ stdin ← user keystrokes (经 daemon 中转)
      └─ stdout → terminal bytes (经 daemon 中转给 xterm.js)
```

**用途**：
- 跑 slash commands（`/plugin update` / `/reload` / `/clear` / `/help`）—— stream-json 模式不支持
- power user 的 raw TTY 体验
- 紧急人工干预活会话

**两通道指向同一 sessionId**，cc 自己协调 jsonl 读写——已实测 PoC step8 PASS。

### **删除原 D-05 的所有 PTY + Hooks 协调路径**

具体不再做：
- ❌ `internal/agent/hookserver.go` HTTP endpoint 做 cc hook callback
- ❌ `internal/cc/hooks.go` 设置 `~/.claude/settings.json` 里的 PreToolUse hook 写回 daemon
- ❌ JSONL chokidar watcher 做实时事件捕获（仅留作 catch-up + cross-session sync 副路径）

**节省 LOC 估算**：~2500-4000（4 路协调 → 单 stdin/stdout JSON RPC + 旁路 PTY）。

## 2. 验证过的事实（来自 PoC step8）

> 这 5 条事实从 `poc/go-quic-wt/step8_dualchannel.go` 实测得出。spec 实施期可直接引用。

### 事实 1：长跑多 turn 在 stream-json 模式工作

```
spawn claude --print --verbose --output-format stream-json --input-format stream-json
write {"type":"user","message":{"role":"user","content":"say AAA"}}
→ assistant text: "AAA"
write {"type":"user","message":{"role":"user","content":"say BBB"}}
→ assistant text: "BBB"
```

单 process，两轮对话，0 重启。

### 事实 2：session_id 跨 turn 稳定

`system/init` 事件每 turn 触发一次，但 `session_id` 字段保持不变（`cc1d6ec1-eada-4b96-a17d-90a8b5c6c37c` 跨两 turn 一致）。

**对实施的暗示**：daemon 第一次见到 init 事件后**记住 sessionID**，后续 turn 的 init 事件不要当成 session boundary，只更新 model / cwd 等元数据即可。

### 事实 3：PTY claude --resume 跟长跑 stream-json 进程能并存

step8 同时跑两个 cc 进程指向同 sessionId，jsonl 文件由 cc 内部协调，无 race 观测。

### 事实 4：PTY 输出含完整 TUI

包括 ANSI box-drawing、Welcome banner（"Welcome back stevenforai!"）、prompt redraw、光标控制。**前端 shell tab 必须用 xterm.js**（200KB JS）才能正确渲染。

### 事实 5：root 用户 + bypassPermissions 需要 IS_SANDBOX=1 绕过

cc 检测到以 root 跑且带 `--dangerously-skip-permissions` 会**直接拒绝退出 1**：

```
--dangerously-skip-permissions cannot be used with root/sudo privileges for security reasons
```

**Mitigation**：tether daemon 在**以 root 跑**时（`os.Geteuid() == 0`）给 cc 子进程加 `IS_SANDBOX=1` env。这是 cc 官方 escape hatch（声明在沙箱内）。**非 root 不注入**——cc 的 root 检查不触发，注了也无害但保持对应实际语义。

```go
if os.Geteuid() == 0 {
    cmd.Env = append(os.Environ(), "IS_SANDBOX=1")
} else {
    cmd.Env = os.Environ()
}
```

PoC step10 已实测此条件注入逻辑（`step10_browser_dual.go` chat + shell 两 handler 均按此模式）。

## 3. Stream-JSON 事件类型（实测）

PoC step8 一个简单两轮对话观察到的全部事件：

| event type | subtype | count | 含义 |
|---|---|---|---|
| `system` | `init` | 2 | 每 turn 一次的会话元信息（model / cwd / tools / plugins） |
| `system` | `hook_started` | 3 | SessionStart hook 启动（用户 settings.json 里 3 个） |
| `system` | `hook_response` | 3 | hook 输出 |
| `assistant` | (none) | 3 | 模型输出消息（含 text + tool_use blocks） |
| `result` | `success` | 2 | 每 turn 结束的统计（tokens, cost, duration） |
| `rate_limit_event` | (none) | 1 | API rate limit 信息 |

**实施期已知事实（PoC step11 + step12 实测）**：

- **stream-json 模式无 `permission_required` 事件**——PoC step11 验证：`--permission-mode default` 跟 `bypassPermissions` 输出 13 个事件**完全一致**，无任何 perm/approval/request 类型事件。`--print` 模式下 cc **自动批准并执行所有 tool 调用**。
- **Permission UI 实施路径** = **cc PreToolUse hook + daemon HTTP callback** —— 详见 **D-05b** addendum。daemon 启动时往 `~/.config/claude/settings.json` 注入一个 `_tether_managed: true` 的 PreToolUse 条目（matcher: `*`），hook 是 tether 自带 Go 编译 binary（~30 LOC，编进 release tarball），通过 HTTP POST 阻塞等 daemon 决策，按返回 exit 0 (allow) / 2 (deny) 给 cc。PoC step12 已端到端验证（DENY + ALLOW 两 pass 全通）。
- **`result.permission_denials[]`** —— deny 路径下 cc 把 tool 调用记录在 result 事件这个字段。daemon 可从这里观察拒绝历史。
- **`partial_message` chunks** — 流式 text chunk（需要 `--include-partial-messages` flag 才有）；v0.1 chat 通道默认**不启用**（事件量 ~10×，UX 收益主要是"打字机效果"，非必需）。

## 4. AgentProvider Go interface 落地

详细 interface 定义在 D-17a §3。这里只列 ClaudeCodeProvider 的 Spawn / Session 实现要点：

```go
// internal/agent/claude_provider.go
type ClaudeCodeProvider struct {
    sandbox bool  // env IS_SANDBOX=1
}

func (p *ClaudeCodeProvider) Spawn(ctx, sid, opts) (Session, error) {
    cmd := exec.CommandContext(ctx, "claude",
        "--print", "--verbose",
        "--output-format", "stream-json",
        "--input-format", "stream-json",
        "--resume", sid,
        // ... model / cwd / etc from opts
    )
    if os.Geteuid() == 0 {
        cmd.Env = append(os.Environ(), "IS_SANDBOX=1")
    } else {
        cmd.Env = os.Environ()
    }
    
    stdin, _ := cmd.StdinPipe()
    stdout, _ := cmd.StdoutPipe()
    if err := cmd.Start(); err != nil { return nil, err }
    
    s := &claudeSession{
        cmd: cmd, stdin: stdin,
        events: make(chan Event, 32),
    }
    go s.readEvents(stdout)
    return s, nil
}
```

session 持有 stdin pipe + events channel，`SendPrompt` 写 JSON 到 stdin，`Events` 返回 channel。

## 5. 跟 D-17a / D-22 的衔接

- **D-17a**：ClaudeCodeProvider 是 D-17a `AgentProvider` interface 的 v0.1 唯一实现。其他 provider（opencode 等）将来按相同 interface 实现各自的 chat 通道。Shell 通道由 daemon 通用 PTY 处理，跟 provider 无关。
- **D-22**：stream-json 事件类型 + tether 内部 normalized Event shape 都在 `internal/wire/types.go` 定义，tygo 生成 TS 给前端。前端按 `Event.Kind` 分发渲染——claude 跟未来其他 provider 共享同一份 wire 类型。

## 6. 反对意见对照

### "失去 PTY raw-mode 体验"

不准确。PTY raw-mode 比 stream-json 多的只是**终端装饰层**（ANSI 颜色、box-drawing、in-place 更新）。**内容信息相同**。

shell 标签页保留 PTY 体验，是 power user 的备份路径。chat 主通道走结构化事件，前端用 React 组件渲染——比 PTY bytes 翻译成 HTML 还干净。

### "long-running stream-json 不能跑 slash command"

对。但这就是 shell 副通道的存在意义——`/plugin update` / `/reload` / `/clear` 在 shell 标签页跑。两通道**互不干扰**——shell 通道改了 plugin 后，chat 通道下一条 prompt 自动用新 plugin（cc 进程内 plugin 状态在 turn 边界更新）。

### "Hook 机制更优雅"

cc PreToolUse hook 走 HTTP callback 回 daemon——这条路在新架构下**一样工作**，但 stream-json 模式下 cc 已经把 hook 事件 inline 进 stdout 流。spec 不再要求 daemon 自己起 HTTP endpoint 做 hook callback；如果用户的 settings.json 里有 hook，cc 自己处理（hook 脚本在 cc 进程上下文跑），事件经 stream-json 进入 daemon。

## 7. 升级路径

stream-json 是 cc 官方公开协议。Anthropic 升级时：

- **小版本（2.x → 2.y）**：通常加新事件 type，不删旧的——daemon 只要忽略未知事件即可向前兼容
- **大版本（2.x → 3.0）**：可能改 envelope shape，daemon 实施期 pin 到具体 cc 主版本，升级路径单独评估

如果未来 stream-json 不能满足（譬如某个高级 hook 仅 SDK 暴露），路径：

1. 内部加 Node sidecar 跑 SDK，stdin/stdout 自定义协议——破坏 single-binary 叙事，但只对 ClaudeCodeProvider 适用
2. 等 Anthropic 把那个 hook 暴露到 stream-json

## 8. Acceptance criteria

- [ ] `internal/agent/claude_provider.go` v0.1 实现，符合 D-17a AgentProvider interface
- [ ] 长跑模式：单 cc 进程跨 ≥ 3 个 user prompt，session_id 稳定（基于 step8 pattern）
- [ ] cc 子进程启动 env 在 uid==0（root）时含 `IS_SANDBOX=1`，非 root 时不注入
- [ ] Shell 通道 daemon endpoint：`/wt/shell?sid=<id>` 起 PTY claude --resume，前端 xterm.js 双向透传
- [ ] PoC step8 升级为 step8b：浏览器实际跑两条 WT stream（chat + shell），cloudcli-style 实测验证
- [ ] 删除 `internal/agent/hookserver.go` + `internal/cc/hooks.go` （改成空 stub 或彻底删除）
- [ ] JSONL watcher 退化为 catch-up 副路径，不再是实时事件主源
