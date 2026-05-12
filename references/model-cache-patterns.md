# Model and Cache Patterns

## Model Workflow

1. 确认 SQL schema。
2. 选择 dialect：MySQL 或 PostgreSQL。
3. 选择 SQLx backend。
4. 用 `rzcli model gen` 生成。
5. 补 repository tests。

命令：

```bash
rzcli model gen -s schema.sql -d target/generated --with-sqlx
```

## REST API Integration

API 与 model 集成推荐保持三层：

1. `.api` 定义 HTTP contract 和 DTO。
2. SQL schema 生成 entity/repository/cache skeleton。
3. REST handler 只做 request -> use case/repository -> response 的编排。

流程：

```bash
rzcli api validate -f api/user.api
rzcli api gen -f api/user.api -d services
rzcli model gen -s schema/users.sql -d crates --name user-model --with-sqlx
```

REST 项目通过 `AppState` 持有 repository：

```rust
use user_model::repository::SqlxUsersRepository;

#[derive(Clone)]
pub struct AppState {
    pub users: SqlxUsersRepository,
}
```

handler 使用 `State<AppState>` 和 `api gen` 自动生成的 request extractor：

```rust
use axum::{Json, extract::State};
use rs_zero::rest::ApiResponse;

use crate::{state::AppState, types};

pub async fn create_user_handler(
    State(state): State<AppState>,
    Json(req): Json<types::CreateUserRequest>,
) -> ApiResponse<types::CreateUserResponse> {
    let _ = (&state, req);
    ApiResponse::success(types::CreateUserResponse::default())
}
```

更完整的项目模板见 `templates/API-MODEL.md`。

边界：

- model generator 不负责凭据、migration、transaction 边界。
- handler 不应直接散落 SQL。
- DTO 和 entity 不必强行共用类型；显式 mapper 更安全。
- cache-aside repository 只缓存主键和唯一索引读取，不缓存普通多行索引结果。

## Cache-aside

rs-zero cache-aside 关注：

- singleflight-style per-key miss merging。
- negative caching。
- TTL jitter。
- cache stats。
- generated cached repository skeleton。

## Local Cache

- `LruCacheStore` 是有界 L1 cache。
- 容量淘汰为 O(1) 风格结构。
- TTL 仍不是 go-zero timing wheel 模型；高容量、高 TTL churn 场景需要压测。

## Redis Cache

启用 feature：

```toml
rs-zero = { version = "0.2.3", features = ["cache-redis", "resil", "observability"] }
```

生产配置关注：

- timeout。
- breaker。
- unavailable policy：fail-open 或 fail-closed。
- metrics/tracing。
- delete retry。

## Redis Cluster vs Application Sharding

Redis Cluster：

- 使用 Redis Cluster 协议。
- 客户端处理 slot、MOVED/ASK、startup nodes。
- `delete_many` 应避免跨 slot 多 key 命令。

应用级分片：

- 应用根据 hash/rendezvous 选择节点。
- Redis 节点彼此不需要组成 Cluster。
- 不处理 Redis Cluster slot 或 MOVED/ASK。

不要把应用级分片描述成 Redis Cluster client。

## External Tests

默认测试不连接外部 Redis。手动验证：

```bash
scripts/external-integration.sh redis-recovery
scripts/external-integration.sh redis-cluster
```
