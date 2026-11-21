# CI Speed Audit 预算细则、迁移清单与交付模板

本文件是 `SKILL.md` 的按需参考（见「按需加载的参考文档」表）：在执行 Procedure 第 3、4 步（逐桶治理、工具链迁移、测试 shard 分片/筛选及最终汇报交付）时查阅，不默认加载。

## 1. 三条最高铁律（不可妥协）

1. **合入前全量兜底**：合入主干（`main`）前必须在 merge queue（或平台等价主干合并门禁，见 `platform-adapters.md`）运行完整测试套件；PR 测试筛选仅缩小 PR 反馈范围，绝不替代合入前全量套件。
2. **绝不掩盖失败**：禁止引入重试（retries）、放宽超时（wider timeouts）或跳过测试（skipped tests）。
3. **绝不减少校验**：不删减必要 lint/typecheck/build/test 覆盖面以换取耗时下降。

## 2. 全链路预算目标与治理清单

> 原则：先审计最近 **150 次** CI 运行（不足 150 次按 `platform-adapters.md` 降级采样），按 `waiting` / `setup` / `running` 拆出 p50 与 p95，**从最大桶开始优化**；硬门禁仅限单次 CI job 执行预算（lint / fmt / check / 单 shard / 最慢 job），**越线即直接判红 FAIL**；`waiting` / `setup` / 缓存命中率为分位诊断目标，本地钩子为本地性能目标，均不做单次判红。

### 预算重定规则（大仓库 / 编译型语言出口）

默认硬门禁预算（Lint / Format `< 1s`、Typecheck / Build `< 5s`、单 shard `< 30s`、最慢 job `< 35s`）以中等规模 TS/JS 仓库为基准。若仓库规模（大型 monorepo）或编译型语言（Rust / C++ / JVM 等）使某项默认硬门禁**物理上不可达**，仅允许按以下三步重定该单项硬门禁预算：

1. **先优化再重定**：必须先完成缓存预热、工具链迁移、并行拓扑与加核/分片；**禁止**对着未优化的慢基线直接放宽预算。
2. **附物理下界证据**：记录该步在热缓存与冷缓存下的实测下界（p50 / p95）及具体物理瓶颈（如全仓链接或不可拆分的类型图大小）。
3. **固化为新硬门禁**：以 `优化后热基线 p95 × 1.2`（向上取整）作为该仓库的新硬门禁写入 CI 预算脚本与回归基线快照；重定后单次超线仍**必须直接判红 FAIL**，且端到端目标优先仍维持 p50 `< 1m` / p95 `< 2m`（靠并行与分片消化）。注：`waiting < 5s` / `setup < 15s` 为分位诊断目标，按分位基线诊断，**不转为单次判红硬门禁**。

### A. Waiting（分位诊断目标 `< 5s`）
- [ ] 评估并迁移至低排队温机池：**默认 Blacksmith runners / warm machines**；等价备选包括 BuildJet、Namespace、Depot 或自建常驻温机池（跨平台方案见 `platform-adapters.md`）。
- [ ] 配置并发策略，自动取消同一 PR/分支上被新提交取代的旧运行（cancel superseded runs）。

### B. Setup（分位诊断目标 `< 15s`）
- [ ] 将基础工具链预构建进镜像/环境（baked toolchain），避免每次运行重复下载安装。
- [ ] 用 `main` 分支预热缓存，使 PR 缓存命中率达到 **`> 95%`**。
- [ ] 显式缓存依赖目录（JS/TS 为 `node_modules`，其他语言见 `stack-profiles.md`）。
- [ ] 仓库检出**默认采用 blobless clone**（`filter: blob:none`）；超大 monorepo 若工作树检出本身占主导，补充 `sparse-checkout` 或浅克隆（`fetch-depth: 1`，但若启用 PR 筛选须确保能获取 merge-base diff）。

### C. Job 拓扑与并行度（最慢 job `< 35s`）
- [ ] `lint`、`typecheck`、`build`、`tests` 全部并行启动。
- [ ] 消除任何必需 job 对另一个 job 的等待（no required job waits on another）。
- [ ] 将多个轻量级小校验合并到一个 job（fold small checks into one job），减少额外 runner 启动与 setup 开销。

### D. 工具链收敛与迁移（默认按语言栈选型）
> 所有工具均为默认选型：若保留旧工具，必须提供**实测理由**；切换前必须**新旧并行运行（side by side）**校验结果一致。JS/TS 默认 Bun + TypeScript 7 完整迁移清单，以及 Go / Rust / Python / JVM 等价工具链对照详见 [stack-profiles.md](stack-profiles.md)。

