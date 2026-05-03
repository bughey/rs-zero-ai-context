# Resilience Patterns

rs-zero 韧性能力集中在 `resil` feature，部分分布式限流依赖 `cache-redis`。

## Circuit Breaker

适用：下游 Redis、DB、RPC、外部 HTTP。

要求：

- 定义 acceptable error。
- 区分业务失败和系统失败。
- breaker rejected 时进入明确降级路径。

## Timeout

所有外部调用必须有 timeout。不要只依赖默认 socket 或 runtime 行为。

## Concurrency Limit

适用：高成本 handler、RPC method、批处理入口。

测试：并发超过限制时应有可解释拒绝。

## Token Limiter Rescue

Redis token limiter 生产配置应开启本地 rescue limiter。

行为：

1. Redis 正常时使用 Redis token bucket。
2. Redis 异常时切到本地 limiter。
3. 后台 ping 监控 Redis 恢复。
4. 恢复后切回 Redis。

外部验证：

```bash
scripts/external-integration.sh redis-recovery
```

## Period Limiter Alignment

需要自然周期时启用对齐：

- 小时。
- 天。
- 指定 UTC offset。

不要用固定 TTL 冒充自然周期窗口。

## Adaptive Shedder

Linux 下默认 CPU provider 基于 `/proc/stat`。非 Linux 平台应明确降级。

外部压测：

```bash
scripts/external-integration.sh linux-cpu
```

## RPC Resilience

- unary：优先 layer/helper。
- streaming：使用观察 wrapper，并说明边界。

## Verification

```bash
cargo test -p rs-zero --features cache-redis,resil --test cache_redis_scripts
cargo test -p rs-zero --features rpc,resil --test rpc_production_integration
cargo test --test resilience_stress
```
