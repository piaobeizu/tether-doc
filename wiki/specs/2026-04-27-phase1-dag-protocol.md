# Tether Phase 1 — DAG Protocol & UI Rendering Spec

**Status:** Draft v0.1
**Date:** 2026-04-27
**Companion docs:** `2026-04-26-tether-go-quic-design.md` (transport layer; unchanged)
**Validation basis:** 4 spikes (`spike-dag-block`, `spike-dag-streaming`, `spike-dag-large`, `spike-dag-tools`) — all green

## 0. Scope

This spec defines the protocol and contracts that turn tether from a "remote cc terminal" into a "remote cc + structured-UI client." It is the minimum set of decisions needed to ship the first end-to-end demo (animation or research workflow).

**In scope:**
- Fenced block schemas: `dag`, `form`, `candidates`, `media` (4 block types for v1)
- Streaming parser rules for tether app
- The shared `dag` skill: slash command surface + state file format
- App rendering contract: component lifecycle, button-to-command resolution, layout default
- Media URL handling (`tether://blob/<sha256>`) and daemon-side blob proxy

**Explicitly out of scope (Phase 2 or later):**
- Skill self-rendered components (custom UI plugins) — Phase 1 uses fixed schemas only
- App-direct skill invocation (bypassing cc) — all interaction goes through cc
- Sandbox / third-party skill security model — Phase 1 trusts installed plugins
- Multi-workspace UI (tab/sidebar)
- DAG patch protocol (we always re-emit full block)
- Audit / version control beyond what git already provides

## 1. Architecture recap

```
┌──────────────────────────────────────────────────────────────────┐
│ tether app  (browser-rendered web technology, not specified here) │
│  • streams cc text + structured fenced blocks                    │
│  • renders blocks via 4 built-in component renderers             │
│  • forwards user actions back as slash commands → cc stdin       │
└──────────────────────┬───────────────────────────────────────────┘
                       │ WebTransport / QUIC
┌──────────────────────▼───────────────────────────────────────────┐
│ tether daemon  (Go; v0.1 design unchanged + blob proxy added)    │
│  • PTY ↔ cc                                                       │
│  • multi-client attach + JSONL replay                            │
│  • blob registry + tether://blob/<sha256> proxy (NEW for Phase 1)│
└──────────────────────┬───────────────────────────────────────────┘
                       │ PTY
┌──────────────────────▼───────────────────────────────────────────┐
│ cc  (orchestrator; runs domain skills + dag skill + MCP servers) │
└─────┬──────────────────┬──────────────────┬─────────────────────┘
      │                  │                  │
   ┌──▼──┐         ┌─────▼─────┐      ┌─────▼─────────┐
   │ dag │         │ domain    │      │ gmi-multimodal│
   │skill│         │ skills    │      │ MCP server    │
   │(md) │         │(animation,│      │(image/video/  │
   │     │         │ research, │      │ audio)        │
   │     │         │ etc.)     │      │               │
   └─────┘         └───────────┘      └───────────────┘
```

Skills are markdown procedure files (polyforge-style). State lives in the filesystem. cc invokes MCP tools and slash commands; cc emits text + fenced blocks. Tether is purely transport + render.

## 2. Wire model (cc → app)

cc's stdout is a text stream. Tether app receives it (post-PTY, JSONL-mapped) as a series of `text_delta` chunks. Each chunk is an arbitrary substring of the assistant's text response — sizes range from 3 chars to 200+ chars and are NOT aligned to any semantic boundary (validated in `spike-dag-block`).

The text stream is treated as **GitHub-flavored markdown** with one extension: fenced code blocks whose language tag is in the recognized set (`dag` | `form` | `candidates` | `media`) are parsed as structured payloads, not rendered as code. All other content (prose, lists, tables, unknown fences) is rendered as markdown.

There are NO out-of-band channels. Everything flows through the assistant text stream.

## 3. Recognized block types

