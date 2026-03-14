# RMCP Streamable HTTP + Bearer 鉴权（中文可复制）

## 目标

实现一个需要 `Authorization: Bearer <token>` 的 Streamable HTTP MCP 服务端，并用客户端携带 token 成功调用工具。

默认约定：
- 监听地址：`127.0.0.1:8001`
- MCP 路径：`/mcp`
- 演示 token：`demo-token`

## 1) `Cargo.toml`

```toml
[package]
name = "rmcp-streamhttp-auth-demo"
version = "0.1.0"
edition = "2024"

[dependencies]
anyhow = "1"
axum = "0.8"
http = "1"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tokio = { version = "1", features = ["macros", "rt-multi-thread", "signal"] }
tokio-util = "0.7"
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "fmt"] }

rmcp = { version = "*", features = [
  "server",
  "client",
  "macros",
  "reqwest",
  "transport-streamable-http-server",
  "transport-streamable-http-client-reqwest"
] }
```

## 2) `src/bin/server.rs`

```rust
use std::sync::Arc;

use anyhow::Result;
use axum::{
    Router,
    extract::State,
    http::{HeaderMap, Request, StatusCode},
    middleware::{self, Next},
    response::Response,
    routing::get,
};
use rmcp::{
    ServerHandler,
    handler::server::{router::tool::ToolRouter, wrapper::Parameters},
    model::{ServerCapabilities, ServerInfo},
    schemars, tool, tool_handler, tool_router,
    transport::streamable_http_server::{
        StreamableHttpServerConfig, StreamableHttpService, session::local::LocalSessionManager,
    },
};
use tokio::sync::Mutex;
use tokio_util::sync::CancellationToken;

const BIND_ADDRESS: &str = "127.0.0.1:8001";
const DEMO_TOKEN: &str = "demo-token";

#[derive(Clone)]
struct TokenStore {
    token: Arc<String>,
}

impl TokenStore {
    fn new(token: &str) -> Self {
        Self {
            token: Arc::new(token.to_string()),
        }
    }
}

fn extract_bearer_token(headers: &HeaderMap) -> Option<&str> {
    headers
        .get("authorization")
        .and_then(|v| v.to_str().ok())
        .and_then(|s| s.strip_prefix("Bearer "))
}

async fn auth_middleware(
    State(store): State<TokenStore>,
    headers: HeaderMap,
    request: Request<axum::body::Body>,
    next: Next,
) -> Result<Response, StatusCode> {
    match extract_bearer_token(&headers) {
        Some(token) if token == store.token.as_str() => Ok(next.run(request).await),
        _ => Err(StatusCode::UNAUTHORIZED),
    }
}

#[derive(Debug, serde::Deserialize, schemars::JsonSchema)]
struct AddRequest {
    value: i32,
}

#[derive(Clone)]
struct CounterService {
    value: Arc<Mutex<i32>>,
    tool_router: ToolRouter<Self>,
}

#[tool_router]
impl CounterService {
    fn new() -> Self {
        Self {
            value: Arc::new(Mutex::new(0)),
            tool_router: Self::tool_router(),
        }
    }

    #[tool(description = "Increment by 1")]
    async fn increment(&self) -> String {
        let mut v = self.value.lock().await;
        *v += 1;
        v.to_string()
    }

    #[tool(description = "Add integer")]
    async fn add(&self, Parameters(AddRequest { value }): Parameters<AddRequest>) -> String {
        let mut v = self.value.lock().await;
        *v += value;
        v.to_string()
    }
}

#[tool_handler]
impl ServerHandler for CounterService {
    fn get_info(&self) -> ServerInfo {
        ServerInfo::new(ServerCapabilities::builder().enable_tools().build())
            .with_instructions("Bearer protected Streamable HTTP MCP server".to_string())
    }
}

#[tokio::main]
async fn main() -> Result<()> {
    let ct = CancellationToken::new();

    let mcp_service = StreamableHttpService::new(
        || Ok(CounterService::new()),
        LocalSessionManager::default().into(),
        StreamableHttpServerConfig {
            cancellation_token: ct.child_token(),
            ..Default::default()
        },
    );

    let token_store = TokenStore::new(DEMO_TOKEN);

    let protected_mcp = Router::new()
        .nest_service("/mcp", mcp_service)
        .layer(middleware::from_fn_with_state(
            token_store.clone(),
            auth_middleware,
        ));

    let app = Router::new()
        .route("/health", get(|| async { "ok" }))
        .route("/token", get(|| async { format!("Bearer {}", DEMO_TOKEN) }))
        .merge(protected_mcp);

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

## 3) `src/bin/client.rs`

```rust
use std::collections::HashMap;

use anyhow::Result;
use http::{HeaderName, HeaderValue};
use rmcp::{
    ServiceExt,
    model::{CallToolRequestParams, ClientCapabilities, ClientInfo, Implementation},
    transport::{
        StreamableHttpClientTransport,
        streamable_http_client::StreamableHttpClientTransportConfig,
    },
};

#[tokio::main]
async fn main() -> Result<()> {
    let mut headers = HashMap::new();
    headers.insert(
        HeaderName::from_static("x-client-name"),
        HeaderValue::from_static("demo-auth-client"),
    );

    let config = StreamableHttpClientTransportConfig::with_uri("http://127.0.0.1:8001/mcp")
        .auth_header("demo-token")
        .custom_headers(headers);

    let transport = StreamableHttpClientTransport::from_config(config);
    let client_info = ClientInfo::new(
        ClientCapabilities::default(),
        Implementation::new("auth-client", "0.1.0"),
    );
    let client = client_info.serve(transport).await?;

    let result = client
        .call_tool(
            CallToolRequestParams::new("increment")
                .with_arguments(serde_json::json!({}).as_object().cloned().unwrap()),
        )
        .await?;
    println!("increment result = {result:#?}");

    client.cancel().await?;
    Ok(())
}
```

## 4) 运行

```bash
cargo run --bin server
```

另一个终端：

```bash
cargo run --bin client
```

## 5) `curl` 验证

无 token（应 401）：

```bash
curl -i -X POST \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"manual","version":"0.1.0"}}}' \
  http://127.0.0.1:8001/mcp
```

有 token（应成功）：

```bash
curl -i -X POST \
  -H "Authorization: Bearer demo-token" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"manual","version":"0.1.0"}}}' \
  http://127.0.0.1:8001/mcp
```

