---
title: Kanban Swarm 实测：把多智能体协作做成一张可恢复的持久任务图
date: 2026-09-16
tags: [hermes, kanban, multi-agent, orchestration, sqlite]
author: 彭梓坚
---

你写过一个脚本，一次 spawn 出八个 `delegate_task` 子代理并行扒数据。跑了一个小时，第六个把主进程弄崩了。重启之后你发现：没有断点，没有中间产物，八个 worker 的上下文全在崩溃那一刻蒸发了，只能从头再来一遍。这不是运气差，这是内进程子代理的结构性代价。

## 进程内的 subagent 群，为什么撑不起长任务

把并行 worker 放进同一个进程，省事，但有三笔账要还。

第一，生命周期和上下文绑死在主进程上。官方文档写得直白：Hermes 进程重启不会恢复运行中的子代理，它的 attempt 会被标记成 `unknown`；`/stop` 会取消运行中的后台子代理，关闭或重置所属会话会丢弃它的活动子代理（`website/docs/user-guide/features/delegation.md:428-443`）。主进程崩，全部丢，这是内进程模型的宿命，不是写得好不好能救回来的。

第二，并行 worker 之间没有可信的共享状态。`delegate_task` 的子上下文被明确禁止改动看板——`kanban_db.py:165-184` 的 `_assert_not_delegated_child_mutation` 直接抛「delegate_task child contexts cannot mutate Kanban tasks or boards」。这意味着八个 worker 想交换中间结论，连一个双方都认账的落点都没有。

第三，跑到一半没法从外面插一句话。官方对照表把 `delegate_task` 定义为 RPC fork→join、父代理阻塞等待、resumability 为 none、不支持 human in the loop（`kanban.md:32-53`）。中断即丢状态，人工介入没有入口。

Kanban 的赌注就是冲这三笔账来的：把协作状态外化成数据库行。每一张卡是一行，每一次交接是一行，黑板上的每一条更新也是一行。任何一方崩了，图还在，谁都能接着跑。要记住边界：Kanban Swarm v1 不是第二个调度器——源码模块 docstring 的原话是它「intentionally does not introduce a second scheduler」。它只是一个把一张小任务图写进既有 Kanban 内核的建图助手，认领、调度、恢复全部沿用既有 dispatcher。

## 拓扑长什么样（root/黑板 → N 并行 worker → verifier → synthesizer）

一次真实建图返回的结构是这样的（隔离库实测，`HERMES_KANBAN_DB` 钉住临时库）：

```json
SwarmCreated = {"root_id": "t_e8af1a02", "worker_ids": ["t_70b37532", "t_a68f2d32"], "verifier_id": "t_ca032e9b", "synthesizer_id": "t_37248cd8"}
```

落库后五张卡的状态：

```
tasks:
  t_e8af1a02  done      swarm-probe  Swarm: probe: verify Kanban Swarm v1 topology in an isolated DB
  t_70b37532  ready     researcher   probe-W1
  t_a68f2d32  ready     writer       probe-W2
  t_ca032e9b  todo      reviewer     Verify swarm outputs
  t_37248cd8  todo      publisher    Synthesize swarm outputs
```

依赖边原文（`task_links` 表，parent → child）：

```
  t_70b37532 -> t_ca032e9b
  t_a68f2d32 -> t_ca032e9b
  t_ca032e9b -> t_37248cd8
  t_e8af1a02 -> t_70b37532
  t_e8af1a02 -> t_a68f2d32
```

五张卡、五条边的形状：root 建成即 done，只当黑板与审计锚点；N 个 worker（这里 2 个）`ready` 并行；verifier（审校卡）门控在全部 worker 之后；synthesizer（汇总卡）门控在 verifier 之后。为什么重要：这张图决定了「谁能并行、谁必须等谁」，也决定了后面讲 gate 时「被门控」和「gate 元数据被校验」是两件完全不同的事。

黑板（blackboard）是这里面最「低科技」的部分。它不加任何新服务，就是 root 卡上一条结构化 JSON 注释，前缀 `[swarm:blackboard] `，读时按 key 合并、同 key 后写覆盖、`_authors` 记录胜出者。实测合并结果长这样：

```json
{
  "topology": {
    "goal": "probe: verify Kanban Swarm v1 topology in an isolated DB",
    "root_id": "t_e8af1a02",
    "synthesizer_id": "t_37248cd8",
    "verifier_id": "t_ca032e9b",
    "worker_ids": ["t_70b37532", "t_a68f2d32"]
  },
  "_authors": {
    "topology": "swarm-probe"
  }
}
```

