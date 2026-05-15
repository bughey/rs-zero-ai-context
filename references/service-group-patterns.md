# Service Group Patterns

rs-zero 支持 go-zero 风格的 service group，用于在同一进程内统一管理 REST、RPC 和后台 worker 生命周期。

## When to Use

使用 service group：

- 一个进程内同时运行 REST API 和 RPC server。
- 服务还需要后台 worker、scheduler、consumer 或 metrics exporter。
- 希望 Ctrl-C / shutdown 时统一停止所有长期任务。
- 任一子服务异常退出时，需要触发全组收敛。

不要用 service group 替代：

- Kubernetes / systemd / supervisor 这类进程级编排。
- 数据库事务、消息确认或分布式一致性边界。
- 有严格启动依赖 DAG 的复杂编排；这类依赖应显式建 readiness/health。

## Core API

常用类型：

- `rs_zero::core::ServiceGroup`
- `rs_zero::core::ServiceGroupConfig`
- `rs_zero::core::ServiceGroupHandle`
- `rs_zero::core::Service`
- `rs_zero::core::ShutdownToken`

基本语义：

- 服务并发启动，不保证启动顺序。
- `ServiceGroup::start()` 默认等待 Ctrl-C。
- `start_with_shutdown(future)` 可用于测试或自定义停止信号。
- 任一服务错误退出时，默认停止全组。
- shutdown 会广播 `ShutdownToken`，再执行 `stop()` hook。
- 停止过程受 `shutdown_timeout` 保护，不应无限等待。

## REST + RPC Example

```rust
use axum::{Router, routing::get};
use rs_zero::{
    core::ServiceGroup,
    rest::{ApiResponse, RestConfig, RestServer, RestService},
    rpc::TonicHealthService,
};

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let rest_router = Router::new().route("/ready", get(|| async {
        ApiResponse::success("ok")
    }));

    let rest = RestService::new(
        "api",
        "127.0.0.1:8080".parse()?,
        RestServer::new(RestConfig::production_defaults("api"), rest_router),
    );
    let rpc = TonicHealthService::new("health-rpc", "127.0.0.1:50051".parse()?);

    let mut group = ServiceGroup::new();
    group.add(rest);
    group.add(rpc);
    group.start().await?;

    Ok(())
}
```

## Worker Example

```rust
use std::time::Duration;
use rs_zero::core::{CoreError, ServiceGroup};

let mut group = ServiceGroup::new();
group.add_fn("worker", |shutdown| async move {
    loop {
        tokio::select! {
            _ = shutdown.cancelled() => break,
            _ = tokio::time::sleep(Duration::from_secs(1)) => {
                // do periodic work
            }
        }
    }
    Ok::<(), CoreError>(())
});
```

## Custom Service

```rust
use rs_zero::core::{Service, ServiceFuture, ShutdownToken};

struct Worker;

impl Service for Worker {
    fn name(&self) -> &str {
        "worker"
    }

    fn start(&self, shutdown: ShutdownToken) -> ServiceFuture<'_> {
        Box::pin(async move {
            shutdown.cancelled().await;
            Ok(())
        })
    }
}
```

## Verification

```bash
cargo test -p rs-zero --test service_group
cargo test -p rs-zero --test service_group_adapters
```

## Production Rules

- worker 必须监听 `ShutdownToken`。
- 不要依赖 service group 启动顺序。
- 对需要准备时间的服务，提供 readiness/health。
- `stop()` 中不要执行无限等待逻辑。
- 设置合理 `ServiceGroupConfig::shutdown_timeout`。
- 子服务错误不要吞掉；让 service group 触发全组收敛。
