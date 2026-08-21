# Agent subprocess lifetime — who owns it, and what a reload actually destroys

> Status: **§6 accepted by the owner, 2026-08-21. Option F is approved to build;
> B+C is the accepted sequel and has NOT started. Nothing is implemented yet.**
> No production file was touched by this document. See §6.0 for exactly what was
> and was not decided — in particular **T and N are still unchosen**, and the one
> experiment that could overturn §6 has not been run (§7.2).
> Work item: tether#134. Read against tether `5f96f7f` (`#211`/`#212`/`#213` landed).
> Measurements taken 2026-08-21 against a purpose-built isolated daemon, never the owner's.

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
  live agent, which I did not measure (§7).

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
4. **The resident cost of one live agent.** The stand-in agent is a Python
   process; its footprint says nothing about the real one. No RSS, no fd count,
   no CPU-at-rest figure. **N cannot be chosen without this.**
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
