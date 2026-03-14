---
name: rmcp-streamable-http
description: Build, debug, and explain MCP Streamable HTTP implementations with the modelcontextprotocol/rust-sdk (`rmcp`) in Rust. Use when requests mention rmcp/rust-sdk, MCP over HTTP or SSE, `streamable_http`, Axum/Hyper MCP server setup, Streamable HTTP client setup, session handling (`mcp-session-id`), auth headers, or migration from stdio transport to HTTP transport.
---

# RMCP Streamable HTTP

Use this skill to implement production-ready Streamable HTTP server/client code with official `rmcp` APIs, and to troubleshoot transport/session/protocol issues.

## Quick Intake
1. Identify target:
- `server` (Axum or Hyper)
- `client` (reqwest transport)
- `both` (end-to-end local demo)
2. Identify auth requirement:
- none
- bearer token middleware
- OAuth flow integration
3. Confirm URL and endpoint:
- default endpoint should be `/mcp`
- examples in this skill use `http://127.0.0.1:8000/mcp`

## Workflow

1. Select cargo features from [streamhttp-recipes.md](references/streamhttp-recipes.md).
2. Start from the minimal template in references instead of ad-hoc code.
3. Keep transport wiring explicit:
- server: `StreamableHttpService::new(...)` + `nest_service("/mcp", service)`
- client: `StreamableHttpClientTransport::from_uri(...)` or `from_config(...)`
4. Run local verification commands from references.
5. If it fails, use [streamhttp-troubleshooting.md](references/streamhttp-troubleshooting.md) before changing architecture.

## Hard Rules

1. Reuse official names/import paths exactly from upstream examples unless the user asks to refactor.
2. Do not invent custom HTTP header names for MCP session/protocol handling.
3. Do not set reserved headers through `custom_headers`:
- `accept`
- `mcp-session-id`
- `last-event-id`
4. Allow `mcp-protocol-version` to be negotiated via initialize result.
5. For stateful server mode (`stateful_mode: true`), expect SSE and session semantics; do not force JSON response behavior.

## Output Contract

When implementing for users, always output:
1. `Cargo.toml` dependency/features snippet.
2. Complete compile-ready Rust file(s), not pseudo-code.
3. Run command(s).
4. Verification command(s) (`curl` or client call).
5. A short troubleshooting section with probable failure modes.

## References

- [streamhttp-recipes.md](references/streamhttp-recipes.md): feature matrix, minimal server/client templates, run commands.
- [streamhttp-troubleshooting.md](references/streamhttp-troubleshooting.md): protocol/session/header/config pitfalls and fixes.
- [streamhttp-quickstart-zh.md](references/streamhttp-quickstart-zh.md): 中文快速上手，包含可直接复制运行的 server/client 示例。
- [streamhttp-auth-bearer-zh.md](references/streamhttp-auth-bearer-zh.md): 中文 Bearer 鉴权示例（服务端中间件 + 客户端 token）。
- [streamhttp-stateless-json-zh.md](references/streamhttp-stateless-json-zh.md): 中文 Stateless + `json_response` 示例（非 SSE JSON 响应）。

## Upstream Baseline

Base guidance on official `modelcontextprotocol/rust-sdk` files on `main` branch (retrieved 2026-03-14):
- `examples/servers/src/counter_streamhttp.rs`
- `examples/servers/src/counter_hyper_streamable_http.rs`
- `examples/servers/src/simple_auth_streamhttp.rs`
- `examples/clients/src/streamable_http.rs`
- `crates/rmcp/src/transport/streamable_http_server/tower.rs`
- `crates/rmcp/src/transport/streamable_http_client.rs`
- `crates/rmcp/src/transport/common/reqwest/streamable_http_client.rs`
