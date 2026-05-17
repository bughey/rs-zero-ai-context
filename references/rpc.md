# RPC Patterns

rs-zero RPC 基于 tonic。AI 生成 RPC 服务时应明确 unary 与 streaming 的韧性边界。

## Workflow

1. 写 `.proto`。
2. 运行 `rzcli rpc gen`。
3. 在 `src/logic/*` 补业务逻辑。
4. 在 tonic server 外层接入 `RpcServerLayerStack`。
5. 补测试、README、RPC.md。



## Handler / Logic Layout

`rzcli rpc gen` 生成的 RPC 项目采用两层结构：

- `src/handler/*`：实现 tonic service trait，处理 `Request<T>` / `Response<T>` / `Status`，并调用 logic。
- `src/logic/*`：承载业务逻辑，方法特征遵循 `.proto` 的 message 形态，例如 `SayHelloRequest -> SayHelloResponse`。
- `src/handler/*` 文件名以 `_handler.rs` 结尾；`src/logic/*` 文件名以 `_logic.rs` 结尾。
- `src/lib.rs`：保留 service struct、`server_layer_stack()` 和直接 message 级调用入口。

示例：

```rust
pub async fn say_hello(
    request: proto::SayHelloRequest,
) -> Result<proto::SayHelloResponse, Status> {
    Ok(proto::SayHelloResponse {
        message: format!("hello, {}", request.name),
    })
}
```

handler 中只做 tonic 适配：

```rust
let reply = logic::hello_logic::say_hello(request.into_inner()).await?;
Ok(Response::new(reply))
```

## RPC Runtime Configuration

`rzcli rpc gen` 生成的 RPC 项目应保留 `etc/<service>.toml`，不要把 listen address 写死在代码里。入口默认使用框架级 `RpcServiceConfig`，不再生成自定义 `src/config.rs`。

配置模式：

```toml
name = "hello-rpc"
mode = "pro"

[server]
host = "127.0.0.1"
port = 50051
timeout_ms = 5000
health = true

[log]
mode = "console"
encoding = "plain"
level = "info"
path = "logs"
rotation = "daily"
compress = false
keep_days = 0
max_backups = 0
max_size_mb = 0

[middlewares]
resilience = true
streaming = true
```

启动时使用：

```rust
use rs_zero::core::{RpcServiceConfig, emit_config_warnings, init_tracing, shutdown_signal};

let app = RpcServiceConfig::load("etc/hello-rpc", "HELLO_RPC")?;
let _ = init_tracing(app.log_config());
emit_config_warnings(&app.validate_features());
let server_config = app.rpc_server_config()?;
```

环境变量用 `__` 覆盖嵌套字段，例如 `HELLO_RPC__SERVER__PORT=50052`。启动后调用 `emit_config_warnings(&app.validate_features())`，当 `[middlewares]` 请求的能力缺少 Cargo feature 时只打印 warning。RPC 服务实现应接收 `app.rpc_server_config()?` 后再构造 service。

## Unary Resilience

优先选择 Tower-first 路径：

- `RpcServerLayerStack`：server-side 推荐入口，挂到 `tonic::transport::Server::builder().layer(...)`，统一处理 timeout、concurrency、breaker、shedder、Redis limiter、metadata 读取和 RPC INFO 日志。
- `RpcClientBuilder`：client-side 推荐入口，按 `RpcClientConfig` 建立 channel，并配合 `request_id_interceptor()` / `trace_context_interceptor()` 传播调用链上下文。
- `RequestContext`：手写边界需要显式注入 headers / metadata / extensions 时使用。

兼容路径：

- `RpcResilienceLayer::run_unary_with_metadata`：未挂 server layer、但已有 `tonic::Request<T>` metadata 时使用。
- `RpcResilienceLayer::run_unary`：无入站 metadata 或仅处理内部消息时使用。
- `RpcUnaryResilienceLayer`：底层 Tower layer，主要给高级手写 service 或旧生成项目使用。

覆盖能力：

- timeout。
- concurrency。
- breaker。
- shedder。
- status mapping。
- metrics/tracing helper。

## Server-side Metadata Rule

新生成代码默认不再要求业务 skeleton 手写 metadata 保留逻辑。推荐做法是在 tonic server 外层挂载 `RpcServerLayerStack`：

```rust
use rs_zero::rpc::{RpcServerConfig, RpcServerLayerStack};

let config = RpcServerConfig::production_defaults("hello-rpc", addr);
let layer = RpcServerLayerStack::new(config.clone()).into_layer();
let service = HelloService::new(config);
let tonic_service = hello_service_server::HelloServiceServer::new(service);

tonic::transport::Server::builder()
    .layer(layer)
    .add_service(tonic_service)
    .serve(addr)
    .await?;
```

在该路径下，trait method 可以直接 `request.into_inner()`，因为 `x-request-id` 与 `traceparent` 已被外层 Tower layer 读取并用于观测和韧性。

只有在未挂 `RpcServerLayerStack` 的手写兼容路径中，才需要在业务处理前保留 metadata：

```rust
let (metadata, _extensions, req) = request.into_parts();
let reply = self.resilience
    .run_unary_with_metadata("GetUser", &metadata, || async move {
        self.handle_get_user(req).await
    })
    .await?;
```

## RPC + Model Integration

RPC 服务调用 model repository 的推荐结构：

- `.proto` 只定义 RPC contract。
- SQL schema 生成 entity/repository/cache skeleton。
- RPC service struct 持有 repository 和 `RpcServerConfig`。
- server 外层挂 `RpcServerLayerStack`，业务 unary 方法只负责从 request message 调用 repository。
- repository 错误显式映射成 `tonic::Status`。

示例：

