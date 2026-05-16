# rzcli Commands

`rzcli` 是 rs-zero 的生成工具。AI 应优先使用它生成 skeleton，再补业务逻辑、测试和文档。

## Version

```bash
rzcli -v
```

开发仓库中：

```bash
cargo run -p rs-zero-cli -- -v
```

## `new-rest`

```bash
rzcli new-rest <service-name> --dir <output-dir>
```

用途：创建 REST 服务骨架。

失败处理：

- 输出目录已有文件时，确认是否可覆盖；不要默认删除用户代码。
- 生成后检查 `Cargo.toml` feature。

## `api validate`

```bash
rzcli api validate -f <file>.api
```

用途：验证 `.api` 语法。

规则：

- `import` 路径按当前 `.api` 文件所在目录解析。
- 生成或导出前先 validate，避免把未解析 import、重复类型或 handler 缺失带到后续文件。
- 当前兼容边界以 goctl compatibility matrix 为准；不要假设所有 goctl 边缘语法都已覆盖。

失败处理：修 `.api`，不要绕过 spec。

## `api format`

```bash
rzcli api format -f <file>.api -o <output>.api
rzcli api format -f <file>.api -o <file>.api --force
```

用途：稳定格式化 `.api`，让 spec review 更清晰。

规则：

- 格式化前先运行 `api validate`。
- `-o` 指向新文件时适合预览格式化结果。
- 覆盖原文件必须显式使用 `--force`。
- 格式化只整理 `.api` 表达，不替代语义修复。

## `api gen`

```bash
rzcli api gen -f <file>.api -d <output-dir>
```

用途：从 `.api` 生成 REST skeleton。

增量生成参数：

```bash
rzcli api gen -f <file>.api -d <output-dir> --dry-run
rzcli api gen -f <file>.api -d <output-dir> --overwrite-handlers
rzcli api gen -f <file>.api -d <output-dir> --force
```

规则：

- 默认安全增量：更新 `src/types.rs`、`src/router.rs`、`src/handler/mod.rs`、`src/logic/mod.rs`，保留已有 handler/logic 和项目文件。
- `--dry-run` 只展示 create/update/skip/conflict 计划。
- `--overwrite-handlers` 覆盖 handler 和 logic 文件。
- `--force` 覆盖全部生成文件。
- handler 入参会根据 request tag 自动生成：`path` -> `Path`，`query` / `form` -> `Query`，否则使用 `Json`；无 request 时无入参。
- 生成结构为 `handler` / `logic` 两层：handler 调 logic，业务逻辑优先写在 `src/logic/*`。

JWT：

- `.api` 中的 `@server(jwt: Auth)` 会在首次生成时接入 JWT 配置。
- secret 读取顺序：`JWT_AUTH_SECRET` -> `[auth].jwt_secret`。
- 过期时间读取顺序：`JWT_AUTH_EXPIRES` -> `[auth].jwt_expires` -> `7200`，单位秒。
- `jwt: AdminAuth` 对应 `JWT_ADMIN_AUTH_SECRET` 和 `JWT_ADMIN_AUTH_EXPIRES`。
- 已有项目后新增 JWT 时，默认不会覆盖 `main.rs` 和 `etc/*.toml`；需要手动合并或使用 `--force`。

完成后：

1. 补业务逻辑。
2. 补测试。
3. 更新 README / API.md。
4. 运行 cargo 验证。

## `api openapi`

```bash
rzcli api openapi -f <file>.api -o <file>.json
```

用途：生成 OpenAPI JSON。

导出范围：

- route method 与 path。
- request 中的 `path`、`query`、`header`、`form`、`json` 字段 tag。
- response schema。
- `@server` 的 prefix、group、middleware、jwt 等可解析元数据。

使用建议：

- 文档生成前先运行 `api validate`。
- `.api import` 文件缺失或相对路径错误时，先修 import，再导出 OpenAPI。
- OpenAPI 是接口文档产物，不替代 `api gen` 的 Rust skeleton。

## `rpc gen`

```bash
rzcli rpc gen -p <file>.proto -d <output-dir>
```

用途：从 proto 生成 tonic-oriented RPC skeleton。

注意：

- 真实 prost build wiring 可能需要应用项目补齐。
- 生成结构为 `handler` / `logic` 两层：handler 负责 tonic 适配，logic 负责 proto message 级业务方法。
- 生成代码默认暴露 `server_layer_stack()`，优先接入 `RpcServerLayerStack`。

## `model gen`

```bash
rzcli model gen -s <schema.sql> -d <output-dir>
```

常用参数：

```bash
--with-sqlx
--with-redis-cache
--dialect postgres
--sqlx-backend mysql
```

用途：从 SQL schema 生成 entity、repository trait、SQLx skeleton 和 cache skeleton。

注意：

- 默认测试不连接真实 DB。
- 需要 Redis cache 时启用 `cache-redis` feature。

## `goctl compat matrix`

```bash
rzcli goctl compat matrix
```

用途：查看 goctl 输入语义兼容范围。

边界：

- rs-zero 追求 goctl 输入语义和工作流兼容，不承诺生成 Go 代码。
- 不承诺 goctl 输出字节级兼容。
- 当前 `.api` 解析覆盖常用 `syntax`、`import`、`type`、`@server`、`service`、`@handler` 和 route 语义。
- 一个解析入口文件内只验证一个 service；多 service 或非常规 goctl 边缘语法应先用 `api validate` 确认。
