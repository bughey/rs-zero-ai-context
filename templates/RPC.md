# {Service Name} RPC

## Proto

```text
proto/{service}.proto
```

## Methods

### {Service}/{Method}

说明方法行为。

Request:

```json
{}
```

Response:

```json
{}
```

## grpcurl

```bash
grpcurl -plaintext 127.0.0.1:{port} list
```

## Resilience

- unary：优先说明使用 `RpcUnaryResilienceLayer` 或 `RpcResilienceLayer::run_unary`。
- 如果 unary 业务调用 model repository，说明 repository 错误如何映射为 `tonic::Status`。
- streaming：说明使用的 observed wrapper 和边界。

## Observability

说明 method metrics、tracing propagation 和错误映射。