All structured blocks share this skeleton:

````markdown
```<kind>
{
  "v": <integer schema version>,
  "instance_id": "<string, stable across re-emissions>",
  ...kind-specific fields...
}
```
````

The fence tag is just the kind name (`dag`, not `dag@v1`). Schema versioning lives in the JSON `v` field. Future incompatible versions can either bump `v` (forcing renderer to dispatch to a v2 handler) or introduce a new fence tag.

### 3.1 `dag` block — node graph view

````markdown
```dag
{
  "v": 1,
  "instance_id": "<workspace-unique stable id>",
  "title": "<short human title>",
  "nodes": [
    {
      "id": "<short node id, unique within DAG>",
      "label": "<human label>",
      "status": "pending|running|approved|rejected|failed|skipped",
      "preview": null | <Preview>,
      "actions": [<Action>, ...],
      "meta": { ... }                  // OPTIONAL, freeform domain extension
    },
    ...
  ],
  "edges": [
    {"from": "<node id>", "to": "<node id>"}
  ]
}
```
````

**Field rules:**

| Field | Required | Notes |
|---|---|---|
| `v` | yes | Always `1` for Phase 1. |
| `instance_id` | yes | Stable across all re-emissions of the same DAG. App uses this as the join key for component update vs new mount. |
| `title` | yes | Renderer displays as DAG header. |
| `nodes` | yes | Array; may be empty. |
| `nodes[].id` | yes | Must be unique within `nodes`. |
| `nodes[].label` | yes | Human-readable; may contain Unicode. |
| `nodes[].status` | yes | Enum, see §3.1.1. |
| `nodes[].preview` | no | `null` or `Preview` object (§3.5). |
| `nodes[].actions` | no | Array of `Action` (§3.6). May be empty or missing. **Renderer MUST tolerate either** (validated in spike-dag-block p2 and spike-dag-streaming). |
| `nodes[].meta` | no | Freeform JSON object for domain skills to attach extra data (frame_id, model used, latency, etc.). Renderer ignores. |
| `edges` | yes | Array; may be empty. Both endpoints must reference existing node ids. |

#### 3.1.1 Status enum

| value | meaning | typical actions to render |
|---|---|---|
| `pending` | Not yet started | run, skip, edit-input |
| `running` | Currently executing | (usually no actions; show spinner) |
| `approved` | User accepted result | redo, edit-after |
| `rejected` | User rejected result; will not be retried | reopen |
| `failed` | Execution error | retry, debug, skip |
| `skipped` | User chose to bypass | reopen |

Renderer SHOULD show a status badge whether or not `actions` is present. When `actions` is empty/missing on a node, render badge only.

### 3.2 `form` block — input collection

````markdown
```form
{
  "v": 1,
  "instance_id": "<unique stable id>",
  "title": "<form title>",
  "fields": [
    {
      "name": "<machine name>",
      "label": "<human label>",
      "type": "text|textarea|number|select|checkbox|file",
      "value": <current value, optional>,
      "placeholder": "<text|null>",
      "options": [{"value": "...", "label": "..."}],   // for select only
      "required": true|false
    },
    ...
  ],
  "submit": {"label": "<button text>", "command": "<slash command template>"},
  "cancel": {"label": "<button text>", "command": "<slash command template>"}
}
```
````

When the user submits, the app interpolates field values into `submit.command`. The interpolation syntax is `{<field-name>}`, e.g.:

```
"command": "/animation:set-prompt --frame={frame} --prompt='{prompt}'"
```

Quoting and escaping rules: the app MUST shell-escape values that contain whitespace or quotes before substitution, OR use a structured wire format (see §6.2 for command argv form). For Phase 1 we use a simple convention: values containing whitespace are wrapped in single quotes and any embedded single quotes are escaped as `'\''`.

### 3.3 `candidates` block — multi-option picker

