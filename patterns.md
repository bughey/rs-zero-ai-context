# rs-zero Code Patterns

## API Spec

```api
type CreateUserRequest {
    Email string `json:"email" validate:"required,email"`
    Name string `json:"name" validate:"required,min=1"`
}

type CreateUserResponse {
    Id i64 `json:"id"`
}

@server(group: user, prefix: /api)
service user-api {
    @handler createUser
    post /users (CreateUserRequest) returns (CreateUserResponse)
}
```

规则：

- route、handler、request、response 都从 `.api` 生成。
- 修改 API 先改 `.api`，再运行 `rzcli api validate` 和 `rzcli api gen`。

JWT 示例：

```api
@server(jwt: Auth)
service user-api {
    @handler profileHandler
    get /api/profile returns (ProfileReply)
}
```

生成项目会读取 `JWT_AUTH_SECRET` 或 `[auth].jwt_secret`，并读取 `JWT_AUTH_EXPIRES` 或 `[auth].jwt_expires`。`jwt_expires` 单位为秒，默认 `7200`。

## Proto Spec

```proto
syntax = "proto3";

package hello.v1;

service Greeter {
  rpc SayHello (HelloRequest) returns (HelloReply);
}

message HelloRequest {
  string name = 1;
}

message HelloReply {
  string message = 1;
}
```

生成：

```bash
rzcli rpc gen -p proto/hello.proto -d target/generated
```

## Cargo Feature Pattern

```toml
[dependencies]
rs-zero = { version = "0.1", features = ["rest", "resil", "observability"] }
```

Redis cache：

```toml
rs-zero = { version = "0.1", features = ["cache-redis", "resil", "observability"] }
```

RPC：

```toml
rs-zero = { version = "0.1", features = ["rpc", "resil", "observability"] }
```

## REST Implementation Notes

- 使用 rs-zero REST 统一响应和错误映射。
- 保留 axum/tower 扩展点。
- 新服务应提供 readiness/health 路径和 `/metrics`。
- 不在 handler 中硬编码外部凭据。

## RPC Resilience Notes

- unary 优先使用 `RpcUnaryResilienceLayer` 或生成代码中的 `RpcResilienceLayer::run_unary`。
- streaming 使用 `ObservedRecvStream`、`record_stream_send`、`run_observed_stream` 等组合式 wrapper。
- 不把 streaming wrapper 描述成完整 tonic stream interceptor 替代品。

## Cache Notes

- 本地 LRU 已有 O(1) 容量淘汰。
- 本地 TTL 仍不是 go-zero timing wheel 模型；高 TTL churn 需要压测。
- Redis Cluster 与应用级分片不同：Cluster 使用 Redis Cluster 协议路由，应用级分片由客户端选择节点。

## Observability Notes

- metrics label 必须低基数。
- 不记录 key、用户 ID、完整 URL、SQL 原文为 label。
- 默认 `MetricsRegistry` 是 in-process registry。
- 高并发或生产标准更高时考虑 `observability-prometheus-client`。
- OTLP 和 Pyroscope collector 测试是 opt-in。

## Test Pattern

```bash
cargo fmt --all -- --check
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace
```

局部 feature：

```bash
cargo test -p rs-zero --features cache-redis,resil --test cache_redis_scripts
cargo test -p rs-zero --features observability,cache-redis,rpc,resil --test observability_integration
```
