# 语言工具链预设与迁移明细（Stack Profiles）

本文件是 `SKILL.md` 的按需参考（见「按需加载的参考文档」表）：在执行工具链选型、依赖缓存设计或 JS/TS 向 Bun 默认栈迁移时读取。

> 通用切换原则：每项工具选型都是**默认项**；若要保留旧工具，必须给出实测数据理由；切换任何工具前必须**新旧并行运行（side by side）**，列出新工具可能漏报的差异并确认信号无衰减后，再切换并删除旧工具。

---

## 1. JS / TS 默认栈：Bun + TypeScript 7

针对 JavaScript / TypeScript 仓库，以下配置为默认目标，除非实测证明旧工具更快或存在不可替代的校验能力：

- [ ] **包管理与脚本**：安装与脚本全部统一为 `bun`，优先锁定精确稳定版；若所需能力仅存在于 canary，**锁定精确 canary 版本**（pin exact canaries），避免漂移。
- [ ] **先确认命令存在**：下文 `bun lint` / `bun fmt` / `bun check` 等子命令迁移前须在锁定版本上逐条执行 `--help` 确认存在；不存在的项保留现有工具（ESLint / Prettier / tsc 等），不删除，不算违规。
- [ ] **依赖缓存**：在 CI 中显式缓存 `node_modules` 与 `~/.bun/install/cache`，由 `main` 分支预热，使 PR 缓存命中率 `> 95%`。
- [ ] **Lint（`< 1s`）**：使用 `bun lint`，直接读取仓库现有的 `.oxlintrc.json`，尽量开启全部有效规则。
- [ ] **Format（`< 1s`）**：使用 `bun fmt`。
- [ ] **Typecheck（`< 5s`）**：在 TypeScript 7 上运行 `bun check`。
- [ ] **Tests（`< 30s / shard`）**：使用 `bun test`，**保留现有 `vitest` imports** 以降低迁移成本；采用模块级隔离而非进程级隔离（isolate modules not processes）；每个测试文件结束后重置状态（reset state per file）；启用随机执行顺序（random order）以暴露隐式状态泄漏。
- [ ] **清理旧工具**：side-by-side 校验信号一致后，删除 `npm`、`yarn`、`pnpm`、`Prettier`、`ESLint`、`oxlint`、`Vitest`、`Jest` 及其孤立配置与 lockfile（删除 `oxlint` 依赖时**保留** `.oxlintrc.json`，供 `bun lint` 继续读取）。

---

## 2. 其他主流语言等价高速工具链对照表

非 JS/TS 仓库沿用同样的三桶审计、并行拓扑、分片均衡与硬门禁方法，但按对应语言的原生高速工具链与缓存边界落地（下列为常见参考，切换前同样需 side-by-side 比对校验覆盖面）：

| 维度 | Go | Rust | Python | Java / JVM |
| :--- | :--- | :--- | :--- | :--- |
| **Lint** | `golangci-lint`（开启缓存、`--new-from-rev` 仅用于 PR 增量提示，全量校验不降级） | `cargo clippy --locked` | `ruff check` | Checkstyle / SpotBugs / Error Prone（增量编译缓存） |
| **Format** | `gofmt` / `goimports` | `cargo fmt --check` | `ruff format --check` | Spotless / `google-java-format` |
| **Typecheck / Build** | `go vet` / `go build`（配合 build cache） | `cargo check` / `cargo nextest --no-run`（共享编译产物） | `ty` / `pyright` / `mypy --cache-dir` | Gradle configuration & build cache / Maven build cache |
| **Tests & Shard** | `go test -shuffle=on -p <cores>` + 按包/文件历史耗时分片 | `cargo nextest run --partition hash:i/n`（避免重复编译） | `uv run pytest -n <cores> --randomly-seed=last` + `pytest-split` | Gradle Test Distribution / JUnit 5 并行 + 按类历史耗时分片 |
| **依赖与构建缓存（Setup）** | `GOMODCACHE` + `GOCACHE`（`main` 预热） | `sccache` / `cargo-chef` + `target/` 增量缓存（`main` 预热） | `uv` 全局缓存 + `.venv`（`main` 预热） | `~/.gradle/caches` 或 `~/.m2/repository` + 远端 build cache |
| **包管理收敛** | 原生 `go.mod` | 原生 `Cargo.lock` | 收敛到 `uv`（替换 `pip` / `poetry` / `pipenv` 冷装） | 锁定 Gradle Wrapper / Maven Wrapper |

### 多语言仓库与 Monorepo 提示

- **多语言混编仓库**：按子目录/语言拆成独立并行 job，各自维护独立缓存键（按对应 lockfile 哈希），禁止让 A 语言的依赖安装阻塞 B 语言的 job。
- **编译型语言（Rust / Go / JVM / C++）的编译复用**：若测试分片需要同一份编译产物，须确保编译缓存（如 `sccache`、Gradle remote build cache、预构建测试二进制）高命中，避免 N 个 shard 重复编译 N 份代码。