````markdown
```candidates
{
  "v": 1,
  "instance_id": "<unique stable id>",
  "title": "<picker title>",
  "selectable": "single" | "multi",
  "options": [
    {
      "id": "<option id>",
      "label": "<short label>",
      "preview": <Preview>,                  // strongly recommended
      "select_command": "<slash command>"    // sent on select
    },
    ...
  ]
}
```
````

Used for "approve which of N candidates" UX. Each option's preview is what the user compares; the command fires when the user selects. For `selectable: "multi"`, the app collects selections and sends each command in turn (or batches via a separate convention; v1 sends individually).

### 3.4 `media` block — single media artifact

````markdown
```media
{
  "v": 1,
  "instance_id": "<unique stable id>",
  "type": "image" | "video" | "audio",
  "url": "<tether://blob/<sha256> or http(s)://>",
  "sha256": "<hex digest, optional but recommended>",
  "title": "<optional caption>",
  "actions": [<Action>, ...]                 // optional
}
```
````

Used to surface a single artifact (final video, generated image set's "winner", etc.). For inline media inside DAG nodes, prefer `Preview` (§3.5) on the node rather than emitting a separate `media` block.

### 3.5 `Preview` object (shared)

Used inside `dag.nodes[].preview` and `candidates.options[].preview`:

```json
// IMAGE
{"type": "image", "url": "tether://blob/<sha>", "sha256": "<hex>", "alt": "<text>"}

// VIDEO
{"type": "video", "url": "tether://blob/<sha>", "sha256": "<hex>", "poster_url": "tether://blob/<sha>", "duration_ms": 30000}

// AUDIO
{"type": "audio", "url": "tether://blob/<sha>", "sha256": "<hex>", "duration_ms": 30000}

// TEXT (for short text previews — bullet of search result, code snippet)
{"type": "text", "content": "<inline text, ≤2KB>"}

// LINK (for external URL — research source)
{"type": "link", "url": "https://...", "title": "<text>", "snippet": "<text>"}
```

`url` MUST be one of:
- `tether://blob/<sha256>` — daemon-managed blob (most common for generated artifacts)
- `https://...` or `http://...` — external URL (research sources, etc.)

`file://` URLs are NOT permitted in the wire format. Skills MUST register local files with the daemon and emit `tether://blob/<sha256>` instead. (See §7.)

### 3.6 `Action` object (shared)

Used inside `dag.nodes[].actions`, `media.actions`, etc.:

```json
{
  "label": "<button text>",
  "command": "<slash command string>",
  "confirm": true | false,    // OPTIONAL, default false
  "style": "primary|danger|default"   // OPTIONAL hint
}
```

When the user activates the button, the app sends `command` as the next stdin line to cc (verbatim, with a trailing newline). If `confirm: true`, the app shows a "are you sure?" overlay before sending.

## 4. Streaming parser

Tether app's parser receives an unbounded stream of `text_delta` chunks and emits a stream of higher-level events to the renderer. It is a stateful FSM with a rolling buffer.

### 4.1 Events emitted to renderer

| Event | Payload | When |
|---|---|---|
| `PROSE_DELTA` | `{text: string}` | New characters of prose (markdown) outside any block |
| `BLOCK_START` | `{kind: "dag"\|"form"\|...; instance_id?: string}` | Recognized fence opener seen and validated |
| `BLOCK_COMPLETE` | `{kind, instance_id, payload: object}` | Block content fully received and JSON-parsed successfully |
| `BLOCK_ERROR` | `{kind?, raw_text: string, error: string}` | Block detected but JSON malformed; renderer falls back to code-block markdown |
| `STREAM_END` | `{}` | Underlying stream closed |

`BLOCK_START` is emitted as soon as the parser commits to "yes this is a block". `instance_id` may not be available yet — the parser MAY defer and only emit `BLOCK_COMPLETE` if the implementation prefers. For Phase 1, emitting only `BLOCK_COMPLETE` (no `BLOCK_START`) is acceptable; renderer treats it atomically.

### 4.2 FSM states

```
S_PROSE       → reading regular text; pass through as PROSE_DELTA
              | sees "```" prefix → push to S_FENCE_OPEN

S_FENCE_OPEN  → reading "```<lang>\n"
              | resolves to recognized lang → push to S_INSIDE_BLOCK
              | resolves to other lang or eof → flush all buffered as PROSE_DELTA, return to S_PROSE

S_INSIDE_BLOCK → accumulating block content
              | sees "\n```" closer at line start → emit BLOCK_COMPLETE, return to S_PROSE
              | stream ends mid-block → emit BLOCK_ERROR, return to S_PROSE
```

### 4.3 Critical implementation rule (validated in spike F1)

The opener token ` ```dag ` (6 chars) CAN split across delta boundaries:
```
delta[0] = "```"
delta[1] = "dag\n{...}"
```

Therefore:
- The FSM input is the **rolling buffer**, not individual delta strings.
- Maintain a carry-over of at least the last 16 chars when transitioning between deltas.
- Match the opener regex `^```([a-z]+)\s*\n` against (carry + new_delta), not just new_delta.

A reference Go implementation will live in `tether/internal/parse/fence.go` (TBW). The same parser is used in tests against the spike output files.

### 4.4 Block boundary cases

**Case: multiple blocks in same response.** Each gets its own `BLOCK_COMPLETE`. App routes by `instance_id`.

**Case: prose interleaved between blocks.** PROSE_DELTAs emitted between BLOCK_COMPLETEs in document order. Layout default (§5.3) renders inline.

**Case: same `instance_id` appears twice in same response.** Second block UPDATES the component (renderer merges/replaces). This is unusual but legal.

**Case: nested fences.** Not supported. If parser sees ` ``` ` inside a block, treat as part of the block content; the closer is identified by `\n``` ` at line start, ` ``` ` mid-line is content. (cc has not been observed to emit nested fences in spikes — F-prior validation.)

**Case: cc emits unrecognized lang tag (e.g., ` ```python `).** Pass through as raw markdown — renderer formats as standard code block.

