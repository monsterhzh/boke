---
title: "Hermes Agent 的 Kanban Swarm：一条命令到底在数据库里写了什么"
date: 2026-09-18
draft: false
slug: hermes-kanban-swarm
tags: ["Hermes", "Kanban", "多智能体", "Agent", "SQLite"]
author: "monsterhzh"
summary: "hermes kanban swarm 用一条命令把“root/黑板 + N 个并行 worker + verifier + synthesizer”写进 SQLite 看板。本文拆开它在库里到底写了什么、黑板协议长什么样，以及它没解决的那部分：verifier 的 gate 是提示词约定、黑板靠 worker 自觉、整块看板只在一台机器上。"
---

如果多智能体的协作状态不是留在某个进程的内存里，而是每一层交接都变成数据库里的一行，会发生什么？

`hermes kanban swarm` 就是这个假设的一份实现。先给结论：它不是新的调度器，也不是新的多智能体框架，而是官方用一条 CLI 命令，把“root/黑板 + N 个并行 worker + verifier + synthesizer”这套最常见的拓扑，直接写进你已有的那块 SQLite 看板。真正值钱的不是“能并行”，而是拓扑本身就是表里的行和边；代价也同样清楚——它只解决拓扑，不解决语义。

## 一条命令建出来的六张卡

先看它实际写进数据库的东西。执行下面这条命令（参数写法按本机实测的版本，和官方文档有出入，后面“坑”那一节说）：

```bash
hermes kanban swarm "为多地域容灾方案做一次评审" \
  --worker researcher:梳理现网故障域 \
  --worker architect:设计切换路径 \
  --worker sre:估算演练成本 \
  --verifier reviewer \
  --synthesizer writer \
  --json
```

