# RPC Patterns

rs-zero RPC 基于 tonic。AI 生成 RPC 服务时应明确 unary 与 streaming 的韧性边界。

## Workflow

1. 写 `.proto`。
2. 运行 `rzcli rpc gen`。
3. 补 tonic service 实现。
4. 接入 resilience layer/helper。
5. 补测试、README、RPC.md。

## Unary Resilience

优先选择：

- `RpcUnaryResilienceLayer`：挂到 tonic/Tower unary service，降低业务漏接风险。
- `RpcResilienceLayer::run_unary`：生成代码和手写 adapter 的兼容 helper。

覆盖能力：

- timeout。
- concurrency。
- breaker。
- shedder。
- status mapping。
- metrics/tracing helper。

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

## Verification

```bash
cargo test -p rs-zero --features rpc,resil --test rpc_production_integration
cargo test -p rs-zero-cli --test goctl_rpc_compat
```