**Case: malformed JSON inside recognized block.** Emit `BLOCK_ERROR` with the raw text. Renderer shows the raw fenced text as a code block. NEVER blank-screen the user.

## 5. App rendering contract

### 5.1 Component lifecycle

The renderer maintains a map: `instance_id → ComponentInstance`.

On `BLOCK_COMPLETE`:
```
if instance_id ∉ map:
  mount new Component(kind, payload)
  map[instance_id] = component
elif map[instance_id].kind == kind:
  map[instance_id].update(payload)            // diff-based re-render
else:
  map[instance_id].unmount()
  mount new Component(kind, payload)
  map[instance_id] = component
```

Component instances persist across cc turns within the same session. Closing/reopening the session forces remount (state recovered via JSONL replay; see §8).

### 5.2 Built-in component renderers

Phase 1 ships exactly 4 + 1:

| Block kind | Component | Description |
|---|---|---|
| `dag` | `DagView` | Node graph (recommended: react-flow / dagre). Each node = card with label + status badge + preview thumb + action buttons. |
| `form` | `FormView` | Stack of labeled inputs + submit/cancel buttons. |
| `candidates` | `CandidatePicker` | Grid or list of options, each with preview + select button. |
| `media` | `MediaView` | Single image/video/audio player with optional title + actions. |
| (markdown) | `MarkdownView` | Standard GitHub-flavored markdown renderer for all prose. |

Each component MUST handle:
- Empty/missing optional fields gracefully (no crashes on missing `actions`, `preview`, etc.)
- Status badges for all 6 enum values in DAG
- Loading state when underlying media URL is being fetched
- Failed-to-load state for media URLs (error badge + retry button)

### 5.3 Layout default: inline interleaving

