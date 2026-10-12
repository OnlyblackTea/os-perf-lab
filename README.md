# os-perf-lab

**「os-perf-bagu」Track 的骨架仓** —— 与发卷平台 [bagu-trainer](https://github.com/OnlyblackTea/bagu-trainer)
对接：平台按 `track.yaml` 渲染看板与体检环境，按 `stages/*/ticket.yaml` 发卷，
判据在**学员自己的机器上**跑。

> **当前状态：设计完成，骨架尚未落地。**
> 本仓目前只有文档（`docs/`）。骨架代码、canonical 测试、`track.yaml`、`stages/*/ticket.yaml`
> 与阶段分支链会随 Stage 1 一起落进来——顺序与判据形状见
> [`docs/设计规格.md`](docs/设计规格.md)，落地清单见
> [`docs/骨架落地清单.md`](docs/骨架落地清单.md)。

> [!WARNING]
> **本规格尚未按题库侧的 `_SCALE.md` 复核规模。** 现有 stage 的改动面普遍只有 1~2 个文件、
> canonical 只有 3~4 条，低于现行下限（改动面 **≥3 文件**、canonical **≥5 条**且红 ≥2 条不同不变量、
> 含边界/拒绝路径）。**开工前必须先复核并扩编**（同 stage 扩面，或与相邻 stage 合并），
> 并把结果写回规格与 ticket 卡。参考已复核的两份：`mq-lab`（6 stage 全部扩面）
> 与 `concurrency-lab`（Stage 1 / Stage 2）。


## 这是什么

一句话定位：给「Linux 性能四板斧背得滚瓜烂熟、却从没亲手 `perf record` 过」的后端候选人，

| 项 | 值 |
|---|---|
| track.id | `os-perf-bagu` |
| 语言栈 | Java 17（`java.util.concurrent` + JDK 内置 `HttpServer`）+ Maven 3.9 + JUnit 5；观测脚本 `scripts/*.sh` 为 bash 5 + `awk`/`grep`/`/proc`（**纯离线**） |
| 基础设施 | 一个 compose：app（轮询 18081）+ pinger（压测源）。**无 DB / 无 Redis / 无外部依赖**；canonical 测试不依赖 compose（进程内起服务） |
| stage 数 | 6（其中 **1 个是「判据最弱」的容量评估阶段**，见 §1.5；无 board-only 阶段） |
| 每 stage 规模 | （见规格） |
| 依赖前置 | 见 §6 `requires`：`java: "17"`、`build: maven`、`docker: "24"`、`python: "3.11"`（脚本用）、`kernel.linux_cgroup: "v2 或 v1"`、`kernel.perf_event_paranoid: "<=1"`（**仅观测练习需要，判据不依赖**） |
| 与 database-bagu 的关系 | 完全互补、零重叠：seckill-lab 问「慢在 SQL 还是锁」，本 Track 问「慢在 CPU / 内存 / IO / 网络 的哪一层」。seckill-lab 的 Stage 3（锁与超卖）讲的是**数据库行锁**，本 Track 的 Stage 5 讲的是**用户态 futex 锁**——两者是同一个词的两层，规格里刻意在八股地图上标注了这处对照 |
| 预计落地工作量 | 骨架 8~10 人日 / canonical 测试 6~8 人日 / 题面与判据 3 人日 / 分支与 pin 1 人日（合计约 18~22 人日，**比 concurrency-lab / mq-lab 贵**，因为骨架要含四个独立的故意缺陷 + 观测脚本） |

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
