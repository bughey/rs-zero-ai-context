---
name: rs-zero-ai-context
description: 用于构建、修改、审查和生成 rs-zero Rust 微服务，覆盖 rzcli、REST .api、RPC .proto、model/cache、服务韧性、可观测性、service group、分布式锁和生产验证。当任务提到 rs-zero、rzcli、go-zero 风格 Rust 服务、REST/RPC 生成、API 与 model 集成或生产就绪分析时使用。
---

# rs-zero-ai-context

用本 Skill 按 rs-zero 的工程方式完成从规格、生成代码、业务实现到验证和生产加固的工作。

## 核心规则

1. 先从契约开始。
   - REST 从 `.api` 开始。
   - RPC 从 `.proto` 开始。
   - model/cache 从 SQL schema 开始。
   - 不要手写生成骨架来绕过 `rzcli`。

2. 优先使用 `rzcli`。
   - 写服务代码前，优先使用 `rzcli new-rest`、`rzcli api validate`、`rzcli api gen`、`rzcli api openapi`、`rzcli rpc gen`、`rzcli model gen`。
   - 修改已有项目时，先 dry-run 或检查生成计划，不要直接覆盖用户业务逻辑。

3. 保持生成边界清晰。
   - REST 和 RPC 生成项目采用 `handler` / `logic` 两层。
   - `handler` 负责框架 extractor、trait、response 适配。
   - `logic` 负责业务代码，方法形态遵循 `.api` / `.proto` 的请求响应定义。
   - 共享 client、repository、配置放入 `AppState` 或服务状态，不散落在 handler 里。

4. 明确 Cargo feature。常用 feature：
   - REST：`rest`
   - RPC / tonic：`rpc`
   - 服务韧性：`resil`
   - Redis cache / limiter / lock：`cache-redis`
   - 数据库：`db-sqlite`、`db-mysql`、`db-postgres`
   - 服务发现：`discovery-etcd`、`discovery-kube`
   - 可观测性：`observability`、`observability-prometheus-client`、`otlp`、`profiling`

5. 显式使用生产默认配置。
   - REST 默认使用 `RestConfig::production_defaults(...)`，除非任务明确需要开发默认配置。
   - RPC unary server 默认使用 `RpcServerLayerStack`；client 使用生成代码或框架支持的 client 路径，并保留 request/trace 传播。
   - Redis limiter、Redis lock、OTLP、profiling、外部 collector 都是 opt-in，必须检查 feature 和配置。

## 参考文件路由

只读取和当前任务相关的参考文件：

- CLI 与生成器行为：`references/rzcli.md`
- REST `.api`、JWT、handler/logic、API 调 RPC：`references/rest.md`
- RPC tonic、Tower-first layer、streaming 边界：`references/rpc.md`
- SQL model、cache-aside、API/RPC 与 model 集成：`references/model-cache.md`
- timeout、breaker、limiter、shedder、CPU/Redis 恢复测试：`references/resilience.md`
- metrics、tracing、request id、OTLP、profiling：`references/observability.md`
- 同进程 REST/RPC/worker 生命周期：`references/service-group.md`
- Redis 分布式锁：`references/distributed-lock.md`
- 常见故障与修复：`references/troubleshooting.md`
- 服务文档模板资源：`assets/API.md`、`assets/RPC.md`、`assets/README-service.md`

## 默认工作流

### 新建 REST 服务

1. 编写或更新 `.api`。
2. 运行 `rzcli api validate -f <service>.api`。
3. 运行 `rzcli api gen -f <service>.api -d <output_dir>`。
4. 在 `src/logic/*` 实现业务代码；`src/handler/*` 只保留适配代码。
5. 如果 `.api` 使用 `@server(jwt: Auth)`，检查 `JWT_AUTH_SECRET`、`JWT_AUTH_EXPIRES`、`[auth].jwt_secret`、`[auth].jwt_expires`。`jwt_expires` 单位为秒。
6. 如果 API 调 RPC，把 channel/client factory 放到 `AppState`；业务逻辑调用 `state.<rpc>()` 或类型化 wrapper，不在每个方法里手写 interceptor。
7. 运行 cargo 验证。

### 修改 REST API

1. 找到现有 `.api`。
2. 先修改 spec。
3. 运行 `rzcli api validate`。
4. 修改已有服务前优先运行 `rzcli api gen --dry-run`。
5. 保留已有 handler/logic 业务代码，只补缺失的生成边界。
6. 如果 route 或 auth 改变，更新测试和服务文档。

### 新建 RPC 服务

1. 编写 `.proto`。
2. 运行 `rzcli rpc gen -p <service>.proto -d <output_dir>`。
3. tonic trait 实现留在 `src/handler/*`；业务代码写入 `src/logic/*`。
4. 从 `etc/<service>.toml` 读取配置，不要把 listen address 写死。
5. unary 韧性与观测默认走 Tower-first server layer。
6. streaming 行为单独记录边界；未实现 unary 同等级 wrapper 时不要声称已覆盖。

### 增加 model/cache

1. 从 SQL schema 开始。
2. 运行 `rzcli model gen -s <schema.sql> -d <output_dir> --with-sqlx`；只有需要 Redis cache-aside 时才加 `--with-redis-cache`。
3. 通过 state 注入 repository。
4. DTO 和 entity 显式转换，不强行复用同一类型。
5. 修改 Redis 行为时补 repository 测试和外部 Redis 测试。

## 生产检查项

声称服务生产就绪前，按需核对：

- Cargo feature 最小且明确。
- 外部调用有 timeout。
- Redis、DB、RPC、外部 HTTP 等关键下游有 breaker。
- 高成本入口有 concurrency limit 或 shedder。
- REST JWT secret 不硬编码；环境变量或 secret manager 优先于本地配置。
- Redis 不可用策略明确。
- 生产使用 Redis token limiter 时启用本地 rescue mode。
- 不混淆 Redis Cluster 与应用级分片。
- 分布式锁使用 TTL、owner-token 解锁；Cluster 场景使用 hash tag。
- metrics label 保持低基数。
- REST -> RPC 的 request id 和 trace context 传播已验证。
- OTLP 和 profiling 为 opt-in，上生产前已连接真实 collector 压测或验证。
- service group 内 worker 监听 shutdown token，不依赖启动顺序。

## 验证命令

迭代阶段运行足够窄但可靠的检查，交付时说明实际执行过的命令。标准全量检查：

```bash
cargo fmt --all -- --check
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace
```

外部检查依赖环境，按需手动运行：

```bash
scripts/external-integration.sh redis-recovery
scripts/external-integration.sh redis-lock
scripts/external-integration.sh redis-cluster
scripts/external-integration.sh otlp pyroscope
scripts/external-integration.sh linux-cpu
```

## 禁止做法

- 不跳过 spec validation 后直接把生成 Rust 当主要修复面。
- 不在未明确要求时覆盖已有业务逻辑。
- 不因为示例提到 Redis、DB、OTLP、profiling 就默认启用对应 feature。
- 不把 request id、用户 ID、cache key、SQL 原文、token、email、手机号放进 metrics label。
- 不把 Redis 锁当成数据库唯一约束、幂等、事务或 fencing token 的替代品。
