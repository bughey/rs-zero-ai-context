# Production Checklist

## Feature

- 确认 `Cargo.toml` 只启用需要的 feature。
- Redis、DB、OTLP、profiling 不应被误加入默认路径。
- profiling 保持 opt-in / experimental。

## REST / RPC

- timeout 已配置。
- breaker 已覆盖关键下游。
- concurrency limit 或 shedder 已覆盖高成本入口。
- RPC unary layer/helper 已接入。
- streaming wrapper 的边界已记录。

## Cache

- Redis unavailable policy 明确。
- Redis breaker 已开启。
- Redis Cluster 与应用级分片选择明确。
- 本地 LRU TTL 边界已说明。

## Limiter / Shedder

- Redis token limiter rescue 已开启。
- period limiter 自然周期需求已对齐。
- Linux CPU provider 已在目标平台验证；非 Linux 已说明降级。

## Observability

- metrics label 低基数。
- tracing propagation 已验证。
- OTLP collector 为 opt-in 验证。
- 高并发 metrics stress 已覆盖或有压测计划。

## External Dependencies

默认 CI 不连接外部服务。生产前按需执行：

```bash
scripts/external-integration.sh redis-recovery
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
