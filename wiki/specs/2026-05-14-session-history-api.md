# Session history API + on-disk format

> Status: **shipped** (2026-05-14, tether 0.4.x).
> Owner: tether daemon.
> Implements: `internal/session/history.go`, `internal/server/session_api.go`.

## Why this exists

The v2 daemon spawns per-session cc / opencode subprocesses. Once a subprocess
exits or the WebTransport client disconnects, the live event stream is gone —
but users want to scroll back through previous turns from the same session.
This doc pins the contract for the durable replay surface.

## On-disk layout

```
~/.tether/sessions/<sid>/history.jsonl
```

- `<sid>` is the **real** agent session id (cc UUID or opencode `ses_*`) — not
  the temporary placeholder Registry uses pre-`system/init`. Re-key happens
  inside `Registry.GetOrSpawnEntry` once the agent emits its init event.
- One JSON object per line; no opening/closing delimiters; file is open-
  append-only. Lossless concat: append survives daemon restart mid-stream.
- Permissions: `~/.tether/sessions/<sid>/` is `0700`, the JSONL file is
  `0600`. The daemon never serves these as static files.

### Per-line schema

```json
{"role": "user", "text": "hello", "ts": 1715688123456}
{"role": "assistant", "text": "Hi! How can I help?", "ts": 1715688123920}
```

| Field   | Type    | Notes |
|---------|---------|-------|
| `role`  | string  | `"user"` or `"assistant"`. No system messages here — those go to the agent via env / settings. |
| `text`  | string  | UTF-8. **HTML escaping disabled** (`json.Encoder.SetEscapeHTML(false)`) so backticks / angle brackets in code blocks survive. |
| `ts`    | int64   | Unix milliseconds at which the message was written / accumulated. |

### Accumulation rules

Streaming providers (cc with `--include-partial-messages`, opencode with
`message.part.delta`) emit dozens of `EventText` per assistant turn. The
HistoryStore buffers chunks in memory keyed by `<sid>` and flushes to
`history.jsonl` exactly once per `EventResult` via `FinalizeAssistant`.

- Per-session in-memory accumulator is capped at `MaxAssistantBufBytes`
  (4 MiB). Beyond that, a `"[... response truncated at <N> bytes ...]"`
  marker is appended and subsequent chunks for the same turn are dropped.
  The daemon logs a warning (`history: assistant response truncated`).
- Corrupt JSONL lines during `LoadHistory` are skipped with a per-line
  warning; the rest of the file still loads. ENOENT is silent (no-history-
  yet is the common case, not an incident).

## HTTP API

All endpoints live under `/api/v1/sessions/`. Auth: cookie middleware
(same as the rest of `/api/v1/*`). WT ticket is NOT required for these
JSON endpoints — they only return history, not live streams.

### `GET /api/v1/sessions`

Lists every session id with on-disk history under
`~/.tether/sessions/`.

```json
["633e5ed8-cada-422a-aee1-c7a3502eb4fd", "ses_01HX9P2VPGYZ8MN7Q4VEY8MJ9V"]
```

Returns `[]` when there's no history yet.

### `GET /api/v1/sessions/<sid>/messages`

Returns the full message log for `<sid>` in chronological order.

```json
[
  {"role": "user", "text": "hello", "ts": 1715688123456},
  {"role": "assistant", "text": "Hi! How can I help?", "ts": 1715688123920}
]
```

**Security**: `<sid>` is validated against the alphabet `[A-Za-z0-9_-]`
with length 8-128. Anything else (path traversal, slashes, URL-encoded
escapes, control bytes) returns `400 invalid sid`. This is defense in
depth on top of the auth gate, so the path is safe even if the cookie
middleware ever regresses.

Returns `[]` for unknown sid (so the client can render a fresh empty
chat without a 404 round-trip).

## Privacy + retention

- **Not encrypted at rest.** Files inherit POSIX `0600` and live under
  the user's home; same trust model as `~/.bash_history`. Encryption is
  tracked in D-13 §2 (out of scope here).
- **No automatic retention.** A long-running daemon will accumulate
  files indefinitely. A future `tether prune-sessions --older-than 30d`
  is filed under v0.5 roadmap.
- **PII**: contents are whatever the user typed and whatever the agent
  replied. Treat them as user data — never include them in telemetry,
  bug reports, or diagnostics unless the user has explicitly attached
  the relevant file.

## What's not stored

- `tool_use` / `tool_result` blocks (only the surrounding text).
- System / status events (`init`, `error`, `rate_limit`).
- WT envelopes (browser-side concern, kept ephemeral).

These can be reconstructed from the agent's own session history (cc
JSONL under `~/.claude/projects/`, opencode's `serve` state) if needed.
The tether-side history is intentionally a thin replay log for the
chat surface — not an archive of the full agent transcript.
