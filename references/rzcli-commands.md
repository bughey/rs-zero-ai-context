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

失败处理：修 `.api`，不要绕过 spec。

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

- 默认安全增量：更新 `src/types.rs`、`src/router.rs`、`src/handler/mod.rs`，保留已有 handler 和项目文件。
- `--dry-run` 只展示 create/update/skip/conflict 计划。
- `--overwrite-handlers` 只覆盖 handler。
- `--force` 覆盖全部生成文件。
- handler 入参会根据 request tag 自动生成：`path` -> `Path`，`query` / `form` -> `Query`，否则使用 `Json`；无 request 时无入参。

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

## `rpc gen`

```bash
rzcli rpc gen -p <file>.proto -d <output-dir>
```

用途：从 proto 生成 tonic-oriented RPC skeleton。

注意：

- 真实 prost build wiring 可能需要应用项目补齐。
- 生成代码默认可接入 rs-zero RPC 韧性 helper。

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

rs-zero 不承诺生成 Go 代码，也不承诺 goctl 输出字节级兼容。
