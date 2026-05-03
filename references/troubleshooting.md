# Troubleshooting

## `rzcli` not found

检查：

```bash
rzcli -v
```

安装：

```bash
cargo install rs-zero-cli --locked
```

开发仓库中：

```bash
cargo run -p rs-zero-cli -- -v
```

## Feature missing

症状：模块或类型无法找到。

处理：检查 `Cargo.toml` 是否启用对应 feature，例如 `cache-redis`、`rpc`、`resil`、`observability`、`otlp`。

## API generation failed

处理顺序：

1. 修 `.api`。
2. 运行 `rzcli api validate -f <file>.api`。
3. 再运行 `rzcli api gen`。

不要绕过 spec 手写生成文件。

## RPC generation failed

检查：

- proto 是否为支持的 proto3 子集。
- import 路径是否本地可解析。
- output dir 是否可写。

## Redis tests skipped

这是预期行为。外部 Redis 测试默认 ignored 或由脚本 gate。

手动执行：

```bash
scripts/external-integration.sh redis-recovery
scripts/external-integration.sh redis-cluster
```

## OTLP or Pyroscope tests skipped

这是预期行为。collector 测试是 opt-in。

```bash
scripts/external-integration.sh otlp pyroscope
```

## Linux CPU test skipped

`linux-cpu` 只有 Linux 平台能验证真实 CPU provider。macOS 或 Windows 应明确 skip，不要伪造通过。

## Metrics cardinality issue

不要使用：

- user id。
- cache key。
- SQL 原文。
- 完整 URL。
- email / phone。

改用 route template、operation、status code 等低基数字段。

## Profiling overhead

profiling 是 opt-in / experimental。启用前必须压测 CPU、内存和采样开销。
