# Bucket 耗时定位、传导量化与常见误判

本文件是 `SKILL.md` 的按需参考（见「按需加载的参考文档」表）：当 `waiting` / `setup` 分位诊断目标未达标、需做两桶传导归因、或遇到缓存虚高/并行饱和等误判症状时读取。所有判定默认基于最近 **150 次** CI 运行的 `waiting` / `setup` / `running` 拆分数据；若历史运行不足 150 次或未使用 GitHub Actions，先按 `platform-adapters.md` 统一拆桶口径并标注临时样本量。

## 1. 用最近 150 次运行定位 waiting 与 setup 桶瓶颈

### waiting 桶（分位诊断目标 `< 5s`）定位与判据

- **定位步骤**：把每次运行的 `waiting` 与提交时间、runner 类型、并发上限对齐，看是“排队等空闲 runner”“并发配额占满”还是“同 PR 旧 run 未取消”占主导；同时对比启用/未启用并发取消策略前后的 p95。
- **判据**：`waiting` p50 `< 5s` 且低占用时段（runner 池 `< 90%`）p95 `< 5s` 为分位诊断目标达标；低占用时段 p95 未达标（`≥ 5s`）且随推送频率/工作时段波动大，指向并发配额或 runner 池不足，需评估低排队温机池（默认 Blacksmith runners / warm machines，备选与跨平台方案见 `budgets-and-checklist.md` 与 `platform-adapters.md`）；若集中在“连续推送后旧 run 仍排队”，则是缺少取消被新提交取代的运行（cancel superseded runs），加并发取消即可消除。

### setup 桶（分位诊断目标 `< 15s`）定位与判据

- **定位步骤**：把 `setup` 拆成克隆（clone）、依赖安装（install）、缓存恢复（cache restore，含基础镜像拉取 / Docker BuildKit 层缓存恢复 / 跨 job artifact 下载）、工具链准备（toolchain setup）四个子段，对最近 150 次运行分别取 p50 与 p95，找占比最大子段；同时记录 PR 缓存命中率与缓存恢复实际耗时。
- **判据**：`setup` p50 `< 15s` 且缓存命中率 `> 95%` 为分位诊断目标达标；分位诊断分开看：缓存命中样本 p95 `< 15s`，缓存未命中样本允许走冷缓存基线（p95 可能 `> 15s`）。clone 子段占主导或偏高 → 启用 blobless clone；install 子段占主导或偏高 → 用 `main` 预热缓存并缓存依赖目录（如 `node_modules` / `GOCACHE` / `.venv`）；toolchain 准备子段占主导或偏高 → 改为 baked toolchain；镜像拉取/层缓存或 artifact 下载占大头 → 按 `budgets-and-checklist.md` 第 I 节治理 BuildKit 远端缓存与制品传递；若缓存命中率达标（`> 95%`）但 `setup` 仍慢，说明是“无效命中”，按第 3 节误判定位继续排查。

> 注：`< 5s` / `< 15s` 是**分位诊断目标**，不是单次硬门禁；端到端唯一 PASS 标准是 p50 `< 1m`、p95 `< 2m`；硬门禁只约束单次 CI job 执行时长（见 SKILL.md 校验清单）。

## 2. 量化 waiting 到 setup 的耗时传导

waiting 发生在 setup 开始之前，setup 不会直接“导致”等待；真正的传导机制是：**runner 被占用时长 ≈ `setup + running`，setup 越慢 → runner 被占越久 → 队列排空越慢 → 后续运行的 `waiting` 越高**。用最近 150 次运行按以下方式量化与判定：

- **量化步骤**：
  1. 为每次运行记录 `waiting`、`setup`、`running` 及其开始时刻当时的「在跑 job 数 / runner 池容量」。
  2. 计算 `waiting` 与「同一窗口内并发运行的平均 `setup`（或 `setup + running` 占用时长）」的相关性，并按 runner 池占用率分桶（如 `<50%` / `50–90%` / `>90%`）比较 `waiting` 的 p95。
  3. 做一次对照校验：仅在缓存/runner 预热命中（`setup` 极短）与冷启动（`setup` 长）两类运行后，比较其后紧邻运行的 `waiting` 分布。
- **判定 setup 是否为 waiting 主因的判据**：
  - `waiting` p95 在池占用率 `>90%` 时显著升高，且与并发运行的平均 `setup` 时长呈明显正相关 → setup 传导成立，优先压 `setup`（baked toolchain、缓存预热）能同时降低 `waiting`。
  - `waiting` p95 高但与 `setup` 时长无关、而与在跑 job 数/配额直接相关 → 主因是容量（runner 池或并发配额不足），应回到第 1 节 waiting 桶判据评估低排队温机池（默认 Blacksmith runners / warm machines）。
  - `waiting` 高且集中于连续推送后旧 run 未结束 → 主因是缺少取消被新提交取代的运行（cancel superseded runs），与 setup 无关。
  - 压完 `setup`（例如命中预热缓存）后 `waiting` p95 无明显下降 → 证伪“setup 是主因”，停止在 setup 上继续投入。

## 3. 常见误判定位

- **缓存命中率虚高但 `setup` 不降**：命中率统计的是“请求到即算命中”，但不代表命中项真正省下了耗时。定位时对比「缓存命中」与「缓存未命中」两种运行下 `setup` 的 p50 与 p95 实际差值：若命中后仍重复做下载解压、恢复依赖目录（如 JS/TS 的 `node_modules`）软链校验或整量 `install` 校验，说明命中的是“无效命中”；此时按真实节省耗时判定缓存价值，而非只看命中率数字。
- **并行度“饱和”但 `running` 内单 job / shard 仍硬门禁越线**：先确认 worker 数是否真的与物理核数匹配（buy cores before shards），再看各 shard/worker 的耗时直方图——若少数 shard 明显拖尾而 CPU 利用率并未跑满，瓶颈更可能是 shard 负载不均、共享资源串行或测试间锁竞争，而非核数不够；此时应改用历史文件耗时做均衡 shard 分片、为每个 shard 分配独立一次性数据库（throwaway DB）并消除共享状态，而不是继续加核或盲目再做 shard 分片。均衡做法见 `jobs-shards-baseline.md`。
- **镜像层缓存与跨 Job 制品误归入 `running`**：若把基础镜像拉取、Docker layer 恢复或跨 job 大体积 artifact 上传/下载混进 `running`，会误判为代码编译或测试慢。拆桶时将镜像拉取与缓存/制品恢复计入 `setup`，实际编译构建计入 `running`；若单次镜像构建层命中率低，优先检查 `Dockerfile` 中是否把高频变动源码 `COPY` 放在依赖安装之前导致整层失效。
