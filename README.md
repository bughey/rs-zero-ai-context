# rs-zero AI Context

把常见 AI 编程助手变成 rs-zero 开发助手：先写规格，优先使用 `rzcli` 生成，按 Cargo feature 接入能力，最后完成 Rust 验证闭环。

## 一键提示

在支持文件操作的 AI 助手中输入：

```text
Set up rs-zero AI tools for this project from https://github.com/bughey/rs-zero-ai-context
```

AI 应完成：

1. 识别当前工具：Codex、Claude Code、Cursor、Copilot 或 Windsurf。
2. 安装对应规则文件。
3. 在需要时读取 `references/` 下的 rs-zero 模式文档。
4. 使用 `rzcli` 和 cargo 命令完成生成与验证。

## 安装内容

```text
AI Assistant
├─ Workflow Layer
│  ├─ AGENTS.md
│  ├─ 00-instructions.md
│  ├─ workflows.md
│  ├─ tools.md
│  └─ patterns.md
└─ Knowledge Layer
   ├─ references/
   │  ├─ rzcli-commands.md
   │  ├─ rest-patterns.md
   │  ├─ rpc-patterns.md
   │  ├─ model-cache-patterns.md
   │  ├─ resilience-patterns.md
   │  ├─ observability-patterns.md
   │  ├─ production-checklist.md
   │  └─ troubleshooting.md
   └─ templates/
      ├─ README-service.md
      ├─ API.md
      ├─ API-MODEL.md
      └─ RPC.md
```

工作流层回答“先做什么、用什么命令、怎么验证”。知识层回答“为什么这样做、具体模式是什么、生产边界在哪里”。

## 手动安装

### Codex

Codex 自动读取项目根目录的 `AGENTS.md`。如果目标项目已有 `AGENTS.md`，合并本仓库的 rs-zero 规则，不要直接覆盖已有规则。

```bash
git submodule add https://github.com/bughey/rs-zero-ai-context.git .codex/rs-zero-ai-context
cp .codex/rs-zero-ai-context/AGENTS.md AGENTS.md
```

### Claude Code

```bash
git submodule add https://github.com/bughey/rs-zero-ai-context.git .claude/rs-zero-ai-context
```

在项目级 Claude 指令中加入：优先读取 `.claude/rs-zero-ai-context/00-instructions.md`，需要详细模式时读取 `.claude/rs-zero-ai-context/references/`。

### GitHub Copilot

```bash
git submodule add https://github.com/bughey/rs-zero-ai-context.git .github/rs-zero-ai-context
ln -s rs-zero-ai-context/00-instructions.md .github/copilot-instructions.md
```

如果 `.github/copilot-instructions.md` 已存在，合并内容，不要覆盖。

### Cursor

```bash
git submodule add https://github.com/bughey/rs-zero-ai-context.git .cursor/rs-zero-ai-context
```

将 `00-instructions.md`、`workflows.md`、`tools.md`、`patterns.md` 添加到 Cursor rules。

### Windsurf

```bash
git submodule add https://github.com/bughey/rs-zero-ai-context.git .windsurf/rs-zero-ai-context
```

将 `00-instructions.md` 作为全局规则，将 `references/` 作为按需参考。

## 使用方式

### 新建 REST 服务

```text
Create a user REST service with rs-zero. Start from an .api spec, use rzcli, and run the required cargo checks.
```

### 新建 RPC 服务

```text
Create a greeter RPC service with rs-zero. Start from a proto file, use rzcli rpc gen, include resilience defaults, and document grpcurl usage.
```

### 接入缓存和韧性

```text
Add Redis-backed cache and token limiter rescue to this rs-zero service. Be feature-aware and keep external Redis tests opt-in.
```

### 生产前检查

```text
Review this rs-zero service for production readiness: features, timeout, breaker, limiter, observability, health checks, and external validation boundaries.
```

## 核心规则

- **spec-first**：REST 先写 `.api`，RPC 先写 `.proto`，model/cache 先写 SQL schema。
- **rzcli-first**：优先用 `rzcli` 生成骨架，不手写大段生成代码。
- **feature-aware**：需要 Redis、RPC、韧性、可观测性、OTLP、DB 时显式声明 Cargo feature。
- **production-first**：默认考虑 timeout、breaker、limiter/shedder、metrics/tracing、health/readiness 和外部依赖边界。
- **verification-first**：完成后运行匹配范围的 `cargo fmt`、`cargo check`、`cargo test`、`cargo clippy`。

## 常用命令

```bash
rzcli -v
rzcli new-rest hello
rzcli api validate -f hello.api
rzcli api format -f hello.api -o hello.formatted.api
rzcli api gen -f hello.api -d target/generated
rzcli api openapi -f hello.api -o openapi.json
rzcli rpc gen -p proto/hello.proto -d target/generated
rzcli model gen -s schema.sql -d target/generated --with-sqlx --with-redis-cache
cargo fmt --all -- --check
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace
```

## 更新

```bash
git submodule update --remote --recursive
```

更新后让 AI 重新读取 `00-instructions.md` 和相关 `references/` 文件。

## 非目标

首版只提供规则、工作流和参考文档，不提供 MCP server、Web UI 或 `rzcli ai init` 命令。

## 相关项目

- rs-zero：Rust-first 的 go-zero 风格微服务框架。
- rs-zero-cli：提供 `rzcli` 生成与兼容命令。
