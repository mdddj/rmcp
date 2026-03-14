# Streamable HTTP Troubleshooting (rmcp)

## Contents
1. `MissingSessionIdInResponse`
2. `SessionExpired (HTTP 404)`
3. `ReservedHeaderConflict`
4. `Unsupported MCP-Protocol-Version`
5. `json_response` vs `stateful_mode`
6. SSE priming and retries
7. Auth failures (`401` / `403`)

## 1. `MissingSessionIdInResponse`

Symptom:
- Client fails after initialize with `MissingSessionIdInResponse`.

Cause:
- Server response did not include `mcp-session-id` and client is not in stateless-allowed mode.

Fix:
1. Ensure server is actually Streamable HTTP and initialize path is correct (`/mcp`).
2. Keep server `stateful_mode: true` for session-based flows.
3. If intentionally stateless, set client config `allow_stateless = true` (default is true in current source).

## 2. `SessionExpired (HTTP 404)`

Symptom:
- Existing session starts failing with `404`.

Cause:
- Server discarded the session.

Fix:
1. Re-initialize and re-send initialized notification.
2. Prefer built-in worker behavior; current transport includes transparent re-initialization logic.
3. If the app keeps long idle sessions, reconnect proactively.

## 3. `ReservedHeaderConflict`

Symptom:
- Client returns error like `ReservedHeaderConflict("accept")`.

Cause:
- These headers are reserved and cannot be set in `custom_headers`:
  - `accept`
  - `mcp-session-id`
  - `last-event-id`

Note:
- `mcp-protocol-version` is allowed because transport negotiation uses it after initialize.

## 4. `Unsupported MCP-Protocol-Version`

Symptom:
- Server returns `400 Bad Request` mentioning protocol version.

Cause:
- Incoming `MCP-Protocol-Version` header is present but not one of known versions.

Fix:
1. Use protocol value negotiated from initialize result.
2. Avoid hardcoding a stale version string across mixed server/client versions.

## 5. `json_response` vs `stateful_mode`

Behavior in current `rmcp` server transport:
1. `stateful_mode: true`
- Stream uses SSE behavior; `json_response` is effectively ignored.
2. `stateful_mode: false` + `json_response: true`
- Server can return plain `application/json` instead of SSE framing.
3. `stateful_mode: false` + `json_response: false`
- Server still returns `text/event-stream`.

## 6. SSE Priming and Retries

Expected behavior in stateful mode:
1. Priming event at stream start (`retry` hint typically present).
2. Priming event can be sent before stream close with custom retry interval.

Implication:
- Client reconnection logic should honor SSE retry hints and last event IDs when available.

## 7. Auth Failures (`401` / `403`)

`401 Unauthorized`:
- Missing/invalid token.
- Verify `.auth_header("token")` and server middleware expectations.

`403 Forbidden` with scope issue:
- Token valid but insufficient scope.
- Check `WWW-Authenticate` header scope hints and request upgraded scope.

