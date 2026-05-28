# tether — MCP integration design spec

**Date:** 2026-05-10
**Author:** wxk + Claude
**Status:** Draft — revised 2026-05-10 after SDK spike; targets v0.3 (v0.1.0 + v0.2.0 already shipped)
**Companion spec:** `marketplace-ws/.workspace/memory/local/2026-05-10-mcp-integration-spec.md` (polyforge side)
**Target version:** tether v0.3. v0.1.0 + v0.2.0 (auth + opencode + multi-tab attach) shipped 2026-05-10. v0.3 is current next milestone.

---

## 0. What this document is

The tether-side counterpart to polyforge-v2's MCP integration spec. Defines:

1. Why tether becomes an MCP host (not just a passive Claude Code wrapper)
2. How tether exposes itself as an MCP server (the strategic value)
3. The Go architecture for MCP host runtime
4. Forward-compat changes that should land early in v0.3 to avoid mid-implementation breakage (originally proposed for v0.1; v0.1+v0.2 already shipped without them)
5. Integration points with existing tether internals
6. Effort estimate, risks, acceptance criteria

The polyforge-v2 spec covers what plugins look like as MCP servers. This spec
covers how tether **runs** those servers and how it **becomes** one.

## 1. Motivation — why tether is an MCP host, not just a user

Claude Code already supports MCP. A user can configure `.mcp.json` and
Claude Code will manage MCP server processes itself. So the obvious question:
**why does tether need to add MCP host runtime?**

Five capabilities are not achievable any other way:

| Capability | Without tether MCP host | With tether MCP host |
|---|---|---|
| **Per-Task MCP server scoping** | All MCP servers always loaded | Activate only servers relevant to active task |
| **Workspace-aware env injection** | MCP servers don't know which task is active | tether substitutes `${TASK_ROOT}` / `${TASK_WORKSPACE}` per task |
| **Unified permission gating** | Two parallel permission systems (Claude Code hook + nothing for non-Claude clients) | Single gateway for all tool calls regardless of source |
| **Audit trail / event sourcing** | Tool calls vanish; no record | All tool calls written to Task history JSONL |
| **Remote MCP exposure** | mobile / web clients cannot use MCP | tether's `/mcp` endpoint via Streamable HTTP (MCP spec 2025-11-25), mobile-reachable |

The fifth point is the strategic differentiation. **tether becomes a remote
AI workspace accessible by any MCP client** — Cursor, Goose, Zed, future
mobile-native AI assistants. This is the "AI-generalized VS Code" vision (D-19)
made concrete: workspace-as-a-service over MCP.

## 2. Architectural overview

tether is **both** an MCP host (managing internal MCP servers) and an MCP
server (exposing aggregated tools to external clients):

```
Claude Code (LLM, subprocess of tether)
    ↕ stream-json (existing, D-05a)
tether (Go binary)
    ├── Claude Code subprocess management (existing)
    │
    ├── MCP host runtime [NEW in v0.3]
    │   ├── Subprocess MCP servers
    │   │   ├── polyforge-core-mcp (stdio)
    │   │   ├── polyforge-coding-mcp (stdio)
    │   │   └── jira-mcp (stdio, community or self-hosted)
    │   ├── Tool registry / router
    │   ├── Permission gateway (generalized from D-05b)
    │   └── Audit hook → Task history
    │
    └── /mcp endpoint via Streamable HTTP [NEW in v0.3]
        ↕
        Cursor / Goose / mobile app
```

Claude Code's `.mcp.json` (auto-injected by tether) points to a single endpoint:

```json
{
  "mcpServers": {
    "tether": {
      "type": "http",
      "url": "https://localhost:8898/mcp"
    }
  }
}
```

tether is an **aggregating MCP server** — Claude Code sees one unified MCP
endpoint; tether internally fans out to all the underlying servers + adds
its own built-in workspace tools.

## 3. Relationship to existing tether design

### 3.1 What the v0.1 design already gets right (no change needed)

| Existing design element | Why it's MCP-friendly |
|---|---|
| **AgentProvider abstraction (D-17a)** | "Which LLM drives the conversation" is orthogonal to MCP. Provider stays as-is. |
| **Wire types via tygo (D-22)** | Adding MCP tool-call wire types is straightforward. |
| **EventBus fan-out (v0.2 §3)** | Exactly the pump infrastructure MCP host needs for tool-call broadcast. |
| **Single-port HTTP/3 mux (§10.B)** | `/mcp` slots into existing routing table. |
| **Skill plugin overlay (D-20)** | Manages Markdown Skill files (different concept from MCP servers). Coexists. |
| **Auth middleware (v0.2 §1)** | MCP endpoints reuse the same JWT cookie auth. |

