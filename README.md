# ci-speed-audit

> 在不掩盖失败、不减少校验的前提下，把仓库 PR CI 提速到 **p50 < 1 分钟、p95 < 2 分钟**。

这是一个 **Cindy / Claude Code / Codex 兼容的 Agent Skill**（`SKILL.md` + 按需加载的 `references/` 参考文档）。它不是命令行工具，而是一份给 AI 编程助手执行的 CI 提速作业指导书：先审计，再按耗时桶逐项治理，全程禁止通过加 retry、放宽超时、跳过测试或删减校验来"变快"。

---

## 这个 Skill 是干什么的

对任意 Git 仓库的 CI 做一次系统化提速改造，核心流程：

1. **审计**：拉取最近 **150 次** CI 运行，把每次耗时拆成 `waiting`（排队）/ `setup`（克隆+装依赖）/ `running`（实际校验）三个桶，分别算 p50 和 p95，定位最大的耗时桶先治理。
2. **逐桶优化**：按默认预算治理 waiting `<5s`、setup `<15s`、最慢 job `<35s`、Lint `<1s`、Format `<1s`、Typecheck/Build `<5s`、Tests `<30s/shard`、本地钩子 `<9s`。
3. **诚实性保障**：merge queue（或等价主干门禁）合入前必须跑全量套件；禁止重试、放宽超时、跳过测试；shard 分片必须证明 100% 测试文件已执行；PR 可达测试筛选必须能通过"故意隐藏真实改动确认 CI 判红"的校验。
4. **交付**：一改一个小 PR 附前后实测数据；外部依赖（账号/Secret/预算）汇总成带月度成本的清单交给你审批；最后输出优化前后总耗时与总成本对比表。

### 两条判定层（不会互相混淆）

| 层 | 对象 | 判定 |
|---|---|---|
| **端到端 PASS** | PR 全流程 | p50 `< 1m` 且 p95 `< 2m`（唯一端到端标准） |
| **硬门禁** | 单次 CI job 执行 | Lint/Format `<1s`、Typecheck/Build `<5s`、shard `<30s`、最慢 job `<35s`，越线即判红 FAIL |
| **分位诊断目标** | `waiting` / `setup` / 缓存命中率 | p50/p95 口径，未达标只预警不判红 |
| **本地性能目标** | 本地 Git 钩子 | `< 9s`，非 CI 硬门禁 |

---

## 怎么用

### 安装

把整个目录放进目标机器的 Agent skills 目录即可，不需要任何运行时依赖：

```bash
# Claude Code（只读取 ~/.claude/skills）
git clone https://github.com/zhibo0502/ci-speed-audit ~/.claude/skills/ci-speed-audit

# Cindy / Codex 等读取 ~/.agents/skills 的 Agent
git clone https://github.com/zhibo0502/ci-speed-audit ~/.agents/skills/ci-speed-audit
```

```powershell
# Windows（PowerShell）。Claude Code 用 .claude\skills，其他 Agent 用 .agents\skills
git clone https://github.com/zhibo0502/ci-speed-audit "$env:USERPROFILE\.claude\skills\ci-speed-audit"
```

```bash
# 不想用 git 也可以直接下载 zip 解压
# curl -L <repo-zip-url> -o ci-speed-audit.zip
# 解压后把目录放到 ~/.agents/skills/ci-speed-audit/ 下即可
```

> 安装后确认目录结构为 `<skills>/ci-speed-audit/SKILL.md` + `references/` 即可，无需额外配置。多机同步只需在每台机器执行一次上述命令。

### 触发方式（Prompt）

在仓库目录下对 AI 助手说任意一句命中 `When to Use` 的话即可，例如：

```
用 ci-speed-audit 审计并提速这个仓库的 CI，目标 PR p50 <1m、p95 <2m。
```

其他触发变体：

- 「把这个仓库的 CI 提速到 1 分钟以内」
- 「优化 CI 耗时，先查清楚慢在排队、setup 还是跑测试」
- 「给 PR 做可达测试筛选，同时保证合入主干前跑全量测试」
- 「把 JS/TS 仓库的 lint / format / typecheck / test 统一迁到 Bun 和 TypeScript 7」
- 「CI 最近又变慢了，先对比回归基线查是哪个桶回归、shard 分片是否失衡」

### 使用示例

**场景**：一个 GitHub Actions 仓库，CI 由 `lint → typecheck → test` 三层 `needs` 串行组成，`npm ci` 无缓存、全量 checkout、还带 `nick-fields/retry`。目标是把它压到 PR p50 < 1m。

**对 Agent 的输入**：

```
用 ci-speed-audit 审计并提速这个仓库的 CI，目标 PR p50 <1m、p95 <2m。
```

**Skill 会按以下顺序执行（摘要）**：

