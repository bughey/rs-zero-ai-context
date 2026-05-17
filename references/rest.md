# REST Patterns

rs-zero REST 基于 axum/tower，目标是保留 Rust 生态扩展点，同时提供 go-zero 风格默认工程体验。

## Service Shape

REST 服务应具备：

- 统一响应。
- 错误映射。
- health/readiness endpoint。
- metrics endpoint。
- tracing 初始化。
- graceful shutdown。
- 可配置 middleware。

## Recommended Workflow

1. 写 `.api`。
2. `rzcli api validate`。
3. 需要 review spec 时运行 `rzcli api format`。
4. `rzcli api gen`。
5. 需要接口文档时运行 `rzcli api openapi`。
6. 补业务逻辑和测试。
7. 补服务说明文档与 `API.md`。
8. 运行验证。

## `.api` Imports and Formatting

`.api` 可以拆分公共类型，再通过 `import` 引入：

```api
syntax = "v1"

import "common/types.api"
```

规则：

- import 路径相对当前 `.api` 文件所在目录解析。
- 生成、格式化或导出 OpenAPI 前先运行 `rzcli api validate -f <file>.api`。
- import 文件缺失、路径大小写不匹配或循环依赖时，先修 spec，不要绕过生成器手写 Rust 类型。
- `rzcli api format -f <file>.api -o <output>.api` 用于生成格式化结果；覆盖原文件需要显式 `--force`。
- 当前 goctl 兼容目标是输入语义和工作流兼容，不承诺字节级输出兼容；一个解析入口文件内只验证一个 service。

推荐顺序：

```bash
rzcli api validate -f <service>.api
rzcli api format -f <service>.api -o <service>.formatted.api
rzcli api gen -f <service>.api -d <output_dir>
```

如果需要覆盖原 spec：

```bash
rzcli api format -f <service>.api -o <service>.api --force
```

## OpenAPI Export

`rzcli api openapi` 用于从 `.api` 导出接口文档：

```bash
rzcli api openapi -f <service>.api -o openapi.json
```

导出内容包括：

- HTTP method 与 route path。
- request 字段 tag：`path`、`query`、`header`、`form`、`json`。
- response schema。
- `@server` 中可解析的 prefix、group、middleware、jwt 等元数据。

OpenAPI 是文档产物，不替代 `api gen`。如果 `.api` 发生变化，先 validate，再 gen 和 openapi，保持 Rust skeleton 与文档一致。


## Framework Service Configuration

`rzcli new-rest` 和 `rzcli api gen` 生成的入口默认使用框架级 `RestServiceConfig`，不要再为 REST/日志/JWT 生成自定义 `AppConfig`：

```rust
use rs_zero::core::{RestServiceConfig, emit_config_warnings, init_tracing, shutdown_signal};
use rs_zero::observability::{MetricsRegistry, metrics_router};
use rs_zero::rest::RestServer;

let app = RestServiceConfig::load("etc/hello-api", "HELLO_API")?;
let _ = init_tracing(app.log_config());
emit_config_warnings(&app.validate_features());

let metrics = MetricsRegistry::new();
let router = router::router().merge(metrics_router(metrics.clone()));
let mut config = app.rest_config();
config.metrics_registry = Some(metrics);
RestServer::new(config, router)
    .serve_with_shutdown(app.addr()?, shutdown_signal())
    .await?;
```

标准配置：

```toml
name = "hello-api"
mode = "pro"

[server]
host = "127.0.0.1"
port = 8080
timeout_ms = 5000
max_body_bytes = 1048576

[log]
mode = "console" # console | file | volume
encoding = "plain" # plain | json
level = "info"
path = "logs"
rotation = "daily" # daily | size
compress = false
keep_days = 0
max_backups = 0
max_size_mb = 0

[middlewares]
metrics = true
resilience = true
```

环境变量按 `__` 覆盖嵌套字段，例如 `HELLO_API__SERVER__PORT=8081`。`RUST_LOG` 仍优先覆盖 `[log].level`。启动后调用 `emit_config_warnings(&app.validate_features())`，当 `[middlewares]` 请求的能力缺少 Cargo feature 时只打印 warning，不中断服务。

## Framework Dependencies

API 调 RPC、连接 model 数据库等依赖应放在框架配置里，而不是业务自定义配置。

静态 RPC endpoint：

```toml
[rpc_clients.hello]
provider = "static"
endpoint = "http://127.0.0.1:50051"
service = "hello-rpc"
connect_timeout_ms = 3000
request_timeout_ms = 5000

[rpc_clients.hello.retry]
enabled = true
max_attempts = 3
initial_backoff_ms = 50
max_backoff_ms = 500

[rpc_clients.hello.deadline]
propagate = true
clip_retries_to_budget = true

[rpc_clients.hello.load_balance]
policy = "static"
```

etcd 服务发现：

