# Tether Release Notes

## v0.3.2 — 2026-05-11

### Summary

v0.3.2 completes the MCP full-stack by exposing a HTTPS `/mcp` endpoint on the
main port (8898) for external clients (Cursor, Goose) and adding a manual Bearer
token store. It also wires the registry observer so the singleton `*mcp.Server`
reflects tool list changes from supervisor reconnects without requiring client
reconnects.

### Added

- **HTTPS `/mcp` endpoint** — external MCP clients connect to `https://<host>:8898/mcp`
  with `Authorization: Bearer <token>`. The same `*mcp.Server` singleton serves
  both the CC loopback (8899) and the HTTPS endpoint (8898).
- **Token CRUD REST API** — three endpoints protected by JWT cookie middleware:
  - `POST /api/v1/mcp/tokens` — create token; returns raw value once
  - `GET /api/v1/mcp/tokens` — list tokens (id, name, created_at; no hash, no raw)
  - `DELETE /api/v1/mcp/tokens/{id}` — revoke token
- **Token store** — `~/.tether/api-tokens.json` (mode 0600); SHA-256 hashed tokens
  (32 random bytes = 256-bit entropy); atomic fsync write; MaxTokens=100 cap.
- **Audit logging** — slog events on every auth decision:
  - `mcp.apitoken.auth_ok` — id, name, remote_addr (host only), request_id
  - `mcp.apitoken.auth_failed` — reason, remote_addr, request_id
  - `mcp.apitoken.created` / `mcp.apitoken.revoked` — id, name
- **Dynamic tool list** — registry `Observer` pattern: `BuildMCPServer` subscribes
  to `registry.Registry` changes; `AddTool`/`RemoveTools` are called automatically
  on supervisor reconnects. Connected clients receive `notifications/tools/list_changed`
  automatically via the go-sdk.

### Changed

- `/mcp` HTTPS endpoint no longer returns 501.
- `BuildMCPServer(gw, bi)` → `BuildMCPServer(gw, bi, reg)`.

### Deferred to v0.3.3

- OAuth 2.1 PKCE flow (`/oauth/authorize`, `/oauth/token`,
  `/.well-known/oauth-authorization-server`).
- Token TTL / scope / dynamic client registration.
- Web UI for token management.
- Rate limiting on `/mcp` Bearer-validation failures (use a reverse proxy
  `limit_req` in the meantime).

### Migration — v0.3.1 → v0.3.2

1. Restart daemon — no config file changes needed.
2. `~/.tether/api-tokens.json` is created lazily on the first `POST /api/v1/mcp/tokens`.
   Back this file up alongside `~/.tether/access-token`.
3. CC loopback (`127.0.0.1:8899/mcp`) is unchanged; existing `~/.claude/settings.json`
   injection continues to work.

### Connecting Cursor / Goose

```bash
# 1. Obtain a session cookie (complete /auth flow in browser once).
# 2. Create a token:
curl -k -H "Cookie: tether_session=<JWT>" \
     -H "Content-Type: application/json" \
     -d '{"name":"cursor-laptop"}' \
     https://127.0.0.1:8898/api/v1/mcp/tokens

# 3. Copy the returned "token" field into your client config.
#    Cursor MCP settings: { "url": "https://<host>:8898/mcp", "headers": { "Authorization": "Bearer <token>" } }
```

### Security notes

- Tokens are all-scope and long-lived. Use one token per external client.
- Revoke and recreate on suspected leak (`DELETE /api/v1/mcp/tokens/{id}`).
- CSRF protection: `WithOriginGuard` rejects POST/DELETE requests with a
  mismatched `Origin` header. Native clients (no `Origin` header) are unaffected.
- Browser-based MCP clients are currently blocked by `WithOriginGuard`. This is
  intentional for v0.3.2; the OAuth 2.1 flow in v0.3.3 resolves it.

### Smoke test

```bash
TETHER_HOST=127.0.0.1 TETHER_PORT=8898 TETHER_JWT=<cookie> \
  ./scripts/v032-smoke.sh
```

---

## v0.3.1 — 2026-05-11

MCP loopback endpoint, builtin workspace tools, CC settings injection.
See PR #66.

## v0.3.0 — 2026-05-10

MCP Host Core: permission migration, registry, gateway, supervisor.
See PR #64, #65.

## v0.2.0 — 2026-05-09

Auth token gate, JWT cookie, opencode provider, multi-tab attach.

## v0.1.0 — 2026-05-09

Initial: HTTP/3 + TCP dual-listener, WebTransport chat/shell, permission UI,
session lock, workspace registry.