```rust
use rs_zero::rpc::{RpcServerConfig, RpcServerLayerStack};
use tonic::{Request, Response, Status};
use user_model::repository::SqlxUsersRepository;

#[derive(Clone)]
pub struct UserService {
    users: SqlxUsersRepository,
    config: RpcServerConfig,
}

impl UserService {
    pub fn new(users: SqlxUsersRepository, config: RpcServerConfig) -> Self {
        Self { users, config }
    }

    pub fn server_layer_stack(&self) -> RpcServerLayerStack {
        RpcServerLayerStack::new(self.config.clone())
    }

    async fn handle_get_user(&self, req: GetUserRequest) -> Result<GetUserReply, Status> {
        let user = self
            .users
            .find_by_id(req.id)
            .await
            .map_err(|error| Status::internal(error.to_string()))?
            .ok_or_else(|| Status::not_found("user not found"))?;

        Ok(GetUserReply {
            id: user.id,
            email: user.email,
            name: user.name,
        })
    }
}

#[tonic::async_trait]
impl user_service_server::UserService for UserService {
    async fn get_user(
        &self,
        request: Request<GetUserRequest>,
    ) -> Result<Response<GetUserReply>, Status> {
        let reply = self.handle_get_user(request.into_inner()).await?;
        Ok(Response::new(reply))
    }
}
```

服务启动时挂 layer：

```rust
let service = UserService::new(users, rpc_config);
let layer = service.server_layer_stack().into_layer();
let tonic_service = user_service_server::UserServiceServer::new(service);

tonic::transport::Server::builder()
    .layer(layer)
    .add_service(tonic_service)
    .serve(addr)
    .await?;
```

不要同时在方法内调用 `run_unary` 又在 server 外层挂完整 `RpcServerLayerStack`；二者二选一，默认选 Tower-first layer。

## Error Mapping

- `not found` -> `Status::not_found`。
- `validation` -> `Status::invalid_argument`。
- `database unavailable` -> `Status::unavailable` 或 `Status::internal`，按业务边界选择。
- 不要把 repository 原始错误直接暴露给客户端。

## Repository Boundaries

- DTO/message 和 model/entity 不必共用类型。
- 显式 mapper 更安全。
- cache-aside 只适合主键/唯一索引读取。
- streaming 不要长时间持有 repository 或 DB pool 句柄。


## RPC Logs and Call Chain

Generated RPC projects should keep `rpc`, `resil` and `observability` enabled. Unary services generated by `rzcli rpc gen` use `handler` / `logic` layout and expose `server_layer_stack()`; mount it on tonic `Server::builder().layer(...)` to emit INFO-level `rpc unary observed` logs when observability is enabled.

Expected fields:

- `rpc.service` / `service`.
- `rpc.method` / `method`, for example `say_hello`.
- `route`, same as method for unified HTTP/RPC search.
- `request_id`.
- `traceparent`.
- `trace_id` / `span_id` when a valid trace context exists.
- `code`, the gRPC status code.

If logs only show `h2::*` DEBUG frames, the service is logging transport internals rather than the rs-zero RPC observation layer. Prefer `RUST_LOG=info,h2=warn,hyper=warn,tower=warn` for application logs.

For API -> RPC chains:

1. Keep RPC channels in API `AppState`; expose `state.hello_rpc()`-style methods that create tonic clients with `request_id_interceptor()` to propagate `x-request-id`.
2. In normal REST handlers, do not call `Channel::connect`, `with_interceptor(...)`, or manually copy `HeaderMap` into tonic metadata; REST metrics middleware scopes the current request id as task-local context for `request_id_interceptor()`.
3. Use `with_rpc_request_id(...)` only for background jobs or async boundaries outside the HTTP handler scope.
4. Use `trace_context_interceptor()` when `otlp` is enabled to propagate W3C TraceContext.
5. On the RPC server side, mount `RpcServerLayerStack`; trait methods can then use `request.into_inner()` safely because metadata is handled by the outer Tower layer.
6. Use `RpcResilienceLayer::run_unary_with_metadata` or `observe_rpc_unary_with_metadata` only for hand-written compatibility paths without `RpcServerLayerStack`.
7. Without OTLP or propagated `traceparent`, logs cannot show a real cross-service `trace_id`; correlate by `request_id` instead.

## Streaming Observation

可用组合：

- `ObservedRecvStream`。
- `record_stream_send`。
- `run_observed_stream`。
- `RpcStreamingObserver`。

边界：

- streaming wrapper 负责 send/recv/finish/timeout 观察边界。
- 它不是完整 tonic stream interceptor 替代品。
- 不应宣称 streaming 已与 unary 达到完全同级自动保护。

## Client Notes

- deadline propagation 需要显式配置。
- retry 只应用于幂等或确认可重试方法。
- discovery/balancer 需要对应 feature 和运行环境。

## RequestContext and RPC Client Builder

`RequestContext` 用于手写边界显式传递低基数字段和调用链上下文：

```rust
use rs_zero::layer::context::RequestContext;
use rs_zero::rpc::RpcClientBuilder;

let context = RequestContext::new("hello-api", "http", "/hello/{name}", "GET")
    .with_request_id("req-1")
    .with_traceparent("00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01");

let builder = RpcClientBuilder::new(rpc_config);
let mut request = tonic::Request::new(HelloRequest { name });
builder.inject_request_context(&context, &mut request)?;
```

普通 REST handler 中通常不需要手动构造 `RequestContext`；REST metrics middleware 会自动设置 task-local request id，`request_id_interceptor()` 会读取并写入 RPC metadata。

## Verification

```bash
cargo test -p rs-zero --features rpc,resil --test rpc_production_integration
cargo test -p rs-zero-cli --test goctl_rpc_compat
```