```toml
[rpc_clients.hello]
provider = "etcd"
service = "hello-rpc"

[rpc_clients.hello.etcd]
endpoints = ["http://127.0.0.1:2379"]
prefix = "/rs-zero"
connect_timeout_ms = 3000
operation_timeout_ms = 3000
```

数据库：

```toml
[database]
kind = "postgres" # sqlite | postgres | mysql
url = "postgres://user:pass@127.0.0.1/app"
max_connections = 20
connect_timeout_ms = 5000
```

代码中从 `RestServiceConfig` 转换：

```rust
let rpc = app.rpc_client_config("hello")?;
let db = app.database_config();
```

feature 规则：

- `[rpc_clients]` 需要 `rpc` feature。
- `provider = "etcd"` 需要 `discovery-etcd` feature。
- `[database].kind` 需要匹配 `db-sqlite`、`db-postgres` 或 `db-mysql`。
- `emit_config_warnings(&app.validate_features())` 只打印 warning；真实连接失败仍应作为启动错误处理。

## Handler Request Extractors

`rzcli api gen` 会生成 `handler` / `logic` 两层，并根据 route request 类型和字段 tag 生成 handler 入参。不要把有 request 的 handler 手写成无参。业务代码优先写在 `src/logic/*`，handler 只做 axum extractor 与 `ApiResponse` 适配。

生成文件命名规则：handler 文件保持 `<handler>_handler.rs`；logic 文件使用 `<handler>_logic.rs`，但 re-export 的业务函数名仍与 handler 函数名一致。

映射规则：

- request 类型包含 `path` tag：生成 `axum::extract::Path<types::Req>`。
- request 类型包含 `query` 或 `form` tag：生成 `axum::extract::Query<types::Req>`。
- request 类型只包含 `json` tag，或没有 `path` / `query` / `form` tag：生成 `axum::Json<types::Req>`。
- route 没有 request：handler 无入参。

示例：

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

生成签名模式：

```rust
use axum::extract::Path;
use rs_zero::rest::ApiResponse;

use crate::logic;
use crate::types;

pub async fn hello_handler(Path(req): Path<types::HelloReq>) -> ApiResponse<types::HelloReply> {
    let reply = logic::hello_handler(req).await;
    ApiResponse::success(reply)
}
```

对应 logic：

```rust
use crate::types;

pub async fn hello_handler(req: types::HelloReq) -> types::HelloReply {
    let _ = &req;
    types::HelloReply::default()
}
```

字段 tag 的 wire name 会参与 `serde(rename)`，例如 `path:"name"` 会让 `HelloReq.name` 正确绑定路径参数。

## JWT Auth

`.api` 中使用 `@server(jwt: Auth)` 启用 REST JWT 生成支持：

```api
@server(jwt: Auth)
service user-api {
    @handler profileHandler
    get /api/profile returns (ProfileReply)
}
```

`rzcli api gen` 会在首次生成项目时通过 `RestServiceConfig` 接入 JWT，并生成 `etc/<service>.toml`：

```toml
[auth]
# Environment variable JWT_AUTH_SECRET takes precedence.
jwt_secret = ""

# JWT expiration duration in seconds.
# Environment variable JWT_AUTH_EXPIRES takes precedence.
jwt_expires = 7200
```

规则：

- `jwt_secret` 支持环境变量和配置文件，环境变量优先。
- `jwt_expires` 单位是秒，默认 `7200`。
- `jwt_expires` 供登录/签发 token 逻辑使用；REST middleware 只校验请求里的 JWT，不负责签发 token。
- 环境变量固定为 `JWT_AUTH_SECRET` 和 `JWT_AUTH_EXPIRES`；`jwt: AdminAuth` 保留为认证域名，不再改变环境变量名。
- 空 `jwt_secret` 视为未配置，不能作为有效 secret。
- `jwtTransition` 会被解析，但当前不会自动启用双 secret 过渡验证。

增量生成注意：

- 首次生成含 JWT 的 `.api` 会写入 `main.rs` 和 `etc/<service>.toml` 的 JWT 配置。
- 已有项目后新增 `@server(jwt: Auth)` 时，安全增量模式默认跳过 `main.rs` 和 `etc/*.toml`，避免覆盖项目配置。
- 遇到这种情况，应手动合并 JWT 配置，或明确使用 `--force` 重新生成。

## Resilience

生产服务优先检查：

- request timeout。
- max concurrency。
- circuit breaker。
- adaptive shedder。
- rate limiter。

REST 中间件应使用低基数 route/method 标识，不使用原始 URL 作为 metrics label。

`RestConfig::production_defaults(...)` 不默认启用 Redis limiter；即使开启 `cache-redis` feature，Redis limiter 也必须显式配置。需要保留生产默认配置并打开 Redis limiter 时，使用 `production_defaults_with_redis_limiter(...)`：

