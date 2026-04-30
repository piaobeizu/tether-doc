# Plan: cc integration via SDK route — implementation

| 字段 | 值 |
|---|---|
| 状态 | draft |
| 起草日期 | 2026-04-30 |
| 作者 | wxk |
| 关联 spec | `<ticket-dir>/.specs/2026-04-30-cc-sdk-route.md` |
| 关联 issue | piaobeizu/tether#13 |
| 模式 | new |

---

## 1. Scope

### 1.1 涉及的 repo

| repo | role | 角色 | 是否本 plan touched |
|---|---|---|---|
| tether | primary | 所有 Go 代码（`internal/backend/claude/`） | yes |
| tether-app | primary | Epic #6/#7 的 UI 代码 | no（spec §3.2 已声明）|
| tether-doc | doc | spec/plan promotion + lessons + retro | yes（仅 `/polyforge:commit` step 0 自动 promote，本 plan 内**无显式 step**） |

### 1.2 M5 gate 检查

workspace.yaml **未声明 `role: config` repo**——本 plan 不触发 commit M5 config-repo-PR 要求。单 PR 可以走完。

### 1.3 总体策略

- **serial-in-tether**：每个 step 在 tether 内顺序构建，后步依赖前步
- **doc 自动促进**：spec/plan 在 `/polyforge:commit` step 0 自动迁入 `tether-doc/wiki/`，本 plan 不安排手工 step
- **测试驱动**：spec §8 的 7 个 scenarios + 3 条断言验证（B' / C' / C''）作为各 step 的"done when X" 验收

---

## 2. Pre-flight

| 检查 | 状态 |
|---|---|
| spec 已通过 review | ok（用户 2026-04-30 "放行"）|
| `tether` worktree 干净，0 ahead | ok |
| `~/.claude/auth/` OAuth 有效（spike 用） | ok（claude.ai enterprise 已登录）|
| Go 1.24.6 可用 | ok |
| `claude` v2.1.123 ELF binary 可用 | ok |
| AnthropicAPI 配额 / 余额 | 未实测；spike 总成本预算 < $5（Haiku 优先） |

---

## 3. Steps

### Step 0: bootstrap Go module + package skeleton

**Repo**: tether
**Commit msg**: `chore(backend/claude): bootstrap go module + package skeleton`

**Files**:
- `tether/go.mod`（新增，`go mod init github.com/piaobeizu/tether`）
- `tether/go.sum`（新增）
- `tether/internal/backend/claude/doc.go`（新增，单行 package comment）
- `tether/.gitignore`（追加 `*.test`、`/cmd/*/spike-out/`）

**Done when**:
- `cd tether && go mod tidy` 无错
- `go build ./...` 通过
- `go vet ./...` 无 warning

---

### Step 1: spawn.go — minimal spawn + stdio capture

**Repo**: tether
**Commit msg**: `feat(backend/claude): spawn cc subprocess with stream-json stdio`

**Files**:
- `tether/internal/backend/claude/spawn.go`（新增）
- `tether/internal/backend/claude/spawn_test.go`（新增）

**API（最小）**:
```go
type SpawnOpts struct {
    ProjectCwd     string  // 容器内的 project 路径（cwd 策略见 Step 8）
    SessionID      string  // 非空 → 加 --resume <sid>
    Model          string  // 例如 "haiku" / "sonnet"
}

type Subprocess struct {
    Cmd    *exec.Cmd
    Stdin  io.WriteCloser
    Stdout io.ReadCloser
    Stderr io.ReadCloser
}

func Spawn(ctx context.Context, opts SpawnOpts) (*Subprocess, error)
```

