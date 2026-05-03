# {Service Name}

## Overview

说明服务职责、主要用户和依赖。

## Features

- REST/RPC endpoint。
- DB/cache 能力。
- resilience 能力。
- observability 能力。

## Cargo Features

```toml
rs-zero = { version = "0.1", features = ["rest", "resil", "observability"] }
```

按实际服务修改 feature，不要保留未使用能力。

## Configuration

```toml
# 示例配置，避免写入真实凭据
```

## Running

```bash
cargo run
```

## API / RPC

- REST：见 `API.md`。
- RPC：见 `RPC.md`。

## Testing

```bash
cargo fmt --all -- --check
cargo test
```

生产或跨模块修改：

```bash
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace
```

## Production Notes

- timeout：说明配置。
- breaker：说明覆盖的下游。
- limiter/shedder：说明策略。
- observability：说明 metrics/tracing/exporter。
- external dependencies：说明 Redis、DB、OTLP、Pyroscope 是否需要 opt-in 验证。
- profiling：如启用，标注 experimental / opt-in。
