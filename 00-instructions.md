# AI Instructions for rs-zero

## Goal

按 rs-zero 的 Rust-first 微服务模式开发服务：先规格，后生成，再实现，最后验证。

## Read Order

1. `workflows.md`：选择任务流程。
2. `tools.md`：查命令。
3. `patterns.md`：查短模式。
4. `references/`：只在需要详细说明时读取具体主题。

## Core Rules

### 1. Spec-first

- REST 从 `.api` 开始。
- REST JWT 从 `.api` 的 `@server(jwt: Auth)` 开始，不手写绕过生成器。
- RPC 从 `.proto` 开始。
- model/cache 从 SQL schema 开始。

### 2. rzcli-first

优先运行 `rzcli`：

```bash
rzcli new-rest <service>
rzcli api validate -f <file>.api
rzcli api gen -f <file>.api -d <dir>
rzcli rpc gen -p <file>.proto -d <dir>
rzcli model gen -s <schema.sql> -d <dir>
```

### 3. Feature-aware

常用 feature：

```text
rest
rpc
resil
cache-redis
db-sqlite
db-mysql
db-postgres
discovery-etcd
discovery-kube
observability
observability-prometheus-client
otlp
profiling
```

使用某能力前，先确认 `Cargo.toml` 已启用对应 feature。

### 4. Production-first

服务代码默认考虑：

- timeout
- breaker
- concurrency limit
- limiter / shedder
- metrics / tracing
- health / readiness
- graceful shutdown
- external dependency boundary

### 5. Verification-first

标准验证：

```bash
cargo fmt --all -- --check
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace
```

局部修改可以先跑局部测试，但最终交付必须说明跑了哪些检查。

## Decision Tree

```text
New REST? -> .api -> rzcli api validate -> rzcli api gen -> implement -> docs -> verify
New RPC? -> .proto -> rzcli rpc gen -> implement -> docs -> verify
Model/cache? -> schema.sql -> rzcli model gen -> repository/cache tests -> verify
Resilience? -> config/layer -> tests for rejection/recovery -> verify
Observability? -> low-cardinality metrics/spans -> exporter opt-in -> verify
Production review? -> feature matrix -> external opt-in checks -> checklist
```

## Do Not

- 不生成空 stub。
- 不跳过验证。
- 不把外部 Redis、DB、OTLP、Pyroscope 测试放进默认路径。
- 不把 profiling 说成默认生产能力。
- 不把 rs-zero 描述成 go-zero 源码 1:1 翻译。
