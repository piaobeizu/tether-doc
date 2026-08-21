# Agent subprocess lifetime — who owns it, and what a reload actually destroys

> Status: **§6 accepted by the owner, 2026-08-21. Option F is approved to build;
> B+C is the accepted sequel and has NOT started. Nothing is implemented yet.**
> No production file was touched by this document. See §6.0 for exactly what was
> and was not decided — in particular **T and N are still unchosen**, and the one
> experiment that could overturn §6 has not been run (§7.2).
> Work item: tether#134. Read against tether `5f96f7f` (`#211`/`#212`/`#213` landed).
> Measurements taken 2026-08-21 against a purpose-built isolated daemon, never the owner's.
>
> **Revision, 2026-08-21 — tether#138, read against tether `4536b9c`.** §7.4's gap
> is half closed: **§2.8 measures a FLOOR for the resident cost of one live agent**
> (1.015 GiB per agent process tree; 2.6 MiB + ~0.5 MiB/session on the daemon
> side) using the three real agent launches the owner authorised. It is a floor,
> not a representative figure, and it is labelled as one everywhere it appears.
> §5-C, §6, §6.0 and §7.4 are updated; **N is still not chosen**, and §7.4 says
> why in three parts rather than being deleted. No code changed.
>
> **Revision, 2026-08-21 — tether#141, read against tether `9968aed`.** §2.8.4's
> `[I]` is now `[M]`: **a single-pid SIGKILL of the agent — the daemon's actual
> reaper — does reclaim all ~700 MiB of MCP children, in under a second, in both
> authorised trials.** New §2.9. But it happens by *stdin EOF and a parent-pid
> watchdog inside third-party servers*, not by kill scope, so §5-C's cap bounds
> resident memory **conditionally on a property of the MCP server set that tether
> neither enforces nor can observe** — and the same section proves the kill scope
> genuinely cannot reach a grandchild. §2.8.4, §5-C and §7.4 are updated;
> §2.8.3's `sessions/` disclosure is corrected. No code changed.

## How to read the claims

Every factual claim carries one tag:

- **[M] measured** — a command was run and its output is quoted or summarised here.
- **[R] read** — read out of the source at `5f96f7f`, with `file:line`.
- **[I] inferred** — a conclusion drawn from [M]/[R] facts, with the gap named.

A spec is not a test. Nothing here is verified by a green suite; the only green
suite involved was the compiler.

## The question on the table

tether#134 records that the agent subprocess is bound to a single chat
connection, so closing that connection kills the agent. The owner has three
questions: **who owns the agent**, **what the cap is**, and **whether the
in-flight turn is worth saving separately from the pending permission request.**

This document answers 1 and 3 with evidence and refuses to answer 2 with a
number, because the number needs a measurement I did not take (§7).

---

## 1. The current lifetime, hop by hop

### 1.1 The chain

Verified fresh at `5f96f7f`, not copied from the work item. [R]

| # | Site | What it does |
|---|---|---|
| 1 | `internal/server/wt_chat.go:255` | `ctx := wtsess.Context()` — the WebTransport session's own context, inside `serveChat` (`:253`) |
| 2 | `internal/server/wt_chat.go:324` | `att, err := reg.Attach(ctx, sid, providerName, wsID)` |
| 3 | `internal/session/attach.go:287` | `func (r *Registry) Attach(ctx context.Context, sid, providerName, wsID string)` |
| 4 | `internal/session/attach.go:406` | `r.spawnEntry(ctx, providerName, cfg, dec.Binding, spawnIfAbsent)` — ctx passed through unwrapped |
| 5 | `internal/session/registry.go:990` | `func (r *Registry) spawnEntry(ctx context.Context, …)` |
| 6 | `internal/session/registry.go:1113` | `sess, err := provider.Spawn(ctx, cfg)` |
| 7 | `internal/agent/claude_provider.go:161` | `cmd := exec.CommandContext(ctx, p.ccPath, args…)` |
| 7' | `internal/agent/opencode_provider.go:78`, `:472` | the same shape for the other provider |

There is **no `context.WithoutCancel` anywhere in production**. [M]

```
$ grep -rn "WithoutCancel" --include='*.go' .
internal/agent/subprocess_wedge_test.go:381: // TestGuardStdout_ExitsWhenTheSessionEndsWithoutCancellation — a session that
internal/agent/subprocess_wedge_test.go:384: func TestGuardStdout_ExitsWhenTheSessionEndsWithoutCancellation(t *testing.T) {
---END-WithoutCancel---
$ grep -rn "context\." --include='*.go' internal/ | grep -c .
604
---END-control---
```

Both hits are one **test function name**, not a call. The 604-hit positive
control proves the grep can see `context.` at all, so the two-hit result is a
real absence and not a broken pattern.

Three other call sites reach `spawnEntry` with a connection-scoped ctx and are
part of the same story: `attach.go:863` (the resume/reopen path),
`attach.go:1282`, and `registry.go:832` (inside `GetOrSpawnEntry`, declared at
`:786`). That last one is reachable in production only through `GetOrSpawn`
(`registry.go:1403` is its sole caller), and `registry.go:1398-1401` records that
`GetOrSpawn` itself has **no** production caller — so treat it as API surface,
not as a fourth live path. [R]

### 1.2 Line numbers in the work item that have moved

`#213` inserted 23 lines into `wt_chat.go` and 164 into `registry.go`, all of
them **below** the `wt_chat.go` citations and **above** the `registry.go` ones.
So the work item's `wt_chat.go` numbers are still exact and its `registry.go`
numbers are all off by the same amount. [M — `git show 2753316:<path>` vs the
working tree]

| Cited at `2753316` | Same text at `5f96f7f` | Delta |
|---|---|---|
| `wt_chat.go:255` — `ctx := wtsess.Context()` | `wt_chat.go:255` | **0** |
| `wt_chat.go:324` — `reg.Attach(ctx, …)` | `wt_chat.go:324` | **0** |
| `attach.go:287` — `Registry.Attach` | `attach.go:287` | **0** |
| `attach.go:406` — `spawnEntry(ctx, …)` | `attach.go:406` | **0** |
| `claude_provider.go:161` — `exec.CommandContext` | `claude_provider.go:161` | **0** |
| `registry.go:855` — `func … spawnEntry` | `registry.go:990` | **+135** |
| `registry.go:978` — `provider.Spawn(ctx, cfg)` | `registry.go:1113` | **+135** |
| `registry.go:1278-1287` — the tether#12 decision | **`registry.go:1414-1423`** | **+136** |
| `registry.go:1340-1346` — the tether#56 leak measurement | **`registry.go:1476-1482`** | **+136** |
| `registry.go:531` — `Entry.Subscribe` | `registry.go:569` | +38 |
| `registry.go:1306` — `Registry.evict` | `registry.go:1442` | +136 |
| `registry.go:1411` — `Registry.teardown` | `registry.go:1547` | +136 |
| (did not exist) — `Entry.backfill` | `registry.go:658` | new |

Anyone quoting this area again should re-derive the numbers; two of the three
files in the chain are being edited weekly.

### 1.3 What the two load-bearing comments actually say

**The decision (`registry.go:1414-1423`)** — this is not an oversight, it is
tether#12 written down:

