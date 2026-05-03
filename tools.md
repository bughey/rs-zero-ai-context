# rs-zero Tools

## Prerequisites

```bash
rustc --version
cargo --version
rzcli -v
```

安装发布版：

```bash
cargo install rs-zero-cli --locked
```

在 rs-zero 仓库开发时可用：

```bash
cargo run -p rs-zero-cli -- -v
```

## Create REST Service

```bash
rzcli new-rest hello-service --dir target/generated
```

## API Commands

验证 `.api`：

```bash
rzcli api validate -f examples/api-hello/hello.api
```

生成 REST skeleton：

```bash
rzcli api gen -f examples/api-hello/hello.api -d target/generated
```

生成 OpenAPI：

```bash
rzcli api openapi -f examples/api-hello/hello.api -o target/openapi.json
```

## RPC Commands

```bash
rzcli rpc gen -p examples/rpc-hello/proto/hello.proto -d target/generated
```

## Model Commands

基础生成：

```bash
rzcli model gen -s examples/model-cache/schema.sql -d target/generated
```

带 SQLx repository：

```bash
rzcli model gen -s examples/model-cache/schema.sql -d target/generated --with-sqlx
```

带 Redis cache skeleton：

```bash
rzcli model gen -s examples/model-cache/schema.sql -d target/generated --with-sqlx --with-redis-cache
```

PostgreSQL：

```bash
rzcli model gen -s schema.sql -d target/generated --dialect postgres --with-sqlx
```

MySQL SQLx backend：

```bash
rzcli model gen -s schema.sql -d target/generated --sqlx-backend mysql --with-sqlx
```

## Compatibility

```bash
rzcli goctl compat matrix
```

rs-zero 追求输入语义和工作流兼容，不承诺生成 Go 文件或字节级兼容 goctl。

## Cargo Verification

快速：

```bash
cargo fmt --all -- --check
cargo test --workspace
```

完整：

```bash
cargo fmt --all -- --check
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace
```

Feature 检查：

```bash
cargo check -p rs-zero --no-default-features
cargo check -p rs-zero --no-default-features --features rest,rpc
cargo check -p rs-zero --no-default-features --features db-mysql,db-postgres,cache-redis,discovery-etcd,discovery-kube,otlp
```

## External Integration

外部依赖测试默认不跑。需要手动执行：

```bash
scripts/external-integration.sh redis-recovery
scripts/external-integration.sh redis-cluster
scripts/external-integration.sh otlp pyroscope
scripts/external-integration.sh linux-cpu
```

说明：

- `redis-cluster` 需要显式提供 cluster startup nodes。
- `linux-cpu` 只有 Linux 会执行真实 CPU pressure；其他平台应明确 skip。
- Pyroscope/profiling 是 opt-in / experimental。

## Post-generation Checklist

1. 检查生成目录。
2. 确认 `Cargo.toml` feature。
3. 补业务逻辑。
4. 补测试。
5. 补 README / API.md / RPC.md。
6. 跑 cargo 验证。
