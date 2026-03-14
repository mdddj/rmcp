# RMCP Streamable HTTP 中文快速上手（可直接复制）

## 目标

在一个 Rust 项目里同时放一个 Streamable HTTP MCP 服务端和一个客户端：
- 服务端地址：`http://127.0.0.1:8000/mcp`
- 客户端调用工具：`increment`、`get_value`

## 1) 创建项目与目录

```bash
cargo new rmcp-streamhttp-demo
cd rmcp-streamhttp-demo
mkdir -p src/bin
```

## 2) `Cargo.toml`

把依赖改成下面这样（完整示例）：

```toml
[package]
name = "rmcp-streamhttp-demo"
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

## 3) `src/bin/server.rs`

```rust
use std::sync::Arc;

use anyhow::Result;
use axum::routing::get;
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
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

const BIND_ADDRESS: &str = "127.0.0.1:8000";

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

    #[tool(description = "Increment counter by 1")]
    async fn increment(&self) -> String {
        let mut v = self.value.lock().await;
        *v += 1;
        v.to_string()
    }

    #[tool(description = "Add any integer value")]
    async fn add(&self, Parameters(AddRequest { value }): Parameters<AddRequest>) -> String {
        let mut v = self.value.lock().await;
        *v += value;
        v.to_string()
    }

    #[tool(description = "Get current counter value")]
    async fn get_value(&self) -> String {
        let v = self.value.lock().await;
        v.to_string()
    }
}

#[tool_handler]
impl ServerHandler for CounterService {
    fn get_info(&self) -> ServerInfo {
        ServerInfo::new(ServerCapabilities::builder().enable_tools().build())
            .with_instructions("A tiny Streamable HTTP counter MCP server".to_string())
    }
}

#[tokio::main]
async fn main() -> Result<()> {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| "info".to_string().into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    let ct = CancellationToken::new();

    let mcp_service = StreamableHttpService::new(
        || Ok(CounterService::new()),
        LocalSessionManager::default().into(),
        StreamableHttpServerConfig {
            cancellation_token: ct.child_token(),
            ..Default::default()
        },
    );

    let app = axum::Router::new()
        .route("/health", get(|| async { "ok" }))
        .nest_service("/mcp", mcp_service);

    let listener = tokio::net::TcpListener::bind(BIND_ADDRESS).await?;
    tracing::info!("server listening on http://{BIND_ADDRESS}/mcp");

    axum::serve(listener, app)
        .with_graceful_shutdown(async move {
            tokio::signal::ctrl_c().await.ok();
            ct.cancel();
        })
        .await?;

    Ok(())
}
```

## 4) `src/bin/client.rs`

```rust
use anyhow::Result;
use rmcp::{
    ServiceExt,
    model::{CallToolRequestParams, ClientCapabilities, ClientInfo, Implementation},
    transport::StreamableHttpClientTransport,
};

#[tokio::main]
async fn main() -> Result<()> {
    let transport = StreamableHttpClientTransport::from_uri("http://127.0.0.1:8000/mcp");

    let client_info = ClientInfo::new(
        ClientCapabilities::default(),
        Implementation::new("demo-client", "0.1.0"),
    );
    let client = client_info.serve(transport).await?;

    let tools = client.list_tools(Default::default()).await?;
    println!("tools = {tools:#?}");

    let r1 = client
        .call_tool(
            CallToolRequestParams::new("increment")
                .with_arguments(serde_json::json!({}).as_object().cloned().unwrap()),
        )
        .await?;
    println!("increment = {r1:#?}");

    let r2 = client
        .call_tool(
            CallToolRequestParams::new("add")
                .with_arguments(
                    serde_json::json!({ "value": 5 })
                        .as_object()
                        .cloned()
                        .unwrap(),
                ),
        )
        .await?;
    println!("add(5) = {r2:#?}");

    let r3 = client
        .call_tool(
            CallToolRequestParams::new("get_value")
                .with_arguments(serde_json::json!({}).as_object().cloned().unwrap()),
        )
        .await?;
    println!("get_value = {r3:#?}");

    client.cancel().await?;
    Ok(())
}
```

## 5) 运行

先开服务端：

```bash
cargo run --bin server
```

再开另一个终端运行客户端：

```bash
cargo run --bin client
```

## 6) 用 `curl` 手工检查 initialize

```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"manual-client","version":"0.1.0"}}}' \
  http://127.0.0.1:8000/mcp
```

## 7) 常见坑

1. 404：确认路径是 `/mcp`，不是 `/`.
2. 保留请求头冲突：不要在 `custom_headers` 里塞 `accept`、`mcp-session-id`、`last-event-id`。
3. 协议版本错误：不要手工硬编码旧版本，优先使用 initialize 协商结果。
4. 如果你要无状态行为，再考虑 `stateful_mode: false` 与 `json_response` 组合。

