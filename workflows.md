# rs-zero AI Workflows

## 1. New REST Service

输入：服务名、路由、请求/响应、是否需要 DB/cache/resilience/observability。

流程：

1. 写 `.api` spec。
2. 验证 spec。
3. 使用 `rzcli` 生成。
4. 实现业务逻辑。
5. 添加测试和文档。
6. 运行验证。

命令：

```bash
rzcli api validate -f <service>.api
rzcli api gen -f <service>.api -d <output_dir>
cargo fmt --all -- --check
cargo test --workspace
```

失败处理：

- spec 失败：先修 `.api`，不要绕过生成器。
- feature 缺失：更新 `Cargo.toml` 后再编译。
- 路由或类型不匹配：以 `.api` 为源头修正。

## 2. Modify REST API

1. 找到现有 `.api`。
2. 修改 spec。
3. 运行 `rzcli api validate`。
4. 运行 `rzcli api gen --dry-run` 预览增量计划。
5. 运行 `rzcli api gen`。
6. 保留已有业务逻辑，补新增 handler/logic。
7. 更新 `API.md` 和测试。

不要直接只改 handler 而不更新 spec。

如果本次新增 `@server(jwt: Auth)`：

- 检查 `main.rs` 是否已接入 `AuthConfig`。
- 检查 `etc/<service>.toml` 是否包含 `[auth].jwt_secret` 和 `[auth].jwt_expires`。
- 生产环境优先使用 `JWT_AUTH_SECRET` / `JWT_AUTH_EXPIRES`。
- `jwt_expires` 单位是秒，默认 `7200`。
- 安全增量模式默认不会覆盖已有 `main.rs` 和 `etc/*.toml`，需要手动合并或明确使用 `--force`。

## 3. New RPC Service

1. 写 `.proto`。
2. 使用 `rzcli rpc gen` 生成 tonic skeleton。
3. 接入 `RpcResilienceLayer` 或 `RpcUnaryResilienceLayer`。
4. streaming 方法使用 observed wrapper，不冒充完整自动 stream interceptor。
5. 生成 `README.md` / `RPC.md`。
6. 运行验证。

命令：

```bash
rzcli rpc gen -p proto/<service>.proto -d <output_dir>
cargo check
cargo test
```

## 4. Model / Cache Repository

1. 确认 SQL schema。
2. 选择 SQL dialect 和 SQLx backend。
3. 需要 Redis cache 时启用 `--with-redis-cache`。
4. 生成 repository skeleton。
5. 添加 cache-aside 测试。

命令：

```bash
rzcli model gen -s schema.sql -d <output_dir> --with-sqlx
rzcli model gen -s schema.sql -d <output_dir> --with-sqlx --with-redis-cache
```

生产注意：

- Redis Cluster 使用真实 cluster client 配置。
- 应用级分片不是 Redis Cluster 协议。
- 外部 Redis 测试保持 opt-in。

## 5. Add Redis Cache / Cluster

1. 确认 feature：`cache-redis`。
2. 选择单节点、Cluster 或应用级分片。
3. 配置 breaker 和 unavailable policy。
4. 补 unit/integration 测试。
5. 外部 Redis/Cluster 验证只在手动或 CI opt-in 中执行。

验证示例：

```bash
cargo test -p rs-zero --features cache-redis --test cache_redis_cluster
cargo test -p rs-zero --features cache-redis,observability --test cache_redis_degradation
```

## 6. Add Resilience

1. 确认 feature：`resil`。
2. REST 优先配置中间件；RPC 优先使用 layer/helper。
3. 需要分布式限流时启用 `cache-redis`。
4. 为拒绝、恢复、超时和降级行为补测试。

重点：

- token limiter Redis 异常时应使用 rescue limiter。
- period limiter 可按自然周期对齐。
- adaptive shedder 在 Linux 才有真实 CPU provider，其他平台需明确降级。

## 7. Add Observability / OTLP

1. 确认 feature：`observability`。
2. 需要成熟 Prometheus client 时启用 `observability-prometheus-client`。
3. 需要 OTLP 时启用 `otlp`。
4. 只使用低基数字段。
5. 外部 collector 测试保持 opt-in。

验证示例：

```bash
cargo test -p rs-zero --features observability,cache-redis,rpc,resil --test observability_integration
cargo check -p rs-zero --no-default-features --features observability-prometheus-client
```

## 8. Production Review

检查：

- Cargo feature 是否最小且完整。
- timeout、breaker、limiter/shedder 是否覆盖关键入口。
- metrics label 是否低基数。
- Redis/DB/OTLP/Pyroscope 是否有 opt-in 验证说明。
- profiling 是否仍标注 experimental / opt-in。
- 文档是否包含配置、运行、测试、回滚。

推荐验证：

```bash
cargo fmt --all -- --check
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace
```