1. 调 `gh api` 拉最近 150 次运行 → 拆 waiting/setup/running → 输出三桶 p50/p95 审计基线
2. 删除 `retry` 步骤和 30min 宽超时（铁律：不掩盖失败）
3. 把 `lint`/`typecheck`/`test` 的 `needs` 链拆开并行，`checkout` 加 `filter: blob:none`
4. 迁移到 `bun install` + `node_modules` 缓存（PR 命中分支缓存、兜底 main 种子缓存）
5. `npm`/`jest`/`eslint`/`prettier` → `bun lint`/`bun fmt`/`bun check`/`bun test`，各步骤写入毫秒级 budget 硬门禁
6. 测试按 `.github/test-times.json` 的历史文件耗时装箱成 4 个 shard，每个 shard 独立 throwaway DB，验证 100% 文件已执行
7. PR 可达测试筛选（Turborepo/Nx/静态依赖图）+ 列出 rerun-everything inputs；merge queue 继续跑全量
8. 收尾输出「优化前 vs 优化后」总耗时与总成本对比表，外部项（如 Blacksmith runners）单独列成带月度成本的审批清单

### 它不会做什么

- ❌ 不会加 `retry` / 拉长 `timeout` / 跳过测试来掩盖失败
- ❌ 不会关掉 lint 规则或删减必要校验来换耗时下降
- ❌ 不会让 PR 筛选替代 merge queue 的全量套件
- ❌ 不会在拿到外部账号/预算前卡住不动——先做掉所有仓库内可落地的改动，再把需要你审批的项列成带月度成本的清单

---

## 参数与可调项

这个 Skill 不是函数调用，"参数"以**触发语中的约束**和**仓库内的实测重定**两种形式存在：

### 1. 触发时可声明的变量

| 变量 | 默认值 | 说明 |
|---|---|---|
| 端到端目标 | `p50 < 1m`、`p95 < 2m` | 唯一端到端 PASS 标准，可在 prompt 中改 |
| 历史样本量 | 最近 `150` 次运行 | 不足 150 次自动按三档降级（见下） |
| 目标语言栈 | 自动检测 | JS/TS 默认 Bun + TS7；其他语言按 `references/stack-profiles.md` 对照表 |
| CI 平台 | 自动检测 | GitHub Actions / GitLab CI / Buildkite / CircleCI 原语映射见 `references/platform-adapters.md` |

### 2. 默认预算（可在实测后重定）

| 项 | 默认值 | 判定层 |
|---|---|---|
| `waiting` | `< 5s` | 分位诊断目标 |
| `setup` | `< 15s` | 分位诊断目标 |
| 最慢 job | `< 35s` | 硬门禁 |
| Lint | `< 1s` | 硬门禁 |
| Format | `< 1s` | 硬门禁 |
| Typecheck / Build | `< 5s` | 硬门禁 |
| Tests | `< 30s / shard` | 硬门禁 |
| 本地钩子 | `< 9s` | 本地性能目标 |

> **预算重定**：大仓库 / 编译型语言物理不可达的单项，只允许在完成缓存/并行/加核优化后，凭实测冷热下界按 `优化后热基线 p95 × 1.2` 重定并固化到基线；禁止对未优化基线直接放宽。

### 3. 历史样本不足时的自动降级

| 可用运行数 N | 做法 |
|---|---|
| `N ≥ 150` | 正式分位基线 |
| `30 ≤ N < 150` | 临时基线，报告标注样本量 |
| `N < 30` | 静态 CI 配置审计 + 优化前后各 5–10 次实测采样 |

---

## 文件结构

```
ci-speed-audit/
├── SKILL.md                              # 主入口：铁律、审计流程、预算表、校验清单、参考索引（135 行）
├── references/
│   ├── platform-adapters.md              # 跨 CI 平台原语映射 + 统一拆桶口径 + <150 次降级（38 行）
│   ├── stack-profiles.md                 # JS/TS Bun+TS7 迁移明细 + Go/Rust/Python/JVM 对照（40 行）
│   ├── budgets-and-checklist.md          # 逐桶清单、预算重定、PR 筛选依赖图、容器/Matrix、PR 与成本模板（142 行）
│   ├── bucket-diagnostics.md             # waiting/setup 定位、传导量化、常见误判（37 行）
│   └── jobs-shards-baseline.md           # 过大 job 拆分、shard 负载均衡、回归基线（52 行）
└── README.md                             # 本文件
```

按需加载设计：`SKILL.md` 常驻上下文，`references/` 只在与当前步骤匹配时才被读取。

---

## 与其它仓库的配合

- **测试文件**：JS/TS 迁移到 `bun test` 时保留 `vitest` imports，无需改测试源码。
- **`AGENTS.md`**：治理完成后会把"change-detector-only tests"反模式写进仓库 `AGENTS.md` 禁止列表。
- **本地钩子**：默认用 `hk`，已有 `lefthook` / `husky` / `pre-commit` 且 `< 9s` 的仓库可沿用。
- **外部依赖**：Blacksmith / 温机池 / Secret / 预算类项会汇总成"需用户审批清单"逐项标明预估月度成本，不会自动替你开通。

---

## License

按你的仓库约定自行选择（建议 MIT）。