Prose and structured blocks are rendered **in document order** within a single scrolling timeline. A DAG block appears between its preceding and following prose paragraphs. This matches cc's natural output flow and is the simplest to implement.

Phase 2 may introduce alternative layouts (DAG pinned to side panel, prose collapsible, multi-column for form-heavy workflows). Not in v1.

### 5.4 Button → command resolution

When the user activates an action button (or submits a form):
1. App resolves the final command string (interpolating form values if applicable).
2. If `confirm: true`, show modal "Send: `<command>`?" with confirm/cancel.
3. App writes the command + `\n` to the cc stdin stream (via daemon).
4. App shows the command in its own message timeline as if the user had typed it (so the conversation history is coherent).
5. App optimistically marks the source component as "pending update" (subtle visual hint) until the next cc response confirms or supersedes.

Step 4 is important: from cc's perspective, button clicks are indistinguishable from typed input. From the user's perspective, the conversation log shows what was sent. This preserves auditability and matches polyforge-style tooling.

### 5.5 Markdown features required

Validated in spike-dag-streaming T4: cc spontaneously uses markdown tables for status summaries. Renderer MUST support at minimum:
- Headings (h1-h6)
- Paragraphs
- Bullet & numbered lists
- Inline code, fenced code blocks (for unknown langs)
- Tables (GitHub-flavored)
- Bold, italic
- Inline links
- Blockquotes

Any standard library (`remark`, `marked`, `markdown-it`) covers this. Do not write a custom markdown parser.

## 6. The `dag` skill

A markdown skill (polyforge-style) that lives in a tether-distributed plugin: `tether-dag/skills/`. Provides the orchestration vocabulary used by all domain skills.

### 6.1 State file location

```
<workspace-root>/.tether/dags/<instance_id>.json
```

Where `<workspace-root>` is the directory cc was launched from (or where the domain skill chooses; typically the project dir). The `.tether/` subdir is owned by the dag skill and auxiliary tether tooling.

### 6.2 State file schema

```json
{
  "v": 1,
  "instance_id": "<same as in the dag block>",
  "title": "<title>",
  "nodes": [...],
  "edges": [...],
  "created_at": "<ISO8601 with TZ>",
  "updated_at": "<ISO8601 with TZ>"
}
```

The `nodes` and `edges` arrays follow the same schema as `dag` block §3.1. The state file IS the DAG block content + 2 timestamps. When the dag skill emits a block, it reads this file, optionally adds a render-only field (none in Phase 1), and emits.

### 6.3 Slash command surface

All commands take `<instance_id>` as their first positional argument. Domain skills MAY define their own commands that wrap these.

**Lifecycle:**

| Command | Effect |
|---|---|
| `/dag:new <instance_id> --title="<title>"` | Create empty DAG. Error if instance_id already exists. |
| `/dag:show <instance_id>` | Re-emit the current dag block. |
| `/dag:list` | List all DAG instances in workspace as a markdown table. |
| `/dag:delete <instance_id>` | Remove state file. Block in app will detach on next ref. |

**Mutation:**

| Command | Effect |
|---|---|
| `/dag:add-node <instance_id> <node_id> --label="<label>" [--status=pending] [--preview-url=tether://...] [--actions='<json array>']` | Add node. Error if node_id exists. |
| `/dag:add-edge <instance_id> <from_id> <to_id>` | Add edge. Error if either endpoint missing. |
| `/dag:set-status <instance_id> <node_id> <status>` | Change status. Re-emit block on success. |
| `/dag:set-preview <instance_id> <node_id> --type=image --url=tether://blob/<sha> [--sha256=...]` | Update preview field. |
| `/dag:set-meta <instance_id> <node_id> --json='<json object>'` | Replace `meta` field. |
| `/dag:remove-node <instance_id> <node_id>` | Remove node and any edges touching it. |
| `/dag:remove-edge <instance_id> <from_id> <to_id>` | Remove specific edge. |

