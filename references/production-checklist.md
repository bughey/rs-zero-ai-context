# Production Checklist

## Feature

- 确认 `Cargo.toml` 只启用需要的 feature。
- Redis、DB、OTLP、profiling 不应被误加入默认路径。
- profiling 保持 opt-in / experimental。

## REST / RPC

- timeout 已配置。
- breaker 已覆盖关键下游。
- concurrency limit 或 shedder 已覆盖高成本入口。
- REST JWT secret 不硬编码，优先使用环境变量或 secret manager。
- 含 `@server(jwt: ...)` 的服务已配置 `[auth].jwt_secret` / `JWT_*_SECRET`。
- `jwt_expires` 单位为秒，签发 token 时应使用该配置，不要散落硬编码 TTL。
- RPC unary server 已接入 `RpcServerLayerStack`；旧 helper 路径有明确兼容说明。
- API 调 RPC 的 client 已放入 `AppState`，并使用 `RpcClientBuilder` / request id interceptor 传播上下文。
- streaming wrapper 的边界已记录。

## Cache

- Redis unavailable policy 明确。
- Redis breaker 已开启。
- Redis Cluster 与应用级分片选择明确。
- Redis distributed lock 如启用，TTL、token 校验解锁、Cluster hash tag 和 Redis down 路径已验证。
- 本地 LRU TTL 边界已说明。

## Limiter / Shedder

- Redis token limiter rescue 已开启。
- period limiter 自然周期需求已对齐。
- Linux CPU provider 已在目标平台验证；非 Linux 已说明降级。

## Service Group

- 同进程多服务已统一接入 `ServiceGroup`。
- REST 使用 `RestService`，RPC 使用 `TonicService` / `TonicHealthService`。
- worker / scheduler 已监听 `ShutdownToken`。
- 不依赖 service group 启动顺序；readiness/health 已覆盖依赖准备。
- `shutdown_timeout` 足够完成清理，但不会无限阻塞退出。

## Observability

- metrics label 低基数。
- tracing propagation 已验证。
- OTLP collector 为 opt-in 验证。
- 高并发 metrics stress 已覆盖或有压测计划。

## External Dependencies

默认 CI 不连接外部服务。生产前按需执行：

```bash
scripts/external-integration.sh redis-recovery
scripts/external-integration.sh redis-lock
scripts/external-integration.sh redis-cluster
scripts/external-integration.sh otlp pyroscope
scripts/external-integration.sh linux-cpu
```

## Verification

```bash
cargo fmt --all -- --check
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace
```

## Documentation

服务文档应包含：

- 配置。
- 运行方式。
- API/RPC 示例。
- 测试命令。
- feature 列表。
- 外部依赖说明。
- 回滚或降级策略。
