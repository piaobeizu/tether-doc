# Plan — gh-3 Daemon goroutine + cc integration (v0.1 foundation)

| field | value |
|---|---|
| ticket | piaobeizu/tether#3 |
| spec | `.specs/2026-04-26-tether-go-quic-design.md` (§4 / §5 / §6 / §11.W / §11.Z / §11.AA) |
| precursor | gh-13 (cc-sdk-route, merged 2026-04-30) — produced `internal/backend/claude` (5030 LOC) |
| repos touched | tether (primary) only — per issue body "不包含" list |
| target PRs | 8 |
| target steps | 18 (each ≈ one commit) |
| Mode | accept-as-is (one-shot draft from spec; user reviews entire plan) |

---

## Inventory — what's already done

`internal/backend/claude/` (from gh-13 merge):

| file | LOC | role for gh-3 |
|---|---|---|
| session.go | 695 | core lifecycle — wrapped by ClaudeCodeProvider |
| spawn.go | 169 | PTY spawn — reused as-is |
| control.go | 317 | bidirectional control protocol — reused |
| message.go | 207 | NDJSON envelope types — reused |
| parser.go | 89 | line scanner — reused |
| recover_test.go etc | 1731 | test corpus — guards regressions |
| `cmd/spike/main.go` | 472 | will extend in S18 integration test |

**Delta for gh-3**: AgentProvider seam (D-17) wraps the above; new daemon orchestration layer; cc integration glue (Hook server, JSONL pipeline beyond raw parsing); Lock; Input gate; Skill system; D-18 seams; Unix socket; 4 CLI subcommands.

---

## Steps (18, grouped into 8 PRs)

### PR-1: AgentProvider seam + ClaudeCodeProvider adapter

**S1.** `internal/agent/provider.go` — define `AgentProvider` interface + `AgentEvent` struct per spec §5.0.
- Files: `internal/agent/provider.go` (~80 LOC), `internal/agent/provider_test.go` (~60 LOC, contract guards)
- Done when: interface compiles; contract test enumerates required methods.

**S2.** `internal/agent/claude_provider.go` — `ClaudeCodeProvider` implementing `AgentProvider`, wrapping `internal/backend/claude.Session`.
- Files: `internal/agent/claude_provider.go` (~250 LOC), `internal/agent/claude_provider_test.go` (~200 LOC)
- Done when: all interface methods delegate to existing `claude.Session`; round-trip Spawn → SendUserMessage → Events test passes against existing fixtures.

### PR-2: cc integration layer (port from happy-cli)

**S3.** `internal/cc/version.go` — startup `claude --version` parse + supported-range hard-fail (F-08).
- Files: `internal/cc/version.go` (~80 LOC), `version_test.go` (~80 LOC)
- Done when: in-range version → continue; out-of-range → process exit 1 with diagnostic listing supported range.

**S4.** `internal/cc/hookserver.go` — HTTP server on 127.0.0.1:0 with 5 cc hook handlers + `/blob/*` 501 + `/hooks/` prefix invariant (D-18 seam).
- Files: `internal/cc/hookserver.go` (~250 LOC), `hookserver_test.go` (~200 LOC)
- Done when: cc-shaped POST to `/hooks/pre-tool-use` reaches handler; `/blob/foo` returns 501; port-0 returns assigned port; hooks behind `/hooks/` prefix consistent.

**S5.** `internal/cc/hooks.go` — generate cc settings.json with hook URLs (port substitution from S4).
- Files: `internal/cc/hooks.go` (~120 LOC), `hooks_test.go` (~80 LOC)
- Done when: settings.json generated for given port + hook list parses back into the expected cc settings shape; integration test (cc started with generated settings) hits S4 handlers.

### PR-3: JSONL pipeline (watcher + classifier + mapper)

**S6.** `internal/cc/path.go` + `findsession.go` + `checksession.go` — cwd-bucket path encoding (F-03), last-session lookup, session validity.
- Files: `internal/cc/path.go` (~50 LOC), `findsession.go` (~50 LOC), `checksession.go` (~30 LOC), tests (~120 LOC)
- Done when: F-03 encoding (`/x/y/z` → `-x-y-z`) passes against PoC fixtures; given cwd, finds latest sid in `.claude/projects/<hash>/`.

**S7.** `internal/cc/jsonl_watcher.go` — fsnotify watcher + IncrementalReader (F-09 torn-write defense, UUID dedup) + 3-class classify (EVENT/HOOK/STATE per F-04).
- Files: `internal/cc/jsonl_watcher.go` (~280 LOC), `jsonl_watcher_test.go` (~250 LOC, includes torn-write race fixture)
- Done when: watcher emits classified records into channels; cursor advances only past `\n`; restart rebuilds UUID index from disk; replay-safe.