### 3.2 What needs forward-compat adjustment

Three changes were originally recommended for v0.1 to avoid breakage during v0.3
MCP integration. Since v0.1.0 + v0.2.0 already shipped without these, they should
land **early in v0.3** (before MCP host work begins) to avoid mid-implementation
breaking changes. Combined cost: ~170 LOC + 4 doc paragraphs, ~1 day.

#### Change 1: Generalize permission API path & schema

**Current (D-05b §3, shipped in v0.1.0):**
```
POST /api/v1/agent/permission/request    {session_id, tool_name, tool_input, hook_event_name}
POST /api/v1/agent/permission/<id>/decide {allow, message}
```

**Recommended adjustment (early v0.3):**
```
POST /api/v1/permission/request          {source, session_id, tool_name, tool_input, source_meta}
POST /api/v1/permission/<id>/decide      {allow, message}
```

Migration: keep `/api/v1/agent/permission/*` as alias for one minor version (v0.3.x)
to avoid breaking any existing clients; new `/api/v1/permission/*` is canonical.

Changes:
- Drop `/agent/` from path (permission is cross-cutting, not agent-specific)
- Add `source` field (initial value `"claude_hook"`; later `"mcp:polyforge-coding"` etc.)
- Move cc-specific `hook_event_name` into `source_meta` opaque object

**Rationale:** v0.3 MCP gateway will gate MCP tool calls. Without this change,
either the path/schema breaks (forcing frontend rewrite) or v0.3 forks a parallel
`/api/v1/mcp/permission/*` endpoint (duplicating UI logic).

#### Change 2: Move permhook to internal/permission/

**Current (shipped in v0.1.0):**
```
internal/agent/permhook/    # contains hook source + manager
```

**Recommended adjustment (early v0.3):**
```
internal/permission/        # cross-cutting permission concern
├── manager.go              # Generic PermRequest / Decision (used by cc hook + future MCP gateway)
├── http.go                 # /api/v1/permission/* handlers
└── cchook/                 # cc-specific embedded hook source
    └── main.go             # original permhook/main.go
```

**Cost:** Pure directory move. ~10 file imports change.

**Rationale:** v0.3 `internal/mcp/gateway/` will import `internal/permission/`.
Doing it via `internal/agent/permhook/` requires either circular imports or
awkward indirection.

#### Change 3: Reserve URL paths and directories

Zero-cost forward reservation:

| Resource | Action (early v0.3) |
|---|---|
| URL path `/mcp` | mux returns 501 until implemented; do not use for anything else |
| URL path `/api/v1/mcp/*` | Same |
| Directory `internal/mcp/` | Create with `README.md` stating "MCP host integration entry point" |
| `internal/agent/provider.go` godoc | Add note: "MCP host integration is orthogonal to AgentProvider" |

### 3.3 What v0.2.0 (shipped) already provides for MCP

v0.2.0 shipped 2026-05-10 with three features that happen to enable MCP work:

| v0.2 feature (shipped) | How it helps MCP |
|---|---|
| Auth middleware | MCP endpoints reuse same JWT cookie |
| EventBus fan-out | MCP tool-call events broadcast via same bus |
| opencode provider | Validates AgentProvider abstraction is real, not just one impl |

No v0.2 design change was needed. v0.2 shipped independent of MCP and
naturally prepared the ground for v0.3.

## 4. Go package layout (v0.3)

