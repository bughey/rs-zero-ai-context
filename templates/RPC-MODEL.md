# RPC + Model Integration Template

本模板用于把 `rzcli rpc gen` 生成的 tonic RPC 服务和 `rzcli model gen` 生成的 model/repository crate 集成起来。

## Scope

适用于：

- RPC unary 方法需要读写数据库。
- RPC service 需要调用 generated SQLx repository。
- RPC service 需要可选 cache-aside repository。

不适用于：

- 自动 migration。
- 自动 transaction 边界。
- 把 model crate 当 ORM 使用。
- streaming 方法里长时间持有 DB 资源。

## Input Specs

`.proto` 示例：

```proto
syntax = "proto3";

package user.v1;

service UserService {
  rpc GetUser (GetUserRequest) returns (GetUserReply);
}

message GetUserRequest {
  int64 id = 1;
}

message GetUserReply {
  int64 id = 1;
  string email = 2;
  string name = 3;
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
rzcli rpc gen -p proto/user.proto -d services
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

RPC 服务依赖 model crate：

```toml
[dependencies]
user-model = { path = "../../crates/user-model" }
rs-zero = { version = "0.2", default-features = false, features = ["rpc", "resil", "observability", "db-sqlite"] }
sqlx = { version = "0.8", default-features = false, features = ["runtime-tokio", "sqlite"] }
```

按实际后端替换 feature：

- SQLite：`db-sqlite` + `sqlx/sqlite`
- MySQL：`db-mysql` + `sqlx/mysql`
- PostgreSQL：`db-postgres` + `sqlx/postgres`
- Redis cache：额外启用 `cache-redis` 和 `resil`

## Service State

```rust
use rs_zero::rpc::{RpcServerConfig, RpcServerLayerStack};
use user_model::repository::SqlxUsersRepository;

#[derive(Clone)]
pub struct UserService {
    users: SqlxUsersRepository,
    config: RpcServerConfig,
}

impl UserService {
    pub fn new(users: SqlxUsersRepository, config: RpcServerConfig) -> Self {
        Self { users, config }
    }

    pub fn server_layer_stack(&self) -> RpcServerLayerStack {
        RpcServerLayerStack::new(self.config.clone())
    }
}
```

如果使用 cache-aside repository：

```rust
use rs_zero::cache::CacheStore;
use rs_zero::rpc::{RpcServerConfig, RpcServerLayerStack};
use user_model::repository::CachedUsersRepository;

#[derive(Clone)]
pub struct UserService<S>
where
    S: CacheStore + Clone,
{
    users: CachedUsersRepository<S>,
    config: RpcServerConfig,
}
```

## Unary Method Pattern

unary server 默认使用 Tower-first 路径：在 tonic server 外层挂 `RpcServerLayerStack`，method 内只处理业务 message。

```rust
use tonic::{Request, Response, Status};

use crate::proto::user_service_server::UserService;
use crate::proto::{GetUserReply, GetUserRequest};
use crate::UserService as AppService;

#[tonic::async_trait]
impl UserService for AppService {
    async fn get_user(
        &self,
        request: Request<GetUserRequest>,
    ) -> Result<Response<GetUserReply>, Status> {
        let req = request.into_inner();
        let user = self
            .users
            .find_by_id(req.id)
            .await
            .map_err(|error| Status::internal(error.to_string()))?
            .ok_or_else(|| Status::not_found("user not found"))?;

        Ok(Response::new(GetUserReply {
            id: user.id,
            email: user.email,
            name: user.name,
        }))
    }
}
```

server 启动时挂 layer：

```rust
use tonic::transport::Server;

let service = AppService::new(users, rpc_config);
let layer = service.server_layer_stack().into_layer();
let tonic_service = user_service_server::UserServiceServer::new(service);

Server::builder()
    .layer(layer)
    .add_service(tonic_service)
    .serve(addr)
    .await?;
```

规则：

- `RpcServerLayerStack` 负责 timeout、concurrency、breaker、shedder、metrics/tracing，并从 tonic metadata 读取 `x-request-id` 与 `traceparent`。
- repository 错误要显式映射为 `tonic::Status`。
- 不要吞掉错误，更不要 panic。
- 默认不要在方法内再套 `run_unary`，避免和外层 `RpcServerLayerStack` 重复。

## Compatibility Helper Pattern

只有未挂 `RpcServerLayerStack` 的旧项目或手写高级场景，才在方法内使用 metadata-aware helper：

```rust
let (metadata, _extensions, req) = request.into_parts();
let reply = self.resilience
    .run_unary_with_metadata("GetUser", &metadata, move || async move {
        self.handle_get_user(req).await
    })
    .await?;
```

## Cache Wiring

```rust
use rs_zero::cache::{CacheAside, CacheAsideConfig, LruCacheStore, TwoLevelCacheStore};
use rs_zero::cache::redis::{RedisCacheConfig, RedisCacheStore};
use user_model::repository::{CachedUsersRepository, SqlxUsersRepository};

let sql_repo = SqlxUsersRepository::new(pool).with_metrics(metrics.clone());
let l1 = LruCacheStore::new(10_000)?;
let l2 = RedisCacheStore::new(RedisCacheConfig::default())?;
let cache = CacheAside::new(TwoLevelCacheStore::new(l1, l2), CacheAsideConfig::default());
let users = CachedUsersRepository::new(sql_repo, cache, "user-rpc");
```

## Streaming Boundary

streaming 方法不要长时间持有 DB 连接或大批量结果。

推荐：

- 分页读取。
- 分段写出。
- 把 DB 访问限制在短循环内。
- 需要 send/recv 观察时使用 `ObservedRecvStream`、`record_stream_send`、`run_observed_stream`。

不要把 streaming wrapper 描述成完整 tonic stream interceptor 替代品。

## Testing

优先补三类测试：

- proto message 到 repository entity 的映射测试。
- unary 方法的成功、not found、repository error 测试。
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
- unary server 优先挂 `RpcServerLayerStack`，不要让业务漏接韧性；旧 helper 只用于兼容路径。
- SQL / RPC metrics label 保持低基数。
- cache key 不进入 metrics label 或日志敏感字段。
