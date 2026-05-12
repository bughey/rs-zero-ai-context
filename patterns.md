# rs-zero Code Patterns

## API Spec

```api
type CreateUserRequest {
    Email string `json:"email" validate:"required,email"`
    Name string `json:"name" validate:"required,min=1"`
}

type CreateUserResponse {
    Id i64 `json:"id"`
}

@server(group: user, prefix: /api)
service user-api {
    @handler createUser
    post /users (CreateUserRequest) returns (CreateUserResponse)
}
```

规则：

- route、handler、request、response 都从 `.api` 生成。
- 修改 API 先改 `.api`，再运行 `rzcli api validate` 和 `rzcli api gen`。

Handler 入参生成示例：

```api
type HelloReq {
    Name string `path:"name"`
}

type HelloReply {
    Message string `json:"message"`
}

service hello-api {
    @handler HelloHandler
    get /hello/:name (HelloReq) returns (HelloReply)
}
```

生成签名应包含 `Path(req): Path<types::HelloReq>`，不要改成无参 handler。`query` / `form` request 使用 `Query(req)`，默认 body request 使用 `Json(req)`。

JWT 示例：

```api
@server(jwt: Auth)
service user-api {
    @handler profileHandler
    get /api/profile returns (ProfileReply)
}
```

生成项目会读取 `JWT_AUTH_SECRET` 或 `[auth].jwt_secret`，并读取 `JWT_AUTH_EXPIRES` 或 `[auth].jwt_expires`。`jwt_expires` 单位为秒，默认 `7200`。

## API + Model Pattern

REST handler 调用 model repository 时，使用 `AppState` 注入 repository，不在 handler 中直接散落 SQL。

```rust
use axum::{Json, extract::State};
use rs_zero::rest::ApiResponse;

use crate::{state::AppState, types};

pub async fn create_user_handler(
    State(state): State<AppState>,
    Json(req): Json<types::CreateUserRequest>,
) -> ApiResponse<types::CreateUserResponse> {
    let _ = (&state, req);
    ApiResponse::success(types::CreateUserResponse::default())
}
```

完整模板见 `templates/API-MODEL.md`。


## API + RPC Pattern

API handler 调 RPC 时，RPC client 必须放入 `AppState`，handler 用 `State<AppState>` 取得 client；不要在 handler 内每次 `Channel::connect`，也不要在 handler 里手写 `HeaderMap` 到 tonic metadata 的 request id 注入。

```rust
use axum::{extract::{Path, State}};
use rs_zero::rest::ApiResponse;

use crate::{state::AppState, types};

pub async fn hello_handler(
    State(state): State<AppState>,
    Path(req): Path<types::HelloReq>,
) -> ApiResponse<types::HelloReply> {
    match state.hello_rpc.say_hello(req.name).await {
        Ok(message) => ApiResponse::success(types::HelloReply { message }),
        Err(status) => ApiResponse::fail("HELLO_RPC_UNAVAILABLE", status.to_string()),
    }
}
```

`AppState` 在 `main` 初始化一次，RPC endpoint 从 `etc/<service>.toml` 或环境变量读取。REST metrics middleware 会在 handler 执行期间自动设置 task-local request id；RPC client 保持 `request_id_interceptor()` 即可把当前 HTTP `x-request-id` 写入 RPC metadata。脱离 HTTP handler 的后台任务才需要显式使用 `with_rpc_request_id(...)`。

## Proto Spec

```proto
syntax = "proto3";

package hello.v1;

service Greeter {
  rpc SayHello (HelloRequest) returns (HelloReply);
}

message HelloRequest {
  string name = 1;
}

message HelloReply {
  string message = 1;
}
```

生成：

```bash
rzcli rpc gen -p proto/hello.proto -d target/generated
```

## Cargo Feature Pattern

```toml
[dependencies]
rs-zero = { version = "0.2.3", features = ["rest", "resil", "observability"] }
```

Redis cache：

```toml
rs-zero = { version = "0.2.3", features = ["cache-redis", "resil", "observability"] }
```

RPC：

```toml
rs-zero = { version = "0.2.3", features = ["rpc", "resil", "observability"] }
```

## REST Implementation Notes