好处是它复用既有的 comment / event / 通知 / 面板，不引入新服务。代价是它的并发语义不是强一致，这个到「什么时候该用」那节会展开。

还有一条强绑定：verifier 卡固定注入 `requesting-code-review`，synthesizer 卡固定注入 `humanizer`（源码写死在 `hermes_cli/kanban_swarm.py:304,323`）。源码没留下选这两个默认值的理由注释，所以「合理不合理」只能算取舍讨论（推断，非事实断言）：`humanizer` 用来去 AI 腔、`requesting-code-review` 用来做代码审查，方向上对；但给聚合卡硬塞一个「去 AI 腔」skill，对非文案类任务是否合适，存疑。

## 原子建图——一个 blocked→done 的 CAS 闩锁

root 卡先以 `blocked` 状态落库，再在同一事务里内联翻成 `done`。这个翻转是一个 CAS（compare-and-swap）：`UPDATE tasks SET status = 'done' … WHERE id = ? AND status = 'blocked'`，只有真的命中那一行（`rowcount == 1`）才继续建后面的图。为什么先 blocked？这是占位闩锁：防止 dispatcher 在「图只建了一半」的时候就把 worker 拉起来乱跑。这也是「建成即完成」的由来——root 不是真任务，是闩锁加黑板。

为什么不直接调用 `kb.complete_task` 而要在事务里内联翻状态？源码 docstring 给了明确理由：后者会自己开事务，并触发提交后副作用——工作区清理、失败计数复位、`recompute_ready`。如果外层事务最后回滚，这些副作用就会误触发。所以建图走「先持久、再催活」的顺序：事务提交之后，才 `recompute_ready` 提升子卡，才发 `kanban_task_completed` 生命周期钩子。

幂等也在这里兜底：同一个 `idempotency-key` 重跑时，从黑板的 `topology` 键恢复已存在的拓扑，不重复建图。为什么重要：这三件事（闩锁、事务边界、幂等）合起来，就是「图可以安全地重跑、可以断点恢复」的机制底座。

## 上手与隔离实测（CLI 与 Python API 两条入口）

先钉一个版本口径。swarm 于 v0.15.0（2026-05-28）引入，官方 release notes 点名了 kanban swarm topology helper；把 Kanban Swarm 记成「v0.16 功能」的说法找不到可靠来源，反向证据倒是有一条——v0.16.0 的主题是桌面应用，其 release 条目里没有 swarm。本文实测环境为 v0.21.0（`hermes --version`）。

CLI 的真实 usage（本机 `hermes kanban swarm --help`）：

```
usage: hermes kanban swarm [-h] [--worker PROFILE:TITLE[:SKILL,SKILL]]
                           --verifier VERIFIER --synthesizer SYNTHESIZER
                           [--tenant TENANT] [--priority PRIORITY]
                           [--created-by CREATED_BY]
                           [--idempotency-key IDEMPOTENCY_KEY] [--json]
                           goal
```

`--worker` 可重复，`--verifier` / `--synthesizer` 必填。一条真实跑通的命令：

```
hermes kanban swarm "probe: verify Kanban Swarm v1 topology in an isolated DB" --worker "researcher:W1 probe" --worker "writer:W2 probe" --verifier reviewer --synthesizer publisher --json
```

Python API 则多了 CLI 给不了的能力：`SwarmWorkerSpec` 能传 `body` / `skills` / `priority` / `max_runtime_seconds`，`create_swarm` 还能传 `workspace_kind` / `workspace_path`。这两条正好接上后面两个坑（worker 正文太薄、CLI 没有工作区参数）。

本文最实用的一段是隔离手法：`HERMES_KANBAN_DB` 指向一个临时 SQLite 文件，就能零副作用地把拓扑、状态、黑板、卡片正文全打印出来验证，不碰生产板。实测里有个反直觉结论：设了这个环境变量之后，显式 `--board blog` 会被静默忽略——命令读的还是临时库（复核实测：`--board blog` 打出来的是隔离库里那几张探测卡，`done=1`、`ready=2`，assignee 是 researcher / writer / reviewer / publisher）；`unset` 之后同一命令才读到真实 blog 板（`orchestrator done=1`、`reviewer running=1`）。源码旁证：`kanban_db.py:713-736` 的 `kanban_db_path()` 把 `HERMES_KANBAN_DB` 列为路径解析第 1 优先级，是给 dispatcher→worker 交接做的防御性设计。为什么重要：你要是想在生产板上做验证实验，这个优先级会悄悄让你「验证了个寂寞」，也可能反过来误伤真数据。

