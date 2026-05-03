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