- 使用 rs-zero REST 统一响应和错误映射。
- 保留 axum/tower 扩展点。
- 新服务应提供 readiness/health 路径和 `/metrics`。
- 不在 handler 中硬编码外部凭据。

## Tower-first Layer Pattern

- REST：优先使用 `RestServer` 或显式 `RestLayerStack`，不要绕过默认 timeout、metrics、breaker、shedder 和 request id scope。
- RPC server unary：优先在 `tonic::transport::Server::builder().layer(...)` 挂载 `RpcServerLayerStack::new(config).into_layer()`。
- RPC client：优先用 `RpcClientBuilder` 建立 channel，并配合 `request_id_interceptor()` / `trace_context_interceptor()`。
- 通用上下文使用 `RequestContext`，负责低基数字段、`x-request-id` 和 `traceparent` 在 HTTP headers / tonic metadata / extensions 间传递。
- 旧 helper `RpcResilienceLayer::run_unary*`、`observe_rpc_unary*`、`RpcUnaryResilienceLayer` 保留给手写高级场景或旧项目兼容，不是新生成代码的默认形态。

## RPC Log Pattern

- `rzcli rpc gen` 生成的 unary skeleton 默认走 Tower-first：service 暴露 `server_layer_stack()`，业务方法只委托 `handle_*`。
- tonic trait impl 可以在方法内使用 `request.into_inner()`；`x-request-id` 和 `traceparent` 已由外层 `RpcServerLayerStack` 读取。
- With `observability`, unary completion emits INFO `rpc unary observed`.
- Search by `rpc.method` / `route` such as `SayHello` or `say_hello`.
- For API -> RPC chains, keep `request_id_interceptor()` enabled; within REST handlers, rs-zero REST middleware automatically scopes the current `x-request-id` for the interceptor. With `otlp`, also use `trace_context_interceptor()`.
- 只有未挂 `RpcServerLayerStack` 的手写兼容路径，才需要 `run_unary_with_metadata` 或 `observe_rpc_unary_with_metadata` 来保留入站 metadata。
- If only `h2::*` DEBUG logs appear, reduce transport noise with `RUST_LOG=info,h2=warn,hyper=warn,tower=warn`.

## RPC Resilience Notes

- unary server 优先使用 `RpcServerLayerStack`。
- unary client 优先使用 `RpcClientBuilder` + interceptor。
- streaming 使用 `ObservedRecvStream`、`record_stream_send`、`run_observed_stream` 等组合式 wrapper。
- 不把 streaming wrapper 描述成完整 tonic stream interceptor 替代品。

## Cache Notes

- 本地 LRU 已有 O(1) 容量淘汰。
- 本地 TTL 仍不是 go-zero timing wheel 模型；高 TTL churn 需要压测。
- Redis Cluster 与应用级分片不同：Cluster 使用 Redis Cluster 协议路由，应用级分片由客户端选择节点。

## Service Group Pattern

```rust
use rs_zero::{
    core::ServiceGroup,
    rest::{RestConfig, RestServer, RestService},
    rpc::TonicHealthService,
};

let rest = RestService::new("api", rest_addr, RestServer::new(RestConfig::default(), router));
let rpc = TonicHealthService::new("health-rpc", rpc_addr);

let mut group = ServiceGroup::new();
group.add(rest);
group.add(rpc);
group.start().await?;
# Ok::<(), Box<dyn std::error::Error>>(())
```

规则：服务并发启动，不依赖启动顺序；worker 必须监听 `ShutdownToken`；任一服务错误默认触发全组 shutdown。

## Observability Notes

- metrics label 必须低基数。
- 不记录 key、用户 ID、完整 URL、SQL 原文为 label。
- 默认 `MetricsRegistry` 是 in-process registry。
- 高并发或生产标准更高时考虑 `observability-prometheus-client`。
- OTLP 和 Pyroscope collector 测试是 opt-in。

## Test Pattern

```bash
cargo fmt --all -- --check
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace
```

局部 feature：

```bash
cargo test -p rs-zero --features cache-redis,resil --test cache_redis_scripts
cargo test -p rs-zero --features observability,cache-redis,rpc,resil --test observability_integration
```