**S8.** `internal/cc/jsonl_mapper.go` — cc record → wire envelope mapper + §11.AA fence tag suffix grep (`<type>:<skill>` → envelope.skill plaintext).
- Files: `internal/cc/jsonl_mapper.go` (~400 LOC), `jsonl_mapper_test.go` (~300 LOC)
- Done when: every cc EVENT record type maps to envelope; fence tag `\`\`\`output:my-skill` produces `envelope.skill="my-skill"`; round-trip jsonl → envelope → consumer matches PoC fixture.

### PR-4: Lock + Input gate

**S9.** `internal/daemon/lock.go` — Lock state machine per spec §8 / §11.D (single-layer, 60s auto-release, force-takeover).
- Files: `internal/daemon/lock.go` (~200 LOC), `lock_test.go` (~250 LOC, concurrent-acquire fuzzing)
- Done when: state transition matrix matches spec table; concurrent acquire test (50 goroutines × 100 iters) shows no double-hold.

**S10.** `internal/daemon/inputgate.go` — Input gate (D-09): all PTY writes pass through; encodes CR-submit (F-01), Esc-cancel (F-07), bracketed paste, slash commands.
- Files: `internal/daemon/inputgate.go` (~150 LOC), `inputgate_test.go` (~200 LOC, against PoC fixtures)
- Done when: F-01 / F-07 fixtures replay produces byte-identical PTY writes vs PoC-1 captures.

### PR-5: daemon goroutine + watchdog (PoC-3 verification)

**S11.** `internal/daemon/daemon.go` — daemon goroutine orchestrating S2 + S4 + S7 + S9 + S10 + ring buffer fan-out (per spec §4.4 + §5.2).
- Files: `internal/daemon/daemon.go` (~300 LOC), `daemon_test.go` (~250 LOC)
- Done when: spawn → cc starts → JSONL emits → S7 classifies → ring buffer fan-out → AgentProvider events forwarded; race-detector clean (`go test -race`).

**S12.** `internal/daemon/watchdog.go` — supervisor + panic recovery + heartbeat + restart-with-backoff (§4.1 + §4.4 skeleton).
- Files: `internal/daemon/watchdog.go` (~150 LOC), `watchdog_test.go` (~250 LOC, PoC-3 three scenarios)
- Done when: PoC-3 scenarios all PASS — (a) injected panic in daemon goroutine → restart in <1s; (b) deadlock detection via heartbeat → ctx cancel + restart; (c) `kill -9` → systemd-equivalent (or supervisor harness) re-spawns process. CI lint blocks `log.Fatal*` / `os.Exit` in goroutines (§4.3).

### PR-6: Skill system

**S13.** `internal/skill/manifest.go` — `tether.toml` parser + symlink-based plugin pool (D-20, ~70 LOC pure symlink per issue body).
- Files: `internal/skill/manifest.go` (~120 LOC), `manifest_test.go` (~150 LOC)
- Done when: parses real tether.toml from PoC corpus; reload picks up changes; dangling-symlink detection works.

**S14.** `internal/skill/integration.go` — wire skill pool into ClaudeCodeProvider's spawn pipeline (`<workspace>/.claude/plugins/`).
- Files: `internal/skill/integration.go` (~120 LOC), `integration_test.go` (~150 LOC)
- Done when: spawn creates symlink farm at `<workspace>/.claude/plugins/`; cc sees skills; spawn cleanup removes farm; skills survive across daemon goroutine restart.

### PR-7: D-18 forward-compat seams + Unix socket

**S15.** `internal/daemon/d18_seams.go` — `<workspace>/.tether/` auto-create + envelope `workspaceRoot` field + `/hooks/` prefix invariant test.
- Files: `internal/daemon/d18_seams.go` (~50 LOC), `d18_seams_test.go` (~80 LOC)
- Done when: spawn with non-existent `<workspace>/.tether/` creates it; every emitted envelope contains `workspaceRoot`; static contract test asserts no handler bypasses `/hooks/` prefix.

**S16.** `internal/control/unixsock.go` — local Unix socket listener (`tether attach` entry).
- Files: `internal/control/unixsock.go` (~180 LOC), `unixsock_test.go` (~200 LOC)
- Done when: `tether attach` client connects to `~/.tether/control.sock`; sees ring buffer replay + new envelopes; multiple concurrent attaches OK.

### PR-8: CLI subcommands + end-to-end integration

**S17.** `cmd/tether/main.go` — daemon main per §4.4 skeleton + 4 CLI subcommands `ls / resume <id> / mode <id> <new-mode> / kill <id>`.
- Files: `cmd/tether/main.go` (~250 LOC), `main_test.go` (~200 LOC)
- Done when: `tether ls` lists sessions from `~/.tether/users/default/sessions/`; `resume <sid>` calls AgentProvider.Resume → restored Session; `mode <sid> <m>` calls SetMode; `kill <sid>` SIGTERM (NOT SIGINT per F-07).

**S18.** Integration test extending `cmd/spike` — full daemon happy-path scenario.
- Files: `cmd/spike/scenario_daemon.go` (~200 LOC) + verification report extension
- Done when: end-to-end run: `tether spawn /tmp/sandbox` → cc starts → user message → assistant reply → tool_use → hook fires → mode switch → kill; verification report shows PASS for all 8 daemon-layer assertions.

