---
name: ci-speed-audit
description: 在不掩盖失败或减少校验的前提下审计并系统提速仓库 CI（目标 PR p50 <1m、p95 <2m），覆盖跨平台三桶耗时审计、runner/缓存/并行拓扑/shard 分片调优、PR 可达测试筛选，以及多语言高速工具链与 JS/TS Bun 默认栈迁移。
version: "0.4.1"
---

# CI 提速与诚实加速（CI Speed Audit）

本技能指导如何在**不让 CI 说谎**的前提下，把代码仓库的 CI 压到 PR 中位数（p50）< 1 分钟、p95 < 2 分钟。核心路径是先审计最近 150 次运行，按最大耗时桶逐项治理，用小颗粒 PR 持续交付前后对比数据与成本账本。SKILL.md 只保留主流程与判定门；深入诊断、shard 分片与交付模板按下方索引按需读取。

本技能**不做**以下事项：不通过重试、放宽超时、跳过测试或删减校验项换取表面变快；也不替代无在线 CI 时的本地纯手工校验流程。

## When to Use

- 「把这个仓库的 CI 提速到 1 分钟以内」
- 「优化 CI（GitHub Actions / GitLab CI / Buildkite / CircleCI 等）耗时，先查清楚慢在排队、setup 还是跑测试」
- 「按诚实 CI 预算给仓库做 runner、缓存、并行拓扑和测试 shard 分片优化」
- 「把 JS/TS 仓库的 lint / format / typecheck / test 统一迁到 Bun 和 TypeScript 7，或给 Go / Rust / Python / JVM 仓库迁移等价高速工具链，并加上 job 超时硬门禁」
- 「给 PR 做可达测试筛选，同时保证合入主干前（merge queue / merge trains）跑全量测试」
- 「CI 最近又变慢了，先对比回归基线查是哪个桶回归、shard 分片是否失衡」

## Prerequisites

- 目标仓库的 CI 历史运行数据读取权限（优先拉取最近 150 次运行，拆分 `waiting` / `setup` / `running`；若历史运行不足 150 次，按 [references/platform-adapters.md](references/platform-adapters.md) 的降级路径处理）。
- 仓库内的 CI workflow / pipeline 配置、包管理与脚本定义、测试及 linter/formatter/typechecker 配置文件。
- **先做完无需用户介入的全部改动**：任何涉及外部账号、Secret、付费 runner 或预算的项（如 Blacksmith runners、warm machines）不得阻塞前期优化；先交付仓库内可落地的改动，再把需人工批准的外部项连同预估月度成本一并列出。
- 切换工具链前确认当前环境可安装目标工具版本（如 JS/TS 栈目标 Bun exact canary 版本），并能与旧工具并行比对。

## Procedure

### 1. 锁定不可违背的三条铁律与总目标

先在方案和 PR 说明中明确以下硬边界（均高于速度本身）：

1. **合入前全量兜底**：任何改动进入主干（`main`）前必须在 Merge Queue（或等价主干合并门禁，如 GitLab Merge Trains，见 `references/platform-adapters.md`）中跑完全量测试套件；PR 级别的测试筛选只用于缩短 PR 反馈时间。
2. **绝不掩盖失败**：禁止添加重试（retries）、放宽超时（wider timeouts）或跳过测试（skipped tests）。
3. **绝不减少校验**：不能通过关掉规则或删掉必要校验来降耗时。
4. **速度目标**：PR 获取结果 **p50 < 1 分钟**，**p95 < 2 分钟**；任何 job 超预算必须直接失败（fail any job exceeding budget）。

### 2. 审计最近 150 次 CI 运行，先治最大耗时桶

1. 拉取仓库最近 **150 次** CI 运行记录，将每次耗时拆为三个桶：`waiting`（排队等待）、`setup`（克隆/装依赖/准备环境）、`running`（实际执行校验与测试）。若未使用 GitHub Actions 或历史运行不足 150 次（`30 ≤ N < 150` 建临时基线；`N < 30` 走静态配置审计 + 5–10 次基准采样），按 [references/platform-adapters.md](references/platform-adapters.md) 执行。
2. 分别计算每个桶以及端到端总耗时的 **p50** 与 **p95**，记录为审计基线（同时记录当前 CI 总耗时与费用基线）。
3. 按占比从大到小排序，**优先治理最大耗时桶**，不要凭直觉跳过瓶颈直接调微小步骤；`waiting` / `setup` 瓶颈与子段定位方法见下方参考文档表。

