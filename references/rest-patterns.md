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
7. 补 `README.md` 与 `API.md`。
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

## Handler Request Extractors

`rzcli api gen` 会根据 route request 类型和字段 tag 生成 handler 入参。不要把有 request 的 handler 手写成无参。

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

use crate::types;

pub async fn hello_handler(Path(req): Path<types::HelloReq>) -> ApiResponse<types::HelloReply> {
    let _ = &req;
    ApiResponse::success(types::HelloReply::default())
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

`rzcli api gen` 会在首次生成项目时接入 `AuthConfig`，并生成 `etc/<service>.toml`：

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
- `jwt: AdminAuth` 对应 `JWT_ADMIN_AUTH_SECRET` 和 `JWT_ADMIN_AUTH_EXPIRES`。
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

## Observability

REST 服务应暴露：

- request total。
- request duration histogram。
- in-flight gauge。
- error/drop 指标。
- tracing span。

不要把用户 ID、token、完整 URL、cache key 放进 label。

## Documentation

`README.md` 必须说明：

- 如何运行。
- 配置项。
- endpoint 示例。
- 测试命令。
- 生产边界。
