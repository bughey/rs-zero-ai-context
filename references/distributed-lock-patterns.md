# Distributed Lock Patterns

rs-zero 的分布式锁目前以 Redis backend 为主，适合跨进程短时互斥。AI 生成业务代码时必须把锁作为保护临界区的辅助能力，而不是数据库事务、唯一约束、幂等或 fencing token 的替代品。

## Feature

```toml
rs-zero = { version = "0.2.1", default-features = false, features = ["cache-redis"] }
```

`cache-redis` 会启用 Redis lock 所需的 Redis client、timeout 和 breaker 能力。

## Basic Usage

```rust
use std::time::Duration;

use rs_zero::lock::{DistributedLock, RedisDistributedLock, RedisLockConfig};

let lock = RedisDistributedLock::new(RedisLockConfig::default())?;
let guard = lock
    .try_acquire("order:{123}", Duration::from_secs(5))
    .await?;

// Critical section must finish before TTL.

lock.release(&guard).await?;
```

规则：

- TTL 必须显式设置，禁止无过期锁。
- 解锁通过 owner token 校验后删除，不能用裸 `DEL`。
- 不要把 owner token、原始 lock key、Redis URL 或用户输入写入日志和 metrics label。
- 临界区必须短于 TTL；第一版不默认自动续租。

## Redis Cluster

Cluster lock key 默认必须带 hash tag：

```rust
lock.try_acquire("order:{123}", Duration::from_secs(5)).await?;
```

不要生成没有 hash tag 的 Cluster lock key。获取锁和 Lua 解锁必须落到同一个 Redis slot。

## Failure Handling

常见错误处理：

- `LockError::Busy`：锁被占用，按业务返回冲突、排队或稍后重试。
- `LockError::OwnerMismatch`：锁已过期或被新 owner 获取，不能假装释放成功。
- `LockError::Timeout` / `Connection` / `BreakerOpen`：Redis 不可用，默认不要进入临界区。

## Production Boundaries

- 分布式锁不是数据库唯一约束、事务或幂等设计的替代品。
- 保护外部资源写入时，应设计 fencing token、版本号或 CAS 检查。
- Redis 单实例锁适合短时互斥；强一致协调场景优先考虑 etcd lease/txn 或数据库事务。
- 第一版不实现 Redlock，也不默认自动续租。
- 真实 Redis 验证使用 `scripts/external-integration.sh redis-lock`，默认 CI 不依赖 Redis。