### 3. 按判定口径逐桶优化

每项工具选型都是**默认项**：若要保留旧工具，必须给出实测数据理由；切换任何工具前，必须**新旧并行运行（side by side）**确认信号一致后再切换删除旧工具。JS/TS 仓库默认按 Bun + TypeScript 7 栈落地，其他语言按等价高速工具链对照表执行（详见 [references/stack-profiles.md](references/stack-profiles.md)）；逐条治理清单见 [references/budgets-and-checklist.md](references/budgets-and-checklist.md)。

| 维度 | 指标与判定口径 | 核心手段（通用方法 + 默认栈） |
| :--- | :--- | :--- |
| PR 端到端 | p50 `< 1m`、p95 `< 2m`（唯一端到端 PASS 标准） | 全链路各桶达标后的综合结果，非独立硬门禁 |
| Waiting | `< 5s`（分位诊断目标） | 低排队温机池（默认 Blacksmith runners / warm machines，备选见 `budgets-and-checklist.md`）、取消被新提交取代的运行（cancel superseded runs，跨平台写法见 `platform-adapters.md`） |
| Setup | `< 15s`（分位诊断目标） | 预构建工具链（baked toolchain）、由 `main` 预热的 PR 缓存（缓存命中率 `> 95%`）、缓存依赖目录（JS/TS 为 `node_modules`）、blobless clone（超大 monorepo 补 `sparse-checkout`） |
| Job 拓扑 | 最慢 job `< 35s`（硬门禁） | `lint` / `typecheck` / `build` / `tests` 全部并行；无任何必需 job 等待另一个 job；把零散小校验合并到一个 job |
| Lint | `< 1s`（硬门禁） | 高速 linter（JS/TS 默认 `bun lint` 读 `.oxlintrc.json`；其他语言见 `stack-profiles.md`） |
| Format | `< 1s`（硬门禁） | 高速 formatter（JS/TS 默认 `bun fmt`；其他语言见 `stack-profiles.md`） |
| Typecheck / Build | `< 5s`（硬门禁） | 高速类型检查/增量编译（JS/TS 默认 TypeScript 7 上 `bun check`；其他语言见 `stack-profiles.md`） |
| Tests | `< 30s / shard`（硬门禁） | 高速测试执行器（JS/TS 默认 `bun test`，保留 `vitest` imports，模块级隔离、单文件重置状态、随机顺序；其他语言见 `stack-profiles.md`） |
| Local Hooks | `< 9s`（本地性能目标） | 本地 Git 钩子强制执行（默认 `hk`，已有达标 `lefthook`/`husky`/`pre-commit` 可沿用）：`pre-commit` 对暂存文件跑 fmt + lint；`pre-push` 跑与 CI 一致的校验 |

> **预算重定规则**：上表数字为默认目标（中等规模 TS/JS 参考）。对大型 monorepo 或编译型语言物理上不可达的单项，仅在完成缓存/并行/加核优化后，凭实测冷热下界与瓶颈说明按 `budgets-and-checklist.md` 重定硬门禁并固化到基线；**禁止**对未优化基线放宽阈值，重定后超线仍必须判红 FAIL。

按以下顺序推进具体落地：

1. **收敛依赖与工具链**：按目标语言选定默认高速工具链并锁定精确版本（JS/TS 默认收敛到 `bun` exact canary，side-by-side 校验通过后删除 `npm` / `yarn` / `pnpm` / `Prettier` / `ESLint` / `oxlint` / `Vitest` / `Jest`，但保留 `.oxlintrc.json`；完整迁移细则与 Go / Rust / Python / JVM 对照见 [references/stack-profiles.md](references/stack-profiles.md)）。
2. **并行与 shard 分片扩展测试**：
   - 先买更多 CPU 核心再做 shard 分片（buy cores before shards），worker 数与核心数匹配。
   - 每个 shard 使用独立一次性数据库（throwaway DB per shard）。
   - 拆分耗时 `> 20s` 的单个测试文件。
   - 基于历史记录的单文件耗时做 shard 分片负载均衡（balance shards from recorded file times，均衡做法与判据见参考文档表）。
   - 增加完整性校验，**证明所有测试文件都实际执行过**（prove all files ran）。