```
tether/
└── internal/
    ├── server/                    # existing
    │   └── mcp_handler.go         # NEW: /mcp endpoint (SSE)
    │
    ├── agent/                     # existing
    │   └── claude_provider.go     # MODIFIED: inject .mcp.json pointing to self
    │
    ├── permission/                # MOVED from internal/agent/permhook/ (Change 2)
    │   ├── manager.go             # generalized in v0.1
    │   ├── http.go
    │   └── cchook/
    │
    └── mcp/                       # NEW (entire tree)
        ├── host/
        │   ├── manager.go         # MCP server lifecycle
        │   ├── server.go          # single MCP server abstraction
        │   ├── stdio.go           # stdio transport
        │   ├── streamable_http.go # Streamable HTTP transport (MCP 2025-11-25)
        │   └── lifecycle.go       # spawn / health / restart
        │
        ├── protocol/
        │   ├── jsonrpc.go         # JSON-RPC 2.0
        │   ├── messages.go        # initialize / tools/list / tools/call
        │   └── progress.go        # progress notifications / cancellation
        │
        ├── registry/
        │   ├── tools.go           # tool name → server routing
        │   └── namespace.go       # collision handling (forced prefixes)
        │
        ├── gateway/
        │   ├── permission.go      # delegates to internal/permission/
        │   └── audit.go           # writes tool-call events directly to Task history JSONL
        │
        ├── builtin/                # tether's own workspace tools
        │   ├── workspace.go       # workspace_read_file / list_files
        │   ├── shell.go           # workspace_run_shell (wraps PTY)
        │   └── state.go           # workspace_get_state
        │
        └── config.go              # MCP config schema
```

## 5. Component breakdown

### 5.1 Manager — MCP server lifecycle

```go
// internal/mcp/host/manager.go
type Manager struct {
    servers map[string]*Server  // server name → process
    mu      sync.RWMutex

    taskCtx     *task.Context        // active Task context for env injection
    history     *history.Logger      // audit logger
    permission  *permission.Manager  // permission gateway
}

type Server struct {
    Name      string
    Transport Transport          // stdio | http_sse
    Tools     []protocol.Tool    // populated post-initialize
    Status    Status             // starting | ready | crashed | shutdown

    pendingCalls sync.Map        // request-id → response channel
    cancelFunc   context.CancelFunc
}

func (m *Manager) Start(ctx context.Context, cfg *Config) error {
    for name, serverCfg := range cfg.Servers {
        envVars := m.expandTaskVars(serverCfg.Env)  // ${TASK_ROOT} etc.
        s, err := m.spawn(ctx, name, serverCfg, envVars)
        if err != nil { return fmt.Errorf("spawn %s: %w", name, err) }
        if err := s.Initialize(ctx); err != nil {
            return fmt.Errorf("initialize %s: %w", name, err)
        }
        m.servers[name] = s
    }
    return nil
}

func (m *Manager) expandTaskVars(env map[string]string) map[string]string {
    out := make(map[string]string)
    for k, v := range env {
        v = strings.ReplaceAll(v, "${TASK_ROOT}", m.taskCtx.WorktreeRoot)
        v = strings.ReplaceAll(v, "${TASK_WORKSPACE}", m.taskCtx.WorkspaceRoot)
        v = strings.ReplaceAll(v, "${TASK_ID}", m.taskCtx.TaskID)
        out[k] = v
    }
    return out
}
```

### 5.2 Stdio transport

```go
// internal/mcp/host/stdio.go
type StdioTransport struct {
    cmd    *exec.Cmd
    stdin  io.WriteCloser
    stdout *bufio.Reader

    sendQueue chan protocol.Message
    recvQueue chan protocol.Message
}

func (t *StdioTransport) Start(ctx context.Context) error {
    if err := t.cmd.Start(); err != nil { return err }
    go t.readLoop(ctx)
    go t.writeLoop(ctx)
    go t.monitorLifecycle(ctx)
    return nil
}

func (t *StdioTransport) readLoop(ctx context.Context) {
    decoder := json.NewDecoder(t.stdout)
    for {
        var msg protocol.Message
        if err := decoder.Decode(&msg); err != nil {
            t.recvQueue <- protocol.Message{Error: err}
            return
        }
        select {
        case t.recvQueue <- msg:
        case <-ctx.Done(): return
        }
    }
}
```

### 5.3 JSON-RPC protocol

Follows MCP standard. Initialize handshake exchanges capabilities; subsequent
calls are tool listing and tool invocation:

```go
// internal/mcp/protocol/messages.go
// Pin to "2025-11-25" (current spec; introduced Streamable HTTP).
const ProtocolVersion = "2025-11-25"

type Initialize struct {
    ProtocolVersion string       `json:"protocolVersion"`
    Capabilities    Capabilities `json:"capabilities"`
    ClientInfo      ClientInfo   `json:"clientInfo"`
}

type Tool struct {
    Name        string                 `json:"name"`
    Description string                 `json:"description"`
    InputSchema map[string]interface{} `json:"inputSchema"`
}

type ToolCallRequest struct {
    Name      string          `json:"name"`
    Arguments json.RawMessage `json:"arguments"`
}

type ToolCallResponse struct {
    Content []Content `json:"content"`
    IsError bool      `json:"isError"`
}
```

