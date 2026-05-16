# Model 与 Cache

当服务需要生成 SQL model、SQLx repository、cache-aside、Redis cache，或需要把 API/RPC 与 model crate 集成时读取本文件。

## Model 工作流

1. 确认 SQL schema 和目标 dialect。
2. 选择后端 feature：`db-sqlite`、`db-mysql` 或 `db-postgres`。
3. 使用 `rzcli model gen` 生成 model 代码。
4. 手动补 repository tests、migration 和运行时连接池接入。

命令：

```bash
rzcli model gen -s schema.sql -d target/generated --with-sqlx
rzcli model gen -s schema.sql -d target/generated --with-sqlx --with-redis-cache
```

边界：

- 生成器不负责凭据、migration、transaction 边界或连接池所有权。
- 生成 repository 是持久化 adapter，不承载业务逻辑。
- DTO 与数据库 entity 应显式转换。
- cache-aside repository 应优先缓存主键和唯一索引读取；普通多行查询除非有明确失效策略，否则不要缓存。

## REST API + model 集成

推荐结构：

1. `.api` 定义 HTTP contract 和 DTO。
2. SQL schema 生成 entity/repository/cache skeleton。
3. `AppState` 持有 repository。
4. `handler` 只做 axum extractor 和 response 适配。
5. `logic` 接收 `.api` request 类型，调用 repository，将 entity 映射为 response DTO，并返回业务错误或响应类型。

生成：

```bash
rzcli api validate -f api/user.api
rzcli api gen -f api/user.api -d services
rzcli model gen -s schema/users.sql -d crates --name user-model --with-sqlx
```

State 模式：

```rust
use user_model::repository::SqlxUsersRepository;

#[derive(Clone)]
pub struct AppState {
    pub users: SqlxUsersRepository,
}
```

Handler 保持薄层：

```rust
pub async fn create_user_handler(
    State(state): State<AppState>,
    Json(req): Json<types::CreateUserRequest>,
) -> ApiResponse<types::CreateUserResponse> {
    match logic::user_logic::create_user(&state, req).await {
        Ok(reply) => ApiResponse::success(reply),
        Err(error) => ApiResponse::fail("MODEL_ERROR", error.to_string()),
    }
}
```

Logic 承载业务流程：

```rust
pub async fn create_user(
    state: &AppState,
    req: types::CreateUserRequest,
) -> anyhow::Result<types::CreateUserResponse> {
    let user = state.users.create(req.email, req.name).await?;
    Ok(types::CreateUserResponse { id: user.id })
}
```

## RPC + model 集成

推荐结构：

1. `.proto` 定义 RPC request/reply message。
2. SQL schema 生成 repository/cache skeleton。
3. RPC service state 持有 repository。
4. `handler` 实现 tonic service trait 并调用 logic。
5. `logic` 接收 proto request message，返回 proto reply message。

生成：

```bash
rzcli rpc gen -p proto/user.proto -d services
rzcli model gen -s schema/users.sql -d crates --name user-model --with-sqlx
```

Logic 示例：

```rust
pub async fn get_user(
    state: &AppState,
    req: proto::GetUserRequest,
) -> Result<proto::GetUserReply, Status> {
    let user = state
        .users
        .find_by_id(req.id)
        .await
        .map_err(|error| Status::internal(error.to_string()))?
        .ok_or_else(|| Status::not_found("user not found"))?;

    Ok(proto::GetUserReply {
        id: user.id,
        email: user.email,
        name: user.name,
    })
}
```

Streaming RPC 中不要跨 yield 长时间持有数据库事务或 borrowed row。需要流式返回时，按 bounded batch 查询或先 materialize 响应项。

## Cache-aside 模式

rs-zero cache-aside 关注：

- per-key miss merging。
- negative caching。
- TTL jitter。
- cache stats。
- generated cached repository skeleton。

本地缓存：

- `LruCacheStore` 是有界 L1 cache。
- 淘汰使用 O(1) 风格结构。
- TTL 不等同于 go-zero timing-wheel 过期模型；高容量、高 TTL churn 场景需要基准压测。

Redis 缓存：

```toml
rs-zero = { version = "0.2", features = ["cache-redis", "resil", "observability"] }
```

生产检查：

- timeout 已配置。
- breaker 已启用。
- unavailable policy 已明确：fail-open 或 fail-closed。
- metrics/tracing 已接入。
- delete retry 行为已验证。

## Redis Cluster 与应用级分片

Redis Cluster：

- 使用 Redis Cluster 协议。
- client 处理 slot、`MOVED`、`ASK` 和 startup nodes。
- multi-key 命令必须避免 cross-slot；需要多 key 时使用相同 hash tag。

应用级分片：

- 应用通过 hash/rendezvous 选择节点。
- Redis 节点不需要组成 Redis Cluster。
- 不处理 Redis Cluster slot 或重定向。

不要把应用级分片描述成 Redis Cluster client。

## 外部测试

默认测试不应依赖 Redis。修改 Redis 行为后手动运行外部检查：

```bash
scripts/external-integration.sh redis-recovery
scripts/external-integration.sh redis-cluster
```