3. **PR 可达测试筛选（PR selection）**：
   - 仅运行本次改动可达的测试（按仓库形态选用可信依赖图并沿**反向依赖传递**覆盖下游，禁止拍脑袋路径正则，无法建可信图时回退跑全量 + shard 控时，选型表见 `budgets-and-checklist.md`）；显式列出触发「全量重跑」的全局输入清单（rerun-everything inputs）。
   - 保持 merge queue（或等价主干门禁）始终执行全量套件。
4. **测试治理（两周观测 / 历史回溯）**：
   - 优先回溯近 14 天已有 CI 失败记录；若无历史记录，再连续两周记录测试失败日志（log test failures for two weeks），严禁无证据凭直觉删测试。
   - 删除只充当“改动探测器”的脆弱测试（change-detector-only tests），并在仓库 `AGENTS.md` 中明令禁止此类写法。
   - **绝对禁止删除**认证/鉴权（auth）、隐私（privacy）、数据迁移（migration）与并发（concurrency）测试。
5. **本地 Git 钩子**：默认 `hk`（已有且 `< 9s` 的 `lefthook` / `husky` / `pre-commit` 可沿用）；`pre-commit` 对暂存文件跑 fmt + lint，`pre-push` 跑与 CI 一致的校验，总耗时 `< 9s`（通过 git hook 强制执行，而非只写进 `AGENTS.md`）。
6. **硬门禁**：为各 job 写入单次执行预算阈值（默认阈值或经实测重定后的仓库阈值），job 超预算即硬门禁 FAIL（fail any job exceeding budget），防止性能回归；同时用最近 150 次运行固化回归基线（方法见 `references/jobs-shards-baseline.md` 第 3 节）。

### 4. 小 PR 渐进交付、先做自助项、给回退与成本账本

1. **一改一小 PR**：每个独立优化项单独提交一个小 PR，附带该改动的前后实测数据（p50 与 p95、缓存命中率或单步耗时）、PASS / FAIL 校验结果与回退方式。
2. **先做完无需用户配合的全部项**：仓库内 workflow、并行拓扑、缓存键、工具链迁移、shard 分片与硬门禁先行落地；最后汇总需要账号、Secret 或预算审批的外部清单，并逐项标明预估月度成本。
3. **收尾总账**：在全部优化完成后，输出**优化前 vs 优化后**的 CI 总耗时（p50 与 p95 及三桶拆分）与总成本对比表（含计费模式与单次 PR 均摊成本）。

## p50/p95 判定快速校验清单

判定分两层：**端到端 PASS**用最近 150 次运行看分位（样本不足时按 `platform-adapters.md` 标注临时基线）；**硬门禁**约束单次 job 执行（fail any job exceeding budget，job 超预算即判红 FAIL）。下列单项硬门禁均支持按预算重定规则替换为经实测固化的仓库阈值：

- [ ] PR 端到端：p50 `< 1m` 且 p95 `< 2m`（唯一端到端 PASS 标准）
- [ ] Lint（JS/TS 默认 `bun lint`）：单次执行 `< 1s`（硬门禁）
- [ ] Format（JS/TS 默认 `bun fmt`）：单次执行 `< 1s`（硬门禁）
- [ ] Typecheck / Build（JS/TS 默认 `bun check`）：单次执行 `< 5s`（硬门禁）
- [ ] Tests（JS/TS 默认 `bun test`）：每个 shard `< 30s`（硬门禁）
- [ ] 本地 Git 钩子（默认 `hk`，备选 `lefthook` / `husky` / `pre-commit`）：单次运行 `< 9s`（本地性能目标，非 CI 硬门禁）
- [ ] 最慢 job `< 35s`（或经实测重定并固化至基线的硬门禁），且无任何必需 job 等待另一个 job
- [ ] `waiting`：p50 `< 5s`，低占用时段（runner 池 `< 90%`）p95 `< 5s`（分位诊断目标）
- [ ] `setup`：p50 `< 15s` 且 PR 缓存命中率 `> 95%`；p95 按缓存命中样本 `< 15s` 与冷缓存基线分开看（分位诊断目标）
- [ ] 硬门禁生效：任一 job 单次超预算即判红 FAIL，未通过重试或放宽超时蒙混

