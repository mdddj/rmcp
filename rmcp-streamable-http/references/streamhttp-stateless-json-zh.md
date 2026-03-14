# RMCP Stateless + `json_response`（中文可复制）

## 目标

使用 Streamable HTTP 但走无状态模式，让服务端对请求直接返回 `application/json`（不走 SSE `data:` 包裹）。

关键配置：
- `stateful_mode: false`
- `json_response: true`

## 1) `Cargo.toml`

```toml
[package]
name = "rmcp-streamhttp-stateless-demo"
version = "0.1.0"
edition = "2024"

[dependencies]
anyhow = "1"
axum = "0.8"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tokio = { version = "1", features = ["macros", "rt-multi-thread", "signal"] }
tokio-util = "0.7"

rmcp = { version = "*", features = [
  "server",
  "macros",
  "transport-streamable-http-server"
] }
```

## 2) `src/bin/server.rs`

```rust
use anyhow::Result;
use rmcp::{
    ServerHandler,
    handler::server::router::tool::ToolRouter,
    model::{ServerCapabilities, ServerInfo},
    tool, tool_handler, tool_router,
    transport::streamable_http_server::{
        StreamableHttpServerConfig, StreamableHttpService, session::local::LocalSessionManager,
    },
};
use tokio_util::sync::CancellationToken;

const BIND_ADDRESS: &str = "127.0.0.1:8002";

#[derive(Clone)]
struct EchoService {
    tool_router: ToolRouter<Self>,
}

#[tool_router]
impl EchoService {
    fn new() -> Self {
        Self {
            tool_router: Self::tool_router(),
        }
    }

    #[tool(description = "simple ping")]
    async fn ping(&self) -> String {
        "pong".to_string()
    }
}

#[tool_handler]
impl ServerHandler for EchoService {
    fn get_info(&self) -> ServerInfo {
        ServerInfo::new(ServerCapabilities::builder().enable_tools().build())
            .with_instructions("Stateless JSON response MCP server".to_string())
    }
}

#[tokio::main]
async fn main() -> Result<()> {
    let ct = CancellationToken::new();

    let service = StreamableHttpService::new(
        || Ok(EchoService::new()),
        LocalSessionManager::default().into(),
        StreamableHttpServerConfig {
            stateful_mode: false,
            json_response: true,
            cancellation_token: ct.child_token(),
            ..Default::default()
        },
    );

    let app = axum::Router::new().nest_service("/mcp", service);
    let listener = tokio::net::TcpListener::bind(BIND_ADDRESS).await?;
    println!("listening at http://{BIND_ADDRESS}/mcp");

    axum::serve(listener, app)
        .with_graceful_shutdown(async move {
            tokio::signal::ctrl_c().await.ok();
            ct.cancel();
        })
        .await?;
    Ok(())
}
```

## 3) 运行

```bash
cargo run --bin server
```

## 4) `curl` 验证（重点看 Content-Type）

```bash
curl -i -X POST \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"manual","version":"0.1.0"}}}' \
  http://127.0.0.1:8002/mcp
```

期望：
1. `HTTP/1.1 200 OK`
2. `content-type` 包含 `application/json`
3. 响应体是普通 JSON（不是 `data: ...` 的 SSE 格式）

## 5) 对照：改成 SSE

如果你把 `json_response` 改成 `false`（且仍然 `stateful_mode: false`），响应通常会变回 `text/event-stream`。

如果你把 `stateful_mode` 改成 `true`，则会回到会话语义，通常也是 SSE 路径。