跑完打开看板，是六张卡：root 卡已经躺在 Done 列，三张 worker 卡全是 `ready`，verifier 和 synthesizer 还在 `todo`。这三个状态不是我编的形容词，而是上游测试里写死的断言（[test_kanban_swarm.py](https://raw.githubusercontent.com/NousResearch/hermes-agent/main/tests/hermes_cli/test_kanban_swarm.py)）：root 的 status 是 `done`，worker 列表（测试里放了两路）是 `["ready", "ready"]`，verifier 与 synthesizer 都是 `todo`；同时 `verifier.parents` 恰好等于全部 worker，`synthesizer.parents` 恰好是 verifier。

为什么重要：绝大多数多智能体方案的拓扑藏在运行时对象里，进程一死只剩日志。这里的拓扑是行和边，你可以用 SQL 查、用 `hermes kanban show` 看，也可以让人手工插一条边进来。整块看板本身是跨全部 Hermes profile 共享的持久化板，每张卡是一个任务，每个 worker 是一个独立的操作系统进程——官方对它的定位原话是让多个具名 agent 协作 “without fragile in-process subagent swarms”（[Kanban 文档](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban)）。

命令还留了一个幂等口子：`--idempotency-key` 只作用于 root 卡，命中已存在的 root 时，拓扑从黑板的 `topology` 键恢复而不是重建。写自动化脚本时这比看起来重要——重跑一次命令不会多出一整套孤儿卡。

## root 卡不是占位，它就是你唯一的黑板

root 卡在创建时先以 `initial_status="blocked"` 落库，再由 `_activate_root_inline()` 在同一个事务里 CAS 翻成 `done`，写进去的完成摘要固定为一句：`Swarm topology planned; root remains the shared blackboard.` 这一步失败，整个图回滚——上游测试专门断言失败时 `SELECT COUNT(*) FROM tasks` 是 0。

所谓“原子建图”不是宣传语，是有测试兜底的不变量。而黑板本身长这样：一个字符串前缀加一段 JSON，作为评论挂在 root 卡上。

```python
# hermes_cli/kanban_swarm.py
BLACKBOARD_PREFIX = "[swarm:blackboard] "
payload = json.dumps({"key": key, "value": value},
                     ensure_ascii=False, sort_keys=True)
```

合并函数 `latest_blackboard()` 把所有带这个前缀的评论按 key 合并，同 key 后写覆盖先写，并把获胜值的作者记进 `_authors`。因为黑板的全部状态就落在既有的 `task_comments` / `task_events` 两张表里，仪表盘、通知器、dispatcher 都不需要新服务（[kanban_swarm.py](https://raw.githubusercontent.com/NousResearch/hermes-agent/main/hermes_cli/kanban_swarm.py)）。

为什么重要：这是全文最可复用的一段。想让并行 worker 交换事实，不必引入 Redis 或消息队列；往 root 卡写一条带前缀的评论就够了，而且这条评论天然带作者、带时间、人也能读到。root 卡“建完即完成”是刻意的——它不产出交付物，只当黑板和依赖锚点，所以在 Done 列看到它不算流水线出错。

## 依赖门控怎么生效：让测试断言替你说

依赖门控就是一条 parent→child 边：所有父卡 done，子卡才由 dispatcher 从 todo 提升为 ready，默认每 60 秒一拍（`kanban.dispatch_interval_seconds`）。时序上，只完成部分 worker 时 verifier 仍是 `todo`；worker 全部完成，verifier 变 `ready`；verifier 完成，synthesizer 才 `ready`。这套规则不依赖谁记住流程，它写在板上。别把它跟 `kanban.auto_promote_children` 混：那个开关管的是 `decompose` 出来的、本来没有父阻塞的子卡要不要自动提升。

还有两处自动挂载值得知道：verifier 卡被强制挂上 `requesting-code-review` 技能，synthesizer 卡被强制挂上 `humanizer`。这条命令默认假设你要的是“代码评审式把关 + 去掉 AI 腔的成稿”；如果你的 verifier 是拿去核对财务数据的，这两个技能并不对口，得自己改卡。

为什么重要：依赖门控解决的是“谁先动”，不是“做得对不对”。它能保证并行 worker 全部落地之后才轮到收敛环节，但不会替你看一眼证据够不够——那是卡体和提示词的活。

## 代价之一：verifier 的 gate 是提示词，不是内核断言

verifier 卡的 body 里写着 `complete only with metadata {"gate": "pass"}`，synthesizer 卡的 body 写着 `Do not start until the verifier has passed the gate.`。听起来像一道硬门槛。

但我在 `recompute_ready()` 一侧没有找到任何校验 `gate` 取值的代码，依赖引擎只看 verifier 卡是不是变成了 `done`；上游测试也是在 verifier 完成（顺带带上 `gate=pass`）之后立刻断言 synthesizer 变 `ready`。也就是说，verifier 只要照常 complete，哪怕压根没写 `gate`，synthesizer 一样会起来。这一点我没有做反例实验来证伪，但从源码看，我没有理由认为它被强制。

要真正拦住，得把判断写进依赖结构：让 verifier 在不通过时 `block` 而不是 `complete`——block 是内核状态，不受提示词约束。

为什么重要：这类“文档里像断言、实现里是约定”的缝，是多智能体流水线最常见的误判来源。你按文档以为写错会被拦下，实际不会。

## 代价之二：熔断、心跳与协议违约

默认每张卡只给 worker 一次机会；要进 Ralph 式目标循环得显式加 `--goal`，循环默认预算 20 轮，耗尽后 block 等人工。worker 迭代预算用到约 90% 时会收到一次完成检查点提示（阈值 `agent.budget_warning_ratio`）。连续非成功尝试达到 `kanban.failure_limit`（默认 2）会 `gave_up` 并自动 block；`ready` 车道还有 respawn guard 拦重跑风暴。任务超过 `kanban.dispatch_stale_timeout_seconds`（默认 4 小时）且最近一小时没有心跳，会被判 `stale` 重新入队；worker 跑完却不调 `kanban_complete` 或 `kanban_block`，会被记成 `protocol_violation`（默认容忍连续 3 次）。心跳在这是续命，不是礼貌。

官方还把 Kanban 明确限定为单机：看板是本地 SQLite 文件，dispatcher 只在本机 spawn worker，跨主机共享一块板“不支持”（[Kanban 文档的 Out of scope 一节](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban#out-of-scope)）。所以 swarm 的规模上限不是模型能力，是一台机器能同时跑多少进程。

工作区还有一层没写进文档的细节。CLI 不传 `workspace_kind` / `workspace_path`，按源码推断每张卡走默认的 `scratch`（任务完成即删）——本机 blog 板的 `board.json` 里 `default_workdir` 是 `null`。**“swarm 的多个 worker 无法共享同一个 repo 工作树”只是我的推断，没有验证。** 想共享，就得建完卡再自己改工作区，或者干脆让卡片用绝对路径读写共享目录。

## 坑：官方示例的 --workers 跑不通

官方文档给的示例是 `--workers researcher,architect,sre`。我在本机（Hermes Agent v0.21.2，upstream rev `5eb99eb2`，`hermes --version` 还提示距上游 1875 个提交）照抄执行，得到的是：

```
hermes: error: unrecognized arguments: --workers researcher,architect,sre
```

`hermes kanban swarm --help` 的原文参数面里只有可重复的单数形式 `--worker PROFILE:TITLE[:SKILL,SKILL]`，外加必填的 `--verifier` 与 `--synthesizer`，以及 `--tenant` / `--priority` / `--created-by` / `--idempotency-key` / `--json`；没有 `--workspace`、`--parent`、`--model`（[kanban_parser.py](https://raw.githubusercontent.com/NousResearch/hermes-agent/main/hermes_cli/kanban_parser.py)）。我只能证明本机这个 rev 上 `--workers` 被拒绝，无法确认文档是否针对另一个版本，所以把它当成版本对齐问题而不是产品缺陷。

两个本机观察值得当复现前提记下。一是 Windows + MSYS 下带空格的参数会被拆开：`--worker researcher:Draft failover options` 报 `unrecognized arguments: failover options`，标题写成无空格形式才通过。二是被 spawn 的 worker 通过 CLI 改看板会被血缘围栏拒绝，原话是 `kanban: delegate_task child contexts cannot mutate Kanban tasks via the CLI`——worker 不能再建一个 swarm，这条 CLI 通路是设计性封死的；按文档，清掉血缘元数据来绕开它属于对抗性行为。

## 什么活该交给 swarm

我的判断标准只有两条：交付物是不是文件，交接是不是真的需要留痕。

一个函数调用就能解决的活——比如“读这份底稿写一段摘要”——套 swarm 只是给自己加负担，你还得给 verifier 和 synthesizer 各付一轮模型开销。反过来，需要多轮才收敛的活更适合交给 `--goal`，代价是轮数不确定：预算耗尽后卡片仍会 block 等人工，而不是继续烧下去。

而当任务能拆成几路互不依赖的事实收集、再收敛成一份结论时，swarm 的形状刚好对得上：worker 并行收集，verifier 统一把关，synthesizer 出终稿，root 卡上的黑板替你留下事实。

有一类活我会谨慎：需要多轮来回讨论、需要探索性试错的。swarm 的 worker 之间只有单向留事实的黑板，没有对话通道——上游只有 `latest_blackboard()` 一个合并函数，worker 侧怎么读到合并结果，我没有找到暴露出来的工具或子命令，推测是读 root 卡评论原文自行解释，**这一点未证实**。

一句话收尾：swarm 把“进程内的临时协作”换成了“数据库里的持久化协作”，你换来可观测与可恢复，付出的是协调逻辑要自己写进卡片——包括那句“这一段我不确定”。