## 四个真实的坑（都有实测证据）

坑一，官方文档示例是错的，且上游至今没修。文档写 `--workers researcher,architect,sre`（`website/docs/user-guide/features/kanban.md:842`，线上同页 2026-09-16 抓取仍未修），本机 v0.21.0 实跑直接退出：

```
hermes: error: unrecognized arguments: --workers researcher,architect,sre
exit=2
```

正确写法是可重复的 `--worker PROFILE:TITLE[:SKILL,SKILL]`。措辞上要精确：不是「官方文档胡说」——同小节的拓扑描述与源码行为一致，是「文档示例与当前 CLI 的精确定义不符」。结论：读文档要顺手 `--help` 对一遍。

坑二，CLI 建的 worker 卡正文太薄。`parse_worker_arg`（`kanban_swarm.py:381-390`）把 body 直接设成标题（`body=parts[1]`），worker 实际读到的只有「标题 + 一段协议尾巴」。用 CLI 建图（隔离库实测，`--worker "researcher:W1 probe"`）落库后，`t_144cc4f2` 这张 worker 卡的完整正文就是：

```
W1 probe

## Swarm protocol
- Swarm root / shared blackboard: `t_7662b20f`.
- Read sibling/parent handoffs from Kanban context before working.
- Put machine-readable facts in completion metadata.
- Put cross-worker notes on the root task using structured comments.
- Goal: probe: review check CLI thin body
```

想让 worker 拿到真正的 prompt，必须走 Python API 自建卡、把 body 写厚。协议尾巴只告诉它「去哪读、往哪写」，不告诉它「做什么」。

坑三，CLI 没有 workspace 参数，默认 scratch。worker 的产物会随临时工作区一起被删掉。清理时机有明确出处：`complete_task` 的数据库事务提交之后触发，有活动子卡时延迟清理；`dir:` 工作区完成时刻意保留（`kanban_db.py:5889-5960`、`kanban.md:65-68`）。所以跨卡交接必须自己给 `dir:` 工作区，否则上一张卡的产物根本活不到下一张卡来取。

坑四，verifier / synthesizer 是单 skill 卡，而 CLI 语义是「请求的 skill 全都不存在就硬失败」。这条有真实前科：synthesizer 卡曾硬编码一个不存在的 `avoid-ai-writing`，结果每次 spawn 都在 CLI 启动阶段报 `Unknown skill(s): avoid-ai-writing` 硬失败，dispatcher 反复重试到上限——修复提交 `69b74c15a3`（PR #34337）把这行改成 `humanizer`。当前机制（`cli.py:8898-8914`）：请求的 skill 部分缺失时降级跳过、只 warning；全部缺失时抛 `ValueError` 硬失败。单 skill 卡一旦名字写错，就是当年那个 bug 的形态。好在两个强绑 skill 在本机默认安装里真实存在，5 个 profile 下都能解析，照抄不会炸——但这是「碰巧现在对」，不是「永远不会错」。

## 什么时候该用 swarm，什么时候别用

对照三件事：

- `delegate_task`：分钟级、内进程、父卡阻塞等返回、无恢复、无人在环。适合一次性的短推理扇出。
- 自建 parent-child 卡链：串行流水线最直白。调研→写作→审校→发布这种纯串行流程用它更合适。
- `kanban swarm`：并行扇出 + 门控 + 汇总，适合「N 个 worker 各查一块、verifier 把关、synthesizer 汇总」的形状。

swarm 真正买到的是四样：并行扇出、一个显式的 gate 契约位（verifier 被要求「证据充分时才带着 `metadata {"gate": "pass"}` 完成」）、可被外部读写的黑板、以及并发上限与成本由既有 dispatcher / 看板控制。但它没买到的，恰恰是很多人以为买了的那一样——这个 gate 不会被内核强制，放行实际由卡状态驱动。实测证据：

```
CASE A — verifier completes with NO gate metadata
  verifier done:     synthesizer = ready  <-- 不写 gate 也放行
CASE A2 — verifier completes with metadata {'gate': 'fail'}
  verifier done(gate=fail): synthesizer = ready  <-- 写错值也放行
CASE B — control: verifier BLOCKS instead of completing
  verifier blocked:  synthesizer = todo  <-- 只有卡状态能拦
```