**Convenience aliases:**

| Command | Effect |
|---|---|
| `/dag:approve <instance_id> <node_id>` | = `set-status approved`. |
| `/dag:reject <instance_id> <node_id>` | = `set-status rejected`. |
| `/dag:start <instance_id> <node_id>` | = `set-status running`. |
| `/dag:done <instance_id> <node_id>` | = `set-status approved`. (Often emitted by domain skills after work completes.) |
| `/dag:fail <instance_id> <node_id> [--reason="<text>"]` | = `set-status failed`; reason stored in `meta.error`. |

### 6.4 Re-emit policy

Every mutation command MUST end by re-emitting the full dag block (with updated state). Spike-dag-streaming validated that cc reliably preserves untouched state across re-emissions.

For multi-step workflows (a single skill turn that adds 5 nodes), the skill SHOULD batch — only emit one final dag block at the end of the turn — to minimize wire traffic and parser work.

### 6.5 Domain skill integration

A domain skill (e.g., `gmi-animation`) integrates with the dag skill in two ways:

**Pattern 1 (preferred): cc invokes dag commands directly.**
The animation skill's markdown contains step-by-step instructions like:
```
1. Run /dag:new storyboard-{project_id} --title="Storyboard for {project_name}"
2. For each frame i in 1..5:
   Run /dag:add-node storyboard-{project_id} f{i} --label="<frame description>"
3. For i in 1..4:
   Run /dag:add-edge storyboard-{project_id} f{i} f{i+1}
4. ...
```

cc reads, executes each slash command in sequence. State file mutated by dag skill. Final block emitted.

**Pattern 2 (allowed): domain skill writes state file directly.**
For high-volume mutations (50+ nodes), invoking 50 slash commands is overhead. Domain skill MAY directly write `<workspace>/.tether/dags/<instance_id>.json` and then run `/dag:show <instance_id>` to emit.

**Anti-pattern: domain skill emits its own dag block.**
Don't. Always go through the dag skill so the state file is canonical.

### 6.6 `allowed-tools` declaration in domain skills

Validated by `spike-q1-nested`: cc's nested skill invocation uses an internal `Skill` tool. Domain skills SHOULD declare it explicitly to ensure availability under stricter permission profiles:

```yaml
---
name: new
description: Create a new animation project
allowed-tools:
  - Skill                                       # for /dag:* invocations
  - Bash(tether-blob-register *)                # for blob registration
  - mcp__gmi_multimodal__image__text_to_image   # provider tools
  - mcp__gmi_multimodal__image__image_to_image
---
```

### 6.7 Latency mitigation: bulk loading

Each nested skill invocation is a full cc inference round-trip. An animation flow with 15 `/dag:*` calls = 15 round-trips ≈ 15-30s of pure DAG-setup latency. If the Phase 1 demo feels sluggish, add a batch command:

```
/dag:bulk-load <instance_id> --json='<full DAG payload>'
```

The skill would parse the JSON, write the state file directly, and emit the block in one call. This is a Phase 1 enhancement, not initial scope — implement only if measurement shows it's needed.

## 7. Media handling — `tether://blob/<sha256>`

### 7.1 Blob registry

The daemon owns a registry at `<workspace-root>/.tether/blobs/registry.json`:

```json
{
  "v": 1,
  "blobs": {
    "<sha256>": {
      "path": "<absolute or workspace-relative path>",
      "mime": "image/png",
      "size_bytes": 145238,
      "registered_at": "<ISO8601>",
      "skill": "<skill name that registered>"
    }
  }
}
```

### 7.2 Registration

When a skill produces a file, it registers it with the daemon before emitting any block referencing the URL. Two interfaces:

**Bash helper (provided as part of tether-dag plugin):**
```bash
tether-blob-register <path>
# prints: tether://blob/<sha256>
```