### 5.4 Tool registry / router

```go
// internal/mcp/registry/tools.go
type Registry struct {
    tools map[string]*ToolEntry  // tool name → server + descriptor
    mu    sync.RWMutex
}

type ToolEntry struct {
    ServerName string
    Tool       protocol.Tool
}

func (r *Registry) Register(serverName string, tools []protocol.Tool) error {
    r.mu.Lock()
    defer r.mu.Unlock()
    for _, t := range tools {
        if existing, ok := r.tools[t.Name]; ok {
            return fmt.Errorf("tool %q already registered by %s", t.Name, existing.ServerName)
        }
        r.tools[t.Name] = &ToolEntry{ServerName: serverName, Tool: t}
    }
    return nil
}
```

### 5.5 Permission gateway — generalized

```go
// internal/mcp/gateway/permission.go
func (g *Gateway) CallTool(
    ctx context.Context,
    toolName string,
    args json.RawMessage,
) (*protocol.ToolCallResponse, error) {

    // 1. Permission check (delegates to internal/permission/)
    decision, err := g.permission.Check(ctx, &permission.Request{
        Source:   "mcp:" + g.registry.LookupServer(toolName),  // e.g. "mcp:polyforge-coding"
        ToolName: toolName,
        Args:     args,
        TaskID:   g.taskCtx.TaskID,
    })
    if err != nil { return nil, err }
    if !decision.Allow {
        g.history.AppendEvent("tool_call_denied", map[string]any{
            "tool": toolName, "source": "mcp", "reason": decision.Reason,
        })
        return errResponse(decision.Reason), nil
    }

    // 2. Audit start
    callID := ulid.New()
    g.history.AppendEvent("tool_call_started", map[string]any{
        "call_id": callID, "tool": toolName, "args": args,
    })

    // 3. Route to actual server
    entry, ok := g.registry.Lookup(toolName)
    if !ok { return nil, ErrToolNotFound }
    server := g.host.GetServer(entry.ServerName)

    resp, err := server.CallTool(ctx, toolName, args)

    // 4. Audit end
    if err != nil {
        g.history.AppendEvent("tool_call_failed", map[string]any{
            "call_id": callID, "error": err.Error(),
        })
        return nil, err
    }
    g.history.AppendEvent("tool_call_completed", map[string]any{
        "call_id": callID, "result_size": len(resp.Content),
    })

    return resp, nil
}
```

The `permission.Request` shape is set in v0.1 (Change 1 in §3.2). v0.3 just
populates `Source` differently — `"claude_hook"` for the existing hook flow,
`"mcp:<servername>"` for MCP gateway.

### 5.6 /mcp endpoint — tether as MCP server

Uses **Streamable HTTP transport** (MCP spec 2025-11-25). Single POST endpoint: each request
is one HTTP call; response is plain JSON or SSE depending on `Accept` header. This replaces
the deprecated 2024-11-05 "HTTP+SSE dual-endpoint" model.

Auth scope: JWT cookie (v0.2). This covers internal clients (Claude Code subprocess, tether
mobile app). External third-party clients (Cursor, Goose) require OAuth 2.1 — deferred to
v0.4 (see §7.5).

```go
// internal/server/mcp_handler.go
// Implements MCP Streamable HTTP transport (protocol 2025-11-25).
// Each POST carries one JSON-RPC request; response is JSON or SSE.
func (s *Server) handleMCP(w http.ResponseWriter, r *http.Request) {
    if r.Method != http.MethodPost {
        http.Error(w, "method not allowed", 405)
        return
    }

    // Auth via JWT cookie (internal clients only; see §7.5 for OAuth 2.1 scope)
    sess, err := s.authSession(r)
    if err != nil { http.Error(w, "unauthorized", 401); return }

    var req protocol.JSONRPCRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "bad request", 400)
        return
    }

    resp := s.dispatchMCP(r.Context(), sess, &req)

    // Respond with SSE if client signals streaming support, plain JSON otherwise.
    if strings.Contains(r.Header.Get("Accept"), "text/event-stream") {
        w.Header().Set("Content-Type", "text/event-stream")
        w.Header().Set("Cache-Control", "no-cache")
        data, _ := json.Marshal(resp)
        fmt.Fprintf(w, "data: %s\n\n", data)
        if f, ok := w.(http.Flusher); ok { f.Flush() }
    } else {
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(resp)
    }
}

func (s *Server) dispatchMCP(ctx context.Context, sess *Session, req *protocol.JSONRPCRequest) *protocol.JSONRPCResponse {
    switch req.Method {
    case "initialize":
        return s.handleInitialize(req)
    case "tools/list":
        return &protocol.JSONRPCResponse{
            ID: req.ID,
            Result: map[string]any{"tools": s.mcpHost.Registry().ListAll()},
        }
    case "tools/call":
        var params protocol.ToolCallRequest
        json.Unmarshal(req.Params, &params)
        result, err := s.mcpHost.Gateway().CallTool(ctx, params.Name, params.Arguments)
        if err != nil { return errorResp(req.ID, err) }
        return &protocol.JSONRPCResponse{ID: req.ID, Result: result}
    }
    return errorResp(req.ID, ErrMethodNotFound)
}
```

