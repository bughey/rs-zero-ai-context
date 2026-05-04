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
3. `rzcli api gen`。
4. 补业务逻辑和测试。
5. 补 `README.md` 与 `API.md`。
6. 运行验证。

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