**Daemon HTTP endpoint (loopback only):**
```
POST http://127.0.0.1:<daemon-port>/blob/register
Content-Type: application/json
{"path": "/abs/path/to/file"}

→ 200 {"sha256": "...", "url": "tether://blob/..."}
```

The daemon computes the sha256 itself (don't trust the skill), then adds to registry.

### 7.3 Resolution (app-side)

When tether app encounters a `tether://blob/<sha256>` URL during render:
1. App requests the blob from daemon over the existing QUIC channel: `GET /blob/<sha256>`.
2. Daemon looks up registry. If unknown sha256 → 404. Otherwise streams the file.
3. App caches by sha256 (content-addressed; immutable).

### 7.4 Security rules

- Daemon NEVER serves a `file://` URL or any path not in registry.
- Registry path field MUST be canonicalized (no `../`, no symlinks outside workspace) before storage.
- File reads MUST be confined to the workspace root or an explicit allowlist (e.g., `~/.cache/gmi-mm/`).
- Registry survives daemon restart (persisted to disk).

### 7.5 Lifecycle

For Phase 1, blobs persist as long as the workspace exists. No GC. (Phase 2: TTL or explicit `tether-blob-unregister`.)

## 8. Replay & app reconnect

Tether v0.1's JSONL replay mechanism plays back the cc session's history when a new client attaches. With dag blocks in the stream, replay behavior:

- The streaming parser runs over the replayed text just like live text.
- Multiple block emissions with the same `instance_id` arrive in sequence; the renderer's component-by-instance_id map ensures the FINAL state wins (intermediate states briefly render but get overwritten — fast enough to be acceptable).
- Prose between blocks renders normally.

**Phase 1: no special replay protocol for dag blocks.** Re-stream + replay-as-live is sufficient.

**Phase 2 candidate:** "fast resume" — daemon reads `.tether/dags/*.json` directly and pushes a `STATE_SNAPSHOT` event to the app on reconnect, before normal JSONL replay starts. This avoids re-rendering intermediate states. Defer.

## 9. Versioning

### 9.1 Schema version (`v` field)

Each block carries `"v": 1`. App's renderer dispatches on `(kind, v)`. Phase 1 only ships v=1 handlers.

### 9.2 Compatibility policy

- Adding optional fields → minor change. Bump nothing. Old clients ignore unknown fields.
- Adding a new enum value (e.g., `nodes[].status: "paused"`) → also minor. Renderer treats unknown enum values as `pending` for safety.
- Removing a field, changing type, breaking enum semantics → major. Bump `v`. Both old and new handlers MAY coexist.

### 9.3 Renderer behavior on unknown `v`

If app sees `v=2` it doesn't have a handler for: emit `BLOCK_ERROR("unsupported version v=2, please update tether app")` and render as code block. Don't try to be heroic.

## 10. Reference plugin: `gmi-animation` (Phase 1 demo)

To validate the spec end-to-end, ship one reference domain plugin:

**Slash commands:**
- `/animation:new <project_id> --prompt="<animation prompt>"`
- `/animation:storyboard <project_id> [--n=5]`
- `/animation:render <project_id> [--frame=<id>]`
- `/animation:stitch <project_id>`

**Internal flow (storyboard):**
1. Use cc parallel tool calling to invoke `gmi-multimodal.image.text_to_image` 5 times with derived per-frame prompts.
2. Each result registers via `tether-blob-register` to get a `tether://blob/<sha>` URL.
3. Construct a DAG via dag skill: 5 nodes f1..f5 with sequential edges, each with image preview.
4. Emit final dag block.

**End-to-end demo target:**
User on a remote tether app: `/animation:new cat-piano --prompt="cat playing piano"` → after ~30s, app shows DAG with 5 image-thumbnail nodes and approve/regen buttons → user clicks "regen" on frame 3 → app sends `/dag:regen` → animation skill regenerates → app updates that one node.

## 11. Open questions / Phase 2 deferrals

These are real but explicitly out of v1 scope.

- **Q1 (RESOLVED 2026-04-27 via `tether/spike-q1-nested/`)**: Nested slash command invocation works. cc has a built-in `Skill` tool that handles it. Sequential multi-invocation preserves order. No workaround needed. **New sub-concern surfaced:** each nested invocation is a separate cc inference round-trip → for an animation setup flow with ~15 dag commands, wall-clock setup is 15-30s and token cost is multiplied. Mitigation: design a batch command `/dag:bulk-load --json='<full DAG>'` for skills that need to populate many nodes/edges at once. Add to spec §6.3 if the demo feels sluggish.
- **Q2**: Skill self-rendered components. Defer to Phase 2; gather user feedback on whether the 4 built-in renderers cover real workflows.
- **Q3**: App-direct skill invocation (skill exposed as MCP server on a separate channel). Defer; current button-to-slash-command path works.
- **Q4**: Permission prompts for parallel MCP calls. If cc fires 5 image-gen calls and each needs approval, mobile UX is bad. Likely solution: configure `--allowed-tools` for animation plugin to skip approval on `mcp__gmi_multimodal__*`. Document in plugin install instructions.
- **Q5**: Plan-mode + dag block coexistence. Untested. May need explicit guidance to skill authors about when to use plan vs DAG.
- **Q6**: Concurrent DAG instances in one workspace (animation + research running side-by-side). Should work via separate `instance_id`s, but UI layout for "multiple top-level views" not specified.
- **Q7**: cc context window pressure when DAG grows large + many tool calls + many turns. Spikes show 30 nodes works fine; 50-100 untested. Watch during implementation.
- **Q8**: Form values containing problematic characters (newlines, single+double quotes, shell metacharacters). The `{field}` interpolation needs a robust escaping rule. Phase 1 uses single-quoted shell escape; Phase 2 may move to a structured stdin protocol (JSON line) instead of slash-command-as-string.

## 12. Implementation checklist

Phase 1 work, ordered by dependency:

- [ ] **Daemon: blob proxy** — registry + HTTP endpoint + resolution. Go, ~3 days.
- [ ] **Daemon: streaming parser** — Go reference impl (`internal/parse/fence.go`); used in tests. ~2 days.
- [ ] **`dag` skill plugin** — markdown skills + state file management + helper bash for blob registration. ~3 days.
- [ ] **`gmi-multimodal` MCP server** — lift prism's GMI adapter into a Python MCP server. ~5 days (depends on GMI SDK familiarity).
- [ ] **`gmi-animation` plugin** — markdown skills using dag + gmi-multimodal. ~3 days.
- [ ] **tether app v0** — connection + replay + parser + 4 renderers + markdown view + button-to-stdin. ~7 days.
- [ ] **Wiring & E2E demo** — install plugins, run animation demo over tether transport. ~2 days.

Total: ~25 person-days for full E2E. Achievable in 4-5 calendar weeks for a focused single developer.

## 13. References

- Companion spec: `2026-04-26-tether-go-quic-design.md` — daemon transport (PTY, JSONL, QUIC, replay).
- Spike validation:
  - `tether/spike-dag-block/FINDINGS.md` — single-turn schema compliance + delta boundary discovery
  - `tether/spike-dag-streaming/FINDINGS.md` — multi-turn instance_id consistency
  - `tether/spike-dag-large/FINDINGS.md` — 30-node drift + tool interleaving (combined report)
  - `tether/spike-q1-nested/FINDINGS.md` — nested skill invocation via cc's built-in `Skill` tool (validates §6.5 Pattern 1)
- Pattern source: `/root/code/golang/gmi/ieops/GMI-marketplace/plugins/polyforge` — reference for markdown-skill + filesystem-state pattern.
- Prism (deprecating): `/root/code/python/agents/prism` — `internal/adapter/gmi/*` will be lifted into `gmi-multimodal` MCP server; rest archived.