**Done when**:
- 能 spawn `claude` 并立即 SIGKILL，`Cmd.Wait()` 返回非 nil error（被 kill 是预期）
- 单元测试覆盖 SessionID 空 / 非空两条路径的 args 构造
- 三个 pipe 在 spawn 后可读写
- 验证 **A.1**：断言 `cmd.ExtraFiles == nil`（不开 fd 3）；scenario 1 运行时 `lsof -p <pid>` 输出无额外 fd
- 验证 **F.2**：故意把 PATH 改成无 `claude` → `Spawn()` 返回 `ErrBinaryNotFound`，错误信息含搜索路径
- 验证 **A.5**：scenario 1 跑 1 分钟，stderr 持续被 drain（用大 stderr 输出测试，不阻塞 stdin/stdout）

---

### Step 2: message.go — NDJSON envelope structs

**Repo**: tether
**Commit msg**: `feat(backend/claude): NDJSON envelope types with version-tolerant parse`

**Files**:
- `tether/internal/backend/claude/message.go`（新增）
- `tether/internal/backend/claude/message_test.go`（新增）

**实现要点**:
- 顶层 `Event` 用 tagged union，按 `type` 字段分发（`system` / `stream_event` / `assistant` / `user` / `result` / `control_request` / `control_response`）
- 不在 union 里的字段保留为 `json.RawMessage` 透传——满足 spec §6.A.2 "unknown-subtype graceful-degrade"
- `system` event 再按 `subtype` 二级分发（`init` / `status` / `api_retry` / `plugin_install` / `hook_started` / `hook_response`）
- 每个 struct 加 `version` 字段（看 jsonl 实测有这个字段）

**Done when**:
- 用 `/tmp/claude-resume-probe/stream1.ndjson`（spec brainstorm 阶段抓到的 72 行）跑 round-trip parse + re-marshal，无字段丢失
- 引入一行未知 `type` 的 fake event，parser 不 panic，graceful 跳过 + 日志

---

### Step 3: parser.go — line-buffered scanner + event dispatcher

**Repo**: tether
**Commit msg**: `feat(backend/claude): NDJSON line scanner with channel dispatch`

**Files**:
- `tether/internal/backend/claude/parser.go`（新增）
- `tether/internal/backend/claude/parser_test.go`（新增）

**实现要点**:
- `bufio.Scanner` with `Buffer(make([]byte, 64*1024), 256*1024)` —— 实测 max line ~13KB，留余量
- `func Parse(r io.Reader) <-chan Event` 返回 channel；scanner 退出时 close channel
- 错误行（非 JSON）日志降噪不阻断（参考 happy `query.ts:120` 的 `logger.debug(line)` 模式）

**Done when**:
- 跑 stream1.ndjson fixture 通过 channel 收到 ≥70 个有效 event
- 跑 fixture + 故意插一行 `not json garbage` 不 panic
- 跑 fixture + 故意截断最后一行（mid-line tearing 模拟），不挂

---

### Step 4: control.go — bidirectional control_request / response

**Repo**: tether
**Commit msg**: `feat(backend/claude): bidirectional control protocol for tool authorization`

**Files**:
- `tether/internal/backend/claude/control.go`（新增）
- `tether/internal/backend/claude/control_test.go`（新增）

**实现要点**:
- 参考 happy `query.ts:170-220`（约 80 行 TS 直译 ~150 行 Go）
- inbound `control_request can_use_tool`: 调用 `ToolAuthorizer interface` 的 callback → 写 `control_response` 回 stdin
- outbound `control_request interrupt`: 生成 random `request_id` → 写 stdin → 在 pendingResponses map 等 matching response → 返回
- 对所有未实现的 `control_request.subtype`：写一个空 `control_response success`，日志 unknown subtype（A.2）
- map 用 `sync.Mutex` 保护，因为 inbound/outbound 跨 goroutine

**Done when**:
- 单测：mock stdin/stdout，模拟 inbound can_use_tool → 验证 callback 被调 + response 写出
- 单测：模拟 outbound interrupt → mock 返回 control_response → 验证 Wait 正确返回
- 单测：unknown subtype inbound → 验证空 success response 写出，不 panic

---

### Step 5: session.go — Session struct + sid discovery + lifecycle

