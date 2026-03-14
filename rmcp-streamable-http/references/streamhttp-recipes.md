# Streamable HTTP Recipes (rmcp / rust-sdk)

## Contents
1. Feature sets
2. Minimal Axum server
3. Minimal Hyper server
4. Minimal client (reqwest transport)
5. Client with auth/custom headers
6. Run and verify

## 1. Feature Sets

Use `rmcp` with explicit features:

```toml
[dependencies]
rmcp = { version = "*", features = [
  "server",
  "macros",
  "transport-streamable-http-server"
] }
tokio = { version = "1", features = ["macros", "rt-multi-thread", "signal"] }
axum = "0.8"
tokio-util = "0.7"
anyhow = "1"
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "fmt"] }
```

Client side (reqwest-backed Streamable HTTP transport):

```toml
[dependencies]
rmcp = { version = "*", features = [
  "client",
  "reqwest",
  "transport-streamable-http-client-reqwest"
] }
tokio = { version = "1", features = ["macros", "rt-multi-thread"] }
serde_json = "1"
anyhow = "1"
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "fmt"] }
http = "1"
```

If you need auth helpers from rmcp, include `"auth"` feature.

## 2. Minimal Axum Server

```rust
use rmcp::transport::streamable_http_server::{
    StreamableHttpServerConfig, StreamableHttpService, session::local::LocalSessionManager,
};
use tokio_util::sync::CancellationToken;

mod common; // your ServerHandler implementation
use common::counter::Counter;

const BIND_ADDRESS: &str = "127.0.0.1:8000";

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let ct = CancellationToken::new();

    let service = StreamableHttpService::new(
        || Ok(Counter::new()),
        LocalSessionManager::default().into(),
        StreamableHttpServerConfig {
            cancellation_token: ct.child_token(),
            ..Default::default()
        },
    );

    let router = axum::Router::new().nest_service("/mcp", service);
    let listener = tokio::net::TcpListener::bind(BIND_ADDRESS).await?;

    axum::serve(listener, router)
        .with_graceful_shutdown(async move {
            tokio::signal::ctrl_c().await.ok();
            ct.cancel();
        })
        .await?;
    Ok(())
}
```

## 3. Minimal Hyper Server

```rust
use hyper_util::{
    rt::{TokioExecutor, TokioIo},
    server::conn::auto::Builder,
    service::TowerToHyperService,
};
use rmcp::transport::streamable_http_server::{
    StreamableHttpService, session::local::LocalSessionManager,
};

mod common;
use common::counter::Counter;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let service = TowerToHyperService::new(StreamableHttpService::new(
        || Ok(Counter::new()),
        LocalSessionManager::default().into(),
        Default::default(),
    ));

    let listener = tokio::net::TcpListener::bind("[::1]:8080").await?;
    loop {
        let io = tokio::select! {
            _ = tokio::signal::ctrl_c() => break,
            accept = listener.accept() => TokioIo::new(accept?.0),
        };
        let service = service.clone();
        tokio::spawn(async move {
            let _ = Builder::new(TokioExecutor::default())
                .serve_connection(io, service)
                .await;
        });
    }
    Ok(())
}
```

## 4. Minimal Client (reqwest transport)

```rust
use rmcp::{
    ServiceExt,
    model::{CallToolRequestParams, ClientCapabilities, ClientInfo, Implementation},
    transport::StreamableHttpClientTransport,
};

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let transport = StreamableHttpClientTransport::from_uri("http://127.0.0.1:8000/mcp");

    let client_info = ClientInfo::new(
        ClientCapabilities::default(),
        Implementation::new("my-streamhttp-client", "0.1.0"),
    );
    let client = client_info.serve(transport).await?;

    let tools = client.list_tools(Default::default()).await?;
    println!("tools = {tools:#?}");

    let result = client
        .call_tool(
            CallToolRequestParams::new("increment")
                .with_arguments(serde_json::json!({}).as_object().cloned().unwrap()),
        )
        .await?;
    println!("result = {result:#?}");

    client.cancel().await?;
    Ok(())
}
```

## 5. Client With Auth / Custom Headers

```rust
use std::collections::HashMap;

use http::{HeaderName, HeaderValue};
use rmcp::transport::{
    StreamableHttpClientTransport,
    streamable_http_client::StreamableHttpClientTransportConfig,
};

let mut headers = HashMap::new();
headers.insert(
    HeaderName::from_static("x-tenant-id"),
    HeaderValue::from_static("tenant-a"),
);

let config = StreamableHttpClientTransportConfig::with_uri("http://127.0.0.1:8000/mcp")
    .auth_header("demo-token") // token only; transport sends Bearer
    .custom_headers(headers);

let transport = StreamableHttpClientTransport::from_config(config);
```

Do not pass these via `custom_headers`: `accept`, `mcp-session-id`, `last-event-id`.

## 6. Run and Verify

Server:

```bash
cargo run -p mcp-server-examples --example servers_counter_streamhttp
```

Client:

```bash
cargo run -p mcp-client-examples --example clients_streamable_http
```

Manual initialize check:

```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"manual","version":"0.1.0"}}}' \
  http://127.0.0.1:8000/mcp
```