全仓检索这个字段名，与 swarm 相关的只有两处：verifier 卡 body 的 prompt 文案（`hermes_cli/kanban_swarm.py:287-292`，源码里写作 `metadata {\"gate\": \"pass\"}`）和测试文件传参（`tests/hermes_cli/test_kanban_swarm.py:255`）。`kanban_db.py`、`tools/kanban_tools.py` 里出现的 `gate` 全是「gated／依赖门控」这类注释用词，没有一处读任务 metadata 里的 gate 字段（仓库里另有一批 `"gate"` 字面量属于 `/goal gate` 质量闸功能，与本文无关）。顺带一提：源码那一行是转义写法 `\"gate\"`，直接 grep `"gate"` 反而找不到它，核对时别漏。所以要把两件事拆开说：verifier 卡确实被依赖图门控在全部 worker 之后，这是机制；`gate` 元数据只是 prompt 层约定，不是机制。把 gate 说成「系统级门控」是错的。

黑板的能力边界同理。8 线程并发写同一 key 的实测结果：

```
CASE C — 8 threads concurrently post key='race'
  write errors: 0
  comments landed on root: 9 total (topology=1 + race=8)
  merged winner for key 'race': {"i": 2}
  _authors['race']: w2
  distinct created_at values: [1789527038]
```

不丢写：每条 comment 与一条 `commented` 事件同事务落库。但同 key 后写覆盖、胜者不可预约——`created_at` 是秒级，同秒并发时合并顺序只由 `list_comments` 的 `ORDER BY created_at ASC` 决定，该查询没有 rowid 兜底（同模块的 `list_comments_after` 明确用 rowid 兜底，两处对「同秒 burst」的处理不一致）。同一实验连跑两次，胜者分别是 w4 和 w2。所以黑板是「无锁追加 + 读时合并」，不会丢，但会覆盖，胜者可追溯、不可预约。安全的用法是每个 worker 用独立 key（比如 `finding:w1` / `finding:w2`）——这是由机制推出的推理，源码和文档都没给这个建议。

## 落地清单与观测面

五步清单：确认版本（`hermes --version`，本文环境为 v0.21.0）→ 建 board → 用 Python API 写厚 body，别指望 CLI 的标题正文 → 给 verifier 写清 gate 判据（什么算过、缺什么算不过，虽然内核不校验，但这是唯一一道质量闸）→ 用 `dir:` 指定一个跨卡共享的工作区，并给建图加幂等键。

观测面：`hermes kanban stats` / `list` / `show <id>` / `tail` / `log <id>` / `runs`，以及 dashboard 的 workers/active、runs/{id}、inspect 端点（来源：官方 kanban 文档）。

成本量级要诚实：每个 worker 是一次完整 agent 会话。本项目自述的流水线记录是单卡约 2–10 分钟、整条 5 卡链 30–60 分钟——注意这是本人流水线自述，本轮没有独立复测，别把它当通用基准。swarm 的并行换来的是墙钟时间缩短，总 token 成本不会少；而 verifier 这个串行瓶颈，决定了最慢的那个 worker 仍然压着整条链的下限。

## 参考资料

- Hermes 官方 Kanban 文档：https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban（抓取 2026-09-16）
- `kanban_swarm.py` 提交历史：https://api.github.com/repos/NousResearch/hermes-agent/commits?path=hermes_cli/kanban_swarm.py（抓取 2026-09-16）
- v0.15.0 release notes：https://api.github.com/repos/NousResearch/hermes-agent/releases/tags/v2026.5.28（抓取 2026-09-16）
- skill 修复提交 69b74c15a3（PR #34337）：https://api.github.com/repos/NousResearch/hermes-agent/commits/69b74c15a3（抓取 2026-09-16）
- 本机源码（安装目录 `C:\Users\peng\AppData\Local\hermes\hermes-agent`）：`hermes_cli/kanban_swarm.py`、`hermes_cli/kanban_db.py`、`cli.py`（仓库根）、`tools/kanban_tools.py`、`website/docs/user-guide/features/kanban.md`、`website/docs/user-guide/features/delegation.md`
- 实测原始输出：`evidence/swarm_probe_output.txt`、`evidence/research_probe_output.txt`、`evidence/review_probe_output.txt`（审校期 CLI 复核，2026-09-16）