### 5.7 Built-in workspace tools

tether exposes its own capabilities as MCP tools:

```go
// internal/mcp/builtin/workspace.go
func RegisterWorkspaceTools(r *registry.Registry, ws *workspace.Manager) {
    r.RegisterBuiltin("workspace_read_file", protocol.Tool{
        Name:        "workspace_read_file",
        Description: "Read a file from the active workspace",
        InputSchema: schemaForReadFile(),
    }, func(ctx context.Context, args json.RawMessage) (*protocol.ToolCallResponse, error) {
        var p struct{ Path string `json:"path"` }
        json.Unmarshal(args, &p)
        content, err := ws.ReadFile(p.Path)
        if err != nil { return nil, err }
        return textResult(content), nil
    })

    r.RegisterBuiltin("workspace_run_shell", protocol.Tool{...},
        // Wraps existing session.PTY, gets multi-attach broadcast for free
        ...)
}
```

These tools become accessible to **any MCP client** connecting to tether's
`/mcp` endpoint — Cursor, Goose, mobile apps, future tools.

## 6. Integration points with existing tether

| Existing component | How MCP host integrates |
|---|---|
| `internal/server/server.go` single-port mux | Add `/mcp` route to existing HTTP/3 + TCP routing |
| `internal/agent/claude_provider.go` | Inject auto-generated `.mcp.json` into Claude Code subprocess env, pointing to localhost `/mcp` |
| `internal/permission/` (post-Change 2) | Gateway directly calls; reuses existing UI flow |
| `internal/session/` (PTY) | builtin shell tool wraps PTY, gets multi-attach broadcast for free |
| `internal/workspace/` REST API | builtin workspace tools wrap REST handlers (don't duplicate logic) |
| `internal/skill/` overlay | Per-Task scenario decides which MCP servers to spawn |

## 7. Tricky implementation points

### 7.1 Bidirectional notifications

stdio MCP servers may push:
- `notifications/log` (logging)
- `notifications/progress` (long-running operations)
- `notifications/tools/list_changed` (server reconfigured)

The tether reader goroutine must distinguish:
- Message with `id` field → response to a pending request (lookup `pendingCalls`)
- Message with no `id` → notification, route to handler

### 7.2 Cancellation

When Claude Code aborts a tool call, tether must propagate cancellation:

```go
func (s *Server) CallTool(ctx context.Context, name string, args json.RawMessage) (*protocol.ToolCallResponse, error) {
    callID := nextID()
    ch := make(chan *protocol.ToolCallResponse, 1)
    s.pendingCalls.Store(callID, ch)
    defer s.pendingCalls.Delete(callID)

    s.transport.Send(buildToolCall(callID, name, args))

    select {
    case resp := <-ch:
        return resp, nil
    case <-ctx.Done():
        s.transport.Send(protocol.Cancelled{RequestID: callID})
        return nil, ctx.Err()
    }
}
```

### 7.3 Server crash recovery

If a stdio MCP server process exits unexpectedly:
1. Mark `server.Status = crashed`
2. Fail all pending calls with `server_crashed` error
3. Append `mcp_server_crashed` event to Task history
4. Restart with exponential backoff (max 3 attempts)
5. On restart success, re-run `initialize` + `tools/list` (tools may have changed)
6. Re-register tools in registry (handle name changes)

### 7.4 Tool name collisions

Two MCP servers both expose `commit`?

Strategy: **forced namespace prefix derived from server name**:

```
polyforge-core    → pf2_core_*
polyforge-coding  → pf2_coding_*
jira              → jira_*
```

Registration fails with an error if collision detected. No silent winner-takes-all.

### 7.5 Authentication isolation

`/mcp` runs on the **same port as the existing HTTP/3+WebTransport server**, but over
**H2/H1** (Streamable HTTP for MCP compatibility; WebTransport stays for chat/shell).

**v0.3 auth scope — internal clients only (JWT cookie):**
- Claude Code subprocess → JWT cookie injected by tether at subprocess spawn
- tether mobile app → JWT cookie from existing v0.2 session

**External clients (Cursor, Goose, third-party MCP) → OAuth 2.1 required, deferred to v0.4.**
MCP spec 2025-11-25 mandates OAuth 2.1 + PKCE for remote HTTP servers. Implementing tether
as an Authorization Server (or proxying to an IdP) is non-trivial. Attempting it in v0.3
alongside the MCP host runtime would overload the milestone. Decision: v0.3 `/mcp` is
explicitly internal-only; §13.2 acceptance criteria scoped accordingly (no Cursor/Goose
end-to-end in v0.3).

Permission UI actor distinction (still applies for internal clients):
- "Claude Code requests Bash with command X" — cc hook flow
- "Mobile app requests workspace_read_file with path Y" — MCP `/mcp` flow

The `source` field added in Change 1 (§3.2) makes this distinction first-class.

## 8. Configuration

MCP servers configured per-workspace, optionally per-Task:

```jsonc
// .workspace/config.json (excerpt)
{
  "mcp": {
    "servers": {
      "polyforge-core": {
        "command": ["pf-v2-mcp"],
        "args": [],
        "env": {
          "POLYFORGE_WORKSPACE": "${TASK_WORKSPACE}"
        }
      },
      "polyforge-coding": {
        "command": ["pf2-coding-mcp"],
        "env": {
          "POLYFORGE_TASK_ROOT": "${TASK_ROOT}"
        }
      },
      "jira": {
        "command": ["npx", "@anthropic/mcp-server-jira"],
        "env": {
          "JIRA_URL": "https://gmicloud.atlassian.net",
          "JIRA_TOKEN": "${env:JIRA_API_TOKEN}"
        }
      }
    }
  }
}
```

`${TASK_ROOT}`, `${TASK_WORKSPACE}`, `${TASK_ID}` resolved by tether at server
spawn time. `${env:VAR}` resolves to host environment variable.

## 9. Per-task lifecycle

When user starts working on a task:
1. tether's MCP host reads task scenario config
2. Spawns relevant MCP servers (lazy: only when first tool call for that scope)
3. Injects task-specific env vars at spawn

When task is paused:
- MCP servers terminated cleanly (initialize → close → exit)
- State preserved in workspace files (per files-as-truth)

When task is resumed:
- Servers respawned with same env vars
- Replay state restored from history (tether doesn't replay tool call results;
  it just restarts the server fresh)

## 10. Go library dependencies

| Concern | Library | Notes |
|---|---|---|
| MCP SDK (Go) — host side | [`github.com/modelcontextprotocol/go-sdk`](https://github.com/modelcontextprotocol/go-sdk) v1.0.0 | Official SDK, maintained with Google, API frozen (v1.0.0 released 2026-04-30). Handles stdio transport, protocol handshake, tools/list, tools/call. 1,400+ known dependents. |
| MCP SDK (Go) — `/mcp` server side | `go-sdk` first; fallback [`github.com/mark3labs/mcp-go`](https://github.com/mark3labs/mcp-go) | Verify at implementation time whether official SDK's Streamable HTTP server transport is complete. mcp-go has native Streamable HTTP and is the fallback. |
| JSON-RPC | Provided by `go-sdk` | No custom code needed |
| Subprocess management | `os/exec` (stdlib) | tether already uses |
| HTTP (Streamable HTTP transport) | `net/http` (stdlib) | tether already uses |
| Supervisor pattern | Custom (~100 LOC) | Restart with backoff |

External dependencies introduced: 1-2 (one or both MCP SDK packages above).

## 11. Effort estimate

| Module | LOC | Effort |
|---|---|---|
| ~~`internal/mcp/protocol/`~~ | ~~500~~ | ~~3 days~~ — **replaced by `go-sdk`** |
| `internal/mcp/host/` (manager + transport) | ~1,200 | 5 days |
| `internal/mcp/registry/` | ~300 | 1 day |
| `internal/mcp/gateway/` (permission + audit) | ~400 | 2 days |
| `internal/mcp/builtin/` (workspace tools) | ~600 | 2 days |
| `internal/server/mcp_handler.go` (`/mcp`) | ~350 | 1.5 days |
| `internal/agent/` mods (`.mcp.json` injection) | ~150 | 1 day |
| Tests (unit + integration + E2E) | ~2,000 | 7 days |
| **Total** | **~5,000 LOC** | **~2.5 weeks** |

Plus forward-compat changes (§3.2): ~250 LOC, 1 day (Note: Change 1 involves dual-schema
handling for old and new permission endpoints; cost revised from original ~170 LOC). Schedule
as the first work item of v0.3 (before MCP host implementation begins).

## 12. Risks & mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Official Go SDK Streamable HTTP server transport incomplete | Medium | Fallback: use `mark3labs/mcp-go` for `/mcp` server side; verify at v0.3 kickoff (30 min spike) |
| MCP server crash floods restart attempts | Medium | Exponential backoff + max 3 retries + circuit-breaker; alert via UI after threshold |
| Auth token in `/mcp` URL leaks via logs | Medium | JWT in cookie, not URL. |
| Permission UI overwhelmed by MCP tool requests | Medium | Per-source rate limit + "always allow this tool from this source" checkbox |
| `${TASK_ROOT}` substitution leaks across tasks | High | Manager spawns fresh server per task transition; never reuse subprocess across tasks |
| External clients (Cursor/Goose) blocked on OAuth 2.1 | Low (v0.3 scoped to internal) | Explicitly deferred to v0.4; v0.3 acceptance criteria excludes external clients |
| Tool name collisions in user config | Low | Forced namespace prefixes; registration fails fast |

## 13. Acceptance criteria (v0.3)

### 13.1 MCP host (incoming, tether → MCP servers)

- [ ] tether spawns configured MCP servers from `.workspace/config.json` MCP section
- [ ] Stdio transport: bidirectional JSON-RPC, including notifications and progress
- [ ] Streamable HTTP transport (MCP 2025-11-25): same protocol semantics, alternate transport
- [ ] Server crash → automatic restart with exponential backoff (max 3 attempts)
- [ ] Server crash event written to Task history JSONL
- [ ] Tool registry rejects name collisions at registration time
- [ ] `${TASK_ROOT}` / `${TASK_WORKSPACE}` / `${TASK_ID}` correctly substituted
- [ ] Per-task MCP server lifecycle: spawn on task start, terminate on pause

### 13.2 MCP server (outgoing, internal clients → tether)

v0.3 scope: internal clients only (Claude Code subprocess + tether mobile app via JWT cookie).
External third-party clients (Cursor, Goose) require OAuth 2.1 — deferred to v0.4.

- [ ] `/mcp` endpoint accepts Streamable HTTP POST requests (protocol 2025-11-25) after JWT auth passes
- [ ] tether's built-in workspace tools available (`workspace_read_file`, etc.)
- [ ] Aggregated tools from internal MCP servers also exposed
- [ ] tether mobile app can connect and invoke tools end-to-end via `/mcp`
- [ ] `tools/list` returns all aggregated tools with namespace prefixes

### 13.3 Permission gateway

- [ ] All MCP tool calls go through `/api/v1/permission/*` (same as cc hook)
- [ ] `source` field correctly populated (`"mcp:<servername>"`)
- [ ] Permission denial returns structured error to MCP client
- [ ] All tool calls (allowed and denied) written to Task history

### 13.4 Integration

- [ ] Claude Code's auto-injected `.mcp.json` points to tether's `/mcp`
- [ ] End-to-end: Claude Code calls `pf2_commit` → tether MCP gateway →
  permission UI → user approves → polyforge-coding-mcp executes git commit →
  result returns to Claude Code
- [ ] All tool calls visible in Task history (`history.jsonl`)
- [ ] tether `doctor` reports MCP server health for each configured server

### 13.5 Forward-compat (early v0.3 prerequisite)

- [ ] `/api/v1/permission/*` paths added (canonical); `/api/v1/agent/permission/*` kept as alias for v0.3.x
- [ ] `permission_request` envelope contains `source` field
- [ ] `internal/permission/` package exists; `cchook` is a subpackage
- [ ] `/mcp` and `/api/v1/mcp/*` reserved (return 501 until implemented later in v0.3)
- [ ] `internal/mcp/README.md` placeholder present

## 14. Connection to polyforge-v2 spec

The polyforge-v2 spec (`marketplace-ws/.workspace/memory/local/2026-05-10-mcp-integration-spec.md`)
defines what runs inside tether's MCP host:

| polyforge spec section | tether spec correspondence |
|---|---|
| §4 Three-server topology | §5.1 Manager spawns these three servers |
| §6 Token budget per tool | §5.5 Permission gateway can enforce per-call budgets |
| §11 Plugin manifest mcp_servers | §8 tether's MCP config consumes these |
| §13 Share drivers as community MCP | §8 jira-mcp via `npx @anthropic/mcp-server-jira` |
| §14 Migration phases | §13 acceptance criteria phased by stage |

The two specs are independent migrations of the same vision. The tether side
can be implemented with a single dummy MCP server (e.g., `polyforge-core-mcp`
stub that returns hardcoded tools) before polyforge-v2's full migration completes.

Recommended sequencing (current state: v0.1.0 + v0.2.0 shipped 2026-05-10):
1. **Early v0.3**: forward-compat changes from §3.2 (~1 day)
2. **v0.3**: tether MCP host + `/mcp` endpoint + builtin workspace tools.
   polyforge-v2 may still be on legacy dispatch protocol — tether's MCP host
   simply doesn't have polyforge MCP servers configured yet.
3. **v0.3.x or v0.4**: polyforge-v2 ships its MCP servers; tether config
   extended to spawn them. Stage 1-2 of polyforge migration begins.

## 15. Open questions

1. ~~**Go MCP SDK status (2026-05)**~~ **RESOLVED**: Use
   `github.com/modelcontextprotocol/go-sdk` v1.0.0 (official, API frozen). For
   `/mcp` server-side Streamable HTTP, verify official SDK coverage at v0.3 kickoff;
   fallback to `mark3labs/mcp-go` if incomplete.
2. ~~**WebTransport vs HTTP+SSE for `/mcp`**~~ **RESOLVED**: `/mcp` uses
   **Streamable HTTP** (MCP spec 2025-11-25, H2/H1). WebTransport stays for
   chat/shell only. HTTP+SSE dual-endpoint model (2024-11-05) is deprecated and
   not implemented.
3. **Tool call timeout policy**: MCP doesn't mandate timeouts. tether should
   set a sensible default (30s? 5min?) per tool category, configurable.
4. **Streaming tool results**: Some tools may want to stream output (long
   shell commands, large file reads). MCP supports progress notifications.
   When to use? (Lean: opt-in per tool; default is single response.)
5. **MCP server discovery beyond config**: Should tether autodiscover MCP
   servers from a registry (community marketplace)? Out of scope for v0.3.
6. **Custom transports (Unix socket, named pipe)**: stdio + Streamable HTTP cover
   most cases. Defer custom transports to v1.0+ as needed.
7. **OAuth 2.1 for external clients (Cursor/Goose)**: Explicitly deferred to v0.4.
   tether must act as Authorization Server or proxy to an IdP. Not in v0.3 scope.

## 16. Vocabulary

| Term | Meaning in this spec |
|---|---|
| **MCP host** | Process managing MCP server lifecycles (tether, in v0.3+) |
| **MCP server** | Process exposing tools (polyforge-core-mcp, etc.; also tether's `/mcp`) |
| **MCP client** | Process calling tools (Claude Code, Cursor, Goose, mobile app) |
| **Aggregating proxy** | tether's role: many MCP servers internal, single `/mcp` external |
| **Stdio transport** | MCP over subprocess stdin/stdout |
| **Streamable HTTP transport** | MCP over HTTP single POST endpoint per MCP spec 2025-11-25 (replaces deprecated 2024-11-05 HTTP+SSE dual-endpoint model) |
| **Forward-compat change** | An early-v0.3 adjustment to v0.1/v0.2-shipped surface, made before MCP host work begins |
| **Built-in tools** | tether's own capabilities exposed as MCP tools (vs. those from spawned servers) |

---

*Last updated 2026-05-10 (revised after SDK spike). Key changes from initial draft:
transport layer updated to Streamable HTTP (spec 2025-11-25); MCP SDK resolved to
`go-sdk` v1.0.0; external OAuth 2.1 deferred to v0.4; v0.3 `/mcp` scoped to internal
clients only. Targets v0.3 (post v0.1.0 + v0.2.0 ship).*
