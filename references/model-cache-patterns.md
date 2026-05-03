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
rs-zero = { version = "0.1", features = ["cache-redis", "resil", "observability"] }
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
