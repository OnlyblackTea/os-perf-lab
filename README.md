# os-perf-lab

**「os-perf-bagu」Track 的骨架仓** —— 与发卷平台 [bagu-trainer](https://github.com/OnlyblackTea/bagu-trainer)
对接：平台按 `track.yaml` 渲染看板与体检环境，按 `stages/*/ticket.yaml` 发卷，
判据在**学员自己的机器上**跑。

> **当前状态：设计完成，骨架尚未落地。**
> 本仓目前只有文档（`docs/`）。骨架代码、canonical 测试、`track.yaml`、`stages/*/ticket.yaml`
> 与阶段分支链会随 Stage 1 一起落进来——顺序与判据形状见
> [`docs/设计规格.md`](docs/设计规格.md)，落地清单见
> [`docs/骨架落地清单.md`](docs/骨架落地清单.md)。


## 这是什么

一句话定位：给「Linux 性能四板斧背得滚瓜烂熟、却从没亲手 `perf record` 过」的后端候选人，

| 项 | 值 |
|---|---|
| track.id | `os-perf-bagu` |
| 语言栈 | Java 17（`java.util.concurrent` + JDK 内置 `HttpServer`）+ Maven 3.9 + JUnit 5；观测脚本 `scripts/*.sh` 为 bash 5 + `awk`/`grep`/`/proc`（**纯离线**） |
| 基础设施 | 一个 compose：app（轮询 18081）+ pinger（压测源）。**无 DB / 无 Redis / 无外部依赖**；canonical 测试不依赖 compose（进程内起服务） |
| stage 数 | 6（其中 **1 个是「判据最弱」的容量评估阶段**，见 §1.5；无 board-only 阶段） |
| 每 stage 规模 | 按 [`_SCALE.md`](./_SCALE.md) 的下限复核后（本轮「**v1.1 规模复核**」，见 §1）：6 个 stage **全部选择「同 stage 扩面」、零合并**。逐 stage 一律 **改动面 3 个文件**且覆盖三类角色（① 核心状态/算法 ② **计数/观测包装器**——把「调用了多少次」变成可断言的量 ③ **调用编排或异常·边界路径**——批次上限、池容量、缓存上界、被拒/耗尽路径）、canonical **6 条**（**红 3 · 单线程基线 2 · 边界/拒绝 1**）、被破坏的不变量 **3 个**（= 红用例数，逐条分属不同不变量）、考点族 **≥2 个**。合计：改动面 **10 → 18 个文件位置**、canonical **23 → 36 条**（红 6 → 18 · 单线程基线 14 → 12 · 边界/拒绝 3 → 6） |
| 依赖前置 | 见 §6 `requires`：`java: "17"`、`build: maven`、`docker: "24"`、`python: "3.11"`（脚本用）、`kernel.linux_cgroup: "v2 或 v1"`、`kernel.perf_event_paranoid: "<=1"`（**仅观测练习需要，判据不依赖**） |
| 与 database-bagu 的关系 | 完全互补、零重叠：seckill-lab 问「慢在 SQL 还是锁」，本 Track 问「慢在 CPU / 内存 / IO / 网络 的哪一层」。seckill-lab 的 Stage 3（锁与超卖）讲的是**数据库行锁**，本 Track 的 Stage 5 讲的是**用户态 futex 锁**——两者是同一个词的两层，规格里刻意在八股地图上标注了这处对照 |
| 预计落地工作量 | 骨架 8~10 → **12~14** 人日 / canonical 测试 6~8 → **11~13** 人日 / 题面与判据 3 → **4.5** 人日 / 分支与 pin 1 → **1.5** 人日（合计 **21 → 29~33 人日**，逐项理由见 §7 的 v1.1 重新估算块；**比 concurrency-lab / mq-lab 更贵**，因为骨架要含**十三个**独立的故意缺陷 + 六个边界守卫 + 六个账本 + 观测脚本，其中三条边界判据要用「确定性序列」构造而不是靠时间） |

## stage 一览

| stage | 分支名 | 依赖 | canonical 测试路径 |
|---|---|---|---|
| 1 | `stage-01-cpu-hot-path-profiling` | `[]`（**DAG 根**） | src/test/java/com/devteam/osperf/stage01/CpuHotPathAcceptanceTest.java |
| 2 | `stage-02-memory-swap-and-oom` | `[os-stage-01]` | .../stage02/MemoryBoundAcceptanceTest.java |
| 3 | `stage-03-disk-io-and-fsync` | `[os-stage-02]` | .../stage03/DiskIoAcceptanceTest.java |
| 4 | `stage-04-network-and-context-switch` | `[os-stage-03]` | .../stage04/NetworkReuseAcceptanceTest.java |
| 5 | `stage-05-lock-contention-and-futex` | `[os-stage-04]` | .../stage05/LockContentionAcceptanceTest.java |
| 6 | `stage-06-pressure-test-and-capacity-planning` | `[os-stage-05]` | .../stage06/CapacityMetricsAcceptanceTest.java |

> 表里的**分支名就是 git 分支名**：平台的 Forge 在 provision 时用
> `base_ref = origin/<分支名>`（起始代码）与 `head_ref = origin/<分支名>-sol`（参考解），
> 两者缺一都会报 `workspace_ref_missing`。分支是**一条链**（每个阶段分支 = 上一阶段 + 本阶段测试），
> 不是兄弟分支——理由见骨架落地后的 `stages/README.md`。

## 目录

```
os-perf-lab/
├── README.md              ← 本文件
├── LICENSE                ← MIT（内容仓许可）
├── .gitattributes         ← eol=lf：digest_pin 要求工作树字节跨平台稳定
├── .gitignore
└── docs/
    ├── 设计规格.md          ← 本 Track 的设计规格快照（见下方提醒）
    └── 骨架落地清单.md       ← 落地前要逐条核对的清单
```

骨架落地后还会出现（形状见设计规格 §2）：

```
├── track.yaml             ← 与平台对接的清单（manifest 惰性声明 / runtime 可执行入口 + digest_pin）
├── pom.xml / go.mod / pyproject.toml
├── src/ 或 internal/      ← 业务实现（学员改动面）
├── src/test/... 或 tests/ ← canonical 验收判据【判定面】
├── stages/<stage-目录>/ticket.yaml   ← 权威工单契约（目录名 = 分支名）
├── stages/README.md       ← 分支链契约
├── scripts/               ← build / digest-pin / check-red-green【判定面】
└── TASKS/                 ← 学习者叙述视图与交付证据
```

## 与规格的关系（重要）

`docs/设计规格.md` 是**创建本仓时的规格快照**，权威副本在题库侧的
`bagu-track-specs/os-perf-lab.md`（设计仍在演进的阶段，改动先落那边，再同步到这里）。

两份文档的分工别搞混：

- **平台契约**（字段、准入门、分支链、锚点算法）由平台仓单方面定义，本仓只遵守、不复制规范正文；
- **题面怎么写**见平台仓 `docs/工单写作规范.md`；
- **每个 stage 该有多大**见题库侧 `_SCALE.md`（改动面 ≥3 文件、canonical ≥5 条、
  红 ≥2 条不同不变量、含边界路径）——本规格已按它复核过。

## 许可

MIT，见 [LICENSE](LICENSE)。