**Repo**: tether
**Commit msg**: `feat(backend/claude): Session lifecycle with sid discovery from system/init`

**Files**:
- `tether/internal/backend/claude/session.go`（新增）
- `tether/internal/backend/claude/session_test.go`（新增）

**API**:
```go
type Session struct {
    SessionID  string  // 从 system/init 提取（spec §4.3）
    ProjectCwd string
    sub        *Subprocess
    parser     <-chan Event
    control    *Controller
    state      SessionState  // idle / streaming / tool-pending / mid-text
    mu         sync.Mutex
}

func New(ctx context.Context, opts SpawnOpts, auth ToolAuthorizer) (*Session, error)
func (s *Session) Send(msg UserMessage) error
func (s *Session) Events() <-chan Event
func (s *Session) Close() error
```

**Done when**:
- Spike scenario 1（完整对话）跑通：`Send` 一条 user msg → 收到 `system/init`（提取 sid）→ `stream_event` × N → `assistant` → `result`
- 单测：session.SessionID 在第一个 system/init 后非空
- 单测：`Close` 干净 SIGTERM 子进程（10s 超时再 SIGKILL）
- 验证 **A.4**：spawn 时故意传 invalid `--resume <bogus-sid>`，`New()` 在 10s 内返回 `ErrInitTimeout`（不挂）
- 验证 **A.3**：`Send()` 后立即 SIGKILL 子进程，下一个 `Send()` 返回 `ErrSubprocessGone`（不返回泛型 EPIPE error）
- 验证 **§7.7**：resume 模式下 stream 出现两次 system/init，session.SessionID 取最后一次值，无 panic
- 验证 **E.1**：grep `internal/backend/claude/` 无任何对 `~/.claude/projects/**` 或 `*.jsonl` 的写入操作（lint 检查或人工 review）

---

### Step 6: Session.Recover() with safe-to-reload state machine

**Repo**: tether
**Commit msg**: `feat(backend/claude): Session.Recover with safe-to-reload gates (C.1-C.5)`

**Files**:
- `tether/internal/backend/claude/session.go`（修改：加 `Recover` 方法 + state machine）
- `tether/internal/backend/claude/recover_test.go`（新增）

**State machine（per spec §6.C.1–C.3）**:
```
states:
  idle           ← 初始 / result 之后
  streaming      ← stream_event 中（mid-text）
  tool-pending   ← assistant tool_use 已发，未收到 user.tool_result
  mid-text       ← 同 streaming 的子状态（保留区分以备日志）

transitions:
  Send(msg) → streaming
  stream_event message_delta stop_reason=tool_use → tool-pending
  user.tool_result received → streaming
  result received → idle

Recover() guard:
  state == idle ? proceed : return ErrUnsafeToReload
```

**实现要点**:
- `Recover(ctx)`: 先杀子进程 → spawn 新 `claude --resume <sid>` → 等 system/init 校验 sid 一致 → 转 idle
- log warning 如果 `Recover` 在非 idle 调用（业务必须先 wait 到 idle）

**Done when**:
- Spike scenario 5（clean resume）：跑完 scenario 1 → `Close()` → `New(opts with SessionID=sid)` → `Send` 新 msg → 验证模型记得前面的对话
- Spike scenario 6（mid-tool kill + recover）：手动构造 tool-pending state → `Recover()` 返回 `ErrUnsafeToReload`
- Spike scenario 6 second-half：等 result → idle → `Recover()` 成功 → 模型不重跑前面的 tool（per spec §7.1 已实测行为）
- Spike scenario **4b**（mid-text recover 被拒）：在 `text_delta` 中途主动调 `Recover()` → 验证返回 `ErrUnsafeToReload`
- 验证 **C.6**：mock 一个未完成的 control_request → 调 `Recover()` → 验证 pending map 被清空，对应 channel 收到 `ErrRecoverInterrupted`
- 验证 idle-wait 硬超时：人为不发 `result` event 120s（mock parser），`Recover()` 仍能强制进 idle 并返回 `ErrStaleStream` warning
- 验证 **多 Session 并发**（spec §10.5）：起两个 Session（不同 sid）并发跑 scenario 1，互不干扰；其中一个 `Recover()` 不影响另一个