> 注意：分位诊断目标未达标（如 `setup` p95 落在允许 `≤ 5%` 的缓存未命中（冷缓存）样本上）按 `references/bucket-diagnostics.md` 作分位诊断预警并继续诊断归因，**不是**硬门禁 FAIL；不要因此误回退已通过硬门禁的小 PR。

## 按需加载的参考文档

以下文档只在执行到对应步骤或命中对应症状时读取，不要默认全部展开：

| 文件 | 何时读取 |
| :--- | :--- |
| [references/platform-adapters.md](references/platform-adapters.md) | 仓库使用非 GitHub Actions 平台（GitLab CI / Buildkite / CircleCI 等）或历史运行不足 150 次时：跨平台原语映射、统一拆桶口径与低样本降级策略 |
| [references/stack-profiles.md](references/stack-profiles.md) | 执行工具链迁移或非 JS/TS 仓库治理时：JS/TS Bun + TS7 默认栈完整迁移明细，以及 Go / Rust / Python / JVM 等价高速工具链与缓存对照表 |
| [references/budgets-and-checklist.md](references/budgets-and-checklist.md) | 执行 Procedure 第 3、4 步时：逐桶可勾选清单、预算重定规则、供应商/工具备选、PR 筛选依赖图、容器/制品/Matrix 治理、小 PR 交付与多计费模式成本总账 |
| [references/bucket-diagnostics.md](references/bucket-diagnostics.md) | `waiting` / `setup` 分位诊断目标未达标或需做传导归因时：两桶瓶颈定位判据、waiting→setup 量化方法、缓存虚高与并行饱和等误判定位 |
| [references/jobs-shards-baseline.md](references/jobs-shards-baseline.md) | 最慢 job / shard 硬门禁越线或收尾防回归时：过大 job 识别拆分判据、shard 历史耗时负载均衡做法、回归基线建立与预警判据 |

## Pitfalls

- **把“关校验/加重试”当提速**：增加重试、拉长超时、跳过不稳定测试或删减校验会掩盖真实缺陷，直接违反铁律，属于无效优化。
- **未做 side-by-side 比对就切工具**：直接替换 linter/typechecker/test runner 可能导致规则漏报或语义偏移；必须新旧并跑核对差异后再删旧工具，且若保留旧工具需给出实测依据。
- **先做 shard 分片后加核**：单机核数不足时盲目拆大量 shard 会成倍放大 `setup` 成本；应先加核并让 worker 数匹配核数，再按录制文件耗时均衡 shard 分片（方法见 `references/jobs-shards-baseline.md`）。
- **shard 分片漏跑与筛选漏测**：未校验「所有文件已运行」可能静默漏掉新增测试；PR 测试筛选若漏列全局配置/依赖等 rerun-everything 输入，会让破坏性改动以 PASS 放行（漏测假绿）。
- **误删高价值测试**：清理 change-detector 测试时若波及 auth / privacy / migration / concurrency，会引入高危回归风险。
- **卡在外部审批不动**：在拿到付费温机池（如 Blacksmith）账号或新预算前停滞不前会拖延收益；应先交付所有不依赖外部审批的提速 PR。

## Verification

1. **诚实性红灯校验（PR 测试筛选）**：故意隐藏/注入一个真实的破坏性代码改动，确认 PR 筛选能命中相关测试并让 CI 判红 FAIL（confirm red）。
2. **工具链硬门禁与本地 Git 钩子性能校验**：在仓库内执行对应语言的高速工具链（JS/TS 默认 `bun lint` `< 1s`、`bun fmt` `< 1s`、`bun check` `< 5s`、`bun test` `< 30s / shard`，或经实测重定的仓库预算；其他语言按 `stack-profiles.md`）及本地 Git 钩子（`< 9s`，本地性能目标、非硬门禁），确认与旧工具 side-by-side 校验信号无衰减。
3. **完整性与硬门禁校验**：确认 merge queue（或平台等价主干合并门禁）触发全量套件、shard 分片运行校验能证明 100% 测试文件已执行、job 超预算会触发硬门禁 FAIL（fail any job exceeding budget）。
4. **最终数据校验**：对比审计基线与优化后的 waiting/setup/running p50 与 p95、PR 端到端 p50（`< 1m`）/ p95（`< 2m`）以及月度 CI 成本账本。
