# 跨 CI 平台适配与低样本降级策略（Platform Adapters）

本文件是 `SKILL.md` 的按需参考（见「按需加载的参考文档」表）：当目标仓库**未使用 GitHub Actions**，或仓库历史 CI 运行记录**不足 150 次**时读取。

---

## 1. 跨 CI 平台等价能力映射

三条铁律（合入前全量兜底 / 绝不掩盖失败 / 绝不减少校验）与三桶审计模型（`waiting` / `setup` / `running`）与平台无关；各平台落地原语对照如下：

| 核心能力 | GitHub Actions（默认） | GitLab CI | Buildkite | CircleCI |
| :--- | :--- | :--- | :--- | :--- |
| **合入前全量兜底（铁律 1）** | Merge Queue（仅组织仓库可用，触发全量套件）；个人账号仓库：分支保护必需检查 + 「Require branches to be up to date」，且 PR 必须跑全量、禁止筛选 | Merge Trains（`CI_PIPELINE_SOURCE == "merge_request_event"` 且 train 跑全量） | 主干合并队列 / 保护分支合入前全量 pipeline | 主干合并前全量 workflow + 分支保护必过项 |
| **取消被取代运行（Waiting）** | `concurrency.cancel-in-progress: true` | `workflow:auto_cancel` + job 级 `interruptible: true` | Pipeline 设置 `cancel_running_branch_builds: true` | Project 设置 Auto-cancel redundant workflows |
| **低排队温机池（Waiting）** | 已有自建 runner 或以省钱为目标：给自建常驻 runner 加实例；否则默认 Blacksmith runners / warm machines（备选 BuildJet / Namespace / Depot） | 自管常驻 GitLab Runner（autoscaler 预留 warm 槽位） | Elastic CI Stack（warm pool / 常驻 agent） | 预留资源类 / 自建常驻 runner |
| **`main` 预热 PR 缓存（Setup）** | `actions/cache`（分支缓存回落 `main` `restore-keys`） | `cache:fallback_keys` 回落 `main` 缓存键 | S3/GCS 缓存插件按 `main` 分支键兜底 | `restore_cache` 多前缀键回落 `main` |
| **Blobless Clone（Setup）** | `actions/checkout` 配 `filter: blob:none` | `GIT_CLONE_FLAGS: "--filter=blob:none"` | `BUILDKITE_GIT_CLONE_FLAGS="--filter=blob:none"` | 自定义 checkout 步骤执行 `git clone --filter=blob:none` |
| **拉取历史运行数据（审计）** | `gh api` / Actions Runs & Jobs API | GitLab Pipelines & Jobs REST/GraphQL API | Buildkite Builds & Jobs REST API | CircleCI Insights & Pipelines API |

### 三桶时间戳统一口径（跨平台一致）

不同平台字段名不同，审计时统一按以下定义拆桶，避免口径漂移：

- **`waiting`**：`job_started_at - job_created_at`（或 `enqueued_at` 到实际分配到 runner 开始执行的时间）。
- **`setup`**：从 job 启动开始，到「检出代码 + 工具链准备 + 缓存恢复（含基础镜像拉取 / BuildKit 层缓存恢复 / 跨 job 制品下载）+ 依赖安装」结束的总耗时。
- **`running`**：实际执行 lint / format / typecheck / build / test 及完整性校验步骤的总耗时。

---

## 2. 历史运行不足 150 次时的降级路径

新仓库、低频仓库或刚切换 CI 平台的仓库往往凑不满 150 次有效运行。按实际可用样本数 `N` 分三档处理，**不要因为凑不满 150 次就卡住审计**：

| 可用运行数 `N` | 审计与基线做法 | 允许推进的动作 | 明确禁止 |
| :--- | :--- | :--- | :--- |
| **`N ≥ 150`**（标准） | 取最近 150 次运行，计算三桶与端到端 p50 / p95，建立正式基线。 | 正常按最大桶排序逐项推进与回归对比。 | — |
| **`30 ≤ N < 150`**（临时基线） | 取全部可用运行计算三桶 p50 / p95，在报告中显式标注 `N=<数量>` 与 p95 置信度偏低。 | 按最大桶排序推进优化；PR 前后对比附实际样本量。满 150 次后升级为正式基线。 | 隐瞒样本不足，把小样本 p95 当长期稳态基线。 |
| **`N < 30`**（冷启动降级） | 1. 先做**静态 CI 配置审计**（查串行 `needs`、retry/宽超时/跳过测试、冷装无缓存、全量克隆、未开 cancel-in-progress）。<br>2. 在优化前后各采集 **5–10 次真实运行**（含冷/热缓存各至少 1 次）记录中位数与极值作为对照。 | 优先落地静态可确定收益的高确定性改动（删 retry、串行改并行、开并发取消、blobless clone、缓存与硬门禁）；积累够 30 / 150 次后补建分位基线。 | 凭空估算或编造 p50/p95 数据；未经实测就做细粒度 shard 装箱。 |
