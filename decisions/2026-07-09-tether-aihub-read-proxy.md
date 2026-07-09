# ADR: tether daemon as a read-only aihub client for Work views (B2)

- **Status**: Accepted (2026-07-09)
- **WI**: tether#15 (`wi_zk6CBXl3`) · spec mem_IxWkuyQH · plan mem_MsXVL40a
- **Owner**: xiaokang.w
- **Relates**: supersedes D-20 *in part* (see below); follows tether#8 (Path B, mem_3PE2qeew); spawned tether#16 (isExempt hardening)

## Context

tether is repositioned from "a cc chat client" to **an industry-agnostic workbench** built on Claude Code + aihub + polyforge + scenario. The workbench needs to surface aihub's *work views* (ready queue, work-item detail, event timeline) in the SPA — these are **polyforge primitives, not coding primitives**, so rendering them is what makes the console domain-agnostic.

This collides with **D-20** (`internal/wire/types.go`), established in tether#8: *"the daemon parses fence markers but never the Content body — daemon-not-aware-of-skill."* tether#8 deliberately chose **Path B** (the agent emits fenced DAG blocks; the daemon stays a pure renderer) and **rejected "Path A: tether becomes an aihub client."** The DAG is agent-pushed and opaque to the daemon.

The MVP question: how does the SPA get queue/wi/timeline data? aihub has no SSE/WebSocket (REST reads only). Two data paths:
- **B1** — SPA calls aihub directly (browser holds an aihub token + CORS).
- **B2** — SPA → tether daemon → aihub (daemon holds the key server-side, proxies reads).

## Decision

1. **Reverse D-20 ONLY for read-only work views.** The daemon becomes an aihub **read-only client for queue / wi / timeline data**, exposed as a small set of curated **GET-only** endpoints `/api/v1/work/{projects,queue,items/{id},items/{id}/events}` (behind the existing auth middleware). The **chat DAG / fenced-block path stays Path B, unchanged** — the daemon still treats fenced bodies as opaque and the agent still emits them. Two data sets, two paths, each optimal; minimal blast radius; no change to shipped tether#8 code; DAG live-ness (agent-pushed) is not regressed to polling.

2. **Data path = B2 (daemon proxy), not B1.** The aihub api_key lives **server-side only** (from `TETHER_AIHUB_URL`/`TETHER_AIHUB_KEY`, falling back to `~/.polyforge/config.toml` `[server].url`+`[auth].api_key`) and is **never sent to the browser**. Read-only is enforced per-handler (net/http `HandleFunc` dispatches all methods, so each handler rejects non-GET with 405). Responses are whitelisted DTOs (`internal/wire/work.go`); upstream errors map to 403/502/503 with generic messages that never echo the key. The SPA has a project selector; each request carries `?project=`. B1 (SPA-direct) is deferred to a future multi-tenant SaaS form.

3. **Trust boundary = single-operator MVP (D-15.6).** tether has **no per-user identity** (a single shared token mints the session cookie) and the daemon is exposed on a public IP for live-verify. aihub's `project_scope` is **single-valued and only RESTRICTS, never grants**, so it cannot bound a multi-project picker per user. Therefore the MVP is an explicit **single-operator tool**: the daemon key == the owner; every token holder is trusted as the owner; the selector shows the owner's readable projects. **Hard prerequisite**: per-user identity (each user's own aihub key/identity) MUST land before tether is exposed to multiple distinct users. This is a gating non-goal, not a soft deferral.

## Consequences

**Positive**
- Proves "tether = aihub's frontend" and "the core is industry-agnostic" using only the coding doing-surface we already dogfood — because the work views are polyforge primitives, the view side is domain-agnostic by construction.
- Read-only + single-operator ⇒ low blast radius; the daemon proxy is unit-tested (read-only 405, key-never-leaked, project-filter) and independently landable.

**Negative / watch**
- The daemon is now an aihub client **for reads** — the D-20 exception is deliberate and scoped; future contributors must keep the fenced-block path Path B and not generalize the coupling.
- **Dual-source staleness**: the same wi appears in the chat DAG (agent-pushed, live) and the Work-tab timeline (polled ~8s). They can visibly disagree; the SPA shows a "last refreshed" marker to make this legible.
- Single-operator only; multi-user needs per-user identity first (see above).
- The pre-existing `isExempt` static-suffix auth exemption (tether#16) means a client-controlled path segment ending in `.js` etc. could bypass cookie auth — low-exploitability for read-only clients, tracked separately.

## Alternatives rejected

- **Unify everything through the daemon (drop Path B):** would rewrite shipped tether#8 DAG code and regress DAG live-ness to polling (no aihub push). Low ROI.
- **B1 (SPA-direct to aihub):** adds a browser→aihub auth surface + CORS; deferred to a multi-tenant form.
- **Generic `GET /api/v1/aihub/*` passthrough:** leaks aihub's full surface to the browser and is harder to keep provably read-only; rejected for curated endpoints.
