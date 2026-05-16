# rs-zero AI Workflows

## 1. New REST Service

输入：服务名、路由、请求/响应、是否需要 DB/cache/resilience/observability。

流程：

1. 写 `.api` spec。
2. 验证 spec，确认 import 相对路径可解析。
3. 需要 review 时格式化 `.api`。
4. 使用 `rzcli` 生成。
5. 需要对外文档时导出 OpenAPI。
6. 实现业务逻辑。
7. 添加测试和文档。
8. 运行验证。

命令：

```bash
rzcli api validate -f <service>.api
rzcli api format -f <service>.api -o <service>.formatted.api
rzcli api gen -f <service>.api -d <output_dir>
rzcli api openapi -f <service>.api -o API.openapi.json
cargo fmt --all -- --check
cargo test --workspace
```

失败处理：

- spec 失败：先修 `.api`，不要绕过生成器。
- import 失败：按当前 `.api` 文件目录检查相对路径，先修 import 再生成。
- feature 缺失：更新 `Cargo.toml` 后再编译。
- 路由或类型不匹配：以 `.api` 为源头修正。

## 2. Modify REST API

1. 找到现有 `.api`。
2. 修改 spec。
3. 运行 `rzcli api validate`。
4. 需要时运行 `rzcli api format`，先输出到临时文件 review。
5. 运行 `rzcli api gen --dry-run` 预览增量计划。
6. 运行 `rzcli api gen`。
7. 需要时重新导出 OpenAPI。
8. 保留已有业务逻辑，新增接口优先补 `src/logic/*`，handler 只做适配。
9. 更新 `API.md` 和测试。

不要直接只改 handler 而不更新 spec。

如果本次改动涉及 `import`：

- import 路径相对当前 `.api` 文件所在目录解析。
- 先修通 `api validate`，再运行 `api format`、`api gen` 或 `api openapi`。
- 不要复制 import 类型到生成文件里规避解析失败。

如果本次新增 `@server(jwt: Auth)`：

- 检查 `main.rs` 是否已接入 `AuthConfig`。
- 检查 `etc/<service>.toml` 是否包含 `[auth].jwt_secret` 和 `[auth].jwt_expires`。
- 生产环境优先使用 `JWT_AUTH_SECRET` / `JWT_AUTH_EXPIRES`。
- `jwt_expires` 单位是秒，默认 `7200`。
- 安全增量模式默认不会覆盖已有 `main.rs` 和 `etc/*.toml`，需要手动合并或明确使用 `--force`。

## 3. New RPC Service

1. 写 `.proto`。
2. 使用 `rzcli rpc gen` 生成 tonic skeleton。
3. 保持 `handler` / `logic` 两层，业务写入 `src/logic/*`。
4. 接入 `RpcServerLayerStack`；旧项目或手写高级场景才使用 `RpcResilienceLayer` / `RpcUnaryResilienceLayer`.
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

## 4. RPC + Model Integration

1. 先确认 proto 和 SQL schema。
2. 运行 `rzcli rpc gen` 和 `rzcli model gen --with-sqlx`。
3. 在 RPC service 中注入 `Sqlx{Entity}Repository` 或 `Cached{Entity}Repository<S>`。
4. unary server 外层挂 `RpcServerLayerStack`，handler 只做 tonic 适配，logic 做业务与错误映射。
5. 把 repository 错误映射为 `tonic::Status`。
6. streaming 方法避免长时间持有 DB 资源，必要时使用 observed wrapper。
7. 补映射、service 和 cache 测试。

参考模板：`templates/RPC-MODEL.md`。

注意：

- 不把 proto message 和 model entity 强行共用。
- 不把 repository 错误吞掉。
- 外部 DB / Redis 测试保持 opt-in。

## 5. API + Model Integration

1. 先确认 REST `.api` 和 SQL schema。
2. 运行 `rzcli api validate`，必要时运行 `rzcli api format`。
3. 运行 `rzcli api gen --dry-run`。
4. 运行 `rzcli api gen` 生成或增量更新 REST skeleton。
5. 运行 `rzcli model gen --with-sqlx` 生成 model/repository crate。
6. 在 REST 项目 `Cargo.toml` 引入 model crate 和对应 DB feature。
7. 建立 `AppState`，持有 `Sqlx{Entity}Repository` 或 `Cached{Entity}Repository<S>`。
8. 在 `main.rs` 初始化 DB pool、repository、metrics，并通过 `.with_state(state)` 注入 router。
9. handler 使用 `State<AppState>` 加生成的 request extractor，调用 logic。
10. logic 调用 repository/cache，并补映射、handler/logic 和 repository/cache 测试。

参考模板：`templates/API-MODEL.md`。

注意：

- 不把数据库凭据写入源码。
- 不把 migration 和 transaction 边界假装交给生成器。
- 不直接在 handler 中散落 SQL；业务逻辑放在 logic，并优先通过生成的 repository。
- 外部 DB / Redis 测试保持 opt-in。

## 6. Add Redis Cache / Cluster

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

## 7. Add Resilience

1. 确认 feature：`resil`。
2. REST 优先配置中间件或 `RestLayerStack`；RPC server 优先使用 `RpcServerLayerStack`，RPC client 优先使用 `RpcClientBuilder`。
3. 需要分布式限流时启用 `cache-redis`。
4. 为拒绝、恢复、超时和降级行为补测试。

重点：

- token limiter Redis 异常时应使用 rescue limiter。
- period limiter 可按自然周期对齐。
- adaptive shedder 在 Linux 才有真实 CPU provider，其他平台需明确降级。

## 8. Add Distributed Lock

1. 确认 feature：`cache-redis`。
2. Redis lock key 必须有稳定 namespace 和 TTL。
3. Redis Cluster lock key 使用 hash tag，例如 `order:{123}`。
4. 临界区必须短于 TTL，不默认自动续租。
5. 处理 `Busy`、`OwnerMismatch`、timeout、connection 和 breaker open 错误。
6. 补单元测试；真实 Redis 使用 opt-in 验证。

验证示例：

```bash
cargo test -p rs-zero --features cache-redis --test redis_lock
scripts/external-integration.sh redis-lock
```

参考：`references/distributed-lock-patterns.md`。


## 9. Add Service Group

1. 确认是否需要同进程运行 REST、RPC、worker 或 scheduler。
2. 使用 `ServiceGroup` 作为生命周期入口。
3. REST 使用 `RestService` 包装 `RestServer`。
4. RPC 使用 `TonicService` 或 `TonicHealthService`。
5. worker 使用 `add_fn` 或实现 `Service`。
6. worker 必须监听 `ShutdownToken`。
7. 补 service group 生命周期测试。

验证示例：

```bash
cargo test -p rs-zero --test service_group
```

参考：`references/service-group-patterns.md`。

## 10. Add Observability / OTLP

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

## 11. Production Review

检查：

- Cargo feature 是否最小且完整。
- timeout、breaker、limiter/shedder 是否覆盖关键入口。
- 分布式锁如启用，TTL、owner token 解锁、Cluster hash tag 和 Redis down 路径是否验证。
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
