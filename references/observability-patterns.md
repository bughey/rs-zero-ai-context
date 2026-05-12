# Observability Patterns

## Metrics

默认：`MetricsRegistry`。

适用：普通服务、测试、轻量部署。

边界：

- in-process registry。
- 高并发和大量 label churn 需要压测。

成熟 adapter：`PrometheusClientMetricsRegistry`。

启用：

```toml
rs-zero = { version = "0.2.3", features = ["observability-prometheus-client"] }
```

## Label Rules

允许：

- method。
- route template。
- status code。
- component。
- operation。

禁止：

- 用户 ID。
- cache key。
- SQL 原文。
- 完整 URL。
- token、email、手机号。

## Tracing

- 使用 TraceContext 传播。
- HTTP header 和 tonic metadata 可承载 W3C `traceparent`。
- REST metrics middleware 会在 handler 执行期间设置 task-local request id；API handler 内通过带 `request_id_interceptor()` 的 RPC client 调用下游时，不需要手写 `x-request-id` metadata 注入。
- Redis/SQL/RPC 默认 adapter 有覆盖，手写 SQL 或未挂 layer 的 tonic stack 仍需显式接入。
- RPC INFO logs use `rpc unary observed` with `rpc.method`, `route`, `request_id`, `traceparent`, `trace_id`, `span_id` and `code`.
- RPC server 优先挂 `RpcServerLayerStack`，由外层 Tower layer 读取 tonic metadata；trait method 内可以直接 `request.into_inner()`。
- 只有未挂 `RpcServerLayerStack` 的手写兼容路径，才使用 `request.into_parts()` 加 `run_unary_with_metadata` / `observe_rpc_unary_with_metadata`。
- RPC client 优先用 `RpcClientBuilder` 建立 channel，并保留 `request_id_interceptor()` 与需要时的 `trace_context_interceptor()`。

## OTLP

启用：

```toml
rs-zero = { version = "0.2.3", features = ["observability", "otlp"] }
```

外部 collector 验证：

```bash
scripts/external-integration.sh otlp
```

## Pyroscope / Profiling

启用 profiling：

```toml
rs-zero = { version = "0.2.3", features = ["profiling"] }
```

边界：

- profiling 是 opt-in / experimental。
- 不应默认启用。
- 上线前必须压测开销。

外部验证：

```bash
scripts/external-integration.sh pyroscope
```

## Verification

```bash
cargo test -p rs-zero --features observability --test metrics_stress
cargo test -p rs-zero --features observability,cache-redis,rpc,resil --test observability_integration
cargo check -p rs-zero --no-default-features --features observability-prometheus-client
```