```rust
use rs_zero::resil::RedisTokenLimiterConfig;
use rs_zero::rest::{RestConfig, RestRateLimiterConfig};

let config = RestConfig::production_defaults_with_redis_limiter(
    "hello-api",
    RestRateLimiterConfig::RedisToken(RedisTokenLimiterConfig::go_zero_defaults()),
);
```

## Observability

REST 服务应暴露：

- request total。
- request duration histogram。
- in-flight gauge。
- error/drop 指标。
- tracing span。

不要把用户 ID、token、完整 URL、cache key 放进 label。

## Documentation

服务说明文档必须说明：

- 如何运行。
- 配置项。
- endpoint 示例。
- 测试命令。
- 生产边界。


## API Context and RPC Clients

当 API 调 RPC 或 model 时，优先建立 `AppState`，把长生命周期依赖放入 state。

规则：

- RPC channel 在 `main` 中初始化一次，放入 `AppState`。
- `AppState` 提供 `hello_rpc()` 这类方法，返回已接入 `request_id_interceptor()` 的 tonic client。
- handler 使用 `State<AppState>` 加生成的 request extractor，例如 `Path(req)`、`Query(req)` 或 `Json(req)`；handler 调 logic。
- router 返回 `Router<AppState>`；合并 metrics/router 后在 `main` 调 `.with_state(state)`。
- endpoint、timeout、凭据等外部依赖配置从 `etc/<service>.toml` 或环境变量读取。
- 不在 handler 内每次创建 `Channel`、读取配置文件或直接调用 `with_interceptor(...)`；RPC 调用放在 logic。
- 不在普通 HTTP handler 中手写 `HeaderMap` -> tonic metadata 的 `x-request-id` 注入。
- REST metrics middleware 会在 handler 执行期间自动设置 task-local request id；`AppState::hello_rpc()` 创建的 client 保持 `request_id_interceptor()`，可让 API 日志和 RPC 日志共享 request id。
- 脱离 HTTP handler 的后台任务或手动异步边界，使用 `with_rpc_request_id(...)` 显式设置调用范围。

示例：

```rust
use hello_rpc::proto::hello_service_client::HelloServiceClient;
use rs_zero::rpc::{RpcClientBuilder, RpcClientConfig, request_id_interceptor};
use tonic::{service::interceptor::InterceptedService, transport::Channel};

#[derive(Clone)]
pub struct AppState {
    rpc_channel: Channel,
}

impl AppState {
    pub async fn connect(endpoint: impl Into<String>) -> Result<Self, tonic::transport::Error> {
        let config = RpcClientConfig::production_defaults(endpoint);
        let rpc_channel = RpcClientBuilder::new(config).connect().await?;
        Ok(Self { rpc_channel })
    }

    pub fn hello_rpc(
        &self,
    ) -> HelloServiceClient<InterceptedService<Channel, impl tonic::service::Interceptor>> {
        HelloServiceClient::with_interceptor(self.rpc_channel.clone(), request_id_interceptor())
    }
}
```

```rust
use axum::{extract::{Path, State}};
use rs_zero::rest::ApiResponse;

use crate::{logic, state::AppState, types};

pub async fn hello_handler(
    State(state): State<AppState>,
    Path(req): Path<types::HelloReq>,
) -> ApiResponse<types::HelloReply> {
    match logic::hello_handler(&state, req).await {
        Ok(reply) => ApiResponse::success(reply),
        Err(status) => ApiResponse::fail(status.code().to_string(), status.message().to_string()),
    }
}
```

```rust
use hello_rpc::proto::SayHelloRequest;
use tonic::Status;

use crate::{state::AppState, types};

pub async fn hello_handler(
    state: &AppState,
    req: types::HelloReq,
) -> Result<types::HelloReply, Status> {
    let mut client = state.hello_rpc();
    let response = client.say_hello(SayHelloRequest { name: req.name }).await?;

    Ok(types::HelloReply {
        message: response.into_inner().message,
    })
}
```

```rust
use axum::{routing, Router};

use crate::{handler, state::AppState};

pub fn router() -> Router<AppState> {
    Router::new().route("/hello/{name}", routing::get(handler::hello_handler))
}
```

## Tower-first REST LayerStack

`RestServer` 默认会应用 rs-zero REST middleware stack。需要显式组合 router 时，可使用 `RestLayerStack`：

```rust
use rs_zero::core::RestServiceConfig;
use rs_zero::rest::RestLayerStack;

let app_config = RestServiceConfig::load("etc/hello-api", "HELLO_API")?;
let config = app_config.rest_config();
let app = RestLayerStack::new(config).layer(router);
```

规则：

- 不要为了手写 axum router 绕过 request id、metrics、timeout、breaker、shedder、auth 和统一错误映射。
- 已有 `RestServer::new(config, router).into_router()` 仍是推荐入口。
- 只有需要显式和其它 Tower/axum layer 编排时，才直接使用 `RestLayerStack`。