- [ ] 包管理与脚本收敛到目标语言高速默认工具并锁定精确版本（JS/TS 为 `bun` exact canary）。
- [ ] Lint（`< 1s`）、Format（`< 1s`）、Typecheck / Build（`< 5s`）、Tests（`< 30s / shard`，模块级/文件级隔离、状态重置、随机顺序）逐项达到硬门禁。
- [ ] Side-by-side 校验通过后，清理并删除被替代的旧工具与冗余配置（JS/TS 删除 `oxlint` 依赖时保留 `.oxlintrc.json`）。

### E. 测试并行与 shard 分片（单 shard `< 30s`）
- [ ] **先加核再做 shard 分片**（buy cores before shards），避免过早分片放大 setup 开销。
- [ ] 将测试 worker 数量与机器核心数精确匹配（match workers to cores）。
- [ ] 为每个 shard 分配独立的一次性数据库（throwaway DB per shard），消除跨 shard 状态竞争：
  - 轻量场景：tmpfs SQLite（如 `/mnt/testdb/test-<shard>.db`）。
  - 服务端数据库：每个 shard 独立 database / schema（如 `CREATE DATABASE ... TEMPLATE` 基于预迁移模板库秒级克隆），或每 shard 独立 ephemeral service container / testcontainers。
- [ ] 找出并拆分运行时间 **`> 20s`** 的单个慢测试文件。
- [ ] 基于历史录制的单文件耗时做 shard 分片均衡（balance shards from recorded file times）。
- [ ] 增加完整性校验步骤，**证明所有测试文件都实际被执行**（prove all files ran，防止 shard 分片脚本漏跑文件）。

### F. PR 可达测试筛选（PR Selection）
- [ ] 基于可信依赖图仅运行本次改动可达的测试（only reachable tests）；共享库/基础包改动必须沿**反向依赖传递**覆盖所有下游测试。
- [ ] 按仓库形态选择依赖图来源（禁止手写拍脑袋的路径正则映射）：

| 仓库形态 | 推荐可达图方案 |
| :--- | :--- |
| **Monorepo（JS/TS / 多语言）** | Turborepo（`--filter=...[origin/main]`）/ Nx（`nx affected`）/ Bazel（`bazel query 'rdeps(...)'`） |
| **单仓 JS / TS** | 静态 import 依赖图 / 基于 AST 的文件级反向依赖解析（若经实测保留 `Vitest`，可追加 `vitest --changed`） |
| **Go** | `go list -deps` 构建包级反向依赖图，仅跑受影响包及下游包测试 |
| **Python** | `pytest-testmon` 或静态 AST import 依赖图 |
| **Rust** | `cargo metadata` workspace crate 依赖图（配合 `cargo nextest` 跑受影响 crate 及下游） |
| **Java / JVM** | Gradle / Maven 模块依赖图（受影响模块及下游模块），或静态 class / import 依赖图 |

- [ ] **降级底线**：若仓库存在大量动态反射/隐式字符串路由导致无法建立可信依赖图，**回退为 PR 跑全量测试**，改靠加核 + 均衡 shard 达标，绝不用不可靠筛选换速度。
- [ ] 显式维护「触发全量重跑」的全局输入清单（rerun-everything inputs：CI workflow 配置、依赖锁文件、全局测试 harness、编译/运行时配置、数据库 migration、公共环境变量/feature-flag schema 等）。
- [ ] **诚实性红灯校验**：故意隐藏/制造一处真实破坏性改动，校验筛选逻辑确实命中并让 CI 判红 FAIL（verify by hiding a real change and confirming red）。

### G. 测试卫生治理（两周观测 / 历史回溯）
- [ ] **优先回溯近 14 天已有 CI 失败记录**进行归类（caught regression vs change detector）；若仓库缺少可回溯的历史失败日志，再开启连续 **2 周** 记录（log test failures for two weeks）。严禁在缺乏失败证据时凭直觉直接删测试。
- [ ] 识别并删除只充当“改动探测器”的脆弱测试（change-detector-only tests）。
- [ ] 将这类反模式明确写入仓库 `AGENTS.md` 禁止列表（ban patterns in `AGENTS.md`）。
- [ ] **绝对保护名单（严禁删除）**：`auth`（鉴权认证）、`privacy`（隐私保护）、`migration`（数据迁移）、`concurrency`（并发正确性）相关测试。