---

### Step 7: end-to-end spike driver — covers all 7 scenarios

**Repo**: tether
**Commit msg**: `feat(backend/claude): spike driver covering scenarios 1-7`

**Files**:
- `tether/cmd/spike/main.go`（新增）
- `tether/cmd/spike/scenarios.go`（新增）
- `tether/cmd/spike/README.md`（新增，跑法 + 期望输出）

**实现要点**:
- 每个 scenario 一个 func，按 spec §8 编号 1-7
- 自动构造 prompt + 自动 approve / deny tools by scenario
- 输出 markdown 报告：`tether/cmd/spike/spike-out/2026-MM-DD-spike-report.md`
- scenario 4（mid-text abort via outbound control_request interrupt）需 control.go 已就位
- scenario 7（hook idempotency）需要先放一个简单的 SessionStart hook（在 spike 临时 settings 里）

**Done when**:
- `go run ./cmd/spike` 跑完 7 个 scenarios，全 PASS
- 跑两次（验幂等性 + JuiceFS 影响）都 PASS
- 报告生成 + check in（gitignore 排除 spike-out 但 commit 一份示例）

---

### Step 8: cwd 策略二选一 + JuiceFS 路径验证

**Repo**: tether
**Commit msg**: `feat(backend/claude): cwd policy decision + jsonl path validation`

**Files**:
- `tether/internal/backend/claude/spawn.go`（修改：固化 cwd 策略，加常量 `DefaultProjectCwd`）
- `tether/cmd/spike/scenarios.go`（修改：加 cwd 实验 scenario 8）
- `tether-doc/wiki/lessons/_raw.jsonl`（追加一行 lesson）

**决策**:
跑实验：从 `$HOME` vs `/some/other/dir` spawn → 各自 jsonl 落到 `~/.claude/projects/-root/` vs `~/.claude/projects/-some-other-dir/`，验证 §6.E.5 的现象。然后**二选一**：
- (a) 固定 `$HOME` cwd——所有 session 进同一 bucket
- (c) 提供 `tether resume <sid>` wrapper——CLI tool 自动 cd

**推荐 (a)**：spike 阶段简单粗暴，wrapper 留 v0.1 后续 sub-ticket。

**Done when**:
- 决策落进 spawn.go 常量 + 注释引用 spec §6.E.5
- tether-doc lessons 加 `cwd-policy-fixed-home` 一行 jsonl
- 此决策若改动，commit msg 必须显式说

---

### Step 9: B' / C' / C'' 实测验证

**Repo**: tether
**Commit msg**: `test(backend/claude): cross-version + sonnet-cost + pod-restart verification`

**Files**:
- `tether/cmd/spike/verify_b_prime.go`（新增——拉两个 CC 版本跑 scenario 1，diff stream）
- `tether/cmd/spike/verify_c_prime.go`（新增——Sonnet 4.6 跑 scenario 5，记录 cost）
- `tether/cmd/spike/verify_c_doubleprime.sh`（新增——k8s SIGTERM 测试 shell 脚本，需 dev cluster）
- `tether/cmd/spike/spike-out/verification-report.md`（新增）

**Done when**:
- B'：两个 CC 版本的 NDJSON envelope schema 兼容（关键字段 `type` / `session_id` / `subtype` 都还在；新版可以多字段，但旧版字段不能消失）
- C'：Sonnet 4.6 resume 成本数据点入 report，与 Haiku 数据点对比
- C''：dev cluster 上 SIGTERM pod → 等 reschedule → 验证 cc startup 能从 JuiceFS 读 jsonl。**扩展 D.1 全路径**：除 `~/.claude/projects/` 外，也验证 `~/.claude/plugins/`、`~/.claude/auth/`、`~/.claude/settings.json` 在 PV remount 后可读（如果 dev cluster 没准备好，本 step 可降级为"手工模拟 PV unmount/remount 时序"）

