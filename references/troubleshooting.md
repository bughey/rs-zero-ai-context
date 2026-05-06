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
3. 需要时运行 `rzcli api format -f <file>.api -o <formatted>.api` 预览格式化结果。
4. 再运行 `rzcli api gen`。

不要绕过 spec 手写生成文件。

## `.api import` not found

检查：

- import 路径是否相对当前 `.api` 文件所在目录。
- 文件名大小写是否和磁盘一致。
- import 文件是否已提交到仓库。
- 是否存在循环 import 或重复类型定义。

处理：先修通 `rzcli api validate -f <file>.api`，再运行 `api format`、`api gen` 或 `api openapi`。不要把 import 类型复制到生成的 Rust 文件里规避解析失败。

## `api format` did not overwrite the file

`api format` 默认写到 `-o` 指定输出。覆盖原 `.api` 必须显式：

```bash
rzcli api format -f <file>.api -o <file>.api --force
```

覆盖前建议先输出到临时文件 review。

## OpenAPI is stale

症状：`openapi.json` 和当前 `.api` 或生成代码不一致。

处理顺序：

1. `rzcli api validate -f <file>.api`
2. `rzcli api gen -f <file>.api -d <output-dir>`
3. `rzcli api openapi -f <file>.api -o <openapi>.json`

OpenAPI 是文档产物，不会自动更新 Rust skeleton。

## goctl compatibility expectation mismatch

rs-zero 追求 goctl 输入语义和工作流兼容，不生成 Go 代码，也不承诺字节级输出兼容。

检查：

```bash
rzcli goctl compat matrix
```

当前 `.api` 解析覆盖常用语法，但一个解析入口文件内只验证一个 service。多 service 或非常规 goctl 边缘语法应先用 `api validate` 确认。

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

## RPC `request_id` is empty

症状：API 日志有 `request_id`，RPC INFO 日志中 `request_id=""`。

常见原因：RPC server 代码在观测前调用了 `request.into_inner()`，导致 tonic metadata 被丢弃。`x-request-id` 和 `traceparent` 都在 metadata 中，不在业务 message 中。

处理：

1. API client 保持 `request_id_interceptor()`，需要 trace 时同时使用 `trace_context_interceptor()`。
2. 普通 REST handler 内不要手写 `HeaderMap` 到 RPC metadata 的 request id 注入；确认 REST metrics middleware 已启用，它会自动把当前 request id 放入 task-local scope。
3. 如果 RPC 调用发生在后台任务或脱离 HTTP handler 的异步边界，使用 `with_rpc_request_id(...)` 显式包裹调用范围。
4. RPC server 侧使用 `RpcRequestParts::from_request(request)` 或 `request.into_parts()` 保留 metadata。
5. 使用 `*_with_parts(...)`、`RpcResilienceLayer::run_unary_with_metadata(...)` 或 `observe_rpc_unary_with_metadata(...)`。
6. 确认生成项目依赖的 `rs-zero` 版本包含该能力；旧版本生成代码即使客户端带了 metadata，server 侧也可能记录为空。