### H. 本地开发钩子（本地性能目标 `< 9s`，非 CI 硬门禁）
- [ ] 通过 Git hook 强制执行本地校验（而非只把要求写进 `AGENTS.md`），总耗时控制在 `< 9s`：**默认使用 `hk`**；若仓库已装 `lefthook` / `husky` / `pre-commit` 且能稳定 `< 9s` 并覆盖以下两个钩子，可直接沿用：
  - `pre-commit`：对暂存（staged）文件执行格式化 + lint（fmt + lint staged）。
  - `pre-push`：执行与 CI 对齐的校验（same check as CI）。

### I. 容器构建、制品传递与 Matrix 矩阵（如适用）
- [ ] **Docker / 容器构建**：启用 BuildKit 与远端/registry 层缓存（如 registry cache / GHA cache / Depot），把变动频率低的依赖安装层前置，并用 `.dockerignore` 裁剪无关上下文；PR 阶段若无需推送镜像，跳过 push 或下沉到主干发布流水线。
- [ ] **跨 Job 制品传递**：减少或消除大体积跨 job artifact 上传/下载（优先复用缓存或在同 job 内完成，避免压缩与网络往返拖慢关键路径）。
- [ ] **多 OS / 多版本 Matrix**：PR 阶段收敛到「主生产目标 + 最小差异维度」矩阵以控时，**合入主干前在 merge queue（或等价主干门禁）跑全矩阵**，严格遵守铁律 1，不削减发布前覆盖面。

---

## 3. 小 PR 交付与回退模板

每个独立改动提交一个小 PR，说明中固定包含以下字段：

```markdown
### 改动目标与命中耗时桶
- 针对桶：`waiting` / `setup` / `running`
- 改动摘要：<一句话说明改动>
- 是否涉及工具切换：是（已附 side-by-side 对比结果）/ 否

### 前后实测数据
| 指标 | 优化前 | 优化后 | 参考值 | 状态 |
| :--- | :---: | :---: | :---: | :---: |
| 本步/本桶 p50 | - | - | - | 数值记录 |
| 本步/本桶 p95 | - | - | - | 数值记录 |
| 缓存命中率（如适用） | - | - | >95% | 达标 / 未达标（分位诊断目标，不单独判红） |
| 若改动受硬门禁约束的 job，单列其单次执行时长 | - | - | 见 SKILL.md 校验清单 | PASS / FAIL（硬门禁） |

### 诚实性与回退确认
- [ ] 无新增重试、无放宽超时、无跳过测试、无减少校验
- [ ] Merge queue（或等价主干合并门禁）全量套件未受削弱
- 回退方式：<明确列出 revert 该 PR 或回退旧配置的步骤>
```

---

## 4. 需用户介入项与最终总账模板

先完成所有不依赖外部审批的改动，再向用户交付以下两张汇总表：

### 需用户审批/配置的外部项（账号 / Secret / 预算）

| 事项 | 计费模式 | 所需权限/资源 | 预期收益（对应耗时桶） | 预估月度成本 |
| :--- | :---: | :--- | :--- | :---: |
| 例：Blacksmith runners / warm machines | 按分钟 / 固定包机 / 混合 | 账号、组织授权、付费预算 | `waiting` 降至 `<5s` | `$X / 月` |

### 优化前后总耗时与总成本对比

> 成本口径说明：按分钟计费（hosted / 托管加速 runner）直接按实测分钟数汇总；**固定月费包机**（自建 runner / 常驻节点池）同时列出固定月费、峰值占用率与按月有效运行次数折算的**单次 PR 均摊成本**，避免包机场景掩盖容量释放与延期扩容收益。

| 维度 | 优化前（150 次基线） | 优化后 | 目标/变化 |
| :--- | :---: | :---: | :---: |
| PR 端到端 p50 | - | - | `< 1m`（端到端 PASS 标准） |
| PR 端到端 p95 | - | - | `< 2m`（端到端 PASS 标准） |
| Waiting p50 与 p95 | - | - | `< 5s`（p50 及低占用时段 p95，分位诊断目标） |
| Setup p50 与 p95 | - | - | `< 15s`（p50 及缓存命中样本 p95，命中率 `> 95%`，分位诊断目标） |
| Running（最慢 job）p50 与 p95 | - | - | 最慢 job `< 35s`（或重定硬门禁） |
| 计费模式（按分钟 / 固定包机 / 混合） | - | - | 口径一致 |
| 月度 CI 总成本 | - | - | 前后差额 |
| 单次 PR 均摊成本（总成本 ÷ 月有效 PR 运行数） | - | - | 前后差额 |