---

### Step 10: 后续 sub-ticket 拆分清单 + handoff cleanup

**Repo**: tether（修改）+ workspace root（删除 handoff md）+ tether-doc（lessons）
**Commit msg**: `docs(backend/claude): post-spike sub-ticket list + handoff retire`

**Files**:
- `tether/internal/backend/claude/README.md`（新增——lib 用法 + 后续 sub-ticket 链接）
- `<workspace-root>/SESSION-HANDOFF-2026-04-30-sdk-migration.md`（**删除**——已被 spec 取代）
- `tether-doc/wiki/lessons/_raw.jsonl`（追加 spike 经验 lesson）

**清单**（写进 README 也开 sub-issue）：
1. cwd 策略 (c) wrapper 实现（如果 step 8 选了 (a)）
2. Session.Recover() 全套 4 caller 接入（plugin reload / pod restart / daemon crash / WT reconnect）—— 本 spike 只验证原语，4 caller 各自集成是独立 ticket
3. hook idempotency 验证（如果将来装 tether-owned hook）—— 本 spike 不装任何 hook
4. JuiceFS 持久化路径部署 manifest（k8s PV YAML）
5. GC stub session 后台扫描器（goroutine）
6. tool authorization UX 与 WT 协议映射（依赖 Epic #6 UI）—— 本 spike 只完成 daemon 端 control_request 握手
7. C.5 reload UX 反馈（"正在重载会话…"）—— 依赖 Epic #6 UI
8. 凭据注入路径（依赖 Epic #5 / 部署 ticket）
9. operational concerns（log volume cap / OOM circuit-breaker / claude version drift detection / pre-flight API balance check）—— 部署/ops 单独 ticket

**Done when**:
- README 写好
- workspace root handoff md 已删
- 7 条 sub-ticket 在 GH 上开好（标记 milestone v0.1，挂 Epic #3）

---

## 4. Risks

| # | 风险 | 概率 | 影响 | 缓解 |
|---|---|---|---|---|
| R1 | `--input-format stream-json` multi-turn 行为与 happy 默认假设有偏差 | 中 | 中—改 architecture | scenario 1 + 2 早跑早验，发现偏差立即调整设计 |
| R2 | `control_request can_use_tool` 协议在新版 CC 里 wire 改了 | 低 | 高—工具授权破 | A.2 versioned parser；happy + companion 两家 RE 互证降低概率；CC v2.1.123 是当前稳定版 |
| R3 | `claude --resume` 跨 cwd 行为与实测不一致（容器环境差异）| 低 | 中—cwd 策略要重选 | step 8 早跑实验，选 (a) 而非 (c) 减少耦合 |
| R4 | C''（k8s pod restart）实测因 dev cluster 不稳跑不通 | 中 | 低—验证降级为推理 | 允许 step 9 C'' 降级为"代码上 review 一遍"，写进 verification-report 标注 |
| R5 | spike 总 token 成本超 $5 预算 | 低 | 低 | 优先 Haiku；Sonnet 仅 C' 单点；Opus 不跑 |
| R6 | spec §10 4 个开放问题在 spike 中暴露新约束 | 中 | 中—spec 要 supersede | 每暴露一个就在 lessons jsonl 记一条；spec 走 supersede chain（`/polyforge:spec --continue`）|

---

## 5. Test strategy

每个 step 的 "Done when" 即验收。汇总：

| 测试类型 | 覆盖 | 在哪个 step |
|---|---|---|
| 单元测试（Go test）| spawn / parser / control / session 各模块独立 | Step 1–6 |
| 集成测试（spike scenarios 1-7）| end-to-end 跨模块 | Step 7 |
| 实测验证（B' / C' / C''）| 跨版本兼容 / 跨模型成本 / 部署时序 | Step 9 |
| 幂等性 / 重跑 | spike 跑两次都过 | Step 7 |

