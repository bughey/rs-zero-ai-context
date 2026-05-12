# AI Instructions for rs-zero (Codex)

Codex 会自动读取项目根目录的 `AGENTS.md`。本文件用于让 Codex 按 rs-zero 的工程规则开发服务。

## Context Roots

从第一个存在的目录解析 rs-zero AI context：

1. `.codex/rs-zero-ai-context`
2. `.github/rs-zero-ai-context`
3. `.cursor/rs-zero-ai-context`
4. `.windsurf/rs-zero-ai-context`
5. `.claude/rs-zero-ai-context`
6. `.`

## File Priority

按需读取：

1. `workflows.md`：任务流程。
2. `tools.md`：`rzcli` 和 cargo 命令。
3. `patterns.md`：短模式和片段。
4. `references/`：详细主题说明。
5. `templates/`：服务文档模板。

不要在每轮对话中一次性加载整个 `references/` 目录。

## Rules

### Spec-first

- REST：先创建或修改 `.api`。
- REST JWT：使用 `.api` 的 `@server(jwt: Auth)`，生成后检查 `JWT_*_SECRET`、`JWT_*_EXPIRES` 和 `[auth]` 配置。
- RPC：先创建或修改 `.proto`，server unary 默认接入 `RpcServerLayerStack`，client 默认使用 `RpcClientBuilder`。
- model/cache：先创建或确认 SQL schema。
- 修改已有服务时，先定位现有 spec，再生成或补齐代码。

### rzcli-first

优先使用 `rzcli`：

```bash
rzcli new-rest <service>
rzcli api validate -f <file>.api
rzcli api gen -f <file>.api -d <dir>
rzcli api openapi -f <file>.api -o <file>.json
rzcli rpc gen -p <file>.proto -d <dir>
rzcli model gen -s <schema.sql> -d <dir>
```

不要手写大段 skeleton 来替代生成器。只有业务逻辑、测试、配置和文档应由 AI 补齐。

### Feature-aware

生成或修改代码时显式检查 Cargo feature：

| 能力 | Feature |
| --- | --- |
| REST | `rest` |
| RPC / tonic | `rpc` |
| 服务韧性 | `resil` |
| Redis cache / limiter | `cache-redis` |
| SQLite | `db-sqlite` |
| MySQL | `db-mysql` |
| PostgreSQL | `db-postgres` |
| etcd discovery | `discovery-etcd` |
| Kubernetes discovery | `discovery-kube` |
| 基础观测 | `observability` |
| 成熟 Prometheus client | `observability-prometheus-client` |
| OTLP tracing | `otlp` |
| profiling | `profiling` |

不要假设所有能力默认启用。

### Production-first

涉及服务路径时默认检查：

- timeout
- circuit breaker
- concurrency limit
- token/period limiter
- adaptive shedder
- metrics/tracing
- health/readiness
- graceful shutdown
- external dependency test boundary

profiling 只能作为 opt-in / experimental 能力描述。

### Verification-first

最小验证：

```bash
cargo fmt --all -- --check
cargo test --workspace
```

生产或跨模块修改优先运行：

```bash
cargo fmt --all -- --check
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace
```

只改生成项目时，先跑该项目的 `cargo check` / `cargo test`，再视影响范围跑 workspace 验证。

## Decision Tree

```text
User request
├─ New REST service -> write .api -> rzcli api gen/new-rest -> implement -> docs -> cargo verify
├─ Modify REST API -> edit .api -> rzcli api validate/gen -> update logic/tests/docs -> cargo verify
├─ New RPC service -> write .proto -> rzcli rpc gen -> mount RpcServerLayerStack -> implement -> docs -> cargo verify
├─ Model/cache -> confirm SQL schema -> rzcli model gen -> wire repository/cache -> tests -> cargo verify
├─ Resilience -> choose layer/config -> add tests -> cargo verify
├─ Observability -> choose registry/exporter -> add low-cardinality metrics/spans -> cargo verify
└─ Production review -> feature matrix -> resilience -> observability -> external opt-in checks
```

## Documentation

为新服务生成或更新：

- `README.md`：概览、配置、运行、测试、生产边界。
- `API.md`：REST endpoint 说明。
- `RPC.md`：RPC method 和 grpcurl 示例。

可参考 `templates/`。

## Avoid

- 空 stub 或只返回 TODO。
- 手写生成器应该生成的骨架。
- 跳过 `cargo fmt`、`cargo test` 或必要的 `cargo clippy`。
- 在未声明 feature 时使用 Redis、RPC、DB、OTLP 或 profiling API。
- 默认连接外部 Redis、DB、OTLP collector、Pyroscope。
- 在 metrics label 中放用户 ID、key、SQL 原文、完整 URL 或高基数字段。
- 宣称 profiling 默认生产就绪。