> serveChat binds the agent subprocess to the client-connection context
> (exec.CommandContext with wtsess.Context()), and both providers close Events()
> when that context is cancelled … That single trigger covers BOTH "session
> ended" and "client disconnected", which is why no idle timer or grace period is
> needed — the connection context already bounds the subprocess lifetime
> (tether#12).

**The cost of removing it naively (`registry.go:1476-1482`)** — tether#56:

> Every session that ended the ORDINARY way — the overwhelming majority —
> therefore left behind a `[claude] <defunct>` zombie, a goroutine parked in
> exec.CommandContext's ctx watchdog, and an unclosed stdin fd, all held until the
> daemon itself exited. Measured on a live daemon at one of each per chat session.

### 1.4 The distinction that makes §5 tractable

Read carefully, those two comments are about **different things**, and the work
item conflates them by one step. [I, from [R] on `registry.go:1442-1450`,
`:1547-1578`, `:2133`]

- tether#56's fix is **`teardown` calling `e.sess.Close()`** — the reaper. It is
  reached from `fanOut`'s defer, i.e. once `Events()` has closed **and drained**.
- tether#12 supplies only the **trigger** that closes `Events()` for a client
  disconnect: the ctx cancel.

So "take the subprocess off the connection ctx and you get tether#56 back" is
true only if you remove the trigger *and supply nothing else*. Supply a
**daemon-owned cancel** for the same session and every hop after it —
`Events()` closes, `fanOut` returns, `teardown` reaps, `evict` unregisters — is
byte-for-byte the path it is today. That is what makes the sid-owned options in
§5 a change of *trigger*, not a rewrite of the reaper.

### 1.5 Confirmation that the reaper works today

After three chat connections came and went on the probe daemon: [M]

```
DAEMON=3650634 CHILDREN=0 ZOMBIES=0
```

No defunct children, no orphans. The tether#56 fix is live and effective. Any
option in §5 has to keep that number at zero.

---

## 2. What actually dies — measured

### 2.1 The harness

Rebuilt from scratch rather than inherited. Safety envelope first: **port 18991,
`HOME=/tmp/t134/home`, never `:443`, never `/root/.tether`, `--mcp-port 18992`**
(the default 8899 would have collided with the owner's live daemon). No
`pkill -f`/`pgrep -f` anywhere — the probe daemon's pid came from a file we
wrote, and was confirmed against `/proc/<pid>/cmdline` before any signal.

- `tether` built at `5f96f7f` with `GOWORK=off go build ./cmd/tether`. `web/dist`
  is not committed, so a stub `index.html` was created for the `//go:embed` and
  removed afterwards; the SPA is irrelevant here because the client is not a
  browser.
- A **stand-in agent** on `TETHER_CC_PATH`: a Python script whose stream-json
  shapes were copied from `internal/agent/fakecc_test.go` (`system/init` →
  `assistant` → `result/success`, nothing emitted before the first prompt,
  cwd-scoped resume markers). It never exits on its own — after the last prompt
  it goes back to blocking on stdin. **So a missing process was killed.**
- A **stand-in permission gate**: a *child of the agent* that POSTs to
  `/api/v1/permission/request` and blocks on the answer with a 65s client
  timeout — the topology and blocking shape of
  `internal/permission/cchook/hook_main.go.txt:20-50`. [R]
- A **WebTransport client** (Go, `webtransport-go v0.10.0`) that dials
  `/wt/chat`, opens the bidi stream, writes one prompt line, drains the daemon's
  per-envelope unidirectional streams, then closes. It needs
  `Origin: https://127.0.0.1:18991` (`internal/server/server.go:124` →
  `mux.go:374-390`) and a 60s `?ticket=` (`internal/auth/jwt.go:24`).
- A process census that walks `/proc` and skips its own pid, so it can never
  match its own command line.

**Deviation to know about:** the daemon ran with
`TETHER_NO_PERMISSION_HOOK=1`, so the real gate binary was neither compiled nor
injected into settings; the stand-in gate was spawned by the stand-in agent
instead. That changes *who spawns the gate's parent's parent*, not the process
topology under test.

### 2.2 The agent dies with the connection — reproduced

One connection, one prompt, then close: [M]

```
== census WHILE the connection is open
pid=3664526 ppid=3650634 state=S cwd=/tmp/t134/home/ws cmd=python3 /tmp/t134/fakeagent --print --output-format stream-json …
COUNT=1
== census AFTER the connection closed
COUNT=0
```

`ppid` is the daemon. The work item's headline is confirmed.

### 2.3 A reconnect respawns — and the positive control that this means something

Three connections on one real sid: [M]

| Connection | Result |
|---|---|
| 1 — fresh | agent pid `3670913`, spawned with `--session-id`; closed → `COUNT=0` |
| 2 — same sid, agent already gone | **new** agent pid `3673061`, spawned with `--resume 04efc9a0-…` |
| 3 — same sid, **overlapping** connection 2 | **no new pid**; census still shows only `3673061` |

Daemon log ordering is the same discriminator the work item names:

```
# connection 2 (respawn): prompt first, resolve second
INFO chat prompt received len=7
INFO serveChat: SessionID resolved sid=04efc9a0-… recovered=false
# connection 3 (adoption): resolve FIRST — the entry was already live
INFO serveChat: SessionID resolved sid=04efc9a0-… recovered=false
INFO chat prompt received len=9
```

Connection 3 is the **positive control**. Without it, "connection 2 respawned"
is compatible with "this code never reuses anything". With it, reuse is proven
to exist, so connection 2's respawn is caused by the agent being **absent**, not
by the code declining to look.

### 2.4 The finding that contradicts the work item: the reap is not prompt, and it is not deterministic

The work item, and `registry.go:1414-1423`, both read as if closing the
connection reaps the agent. It does — **eventually.** Measured reap latency from
the moment the client stops speaking, three arms, three trials each: [M]

| Arm | What the client does | Reap latency |
|---|---|---|
| `graceful` | `CloseWithError(0,"")`, then exit immediately | **0.03s, 43.21s, 0.03s** |
| `linger` | `CloseWithError(0,"")`, stay alive 1.5s, then exit | **1.54s, 1.54s, 1.54s** |
| `vanish` | never close; the client process is SIGKILLed | **41.18s, 41.18s, 41.47s** |

An earlier 5-trial run of the same `graceful` shape with a surviving gate child
gave `42.30s, 0.04s, 0.04s, 0.04s, 0.04s` — one long tail in five. [M]

Three things follow.

1. **The `linger` arm reaps at 1.54s ≈ the linger itself, not at the close.** So
   what cancels `wtsess.Context()` in this setup is the *QUIC connection going
   away*, not the WebTransport close capsule. [M for the observable; **[I] and
   unresolved** for which of "the capsule was never flushed" and "the server does
   not cancel on the capsule" is responsible — see §7.]
2. **A client that vanishes costs ~41s.** The daemon offers
   `MaxIdleTimeout: 90s`, `KeepAlivePeriod: 15s`
   (`internal/server/server.go:109-116` [R]); the Go probe's default offer is
   30s, and QUIC negotiates the minimum, so ~41s is 30s plus loss-probe slack.
   **[I]** A browser that offers a larger idle timeout would sit closer to the
   daemon's 90s. This is the phone-loses-signal case, i.e. this product's
   primary case.
3. **Even a graceful close was not observed 1 time in 3.** So the window is not
   "only for crashes".

⇒ **The status quo already grants a grace period of 0 to ~43 seconds, and its
length is a QUIC timeout that nobody chose.** The claim in
`registry.go:1414-1423` that "no idle timer or grace period is needed" is
accurate about *code* and misleading about *behaviour*: there is a grace period,
it is just inherited rather than specified. This is the single most important
input to the owner's decision, because it means §5's options are not "add a
grace period where there is none" — they are "replace an accidental one with a
chosen one".

And it has a user-visible consequence, confirmed independently in §3.1: **a
reload sometimes keeps your in-flight turn and sometimes kills it**, decided by
whether a UDP close got out. A user cannot learn a rule from that.

### 2.5 The permission gate survives its parent — and so does the request

One connection; the agent spawns the gate on its first turn; close the
connection; then census, then answer the request. [M]

```
== census WHILE the connection is open
pid=3682433 ppid=3680449 … /tmp/t134/fakeagent …        # agent, child of the daemon
pid=3682434 ppid=3682433 … /tmp/t134/hookprobe.py       # gate, child of the agent
DAEMON=3680449 CHILDREN=1 ZOMBIES=0

== census AFTER the connection closed
pid=3682434 ppid=1     … /tmp/t134/hookprobe.py         # gate SURVIVED, reparented to init
DAEMON=3680449 CHILDREN=0 ZOMBIES=0                     # agent gone

== POST /api/v1/permission/<id>/decide, 13s after the close
DECIDE-HTTP=204

== gate-process log
[hook] pid=3682434 ppid=3682433 start t=1787284379.179
[hook] status=200 body={"allow":true,"message":"probe decides"} t=1787284392.114
[hook] exit t=1787284392.114
```

Four facts, and the work item guessed the wrong branch on the first two:

1. **The gate process survives, structurally rather than by luck.** `Spawn` sets
   no `SysProcAttr`/`Setpgid` (`claude_provider.go:161-176` [R]), so
   `exec.CommandContext`'s default cancel is `Process.Kill()` against **one
   pid**. A grandchild is never in the kill scope. [R + M]
2. **`internal/permission/http.go:64-65`'s `r.Context().Done()` therefore never
   fires**, so `m.Decide(req.ID, false, "client disconnected")` is never called
   and the entry is **not** removed. The `204` is the proof: `Manager.Decide`
   returns false for an unknown id and `http.go:97-100` answers `404`
   (`manager.go:91-104` [R]). We got `204`.
3. **The only route out is the 60s timer** armed in `Manager.Add`
   (`manager.go:16` `Timeout = 60 * time.Second`, `:78-85` [R]) — 5s before the
   gate's own 65s client timeout. So the request outlives its agent by up to a
   minute.
4. **The decision reaches the gate and then goes nowhere.** The gate got
   `allow:true` and exited 0. Its exit code is read by the agent that spawned it,
   and that agent is dead. [M for the delivery; [I] for "nothing acts on it",
   from the gate contract at `hook_main.go.txt:45-49` [R].]

### 2.6 The reloaded page *does* get the request back

This is where the work item is most wrong, and it matters because it is wrong in
the direction that makes the bug worse, not milder.

Connection 1 raises a request and closes (agent dies, gate survives, request
still pending). Connection 2 reconnects on the same sid — the reload. The gate
is armed exactly once, so connection 2 creates no new request. [M]

```
=== CONN1
PROBE envelope kind=permission payload={"id":"9fbd397a7030d0df",…}
=== CONN2 — the reload
PROBE envelope kind=message    payload={"sessionId":"52a44d9e-…","type":"session_ready"}
PROBE envelope kind=permission payload={"id":"9fbd397a7030d0df",…}   ← the SAME id
BACKFILL-SEEN=yes

# daemon
INFO replayed outstanding envelopes to a new subscriber count=1
```

`Entry.backfill` (`registry.go:658-678`) reads `Registry.PendingBackfill`
(`registry.go:167`, wired at `mux.go:157`), which snapshots
`permission.Manager.Pending()` — **daemon-wide, not per-entry** [R]. So it does
not care that the reload got a brand-new `Entry`. The reload gets the request.

And the frontend keeps it now: `web/src/lib/store.ts:1038` makes `loadHistory`
*filter* rather than clear —
`s.pendingPermissions.filter(p => p.sessionId != null && p.sessionId === s.sessionId)`
[R]. (Team memory from tether#124 records `loadHistory` resetting it to `[]`;
that was true then and `#213` changed it. Anyone working from that memory should
re-read the line.)

Two caveats on this measurement, both honest:

- Connection 2 here **adopted a live agent** (no new pid; `SessionID resolved`
  logged before `chat prompt received`) because it landed inside the accidental
  window of §2.4. So this run demonstrates the backfill *and* the adoption at
  once. Outside the window the backfill still fires — it is subscription-time
  and entry-independent [R] — but it rides `subCh` and so is only forwarded
  after `Resolve` returns (`wt_chat.go:603-614`), which on the `--resume` path
  waits for `system/init`, which under `--input-format stream-json` waits for
  the first prompt. **I did not measure the outside-the-window case**, so the
  work item's "invisible until the user types" stands as [R], not [M].
- `internal/session/registry.go:599-609` already says a reload is not repaired
  by the backfill. That is right about *usefulness* and wrong about *delivery*:
  the replay does happen.

⇒ **The post-reload failure is not a missing prompt. It is a prompt that is
present, answerable, and inert.** By this repo's own standard —
"a notice a user has caught lying is one they stop reading"
(`wt_chat.go:592`) — that is the worse of the two failures.

### 2.7 Two producers, only one affected

`permission.Manager.Check` has exactly one caller,
`internal/mcp/gateway/gateway.go:82` [M grep]. Its ctx is the MCP tool call's,
not a chat connection's, and the thing waiting for the decision is the daemon
itself. So **MCP-sourced permission requests are already daemon-owned and
survive a chat reload with their consumer intact.** [I from [R]] Only the
agent-spawned gate path has the dead-consumer problem. Any fix must not treat
the two as one case.

### 2.8 The resident cost of one live agent — a FLOOR, and the daemon's share

Added 2026-08-21 by tether#138, read against tether `4536b9c`. This is §7.4, the
one input §5-C's cap N has, and §7.4 is rewritten rather than removed because
only half of it is now closed: **the floor is measured; the representative
figure is not, and it needs the owner's quota.**

#### 2.8.0 Why one number here is two numbers

The cap is about what has to stay resident per live session, and that splits into
a part tether owns and a part it only spawns:

- **the daemon's own share** per live chat session — free to measure, because the
  thing under the microscope is the daemon, so the §2.1 stand-in agent is a
  perfectly good load;
- **the agent process's own footprint** — costs real API quota, because a Python
  stand-in's footprint says nothing about the real provider's (that is exactly
  what §7.4 said).

They came out **a factor of ~400 apart** — 2.6 MiB against 1,040 MiB — which is
the finding.

#### 2.8.1 The daemon's share — free, 3 trials per point [M]

Harness as §2.1 (isolated `HOME`, high port, `--mcp-port` moved off 8899, no
`pkill -f`/`pgrep -f`, stand-in agent on `TETHER_CC_PATH`), on port **19181**
with `--mcp-port 19182`; the owner's live daemon was confirmed listening on
`:443` and `127.0.0.1:8899` at the time and was never touched. Goroutines were
counted from a `SIGQUIT` dump under `GOTRACEBACK=all` (`^goroutine \d+`
occurrences), which is terminal, so each count is its own daemon run; RSS is
paired *within* one run (baseline, then n live sessions) so run-to-run variance
cancels.

| Live chat sessions | goroutines (3 trials) | Δ daemon `VmRSS` (3 trials) | daemon fds | daemon OS threads |
|---|---|---|---|---|
| 0 | 20, 20, 20 (and 20, 20, 20 in the second batch) | baseline 18,696–18,988 kB | 9, 9, 9 | 7–9 |
| 1 | **46, 46, 46** | **+2652, +2576, +2712 kB** | 12, 12, 12 | 10–13 |
| 3 | **74, 74, 74** | **+3588, +3384, +3920 kB** | 18, 18, 18 | 11–13 |

- **Goroutines: +14 per live session, exactly, with zero spread over 3 trials at
  each point.** The other +12 is a one-off. Diffing the dumps by `created by`
  site gives, per session: 2 × `agent.(*ClaudeCodeProvider).Spawn` (`readLoop`
  and `guardStdout`), 1 × `session.(*Registry).spawnEntry` (`fanOut`), 1 ×
  `server.serveChat` (`readPrompts`), 1 × `buildMux.handleWTChat.func10`, 1 ×
  `os/exec.(*Cmd).Start`, and 7 in the transport (`quic-go.(*Conn).run`,
  `baseServer.handleInitialImpl`, `http3.rawConn.handleControlStream`,
  `webtransport-go.(*Server).Serve`, 3 × `ServeQUICConn`, `newSession`) — 14.
  The remaining +12 is `runtime.gcBgMarkStartWorkers`, which equals GOMAXPROCS
  on this 12-core box, appears when the first session makes the heap grow enough
  to start a GC, and does **not** repeat.
  **This decomposition was derived from the 1-session dumps and its consequence
  — `20 + 12 + 3×14 = 74` — written down before the 3-session dumps were read.
  All three came back 74.**
- **RSS: not linear, and small.** First session ≈ **+2.6 MiB** (median 2652 kB);
  sessions 2 and 3 cost ≈ **+0.5 MiB each** (medians: (3588−2652)/2 = 468 kB;
  worst pairing of the observed extremes, 672 kB). Most of the first session's
  cost is the one-off heap growth the 12 GC workers belong to.
- **fds: +3 per session, exactly linear** (9 → 12 → 18). Consistent with cc's
  stdin/stdout pipes plus one.
- **OS threads are not a per-session constant** — they are the Go scheduler's
  business and moved 7↔13 independently of n. Reported so nobody derives a rate
  from them.

**Positive control on the dump parser, and why it is not optional.** The three
frames `fanOut` / `readLoop` / `guardStdout` are 0 at 0 sessions and exactly n
at n sessions, in every trial. That control exists because the first attempt at
this measurement produced a **+12 goroutine delta for a session whose agent had
not started at all** — the stand-in was not executable, `Spawn` returned
`permission denied`, and the transport goroutines alone moved the count. So a
run is admissible only when the process census sees exactly n stand-in processes
*and* those three frames appear exactly n times. Both were true for all six runs
reported above. Without that gate this table would have had a plausible, wrong
number in it.

#### 2.8.2 The real agent's footprint — a FLOOR, 3 launches [M]

**Three** real provider launches, the number the owner authorised, each with the
shortest possible prompt — a single `.`. No attachment, no tool call, one turn,
then measured at rest. Provider `2.1.237` (334,715,184 bytes on disk), resolved
the way `internal/server/cc_resolve.go:19-22` resolves it [R] — `TETHER_CC_PATH`
unset, so the `PATH` lookup, landing on
`/root/.local/share/claude/versions/2.1.237`.

Spawned with **the daemon's own argv, verbatim** from
`internal/agent/claude_provider.go:135-159` — `--print --output-format
stream-json --input-format stream-json --verbose --include-partial-messages
--permission-mode default --session-id <minted uuid>` — with `IS_SANDBOX=1`
(`buildEnv`'s rule as root, `:217-223`) and cwd set to a probe workspace.
Measured at rest, i.e. after `result/success` arrived and the process went back
to blocking on stdin.

| | agent process alone | whole process tree (10 processes) |
|---|---|---|
| `VmRSS` | 353,632 / 342,316 / 321,736 kB → **median 334 MiB** | 1,064,540 / 1,056,700 / 1,092,268 kB → **median 1.015 GiB** |
| `VmHWM` (peak) | 370,484 / 367,692 / 331,872 kB → median 359 MiB | 1,276,384 / 1,228,460 / 1,199,412 kB → median **1.171 GiB** |
| fds | 31 / 32 / 32 (`ls /proc/<pid>/fd \| wc -l`: 31 / 31 / 32) | 170 / 170 / 170 — **spread 0** |
| OS threads | 16 / 13 / 16 | 83 / 76 / 79 |
| CPU at rest | — | **1.30% / 0.33% / 0.73%** of one core, 30 s window |

CPU at rest is two `/proc/<pid>/stat` `utime+stime` reads summed over the tree,
30 real seconds apart — not `top`'s instantaneous figure. Its own positive
control: the same meter reads **0.00%** on a `sleep` and **92.3%** on a
deliberate spin loop, so 0.33–1.30% is a real small number and not a broken
meter. The three-fold spread across trials is why the range is reported and no
mean is offered — §2.4's lesson applied to a second quantity.

**The tree, identical in shape in all three trials.** The agent is not one
process. It spawned four stdio MCP servers as **nine** child processes. Where
each of the four is configured was not enumerated — `settings.json`'s
`mcpServers` names exactly one server and it is *http* (the tether loopback), so
the stdio set comes from the user/plugin config [M for the process list, [I] for
which file declares each]:

```
agent                                          RSS 353,632 kB  fd 31  thr 16
├─ uv tool uvx --from mcp-atlassian …           RSS  76,492 kB
│  └─ …/bin/python …                            RSS 128,104 kB
├─ npm exec chrome-devtools-mcp@latest          RSS  99,644 kB
│  └─ sh -c chrome-devtools-mcp                  RSS   1,844 kB
│     └─ chrome-devtools-mcp                      RSS 140,428 kB
│        └─ node …/chrome-devtools-mcp/…           RSS 135,468 kB
├─ …/polyforge/1.1.7/bin/polyforge              RSS  30,368 kB
└─ node /usr/bin/codegraph serve --mcp          RSS  44,856 kB
   └─ …/codegraph-linux-x64/…                    RSS  53,704 kB
                                       tree total 1,064,540 kB   (trial 1)
```

Children alone: 710,908 / 714,384 / 770,532 kB → **median 698 MiB, about two
thirds of the tree**, and it is **per agent process** — nothing here is shared
between two live sessions.

**Why this is a floor and not the number.** The prompt was one character; the
transcript is one exchange; no file was attached and no tool ran. Everything a
real session accumulates — a long transcript, file contents, tool results — adds
to the agent process's own RSS, and none of it was present. The one-character
turn nevertheless reported `input_tokens` 11,985–12,171 and
`cache_read_input_tokens` 112,504–499,919, i.e. the system prompt and the tool
definitions are already loaded at the floor; that is the part a representative
measurement would *not* change much. **A representative figure is therefore
strictly larger than these numbers, and by an amount this measurement cannot
bound.**

**What a representative measurement would need, and why it needs separate
approval.** Several launches driven through a handful of real turns *with* tool
calls and file reads, sampled at rest between turns — tens of minutes of model
time per launch instead of the 85–170 s these three took (the quota actually
spent: `output_tokens` 1,306 / 1,772 / 6,541, the largest including 3,499
thinking tokens). And it has to be repeated against a **representative MCP
configuration**, because 698 of the 1,040 MiB is MCP servers and that set is a
property of the machine, not of tether.

#### 2.8.3 Deviations, disclosed

- **The daemon was not in the loop for the real-agent arm.** `buildEnv` is
  `os.Environ()` plus `IS_SANDBOX` (`claude_provider.go:217-223` [R]), so the
  agent inherits the daemon's `HOME` — there is no way to give the daemon an
  isolated `HOME` and the agent the real one. A daemon run faithful enough to
  authenticate the real provider would have had to run with the owner's real
  `HOME`, putting `~/.tether` and `~/.claude/settings.json` in the daemon's own
  write path. Replicating the spawn instead kept both out of it. What is
  replicated (argv, `IS_SANDBOX`, cwd) is read from source; what is skipped is
  the daemon's `cfg.Env` extra, which is `TETHER_DAEMON_PERM_ENDPOINT`
  (`session/registry.go:999-1006` [R]) — and its absence is *safe by design*:
  with it unset the PreToolUse hook exits 2 before opening a socket
  (`cchook/hook_main.go.txt:23-26` [R]), so a tool call could not have reached
  the owner's live daemon. No trial produced any hook output on stderr, which
  that path always writes, so no tool call occurred in any of the three.
- **The provider did connect to `http://127.0.0.1:8899/mcp`**, the loopback MCP
  endpoint the owner's `settings.json` names, i.e. to the owner's live daemon.
  That is a tool listing, it is read-only, and it is what a daemon-spawned agent
  does — so it is fidelity, not contamination. Named because it is a real
  interaction with a process this work item did not own.
- **The provider wrote its own transcript.** Accounted for exactly, by listing
  directory names only and opening no file: **one new project directory**
  (`projects/-root-…-probe138-home-ws`, 7 entries, keyed on the probe's cwd,
  37 project directories in total afterwards) and **zero additions under
  `sessions/`** — the latter checked by matching the three session ids this probe
  minted against the 146 entry names, because "the name does not contain
  `probe138`" would prove nothing in a tree whose entries are named by session
  id. Nothing pre-existing in either tree was written, renamed, truncated or
  deleted, and the new directory was **left in place** — deleting it would itself
  have been a write to that tree. (Incidental confirmation of the fake-agent
  contract's fact ①: all three ids came back on `result` verbatim, so
  `--session-id` is still adopted at `2.1.237`.)
- The stand-in arm ran with `TETHER_NO_PERMISSION_HOOK=1` and
  `--skip-mcp-inject`, as in §2.1.

#### 2.8.4 The reaping caveat — and why §1.5's census cannot see it

All three trials were torn down by **closing stdin**, the ordinary end of a
stream-json session, and afterwards every one of the 30 recorded pids across the
three trees was gone (checked by recorded pid plus `/proc/<pid>/stat` start time,
never by pattern; a positive control confirmed the checker can see a live pid).
So on *that* path the agent shuts its MCP children down itself and nothing leaks.

**The daemon does not tear down that way.** Its reaper is
`exec.CommandContext`'s default cancel — `Process.Kill()` against **one** pid,
with no `SysProcAttr`/`Setpgid` (§2.5 fact 1 [R]) — which is precisely the scope
that left the permission gate alive and reparented to init in §2.5 [M].

**This paragraph used to end in an `[I]`** — "under the daemon's real reaper the
agent's ~698 MiB of MCP children are therefore orphaned rather than reclaimed".
**tether#141 measured it and that inference is WRONG: they are reclaimed, in
under a second. See §2.9.** The reasoning was sound about kill *scope* and wrong
about *outcome*, because it stopped one hop short: nothing has to be in the kill
scope for a stdio server to die — the dead agent's fds close, the servers' stdin
pipes hit EOF, and they exit themselves.

What survives from this paragraph unchanged is the part about the meter:
§1.5's `CHILDREN=0 ZOMBIES=0` **is** blind to those processes, and that is not a
guess — §2.5's own output shows the surviving gate process listed at `ppid=1`
**on the same line as `CHILDREN=0`**, so that counter demonstrably counts the
daemon's *direct* children only, and the agent's MCP servers are grandchildren of
the daemon. §2.9 explains why that blindness still matters even though the
outcome turned out benign.

### 2.9 Single-pid SIGKILL **does** reclaim the MCP children — by EOF, not by kill scope

Added 2026-08-21 by tether#141, read against tether `9968aed`. This closes
§2.8.4's `[I]` and §7.4's third sub-bullet. **Two** authorised real launches, one
character of prompt each.

#### 2.9.0 The answer, first

**All ~700 MiB comes back.** In both trials, every one of the agent's nine
descendants was gone within **1 s** of a single-pid SIGKILL against the agent, and
the resident memory they held went with them. Trial 2 additionally polled at 50 ms
resolution, which puts the slowest of the nine at **0.866 s**; trial 1 had no fine
poll, so it is only bounded to `(0.2 s, 1.0 s]` — consistent, and stated as a
bound rather than borrowed from trial 2. [M]

| | trial 1 | trial 2 |
|---|---|---|
| tree at rest, before the kill | **10 processes**, `VmRSS` **1,091,388 kB** | **10 processes**, `VmRSS` **1,063,384 kB** |
| of which the agent process | 297,468 kB | 296,108 kB |
| of which the 9 descendants | **793,920 kB** | **767,276 kB** |
| still alive at **t+0.2 s** | 3 procs, **345,540 kB** | 3 procs, **322,024 kB** |
| still alive at t+1 s | **0**, 0 kB | **0**, 0 kB |
| still alive at t+5 / 15 / 60 / 120 s | 0, 0, 0, 0 | 0, 0, 0, 0 |

**The two trials agree**, on the count, on the shape of the tree, and — see
§2.9.3 — on *which three* processes are the slow ones. Per §2.4's lesson two
trials was the floor, not a formality; they did not disagree, so there is no
disagreement to report.

#### 2.9.1 What the arm did, and the three things it deliberately did not do

The reaper being reproduced is `exec.CommandContext`'s default cancel:
`cmd.Process.Kill()`, i.e. `kill(<one pid>, SIGKILL)`, with `Spawn` setting no
`SysProcAttr`/`Setpgid` (`claude_provider.go:161-176` [R]). So the arm is
`os.kill(agent_pid, SIGKILL)` and **nothing else**:

- **not** the process group — that is not what the daemon does, and it would have
  answered a different question;
- **not** a stdin close first — that is the *graceful* path §2.8.4 already
  measured (30/30 gone), and mixing it in would have conflated the two;
- **not** a pattern match, ever. Every process was recorded before the kill as
  `pid` + `cmdline` + `/proc/<pid>/stat` field 22 start time, and every later
  observation matched on **pid + start time**. This machine runs the owner's live
  sessions with their own MCP children; a pattern would have swept those in, and
  pids get reused. Nothing was signalled that was not both in a file this probe
  wrote and confirmed live against `/proc/<pid>/cmdline` first.

The agent was measured **at rest** — after `result/success`, back to blocking on
stdin, with a draining thread on its stdout so it could never be blocked on a
full pipe while being called idle.

#### 2.9.2 The negative control — the kill scope really is one pid

`SURVIVORS=0` from a checker that cannot see a survivor looks exactly like
`SURVIVORS=0` from a clean reap. So the same driver, the same census, the same
sampler and the same kill were run first against a **stand-in topology**: a
parent that spawns four children on dedicated stdin pipes, three well-behaved
(one of them with a grandchild of its own) and one **deliberately deaf to EOF**
— it `dup2`s `/dev/null` over fd 0, so its copy of the pipe read end is closed
and EOF can never reach it. Twice: [M]

```
>>> os.kill(51110, SIGKILL)  -- single pid, no process group, stdin NOT closed
  pid=51111 gone    srv-a-wellbehaved
  pid=51112 gone    srv-b-nested
  pid=51115 gone    srv-b-nested-gk          <- grandchild, also EOF
  pid=51113 gone    srv-c-wellbehaved
  pid=51114 alive   ppid_now=1  rss_now=12324   srv-d-STUBBORN
  ALIVE=1 ZOMBIE=0 GONE=5  at t+0.2s AND at t+1s, +5s, +15s, +60s, +120s
```

Three things this pins down, and none of them is decoration:

1. **The sampler can see a survivor** — so §2.9.0's zeroes are real zeroes.
2. **The kill scope is exactly one pid.** A child that does not choose to exit
   is *not* killed by this signal; it is reparented to `ppid=1` and keeps its
   RSS indefinitely. §2.8.4's structural reading was right.
3. Therefore the real agent's children died **because they chose to**, not
   because the kill reached them. That distinction is the whole content of
   §2.9.4.

#### 2.9.3 Per server, with the latency — because "gone" needs a *when*

The work item asked for per-server reporting rather than an aggregate, and it was
right to: the four servers do not behave the same. Trial 2 additionally polled
every 50 ms from the instant of the signal, so the disappearance is a latency and
not one in/out bit against a fixed wait — §2.4's lesson applied to a third
quantity. Grouped by the MCP server each process belongs to: [M]

| MCP server | process | `VmRSS` before (t1 / t2) | gone at (t2, ±50 ms) |
|---|---|---|---|
| **polyforge** (stdio, connected) | `…/polyforge/1.1.7/bin/polyforge` | 30,336 / 29,480 | **0.054 s** |
| **codegraph** (stdio, connected) | `node /usr/bin/codegraph serve --mcp` | 45,332 / 45,324 | **0.054 s** |
| | └ `…/codegraph-linux-x64/node --liftoff-only …` | 54,184 / 54,676 | **0.055 s** |
| **chrome-devtools** (stdio, pending) | `npm exec chrome-devtools-mcp@latest` | 158,512 / 157,412 | **0.160 s** |
| | └ `sh -c "chrome-devtools-mcp"` | 1,980 / 1,884 | **0.106 s** |
| | └ └ `chrome-devtools-mcp` | 161,664 / 160,040 | **0.055 s** |
| | └ └ └ `node …/telemetry/watchdog/main.js --parent-pid=<the one above>` | 152,708 / 151,544 | **0.866 s** ← slowest |
| **atlassian** (stdio, pending) | `uv tool uvx --from mcp-atlassian mcp-atlassian` | 68,276 / 45,884 | **0.413 s** |
| | └ `…/bin/python …/bin/mcp-atlassian` | 120,928 / 121,032 | **0.413 s** |
| *(the agent itself)* | the provider, `2.1.237` | 297,468 / 296,108 | 0.002 s (it was the target) |

A fifth server, **github**, is declared and reported `"status": "failed"` in the
`system/init` line in both trials, so it contributed no process. The four that did
are the same four §2.8.2 found, in the same nine-process shape. [M]

**The three processes alive at t+0.2 s are the same three in both trials**: the
`mcp-atlassian` pair and the `chrome-devtools-mcp` telemetry watchdog — 345,540 kB
and 322,024 kB respectively, i.e. **about a third of the tree is still resident
one fifth of a second after the kill**. That is the number a cap implementation
has to care about, not the t+1 s zero:

- **[I]** a cap that evicts and immediately admits a replacement will briefly hold
  roughly (cap + 1) agents' worth of memory, for about a second. The gap named:
  nothing here measured back-to-back eviction and admission; this is arithmetic on
  the observed decay, not a measurement of it.

#### 2.9.4 Why this is a **conditional** yes, and what the condition is

The mechanism is **not** the kill. §2.9.2 proves the signal reaches one pid. What
kills the children is what happens *after* the agent dies: the kernel closes the
dead agent's fds, which are the write ends of each server's stdin pipe, the
servers read EOF, and they exit. [I — from [M] on both arms: the same signal that
cleared the real tree left the EOF-deaf stand-in child alive for 120 s. Gap named:
no `strace`, so "it exited on EOF" is inferred from the contrast, not observed at
the syscall.]

And **at least two different third-party mechanisms** are doing that work, which
is the reason to state the yes conditionally:

- the fast six (54–160 ms) are consistent with a direct read of a closed stdin;
- `mcp-atlassian`'s pair goes at 413 ms — the `uv` wrapper and its python child
  hold the *same* pipe, and they go together;
- the `chrome-devtools-mcp` **telemetry watchdog** goes last, at 866 ms, and it is
  the one process in the tree whose own argv says it is not watching stdin at all:
  `--parent-pid=<the chrome-devtools-mcp pid>`. It exits because it notices its
  parent is gone, on its own polling period. [R on the argv, [M] on the latency.]

⇒ **The reclamation is a property of the MCP server set, not of tether.** Every
server on this machine happens to shut down on its own when the agent dies, by one
of two mechanisms it chose for itself. tether does not require that, does not
verify it, and — per §2.8.4 — cannot even see it, because these are the daemon's
grandchildren and §4's invariant 3 (`CHILDREN=0 ZOMBIES=0`, the work item calls it
§4.3) counts direct children only. One EOF-deaf or slow-draining server in
someone's config, and the stand-in arm of §2.9.2 *is* what the daemon's reaper
produces: an orphan at `ppid=1` holding its RSS until the box reboots, invisible
to every counter this spec has.

#### 2.9.5 The contrast with §2.8.4's stdin-close arm

Same tree, same census, two different teardowns:

| | §2.8.4 (tether#138) | §2.9 (tether#141) |
|---|---|---|
| what was done | **closed the agent's stdin** | **SIGKILL, one pid; stdin left open** |
| who shuts the servers down | the agent, orderly, while alive | nobody — each server notices for itself |
| is this what the daemon does? | **no** | **yes** — `exec.CommandContext`'s default cancel |
| recorded pids surviving | 0 of 30 (3 trials) | **0 of 20** (2 trials) |
| how long it took | not measured per process | **≤ 0.87 s**, per process in §2.9.3 |

The step the difference comes from is **whether the agent gets to run any shutdown
code at all**, and the finding is that on this machine it does not need to. The
two arms agree on the outcome and disagree completely on the mechanism — which is
exactly why measuring the graceful arm did not answer this question.

#### 2.9.6 Deviations and disclosures

- **The daemon was not in the loop**, for the same reason §2.8.3 gives: `buildEnv`
  is `os.Environ()` plus `IS_SANDBOX` (`claude_provider.go:217-223` [R]), so the
  agent inherits the daemon's `HOME`, and a daemon faithful enough to let the real
  provider authenticate would have had to run with the owner's real `HOME` —
  putting `~/.tether` in its write path. The spawn was replicated instead: argv
  verbatim from `claude_provider.go:135-159`, `IS_SANDBOX=1`, cwd a probe
  workspace, `TETHER_DAEMON_PERM_ENDPOINT` explicitly unset (so the PreToolUse
  hook exits 2 before opening a socket, `cchook/hook_main.go.txt:23-26` [R] — and
  both trials produced **0 bytes of stderr**, which that path always writes, so no
  tool call occurred in either). **No daemon was started and no port was bound by
  this work item at all**, which is a stronger statement than "a high port was
  used". The owner's daemon was `pid=1912269` on `:443` and `127.0.0.1:8899`
  before and after, with an unchanged start time — never restarted, never bound
  over. [M]
- **Why the daemon's absence does not weaken the result.** The quantity under test
  is whether a grandchild of the daemon survives `kill(agent_pid, SIGKILL)`. The
  agent's parent is not in that causal chain: the servers' stdin pipes are created
  by the agent, and `Process.Kill()` is the same syscall from any parent. What a
  real daemon would add — `TETHER_DAEMON_PERM_ENDPOINT`, and its own pipe on the
  agent's stdin/stdout — touches the agent's own fds, not its children's.
- **The provider inherited this session's own env**, including `CLAUDECODE`,
  `CLAUDE_CODE_ENTRYPOINT`, `CLAUDE_CODE_SESSION_ID` and six more `CLAUDE*` keys,
  because `buildEnv` is `os.Environ()` and the driver's own parent is a session of
  the same provider. A daemon would not have those. Named because it is a plausible
  cause of a config difference, and one showed up: see the next point.
- **No connection to `127.0.0.1:8899` was observed.** §2.8.3 disclosed that the
  provider connected read-only to the owner's live daemon's MCP loopback. In these
  two trials the `system/init` line's `mcp_servers` list contains
  `plugin:polyforge:polyforge`, `atlassian`, `chrome-devtools`, `github`,
  `codegraph` — **no http/tether entry at all** [M]. So this probe has no evidence
  of that connection, and did not otherwise look for it (no packet capture). This
  is reported as a difference, not as a correction of §2.8.3: the env above and
  any config change since are both unexcluded explanations.
- **The provider wrote to its own config tree, and it was left in place.**
  Accounted for by a full before/after listing of **names**, plus `stat` metadata
  on the `sessions/` entries — **no file in either tree was opened**, nothing was
  copied, and nothing was written, renamed, truncated or deleted by this probe.
  `projects/`: 37 → 38, the one addition being `-root-tether141lab-ws`, keyed on
  the probe's cwd, exactly as §2.8.3 describes. `sessions/`: **146 → 148, the
  additions being `58058.json` and `72081.json`** — the probe's two agent pids.
  Deleting either would itself have been a write to a read-only tree, so both were
  left alone.
- 🔴 **§2.8.3's `sessions/` disclosure is corrected.** It claims "**zero additions
  under `sessions/`** — the latter checked by matching the three session ids this
  probe minted against the 146 entry names, because 'the name does not contain
  `probe138`' would prove nothing in a tree whose entries are named by session
  id." **That tree's entries are not named by session id.** All 148 entries are
  `<pid>.json`, and none of the 148 contains a session id [M]. So the check was
  keyed on a field that does not appear in the data and could not have detected an
  addition whatever happened; the 146 it quotes is the same 146 this probe measured
  *before* its own run, so §2.8's own additions, if any, are inside it. The
  correct form of the check is the full before/after name diff above. `stat` also
  shows these are **live** files — one unrelated entry's mtime moved during this
  probe's own run — so `sessions/<pid>.json` is per-process state written by
  whatever instance is running, not a per-run transcript, and the count is
  therefore not a stable baseline for anyone else's before/after either. The
  general shape is worth keeping: **a negative check has to be keyed on the field
  the data is actually keyed on, or it is a green light wired to nothing** — the
  same failure mode as §2.8.1's "+12 goroutines for an agent that never started".
- Two trials, not three. §2.4's lesson is that one is not enough; the work item
  authorised two, they agreed, and the stand-in arm (which costs nothing) was run
  twice as well.

---

## 3. The two consequences, separated

### 3.1 (a) The in-flight turn

**Lost, non-deterministically.** [M §2.2, §2.3, §2.4, §2.6]

- If the reload's reconnect lands *after* the reap, the agent is gone. `--resume`
  restores the conversation from disk; it does not restore a turn that was being
  generated. The tokens that had already arrived are on disk
  (`emitSegments` writes to `HistoryStore` before every `broadcast` — team
  memory, tether#119); the rest were never produced.
- If it lands *inside* the accidental 0–43s window, the entry is still live, the
  reconnect **adopts** it, and the turn keeps going. Measured (§2.3 connection 3,
  §2.6 connection 2).

The user-facing shape is therefore not "reload kills your answer" but "reload
kills your answer unless it doesn't". No message is shown either way.

### 3.2 (b) The pending permission request

**Not lost. Delivered. Inert.** [M §2.5, §2.6]

| Sub-claim in the work item | Measured outcome |
|---|---|
| the gate dies with the agent | **no** — it survives, reparented to init |
| `r.Context().Done()` fires and deletes the entry | **no** — it never fires |
| the request is dropped from `pending` | **no** — `decide` → `204` at +13s; only the 60s timer removes it |
| a reload cannot get the request back | **no** — the backfill delivers it, and the store now keeps it |
| answering it accomplishes anything | **correct — and it is the only sub-claim that survives** — the gate gets `allow:true` and exits into a dead parent |

### 3.3 The answer to the owner's question 3

**They are not separable, and (b) reduces to (a).**

The work item's candidate for (b) — "move ownership of the request from the
agent process to the daemon, together with its timeout, and let the agent die" —
is **already true in every respect that matters**: `permission.Manager` is
daemon-side, the timeout is daemon-side (`manager.go:16`), the request outlives
the agent by up to 60s, and it is even re-delivered to the next client. Building
that would buy nothing.

What is missing is not ownership of the *request*. It is a **consumer for the
decision**. The consumer is the agent's own gate subprocess exit code, and that
consumer is meaningless once the agent is dead. So:

- Any fix that makes a post-reload permission decision *mean something* requires
  the agent to be alive ⇒ that is exactly problem (a).
- The only thing worth doing for (b) *alone* is **negative**: stop showing an
  answerable prompt for a session whose agent is gone. That is a small, cheap,
  independent change (§5, option F) and it is worth doing whatever is decided
  about (a), because today the UI invites an action that does nothing.

⇒ Splitting this into two work items is reasonable, but not along the line the
work item drew. The split is **"choose the lifetime" (large)** and **"stop
lying about dead requests" (small)** — not "save the turn" and "save the
request".

---

## 4. What any option has to keep true

Extracted from the two comments in §1.3, plus §1.5. An option that cannot answer
all five is not a candidate.

1. **"Session ended" is reaped.** The agent exits on its own → `Events()` closes
   → `fanOut` returns → `teardown` → `evict`. *(Note: this case never depended on
   the connection ctx.)*
2. **"Client disconnected" is reaped**, eventually and boundedly.
3. **Zero defunct children, zero parked ctx-watchdog goroutines, zero unclosed
   stdin fds after a session ends** — §1.5's `CHILDREN=0 ZOMBIES=0`.
4. **`cmd.Wait` still runs only after every read of the process's stdout has
   completed** — the os/exec precondition argued at `registry.go:1498-1527`. Any
   new reaper must go through `teardown`, not call `Close` from somewhere new.
5. **Daemon shutdown reaps everything**, since a session with no connection has
   nobody else to notice.

---

## 5. Options, each with its reaping story

### A. Connection-owned — the status quo

- **Reaps it:** the ctx cancel, i.e. whenever the transport notices.
- **§4:** all five, today. [M §1.5]
- **Misfire mode:** the transport does not notice → **~41s of extra life,
  measured** (§2.4), then reaped by the QUIC idle timeout. Bounded by
  `MaxIdleTimeout: 90s`.
- **Cost to build:** zero.
- **What it costs the user:** the in-flight turn, non-deterministically; and an
  inert permission prompt after a reload.
- **Honest summary:** this is not "no grace period". It is "an unspecified grace
  period of 0–43s, and a coin flip visible to the user".

### B. sid-owned with idle eviction — *recommended, see §6*

The agent's ctx becomes a **per-session** ctx the `Registry` owns (created in
`spawnEntry`, cancel stored on the `Entry`). `serveChat` stops being the owner.
A reaper cancels it.

- **Reaps it:** an explicit reaper, on `(no subscribers on the Entry) AND (no
  turn in flight) AND (idle for T)`. Both terms already exist:
  `Entry.Unsubscribe` (`registry.go:682`) knows when the last subscriber leaves,
  and `Entry.turnsInFlight` / `clearTurns` (`registry.go:535-553`) is the
  in-flight signal `teardown` already consults. Cancel → `Events()` closes →
  `fanOut` → `teardown` → `evict`: **the existing reaper, unchanged.**
- **§4:** (1) unchanged — never depended on the ctx. (2) becomes the reaper's
  job; this is the new obligation and the one that must be pinned by a test whose
  failure is the tether#56 leak. (3) preserved because `teardown` is untouched.
  (4) preserved for the same reason. (5) needs an explicit cancel-all on shutdown
  — new, and one line's worth of design, several lines' worth of test.
- **Misfire mode:** the reaper never fires ⇒ **exactly tether#56, one leak per
  abandoned session, until the daemon exits.** This is why B must ship *with* C's
  cap, not after it: the cap converts an unbounded leak into a bounded one.
- **Cost to build:** the ctx re-parenting is small. The reaper, the timer
  bookkeeping, the shutdown path and the tests that make the reaper's absence
  fail loudly are the real work. Call it the largest of these options short of D.
- **The number T:** must be *chosen*, and choosing it is now easier, not harder,
  because §2.4 says the honest baseline is not 0 but "up to 43s, sometimes".
  Anything ≥ that is not a new risk class, it is the same one named out loud.

### C. sid-owned with a hard cap

B, plus a maximum number of simultaneously live agents; over the cap, evict the
longest-idle session with no subscribers and no turn in flight.

- **Reaps it:** the cap, in addition to B's idle timer. It is the *backstop*, so
  its value only has to be safe, not optimal.
- **§4:** as B.
- **Misfire mode:** the cap evicts a session a user is about to come back to ⇒
  they get `--resume` instead of adoption, i.e. today's behaviour. **A misfiring
  cap degrades to the status quo, which is what makes it the right backstop.**
- **Cost to build:** small *on top of* B; meaningless without B.
- **On the number:** the owner's daemon has **92** entries under its sessions
  directory [M — `ls -1 | wc -l`, no file read] and the transcript store has 97
  files across 36 project directories [M]. **That is history, not concurrency,
  and it does not constrain the cap.** The cap needs the resident cost of one
  live agent, and §2.8 now measures a **floor** for it: **1.015 GiB resident per
  live agent tree** (334 MiB of it the agent process itself, ~698 MiB its MCP
  server children), against **2.6 MiB for the first session and ~0.5 MiB per
  session after that** on the daemon side. **The daemon is not the constraint;
  the agent is, by a factor of ~400.** Two consequences for how N is
  written down:
  - **N is not a constant, it is a memory budget divided by that floor.** At the
    floor, a host that reserves *M* GiB for live agents supports N ≈ M / 1.02, and
    the floor only moves up. On the machine these numbers came from (48,168 MiB
    total, 35,314 MiB available at the time [M — `free -m`]) that is still tens; on
    a 2–4 GiB host it is 1–3. Expressing the cap as a reserved-memory figure
    rather than an integer is what makes it a safe backstop on hosts nobody has
    measured — which is exactly what this option claims to be.
  - **The cap does bound resident memory — conditionally.** This was open until
    tether#141 measured it (§2.9): a single-pid SIGKILL of the agent, which is
    exactly the daemon's reaper, reclaimed **all** of the ~700 MiB of MCP
    grandchildren in **under a second**, in both trials. So an eviction really
    does give the memory back and the cap is a real backstop. Three riders,
    all load-bearing for whoever writes the cap:
    1. **The condition is a third-party property.** Nothing was killed except one
       pid; the servers exited *themselves*, six of them on stdin EOF and one
       (`chrome-devtools-mcp`'s telemetry watchdog) on a parent-pid poll. tether
       does not require this, cannot enforce it, and per §2.8.4 cannot see it —
       §4's invariant 3 counts direct children only. §2.9.2's stand-in shows what
       one EOF-deaf server would produce: an orphan at `ppid=1` holding its RSS
       until the box reboots. **If the cap is documented as a memory backstop, that
       caveat belongs in the same sentence.**
    2. **Reclamation is not instantaneous.** About a third of the tree
       (322–346 MiB) was still resident 0.2 s after the kill. A cap that evicts
       and immediately admits will briefly hold ~(N+1) agents' worth.
    3. **It is still a floor** (§2.8.2), so N is still a reserved-memory figure
       divided by a number that only moves up — not an integer.

### D. Explicit start/stop

The user (or the UI) starts and stops an agent; connections attach to whatever
is running.

- **Reaps it:** the user. Plus B's idle timer and C's cap as a safety net,
  because a user who closes the tab and goes home will not press stop.
- **§4:** (2) is not answered by the user, so this **requires** B and C anyway.
- **Misfire mode:** the user never stops it ⇒ B/C carry it, so the failure is
  theirs, not this option's.
- **Cost to build:** B + C + protocol + UI. Largest.
- **Verdict:** a product feature layered on B/C, not an alternative to them.

### E. Workspace-owned

One agent per registered workspace, shared by every session in it.

- **Reaps it:** whatever reaps a workspace — nothing today.
- **§4:** cannot answer (1) coherently: `cc --resume` is keyed on a *session*,
  and `spawnEntry` registers per sid, so "the workspace's agent" has no defined
  transcript. It also collides with the ownership gate (`admitChat` →
  `OwnedByOther`, `wt_chat.go:309`).
- **Verdict:** rejected on mechanism, not on cost.

### F. Leave the agent mortal; stop lying about dead requests

Do **nothing** about the lifetime. Instead:

1. When a session's agent is gone, do not offer its pending requests as
   answerable. Either withdraw them from `Manager` at `teardown` (the one place
   that knows a session ended and already runs once per session), or tag them so
   `PendingBackfill` and the store render them as expired rather than actionable.
2. Tell the reader, once, that the turn did not survive the reload — the same
   discipline tether#124 applied to dropped envelopes: say only what is true.

- **Reaps it:** nothing new; A's reaper is untouched.
- **§4:** all five, unchanged.
- **Misfire mode:** a request is withdrawn for a session that was actually
  adopted inside the §2.4 window ⇒ a prompt disappears that *could* have been
  answered. Avoidable: `teardown` runs only when `Events()` has closed, which is
  strictly after the agent is really gone.
- **Cost to build:** smallest of all. Touches `teardown`, `PendingBackfill`, and
  the store's permission reducer.
- **What it does NOT fix:** the in-flight turn. It converts a lie into an honest
  loss.

⇒ **The work item's own candidate — "move request ownership to the daemon and
let the agent die" — is not on this list, because §2.5 measured it as already
done.** F is what that candidate becomes once you know that.

---

## 6. Recommendation

### 6.0 What the owner actually decided — 2026-08-21

Recorded so that "recommended" below is not later read as either more or less
than it is.

**Accepted:**

- **F ships now, as its own work item.** Approved to build. Its scope is §5-F:
  stop offering a dead session's pending requests as answerable, and say once
  that the turn did not survive. It does not prejudice the lifetime decision.
- **B+C is the accepted direction for the lifetime, as one piece of work, after
  F.** Not started. Filed separately so it cannot be picked up piecemeal — §6.4
  is the reason B alone is not acceptable.
- **A is rejected as the resting place**, on §2.4's grounds: staying on it keeps
  an accidental 0–43s grace period and a reload whose effect the user cannot
  predict. E stays rejected on mechanism (§5-E). D remains a later feature.

**NOT decided, and deliberately still open:**

- **T (the idle-eviction interval) and N (the cap).** §6.5 stands: neither is
  picked here. N specifically **cannot** be picked yet — it needs §7.4, the
  resident cost of one live agent, which is filed as its own measurement task.
  **Update 2026-08-21 (tether#138):** that task ran and §2.8 measures a **floor**
  — 1.015 GiB per live agent tree, 334 MiB for the agent process alone, against
  2.6 MiB + ~0.5 MiB/session on the daemon side. N is still **not** picked, for
  two named reasons rather than for want of any number: the floor is a floor and
  the representative figure needs quota the owner has not granted (§2.8.2), and
  §2.8.4 had not established that a cap bounds resident memory at all. What the
  floor **does** settle: the daemon's own per-session cost is negligible, and N
  has to be written as a memory budget rather than a portable integer (§5-C).
  **Update 2026-08-21 (tether#141):** the second of those two reasons is now
  closed — §2.9 measures that the daemon's own single-pid SIGKILL reclaims **all**
  ~700 MiB of MCP grandchildren in under a second, so a cap **is** a real memory
  backstop. It does so because those servers exit themselves, not because the kill
  reaches them, so the three riders in §5-C travel with any figure that gets
  written down. N is still not picked; the remaining blocker is the representative
  figure, i.e. quota.
- **Whether §6 survives contact with a real browser.** §7.2 is the cheapest
  experiment that could overturn point 2 of §6, and it **has not been run** —
  this machine has no display server, so it needs a machine that has one. Until
  then §2.4's ~41s is a Go-client number and, per §7.2, must not be quoted as
  the browser's.

⇒ Reading guide: everything below §6.0 is the analysis that produced the
decision, not a second decision. If §7.2 later shows a browser's close is
reliably observed, re-read §6 point 2 before building B+C — F is unaffected
either way, which is why it went first.

---

**Ship F now, independently. Then do B+C as one piece of work.**

The reasoning, in the order it actually runs:

1. **F is unconditional.** Whatever is decided about the lifetime, the UI must
   not present an answerable prompt for a tool call that cannot exist. That is
   true under A, B, C and D alike, it is the cheapest change on the list, and it
   is the only one whose value does not depend on a number nobody has measured.
   It does not prejudice the lifetime decision either way.

2. **A is not the safe default it looks like.** The comment at
   `registry.go:1414-1423` earns its confidence from the *resource* argument, and
   that argument is sound. But §2.4 shows the *behavioural* claim underneath it —
   "the connection context bounds the subprocess lifetime" — bounds it at
   somewhere between 0.03s and ~43s depending on whether a UDP packet got out.
   Staying on A is therefore not "declining to add a grace period"; it is
   choosing to keep an accidental one, and to keep shipping a reload whose effect
   the user cannot predict. Once that is on the table, the conservative option
   changes sides.

3. **B is cheaper than the work item fears, for one specific reason.** tether#56
   is a fact about `teardown`, not about the ctx (§1.4). Re-parent the ctx to the
   session and every hop after the cancel is the path that runs today. The new
   surface is the trigger and its tests, not the reaping machinery. That is a
   real cost, but it is a bounded one, and it is the cost of *specifying* a
   window that already exists.

4. **B without C is not acceptable, so they are one piece of work.** B's misfire
   mode is precisely the tether#56 leak. C turns "leak until the daemon exits"
   into "at most N live agents, and the (N+1)th eviction degrades to today's
   `--resume`". A misfire that degrades to the status quo is the strongest
   property any of these options has, and only C has it.

5. **Do not pick T or N in this document.** §2.4 gives T a defensible floor
   (anything up to ~45s is not a new risk class — the daemon already does that
   when a client vanishes) and §7 says what N needs. Picking either from the
   armchair would be the mistake this repo has already written up twice.

6. **D is a feature to consider after B+C, not instead of them.** E is rejected
   on mechanism.

### What would change this recommendation

- **A measurement that the resident cost of one live agent is large** (say, RSS
  in the hundreds of MB with the model context loaded) would push N down toward
  2–3, at which point B+C stops being "sessions survive reloads" and becomes
  "the most recent two sessions survive reloads", and F alone might be the whole
  answer.
  **This trigger is now met — at the floor, before any representative
  measurement (§2.8.2, tether#138).** A started, connected, *idle* agent given a
  one-character prompt is **334 MiB** on its own and **1.015 GiB** as the process
  tree that actually has to stay resident. So the "hundreds of MB" branch is the
  live one, and the qualifier the bullet hedged with — "with the model context
  loaded" — turns out not to be needed to reach it. What that does **not** settle
  is the direction of the conclusion, because the bullet quietly assumed a fixed
  memory budget: at 1.02 GiB per agent, N is 1–3 on a 2–4 GiB host and still tens
  on the 47 GiB box these numbers came from. ⇒ the honest form of this bullet is
  not "N drops to 2–3" but **"N stops being a portable integer"**; see §5-C.
  On the smaller hosts, F alone being the whole answer is exactly right.
- **Evidence that the §2.4 long tail does not occur with a real browser** — i.e.
  that Chrome's WebTransport close is reliably observed and cancels
  `wtsess.Context()` promptly. That would restore A's determinism, remove the
  "coin flip" argument, and leave only the in-flight-turn loss, which F makes
  honest. This is the single cheapest experiment that could overturn point 2, and
  it is the first thing I would run next (§7).
- **Evidence that reloads overwhelmingly happen when no turn is in flight** would
  shrink the prize B is chasing to nearly nothing.
- **A decision that in-flight turns are cheap to re-ask** (short answers, cheap
  tokens) would do the same. The opposite — long agentic turns with tool calls —
  makes B more valuable, and that is the direction this product is going.

---

## 7. What I did not establish

Listed because an honest gap is worth more than a confident guess.

1. **Why a graceful WebTransport close is sometimes not observed by the daemon.**
   Measured: it happened 1 time in 3, and the `linger` arm shows the reap
   tracking *process exit* rather than the close call (§2.4). Not established:
   whether the close capsule was never flushed by the Go client, or whether
   `webtransport-go` 0.10.0's server does not cancel the session context on it.
   Two different fixes hang on that answer.
2. **What a real browser does.** Everything in §2 used a Go client. I have **no**
   measurement of Chrome's reload/close against this daemon, and Chrome
   negotiates its own idle timeout. The ~41s figure is specific to a peer whose
   idle timeout is 30s; the daemon's own offer is 90s, so a browser could sit
   longer. **Do not quote 41s as the browser number.**
3. **The backfill's visibility on the true respawn path.** §2.6's reload adopted
   a live agent. I did not construct a run where the reconnect lands *after* the
   reap and then check whether the replayed request is on screen before the user
   types. The work item's claim there remains [R].
4. **The resident cost of one live agent — half closed, half still open.**
   *Was:* "The stand-in agent is a Python process; its footprint says nothing
   about the real one. No RSS, no fd count, no CPU-at-rest figure. N cannot be
   chosen without this."
   **Measured floor (§2.8, tether#138, 3 authorised launches):** a started,
   connected, idle real agent given a one-character prompt is **334 MiB `VmRSS`**
   as a process, **1.015 GiB** as the 10-process tree it actually is (median of
   3; ranges in §2.8.2), **31–32 fds** on the agent and **170** across the tree,
   and **0.33–1.30% of one core at rest**. The daemon's own share of a live
   session is **+14 goroutines, +3 fds, +2.6 MiB for the first session and ~0.5
   MiB for each one after it**.
   **Still not established, and this is why the item is not deleted:**
   - **The representative figure.** Everything above was taken at the smallest
     context the process can hold — one character in, one turn, no attachment, no
     tool call — so it is a lower bound and the gap to a working session's
     footprint is unbounded by anything here. Closing it means driving several
     launches through real turns with tool calls and file reads, which is **model
     time the owner has to approve separately**; the three launches spent here
     were the whole authorisation.
   - **Whether that cost is per-machine or per-product.** 698 of the 1,040 MiB is
     the four stdio MCP servers *this* machine configures, spawned per agent
     process and shared with nothing. A representative figure needs a
     representative MCP configuration, and nobody has said what that is.
   - ~~**Whether the daemon's reaper reclaims it** (§2.8.4). Measured only for a
     graceful stdin close, which is not what the daemon does.~~ **Closed by
     tether#141 — §2.9: it does, entirely, in under a second, on the daemon's own
     single-pid SIGKILL path.** What replaces it is narrower and does not block
     choosing N: **the reclamation is the MCP servers' own doing, not the kill's**,
     so it holds for *this* machine's server set and is neither required nor
     observable by tether (§2.9.4). Whether any server anyone actually configures
     fails to exit is unmeasured and unmeasurable from here — it is a property of
     other people's machines.
   ⇒ **N can now be bounded, but still not chosen** — see §5-C for the shape the
   floor forces on it (a memory budget, not an integer) and §6.0 for what the
   owner has and has not decided.
5. **Whether `--resume` ever recovers a turn that was mid-generation.** Read as
   "no" from what `--resume` is, but not tested: nothing here interrupted a
   *long* turn and then resumed. The stand-in agent's turns complete in
   milliseconds.
6. **The opencode provider.** `opencode_provider.go:78`/`:472` were read, not
   exercised. Its `Interrupt()` deliberately kills `serve` *without* closing
   `Events()` (`registry.go:1543-1546` [R]), which means it has a hibernation
   state chat does not, and any option in §5 has to be re-argued for it.
7. **The shell surface.** `handleWTShell` builds its PTY env from
   `reg.PermEndpoint` (`wt_shell.go:83`, `:265` [R]) and so is a third producer
   of permission requests. Not measured, not analysed.
8. **Concurrency.** Every measurement was one session at a time. Nothing here
   says how the reaper, the cap, or the adoption path behave with several live
   sessions and overlapping reconnects.
9. **The 92 sessions on the owner's daemon.** Counted, not inspected — `ls -1 |
   wc -l` on the directory, no file opened, nothing copied. I do not know how
   many of those are resumable, nor how many a user would want live at once.

---

## Appendix — reproducing §2

The harness lives outside the repo (`/tmp/t134`) and is not committed; it is
described completely enough in §2.1 to rebuild. The safety envelope is not
optional: **port 18991 and an isolated `HOME`, never `:443`, never
`/root/.tether`, `--mcp-port` overridden off 8899, and no `pkill -f`/`pgrep -f`
on this machine** — the pattern matches sibling agents' processes. Every probe
process was accounted for at the end:

```
=== FINAL CENSUS: anything of mine left?
COUNT=0 MATCHING=/tmp/t134
=== positive control: the census CAN see a process
pid=3730046 ppid=3729892 state=S cmd=sleep 30
COUNT=1 MATCHING=sleep 30
=== control ended; recheck
COUNT=0 MATCHING=sleep 30
=== port 18991/18992 listeners
---END-portcheck (empty = free)---
```

The positive control is there because `COUNT=0` from a census that cannot see
anything would look identical to `COUNT=0` from a clean machine.

## Appendix B — reproducing §2.8 (tether#138)

Same envelope, different ports: **19181** and `--mcp-port 19182`, isolated
`HOME`, never `:443`, never `/root/.tether`, no `pkill -f`/`pgrep -f`. The
harness is again outside the repo and not committed; §2.8.1–2.8.3 describe it
completely enough to rebuild. The pieces:

- the §2.1 stand-in agent, re-derived from `internal/agent/fakecc_test.go` at
  `4536b9c` (per-turn order `system/hook_started` → `hook_response` → `init` →
  `stream_event*` → `assistant` → `result/success`; nothing before the first
  prompt; never exits on its own);
- a `/wt/chat` client built from `poc/go-quic-wt/step2_client.go`'s shape — the
  PoC module was **copied out of the repo** and built there, so nothing was added
  to the tether tree (confirmed afterwards: `git status --short` empty, and the
  `web/dist` stub the `//go:embed` needs was removed);
- goroutine counts from `SIGQUIT` under `GOTRACEBACK=all`, diffed by
  `created by` site so the answer is *which* goroutines, not just how many;
- a process census that skips its own pid and its own script name, plus a
  survivor check that matches **recorded pid + `/proc/<pid>/stat` start time**
  rather than a pattern — this machine also runs the owner's live sessions with
  their own MCP children, and a pattern would have swept those in.

Three positive controls, because three different zeroes are claimed:

| The zero being claimed | Its control | Result |
|---|---|---|
| "no probe process is left" | start a `sleep 30`, census, reap, census again | `COUNT=1` → `COUNT=0` |
| "the idle agent uses ~0 CPU" | same meter on a deliberate spin loop | `0.00%` on `sleep`, **92.3%** on the spinner |
| "the goroutine dump shows n sessions" | count `fanOut`/`readLoop`/`guardStdout` frames | `0` at 0 sessions, exactly `n` at n |

Final state:

```
=== FINAL CENSUS: anything of mine left?
COUNT=0 MATCHING=tetherd server
COUNT=0 MATCHING=<probe>/standin
COUNT=0 MATCHING=<probe>/bin/wtc
COUNT=0 MATCHING=probe138
=== real-agent trees: 30 recorded pids across 3 trees
SURVIVORS=0
=== positive control: the census CAN see a process
pid=4101201 ppid=4101192 state=S cmd=sleep 30
COUNT=1 MATCHING=sleep 30
=== control ended; recheck
COUNT=0 MATCHING=sleep 30
=== port 19181/19182 listeners
---END-portcheck (empty = free)---
```

The owner's daemon was `pid=1912269` on `:443` and `127.0.0.1:8899` before and
after, i.e. never restarted and never bound over.

## Appendix C — reproducing §2.9 (tether#141)

Harness outside the repo and not committed; §2.9.1–2.9.6 describe it completely
enough to rebuild. **No daemon was started and no port was bound**, so the port
half of the envelope is vacuous here rather than satisfied — see §2.9.6 for why
the daemon is not in the causal chain. The rest of the envelope held: never
`:443`, never `/root/.tether`, no `pkill -f`/`pgrep -f`, no file opened under the
provider's `projects/` or `sessions/`, nothing deleted from either. Four pieces:

- one driver used for **both** arms, so the census, the kill and the sampler are
  literally the same code in the control and in the measurement;
- the stand-in topology of §2.9.2, whose fourth child `dup2`s `/dev/null` over
  fd 0 and is therefore structurally incapable of seeing EOF — the negative
  control;
- a survivor checker keyed on **recorded pid + `/proc/<pid>/stat` field 22 start
  time**, never on a name or pattern;
- a reaper that will only signal a pid it finds in one of the probe's own JSON
  census files, and only if that pid's start time still matches.

Coarse sample grid `t+0.2 / 1 / 5 / 15 / 60 / 120 s` in every run; trial 2 and
both stand-in runs additionally polled every 50 ms for the per-process latency in
§2.9.3. Trial 1 had no fine poll, so its latency is only bounded to
`(0.2 s, 1.0 s]` — consistent with trial 2's 0.866 s maximum.

Three zeroes are claimed, so three controls:

| The zero being claimed | Its control | Result |
|---|---|---|
| "no descendant survived the SIGKILL" | the EOF-deaf stand-in child, killed by the same code | `ALIVE=1` at every sample, `ppid=1`, 12,324 kB held — twice |
| "no probe process is left" | start a `sleep 30`, census, reap, census again | `COUNT=1` → `COUNT=0`, and the pid+start-time checker `alive` → `gone` |
| "the recorded pids are all gone" | the same checker that reported the stand-in survivor | `RECORDED_TOTAL=32 SURVIVORS_TOTAL=0` |

Final state:

```
=== recorded pids, matched on pid + /proc start time
   real.t1.samples.json        recorded=10  survivors=0
   real.t2.samples.json        recorded=10  survivors=0
   standin.dry1.samples.json   recorded=6   survivors=0     (control reaped by hand)
   standin.dry2.samples.json   recorded=6   survivors=0     (control reaped by hand)
   RECORDED_TOTAL=32 SURVIVORS_TOTAL=0
=== read-only substring census on this work item's own lab path
   COUNT=0 MATCHING=<lab dir>
   COUNT=0 MATCHING=standin_
   COUNT=0 MATCHING=driver.py
=== positive control: the census CAN see a process
   pid=86143 ppid=86141 state=S cmd=sleep 30
   COUNT=1 MATCHING=sleep 30      status_of(control) = alive
=== control ended; recheck
   COUNT=0 MATCHING=sleep 30      status_of(control) = gone
```

⚠️ One harness note worth keeping, a level up from tether#134's "the census
matched itself": the first run of that census reported `COUNT=1` on the lab path
and the hit was **the invoking shell**, whose command line contained the path.
The census skips its own pid, not its parent's. Re-invoking it by a command whose
text does not contain the lab path gave `COUNT=0`. Both are quoted here because
"the census counted the thing that launched it" is a different bug from "the
census counted itself", and only one of them is fixed by `skip os.getpid()`.

The two stand-in control survivors were killed by hand afterwards, by recorded
pid with a start-time check, and `AFTER-REAP-ALIVE=0` in both cases.
