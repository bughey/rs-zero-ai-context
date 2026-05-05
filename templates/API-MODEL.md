# API + Model Integration Template

本模板用于把 `rzcli api gen` 生成的 REST 项目和 `rzcli model gen` 生成的 model/repository crate 集成起来。

## Scope

适用于：

- REST handler 需要读写数据库。
- REST handler 需要使用 generated SQLx repository。
- REST handler 需要可选 cache-aside repository。

不适用于：

- 自动迁移管理。
- 自动事务边界。
- 自动凭据注入。
- 把 model crate 当 ORM 使用。

## Input Specs

`.api` 示例：

```api
type CreateUserRequest {
    Email string `json:"email"`
    Name string `json:"name"`
}

type CreateUserResponse {
    Id i64 `json:"id"`
}

service user-api {
    @handler CreateUserHandler
    post /users (CreateUserRequest) returns (CreateUserResponse)
}
```

`schema.sql` 示例：

```sql
CREATE TABLE users (
    id INTEGER PRIMARY KEY,
    email TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL
);
```

## Generation

```bash
rzcli api validate -f api/user.api
rzcli api gen -f api/user.api -d services
rzcli model gen -s schema/users.sql -d crates --name user-model --with-sqlx
```

需要 Redis cache-aside 时：

```bash
rzcli model gen \
  -s schema/users.sql \
  -d crates \
  --name user-model \
  --with-sqlx \
  --with-redis-cache
```

## Cargo Wiring

REST 项目依赖 model crate：

```toml
[dependencies]
user-model = { path = "../../crates/user-model" }
rs-zero = { version = "0.2.3", default-features = false, features = ["rest", "observability", "db-sqlite"] }
sqlx = { version = "0.8", default-features = false, features = ["runtime-tokio", "sqlite"] }
```

按实际后端替换 feature：

- SQLite：`db-sqlite` + `sqlx/sqlite`
- MySQL：`db-mysql` + `sqlx/mysql`
- PostgreSQL：`db-postgres` + `sqlx/postgres`
- Redis cache：额外启用 `cache-redis` 和 `resil`

## AppState

```rust
use user_model::repository::SqlxUsersRepository;

#[derive(Clone)]
pub struct AppState {
    pub users: SqlxUsersRepository,
}

impl AppState {
    pub fn new(users: SqlxUsersRepository) -> Self {
        Self { users }
    }
}
```

如果使用 cache-aside repository：

```rust
use rs_zero::cache::CacheStore;
use user_model::repository::CachedUsersRepository;

#[derive(Clone)]
pub struct AppState<S>
where
    S: CacheStore + Clone,
{
    pub users: CachedUsersRepository<S>,
}
```

## main.rs Wiring

```rust
mod handler;
mod router;
mod state;
mod types;

use anyhow::Result;
use rs_zero::core::{LogConfig, init_tracing, shutdown_signal};
use rs_zero::db::{DatabaseConfig, connect_sqlite_pool};
use rs_zero::observability::{MetricsRegistry, metrics_router};
use rs_zero::rest::{RestConfig, RestMetricsConfig, RestMiddlewareConfig, RestServer};
use state::AppState;
use user_model::repository::SqlxUsersRepository;

#[tokio::main]
async fn main() -> Result<()> {
    let _ = init_tracing(LogConfig::from_env());
    let metrics = MetricsRegistry::new();

    let pool = connect_sqlite_pool(&DatabaseConfig::sqlite("sqlite://app.db")).await?;
    let users = SqlxUsersRepository::new(pool).with_metrics(metrics.clone());
    let state = AppState::new(users);

    let router = router::router()
        .merge(metrics_router(metrics.clone()))
        .with_state(state);

    let config = RestConfig {
        metrics_registry: Some(metrics),
        middlewares: RestMiddlewareConfig {
            metrics: RestMetricsConfig { enabled: true },
            ..RestMiddlewareConfig::default()
        },
        ..RestConfig::default()
    };

    let addr = "127.0.0.1:8080".parse()?;
    RestServer::new(config, router)
        .serve_with_shutdown(addr, shutdown_signal())
        .await?;
    Ok(())
}
```

## Handler Pattern

```rust
use axum::{Json, extract::State};
use rs_zero::rest::ApiResponse;
use user_model::entity::Users;

use crate::{state::AppState, types};

pub async fn create_user_handler(
    State(state): State<AppState>,
    Json(req): Json<types::CreateUserRequest>,
) -> ApiResponse<types::CreateUserResponse> {
    let user = Users {
        id: 0,
        email: req.email,
        name: req.name,
    };

    state
        .users
        .save(&user)
        .await
        .expect("replace with service error mapping");

    ApiResponse::success(types::CreateUserResponse { id: user.id })
}
```

注意：示例中的 `expect` 只是占位。真实服务必须把 repository 错误映射为明确的业务错误或统一 REST 错误，不要 panic，也不要吞掉错误。

## Router State

如果 handler 使用 `State<AppState>`，需要让 router 带上 state。推荐先合并无状态路由，再调用 `.with_state(state)`：

```rust
let router = router::router()
    .merge(metrics_router(metrics.clone()))
    .with_state(state);
```

如果编译器提示 state 类型不匹配，再把生成的 `router.rs` 调整为显式 state 版本：

```rust
use axum::{Router, routing};

use crate::{handler, state::AppState};

pub fn router() -> Router<AppState> {
    Router::new().route("/users", routing::post(handler::create_user_handler))
}
```

安全增量模式会继续更新 `router.rs`，所以这类手动改动要谨慎。优先保持生成路由结构简单，把业务 wiring 放在 `main.rs` / `state.rs`。

## Cache Wiring

```rust
use rs_zero::cache::{CacheAside, CacheAsideConfig, LruCacheStore, TwoLevelCacheStore};
use rs_zero::cache::redis::{RedisCacheConfig, RedisCacheStore};
use user_model::repository::{CachedUsersRepository, SqlxUsersRepository};

let sql_repo = SqlxUsersRepository::new(pool).with_metrics(metrics.clone());
let l1 = LruCacheStore::new(10_000)?;
let l2 = RedisCacheStore::new(RedisCacheConfig::default())?;
let cache = CacheAside::new(TwoLevelCacheStore::new(l1, l2), CacheAsideConfig::default());
let users = CachedUsersRepository::new(sql_repo, cache, "user-api");
```

## Testing

优先补三类测试：

- DTO 到 repository entity 的映射测试。
- handler 成功和 repository 错误测试。
- cache-aside 命中、miss、失效测试。

验证：

```bash
cargo fmt --all -- --check
cargo test
cargo clippy --workspace --all-targets -- -D warnings
```

外部 DB / Redis 测试保持 opt-in，不放进默认路径。

## Production Notes

- 凭据从环境变量、配置文件或 secret manager 读取，不写入代码。
- migration 由应用或部署流程负责，model generator 不负责执行迁移。
- transaction 边界由业务 use case 决定，不放进通用 repository 模板里。
- SQL metrics label 保持低基数，不记录 SQL 原文或用户输入。
- cache key 不进入 metrics label 或日志敏感字段。