外部依赖测试：
- 真 Anthropic API 调用（用 Haiku 控制成本）
- JuiceFS 挂载点 I/O（容器内默认就有）
- claude binary v2.1.123 可执行

---

## 6. Rollback

本 ticket 是**纯新增代码 + 新 go module**——无 prod 风险（tether v0.1 还没发布）。

| 场景 | rollback |
|---|---|
| Step 0–7 任一 step 实测发现根本性问题 | `git reset --hard origin/main`，spec supersede 重写 |
| Step 8 cwd 策略选错 | 单文件 revert spawn.go，重新选 |
| Step 9 实测结果不符断言 | 记 lesson + spec supersede，新 ticket 解决根因 |
| Step 10 sub-ticket 拆分有遗漏 | GH issue 后期补开 |

无 DB migration、无 schema 变更、无外部 API 契约变更——rollback 永远是 `git revert`。

---

## 7. Open dependencies（不卡 plan 启动，但 pre-merge 前要解）

| 项 | 解到 / 谁解 |
|---|---|
| spec §10.1 hooks/MCP/settings 继承策略 | spike 阶段定，写进 README |
| spec §10.2 工具授权 UX → WT 协议映射 | 依赖 Epic #6 ticket（不阻塞本 PR）|
| spec §10.3 凭据注入路径 | 依赖部署 ticket / Epic #5（spike 用本地 OAuth，commit 前写 README 标注）|
| spec §10.4 cwd 策略二选一 | step 8 显式落决策 |
| spec §10.5 一用户多 Session 模型 | Step 6 done-when 加并发 Session 验证 |

---

## 8. 单 PR 可行性

- **steps**: 11（< 15 警戒线，ok）
- **repos**: 1 + 1 doc auto-promote = effective 1（< 5 警戒线，ok）
- **M5 gate**: 无 config repo，不触发
- **总代码量估计**: ~1500–2500 行 Go（含测试）
- **单 PR 单独 reviewable**: 是

---

## 9. Estimated timing

| 阶段 | 估时（专心做）|
|---|---|
| Step 0–1 | 1h（go mod init + spawn.go）|
| Step 2–3 | 2h（envelope + parser）|
| Step 4 | 2h（control 双向）|
| Step 5 | 2h（session lifecycle）|
| Step 6 | 3h（state machine + recover）|
| Step 7 | 2h（spike driver glue）|
| Step 8 | 1h（cwd 决策 + 实验）|
| Step 9 | 2h（实测验证 + report）|
| Step 10 | 1h（README + cleanup + sub-tickets）|
| **合计** | **~16h spike + lib core** |

**关于 handoff "1–2h spike" 的对照**：handoff 原话指的是**单文件 PoC**（验证 happy 路线能跑通即可）。本 plan 是**完整 lib + tests + 实测验证**——包括 typed errors、state machine、idempotent hooks 验证、跨版本/跨模型/k8s 实测——这些是把"spike PoC"变成"v0.1 可投产 lib"的额外投资，不是 scope creep。如果接受 handoff 原 scope（PoC level），可以跳过 §6 的 A.3/A.4/A.5/F.2 typed-errors + Step 9 跨版本验证，把工作量压回 ~6–8h；但代价是 v0.1 daemon 上线时这些缺陷会浮出。**本 plan 选完整版**。

---

## 10. References

- spec：`<ticket-dir>/.specs/2026-04-30-cc-sdk-route.md`
- happy 源码（参考实现）：`/tmp/happy-study/happy-cli/src/claude/sdk/`
- spec brainstorm 录：30+ 轮论证（A/C/D/E/F/G + cwd-gap + multi-turn 澄清）
- 实测 fixture：`/tmp/claude-resume-probe/stream1.ndjson` + `/tmp/claude-resume-probe/c840b5fb-…jsonl`