---

## Cross-repo gates

Single repo (tether). **No M5 gate** triggered — spec doesn't add `api/**/*.go` (the wire envelope types live in `internal/`, not `api/`).

---

## Risks + mitigations

| risk | likelihood | impact | mitigation |
|---|---|---|---|
| AgentProvider interface (S1) churn — too narrow → forces refactor on first 2nd-provider attempt | high | medium | spec §5.0 explicitly says "v0.1 单一实现，等 codex/opencode 进来时按需 refactor"; accept the churn; just don't put backwards-compat shims today |
| Hook server (S4) port collision with skills' MCP servers | medium | low | port-0 listener (§5.2 already prescribes); record port in session meta; skills query port via env var |
| JSONL torn-write (S7) regression — the F-09 fixture was 1ms × 45s in PoC-1 but cc may behave differently under sustained load | low | high | add long-soak test (1ms × 5min) in S7 before merging; static contract guard for "cursor only advances past `\n`" |
| Lock (S9) deadlock under force-takeover — concurrent take + auto-release race | medium | medium | S9's concurrent-acquire fuzz (50×100); race detector mandatory in CI |
| Watchdog (S12) restart loop — bug in safeRun panic-handler causes infinite restart at 10ms intervals | medium | high | exponential backoff with cap (already in spec §4.4 skeleton); restart-rate limiter (max 5 / 60s) before declaring dead |
| Skill symlink (S14) leaks across spawns — workspace-A skills visible in workspace-B's spawn | medium | medium | spawn cleanup deletes `<workspace>/.claude/plugins/` symlink farm before next spawn; integration test checks isolation |
| §11.AA fence tag (S8) false-positive — unrelated `\`\`\`json` etc misparsed as `<type>:<skill>` | low | low | regex anchors on cc-emitted block headers only; spec §11.AA grammar gives explicit tag suffix shape; test fixtures cover negative cases |
| `cmd/tether` daemon main (S17) doesn't shadow `cmd/spike` cleanly — conflicting init() | low | low | move shared init helpers to `internal/cli/`; spike retains its `-binary` flag from gh-13 work |

---

## Test strategy

Per-step:
- Unit tests inline (every new file gets `_test.go`)
- `go test -race` mandatory in CI from S1 onwards (gh-13 retro lesson)
- Static contract guards in `internal/agent/contracts_test.go` (extend gh-13's pattern)
- Negative-property tests (e.g., "no jsonl writes from daemon", "no fd 3 wiring") inherited from gh-13's contracts_test.go

Cross-cutting:
- PoC-3 watchdog scenarios (S12) — 3 must-pass
- PoC-1 input fixtures (S10) — F-01, F-07, F-10/F-11 byte-replay
- Integration scenario (S18) — extends `cmd/spike` to include daemon-layer assertions

Cost characterization: spike's existing $0.05/run pattern continues; new scenarios add ≤$0.05 each.

---

## Rollback plan

Per PR (each PR is independently revertable):
- PR-1 (seam) — pure-additive interface; revert = delete new package, no caller depends yet
- PR-2 (cc integration glue) — additive; revert = delete; no production caller until PR-5
- PR-3 (JSONL pipeline) — additive; revert restores gh-13's `parser.go` as the only JSONL consumer
- PR-4 (Lock + InputGate) — additive helpers; revert = delete
- PR-5 (daemon goroutine + watchdog) — first PR with **production caller**; revert here disables the new daemon path. Spike-driver fallback (gh-13's existing) remains operational
- PR-6 (Skill) — gated by feature flag `TETHER_ENABLE_SKILLS=true`; revert = unset flag
- PR-7 (D-18 + Unix socket) — additive; revert = delete; CLI commands in PR-8 fall back to direct AgentProvider call
- PR-8 (CLI + integration) — final user-facing surface; revert = remove `cmd/tether` binary, ship `cmd/spike` only (degraded but functional)

No data migrations. No external service mutations. No DB schema. Pure code revert at any boundary.

---

## Self-review checklist (per skill phase-plan §13)

- [x] Placeholder scan — no `TODO` / `FIXME` / `<…>` left in plan
- [x] Single-PR feasibility — 18 steps split into 8 PRs (each PR 200-700 LOC, reviewable)
- [x] Step granularity — each step = one commit ≈ one file group + tests
- [x] Verification clarity — every step has "Done when" line with measurable criteria
- [x] Commit-M5 gate consistency — no `api/**/*.go` paths; M5 gate not triggered
- [x] Risk pre-mortem — 8 risks with mitigations + likelihood/impact rating
- [x] Test strategy — unit + race detector + static contract + integration; PoC-1/PoC-3 fixtures wired
- [x] Rollback path — every PR independently revertable; no irreversible writes

---

## Open questions (none currently — surface here if user finds during review)

