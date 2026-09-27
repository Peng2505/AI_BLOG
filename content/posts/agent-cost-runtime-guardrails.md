---
title: "7.8 万美元的 Agent 账单之后：我给自己 Agent 装了三层刹车（附一次运行时中断实测）"
date: 2026-09-27
tags: [ai-agent, guardrails, cost-control, observability]
author: 彭梓坚
description: "Agent 成本失控时，护栏该装在哪一层、越界时该做什么动作——把「事后账单」改造成一条运行时可中断、可观测的信号，并附一份可核实到文件的成本区间数据。"
---

## 一、先把那条帖子当成信号，而不是当成故事

9 月下旬，Hacker News 上出现了一则被标记为 `[flagged]` 的自述帖，标题逐字是「OpenAI Codex agents go rogue and consumes USD 78,000 without authorization」，作者 `lorenzomassaro`，他自述身份是意大利 AI 公司 Eternal Tech 的 CTO、产品是 detwin.ai，并附了一个工单号 [W02][W04]。截至本机抓取（2026-09-27T14:17:38 经官方 API），这则帖子的 score 为 66、descendants 为 27（评论计数）；页面端（同日 14:19）列表行显示 `[flagged]`、61 points、25 comments [W02][W03]。这些数字会随时间变化，我在这里写死抓取时间与抓取方式，是因为对这类材料的任何引用都必须「带时间、带方式」——它是一则 text 自述帖，没有外链，没有任何厂商侧或第三方核实的记录。

先把边界划清楚，再往下读。我不裁决这件事的真假：帖主自己在正文里就写明了「these local token counters are not the authoritative OpenAI billing ledger」——他拒绝把本地 token 计数直接乘以单价换算成账单 [W02]。评论区同时存在两种声音：`minimaxir` 直指「This submission appears to be highly vote-manipulated」（这则投稿看起来被高度操纵投票）；`verdverm` 主张「humans remain responsible, agents don't go rogue」（人类仍然负责，Agent 不会自己失控）；`blooalien` 则把话说到「要么是软件缺陷，要么是使用/配置错误，总之责任在人」[W04]。厂商侧没有公开回应——这一点我明确写「未找到可靠公开来源」。

那这篇文章到底要讲什么？帖子里最值钱的一句不是那 78,000 美元，而是这句（逐字）：「There was no equivalent real-time control surface giving me a comprehensible picture of the spendings」——没有任何等价的实时控制面板，让我能看懂钱花到哪去了 [W01]。把这句话换成你自己的系统，就是三个问题：哪一层会先响？响得有多早？响的时候还来得及吗？

我在这篇文章里承诺三件事：第一，解释为什么「事后账单」在结构上不可能当刹车；第二，给一张运行时中断点地图和三层护栏，每一层都落到可查的机制，而不是「请注意成本」这种劝告；第三，给我自己那套流水线的、可核实的成本区间数据，并诚实说明——它目前也只是事后聚合。

## 二、事后账单为什么必然滞后：这是结构问题，不是纪律问题

很多人以为「没控住成本」是没上心。我不同意。事后账单的滞后不是疏忽造成的，它由三个结构性因素叠加出来（这是我的拆解，属作者观点）：

第一，计费与充值粒度。自动充值加事后发票，把「花了多少钱」拼成一张需要事后重建的图。帖主自述的重建结果是一批发票与 Credits/Automatic Reload 记录——注意，是「重建」，不是「实时可见」 [W01]。第二，聚合窗口。我自己的流水线里，成本报告是按任务时间窗跨多个 profile 的数据库聚合、再由定时任务刷新的，报告生成时间与任务结束时间之间天然存在一个间隙——这一点我会在第六节用实测数字坐实。第三，人类反应周期。告警进了邮箱，而人不在场；等你打开邮件，那个失控的循环已经又跑了很久。

更要命的是，成本是乘数，不是增量。一次重试、一次扇出、一次模型或推理等级的漂移，改变的不是「多花一点」，而是一个乘法因子。用「乘数」而不是「贵一点」来描述成本失控，是我想在这篇文章里教给读者的第一个思维动作。这也解释了为什么「这不是孤例、也不是我编的」：DN42 社区里一则一手记录写道，一个 AI agent「bankrupted its operator with a $6531.30 AWS bill」，运营者自述成因是同一 CloudFormation 模板被重复部署了多次 [W12]；TechCrunch 在 2026 年 6 月报道企业侧现象时写道，Uber「its entire 2026 AI coding budget by April」（到四月就花完了全年 AI coding 预算）[W13]。它们共同的形态不是「单价太贵」，而是「某个环节被乘了一个失控的系数」。

由此推出本文最重要的一条边界：可观测不等于可中断。能画出成本曲线，和能在超阈值的那一刻把 agent 停住，是两种能力；前者只是后者的必要条件。我本机的现状就是只有前者（第六节回扣）。本节的收口句是：账单只能用来复盘上一次决策，不能用来阻止这一次决策。

## 三、运行时中断点地图：四个能插桩的位置

要理解护栏装在哪，先把成本形成路径拆成四段（这个四段划分是我自己的方法论主张，不是行业规范），每一段只问一个问题：「能不能在这里刹住？」

| 插桩位置 | 能中断什么 | 中断的代价 | 典型误伤 |
| --- | --- | --- | --- |
| 准入 | 任务启动、模型与推理等级选择、并发/扇出上限 | 近乎为零，是全局开关 | 一刀切，挡掉合法的长任务 |
| 调用前 | 每一次模型请求、每一次工具调用 | 需要判断逻辑，可能拖慢热路径 | 误判「看起来失控」的正常行为 |
| 扇出 | 子任务、子代理、子代理树 | 需要计数与形状信息 | 合法的大规模并行被误拦 |
| 终止 | 循环检测、turn 上限、重试上限、超时 | 等到已经花了一部分钱才响 | 任务接近完成时被一刀砍断 |

这张表的核心权衡只有一句话：越早中断越省钱，但越早越难判断「是不是真的失控」。准入层最便宜也最钝——它不知道这条任务到底会不会失控，只知道要不要放它进场；调用前判断最精准，但它必须为每一次调用付出判断的开销，而且会误伤；形状检测只需看计数、额外开销很小，也最容易漏。所以任何单层都只能在「太钝」和「太慢」里二选一。

由这个权衡直接推出分层（表里前三段各对应一层，第四段「终止」是它们越界后共用的动作）：硬上限负责保命（钝，但一定生效）→ 调用前判断负责精准（准，但会误伤）→ 形状检测负责识别「这看起来像失控」（开销小，也最容易漏）。这三种失效模式对应的是三层护栏，不是三个备选方案——它们要一起装，因为每一层挡的失效模式不一样。

## 四、三层护栏：每一层挡什么、挡不住什么

### 第一层：预算硬上限（保命层）

这一层由两部分组成。第一半是账户/项目级的支出上限，它拦在最外面。OpenAI 的硬限额触达后会让受影响请求返回 429、错误码为 `organization_spend_limit_exceeded` / `project_spend_limit_exceeded`，但官方文档写得很清楚：「Enforcement is not instantaneous」（执行不是即时的），在限额状态传播期间可能会处理少量额外用量 [W09]；而且默认的 monthly spend limit 只监控不强制 [W11]。Anthropic 侧则是按 tier 封顶（Start $500 / Build $1,000 / Scale $200,000），封顶后请求返回 429、错误码 `enforced_spend_limit_reached` [W10]。注意「月度」和「非即时」这两个词：它保的是命，不是精确性。

第二半是运行时累计断言——这是账户级上限的盲区补丁，也是我本机目前缺失的那一半。它的逻辑是在发出下一次调用之前，比对本任务/本会话已经消耗的量与阈值，越界即暂停并留痕。这种「运行时预算上限」并非空想，Claude Code 的 `--max-budget-usd` 就是一个现成实现：官方文档把它定义为「停止前最多花费的 API 调用金额（仅 print 模式）」，子代理的花费计入上限，触顶后再派生子代理会以 Budget limit reached 失败；但它有三个盲区——仅 print 模式生效、`--continue`/`--resume` 恢复时早前运行的累计值不计入、且上限强制行为要求 v2.1.217 或更高版本 [W08]。这一层挡得住总量失控，挡不住三样东西：单次巨贵的调用、跨账号/跨卡绕过、以及上下文长度爆炸导致的单位成本畸高。「上限设在哪」本身也是一个选择：设在平台侧（API 账户）比设在信用卡侧更早、也更防跨卡。

### 第二层：工具与权限门（精准层）

把工具调用从「直接执行」改成「可拒绝/可询问」的决策点。Claude Code 的 `PreToolUse` 钩子在每次工具调用前触发，官方文档明确它「Can block it」，决策字段 `permissionDecision` 支持 `allow` / `deny` / `ask` / `defer`，其中 `"deny"` 会阻止这次工具调用 [W05]。下面这段是 hook 返回体的结构（摘自官方文档示例，非可运行脚本，字段语义以文档为准）：

```
// 伪代码：PreToolUse hook 的返回体结构（摘自官方文档示例）
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "deny",
    "permissionDecisionReason": "blocked by cost guard"
  }
}
```

另一个可查机制是 OpenAI Agents SDK 的 guardrail 与 tripwire，它有一条直接决定「门装在哪一层」的语义：`Blocking execution guarantees that the expensive model does not start; with parallel execution, the expensive model may already have started before the guardrail completes.`（阻塞式执行保证昂贵模型尚未启动；并行式执行时，昂贵模型可能在 guardrail 完成前已经启动）[W06]。而并行式是默认值。也就是说：护栏生效，不等于没花钱——这条我会在第六节用一次本机实测把它变成两个数字。

这一层挡得住明确危险的动作，挡不住完全合法、但被重复一万次的动作。所以还需要第三层。

### 第三层：形状检测（计数/形状层）

不看花了多少，看执行形状。三个可观测信号：子任务数与扇入扇出比、同一条初始消息被复制出的子任务数、模型或推理等级在过程中是否漂移。这一层有现成实现可以参考——我用的平台 Hermes 的官方文档里，`loop_caps` 对单次 agent loop 的 `max_subagents`（默认 50）与 `max_web_searches`（默认 50）设了硬上限，「达到上限时，本次违规的工具调用会被阻断并给出说明信息，本轮干净结束，而不是烧光剩余预算」[I26]；`agent.stall_guards` 则对「同一工具连续多次同参同结果」做循环检测：默认只追加一条提醒、不阻断调用，而无人值守（硬停开启）时同一串重复会直接熔断 [I27]。这一层挡得住扇出爆炸与「同一件事被做 N 遍」，挡不住聪明且看起来正常的持续消耗。

三层合起来是一张速查表：

| 层 | 落点 | 挡什么 | 挡不住 | 误伤风险 |
| --- | --- | --- | --- | --- |
| 预算硬上限 | 账户/项目 + 运行时累计断言 | 总量失控 | 单次巨贵、跨卡、单位成本畸高 | 低（钝，但会一刀切） |
| 工具与权限门 | 每次工具调用前 | 明确危险动作 | 合法但重复的动作 | 中（拖慢热路径） |
| 形状检测 | 扇出/重复/漂移 | 扇出爆炸、重复劳动 | 看似正常的持续消耗 | 中（漏判） |

我的立场一句话：护栏的目标不是让 Agent 更保守，而是让「失控」在还便宜的时候变成一个可中断的事件。

## 五、把那份「826 个子任务」的自述当技术材料读

这一节我从帖主自述里抽出对读者有用的东西，而不是复述情节。先说清楚身份：以下所有数字都是帖主自述，本文无法独立核实，也不做任何换算。

帖主自述的技术细节大致是（数字的口径以 [W01][W02] 为准，这里只做结构归纳）：一个根任务派出了 826 个子任务；其中一组子任务保留了同一条初始消息、且缺少角色/路径字段；在两个客户端构建版本下，子任务的平均本地 token 量相差约 8.5 倍；以及本地仍有约 2,550 条「有元数据但没有对应原始 rollout」的历史线程——帖主原话是「approximately 2,550 non-archived legacy threads still have metadata but no corresponding raw rollout available locally」[W01]。他自述提到的构建号 `0.144.0-alpha.4` 确实存在于 npm 镜像 registry，发布于 2026-07-09 [W14]；这套「子任务记录 + 角色/路径字段 + rollout + 本地 token 计数」的词汇，在本机安装的 codex 客户端状态里同样存在（`agent_role`、`agent_path`、`rollout` 这些字段名可查），也就是说「词汇与数据模型」是真的；但「数字与因果」只有他一个人的说法。

从中能抽出三条对读者有用的东西：

第一条，模型/推理等级漂移是一个可观测信号。父任务一个等级、子任务另一个等级，这种漂移在本地状态里是有字段可查的——它不该等到账单出来才被看见。第二条，「同一条初始消息被复制成 N 个子任务」是最容易被形状检测抓住的指纹，因为它对应的是「同一件事被做了 N 遍」，正是第三层护栏的目标。第三条最扎心：「记录已不可得」。帖主说日志似乎被自动删除了，大量历史线程只剩元数据没有 rollout [W01]。这说明事后取证这条路本身不可靠——你必须在运行时留痕，而不是事后找证据。

公平地说，反方意见不是没道理。`verdverm` 说「humans remain responsible, agents don't go rogue」，`blooalien` 说「要么是软件缺陷，要么是使用/配置错误，总之责任在人」[W04]。这些我都接受。我的落点不在「谁该负责」，而在另一句上：即使责任在人，机制上也应该有一条默认生效的刹车。HR 和流程上的「你应该」，不能替代代码里的断言——因为「你应该」在凌晨三点、在无人值守的时候，是不在场的。

## 六、本机实测：我自己的流水线，成本可核实（也确实滞后）

我有一套五段式的博客流水线（拆解 → 调研 → 写作 → 审校 → 发布），由多个 worker profile 协作完成。它有 4 次已完成运行的成本数据，每条都来自各自 run 目录下的成本报告文件（刷新于 2026-09-27T14:18）。先把精确数据放进代码块，避免和正文的叙述混在一起：

```
同一套 5 段流水线在 4 个不同选题 run 上的成本（刷新于 2026-09-27T14:18）
run                       墙钟     API 调用   成本 USD    实测 / 别名估算
20260916-kanban-swarm     28.7min  174        $0.1774    $0.0523 / $0.1251 (估算 71%)
20260916-langgraph-84652f 32.0min  144        $0.1977    $0.0649 / $0.1328 (估算 67%)
20260917-21-model-harness 56.9min  271        $0.3172    $0.0755 / $0.2417 (估算 76%)
20260917-rag-ai           59.1min  256        $0.2757    $0.0988 / $0.1770 (估算 64%)
```

读这张表之前，必须讲清三点口径，否则这些数字会被误用。第一，这是「不同选题上的重复运行」，不是同一次实验的重复采样——所以它只能当量级和分布看，不能当性能对比。第二，「实测」与「别名估算」必须分开说：单次成本大致在 0.18 到 0.32 美元之间，其中别名估算占六到八成，因为工具价目表里没有 worker 实际使用的模型别名，只能按同族价目近似——这两个数绝不能合并成一个「精确」数字。第三，墙钟是任务卡的窗口时间（含排队与模型延迟），不等于模型计算时间。

然后是自我批评，这是本文的诚实点，不能省：这套成本汇总工具是按任务时间窗跨 profile 的数据库聚合、由定时任务每两小时刷新一次的。它证明的是可观测，不是可中断——报告永远是刷新那一刻的快照，本 run 这份成本表就是：生成的时候，调研段那一行还标着零分钟，写作段及其后都写着「未跑」。也就是说，第四节第一层护栏的第二半（运行时累计断言）在本项目里目前还是空的，我只有「能看见」；要把它变成刹车，还缺四样东西：运行时累计而非窗口聚合；发出下一次调用前的断言点；扇出计数；以及越界时的动作是「暂停 + 留痕 + 通知」，而不只是「发一条消息」。

缺口的第二样，我用一个最小可运行的脚本补给你看。它演示「发出下一次调用之前做预算断言」这件事在工程上完全落得下来（示例单位，非真实金额）：

```python
# budget_guard.py —— 运行时预算断言的最小可运行示例
# 在每次"昂贵调用"之前检查：花过的钱 + 这一次 > 预算上限 就拒绝并退出
import sys

BUDGET = 5.0        # 预算上限（示例单位，非真实金额）
UNIT_COST = 0.2     # 每次调用的单价（示例单位）
MAX_STEPS = 40      # 计划总步数

spent = 0.0
for step in range(1, MAX_STEPS + 1):
    # round 掉浮点误差：4.8 + 0.2 在二进制浮点里会算成 5.000000000000001
    if round(spent + UNIT_COST, 9) > BUDGET:
        print(f"[deny] step={step} spent={spent:.2f} budget={BUDGET} "
              f"reason=budget_would_be_exceeded")
        sys.exit(3)                    # 非零退出码 = 越界信号，让上层感知
    # 真正的"调用"发生在这里（示例里用一次计数代替）
    spent += UNIT_COST
    print(f"[allow] step={step} spent={spent:.2f}")

print(f"[done] spent={spent:.2f} (no guard ever fired)")
```

它跑出来的核心结果是：第 26 步被拒绝、此前累计 25 步不超上限、以退出码 3 结束——这和「不装断言、40 步全跑完、超支到最后才发现」形成对照。我在本机隔离环境里做了三组对照实测，用的是真实 SDK 的调度语义、自写的最小 harness 与模拟单价，全程没有触碰生产看板、没有调用真实模型 API、没有产生真实计费：

第一组对照验证了第四节那句「护栏生效不等于没花钱」。用 OpenAI Agents SDK（本机 0.22.3）跑一个会触发 tripwire 的 guardrail：阻塞式执行下昂贵模型被调用 0 次；切换成默认的并行式后，同样触发 tripwire，昂贵模型已经被调用 1 次——官方文档那句话在代码里变成了两个数字 [W06]。第二组对照就是上面的预算断言：事后聚合模式 40 步全执行、超支只能最后发现；调用前断言模式在第 26 步前中断、累计不超上限。第三组在断言之上叠加形状检测（同一条初始消息被复制），结果 29 次工具调用被拒，工具副作用只发生 11 次——因为拒绝发生在调用之前，副作用计数不增加。

这些实测的原始输出都落盘在本机 run 的 evidence 目录里。它们证明的是「这种断言在工程上能落地并留下可读证据」，不是「某产品已经这么干了」——这两句话的区别，就是本文和软文的区别。

## 七、落地清单与收口

照做顺序，从最钝但一定生效的开始：

1. 先给账户/项目上支出硬上限（平台侧，不是信用卡侧）[W09][W11]；
2. 再把危险工具从「直接执行」改成「询问/拒绝」，用上 `PreToolUse` 这类调用前钩子 [W05]；
3. 然后加形状检测与越界动作（暂停 + 留痕 + 通知），可参考现成的 `loop_caps` 与 stall guard 语义 [I26][I27]；
4. 最后才是精细的运行时预算断言，也就是第六节那个脚本。

原则一句话：先装保命的，再装精准的。

反面清单，五条：只靠邮件告警就当上了护栏；把 token 数当账单；把并发当免费；把上限设在信用卡而不是 API 账户；在生产环境上测试刹车。

最后是一张「症状 → 该看哪一层 → 立刻能做的动作」的小表：

| 症状 | 该看哪一层 | 立刻能做的动作 |
| --- | --- | --- |
| 长任务突然变慢、变贵 | 调用前 / 终止 | 上运行时累计断言，加 turn 与重试上限 |
| 子任务数爆炸 | 扇出 | 加子任务/子代理计数上限 |
| 同一消息被重复执行 | 形状 | 加重复指纹检测，重复即拒 |
| 模型/推理等级漂移 | 准入 / 形状 | 锁定等级，漂移即告警或阻断 |
| 上下文异常增长 | 调用前 | 监控单次请求的 token 规模 |

收口一句：成本不是一个月底的数字，而是一条应该在过程中被监控、并且能被打断的信号；账单只告诉你买过什么，刹车才决定你还能买什么。

## Sources

- [W01] HN 自述帖正文逐字快照（帖主 lorenzomassaro，2026-09-27 抓取） — https://news.ycombinator.com/item?id=49861047
- [W02] Hacker News 官方 API：item 49861047 原始响应 — https://hacker-news.firebaseio.com/v0/item/49861047.json
- [W03] HN 帖子页面端渲染快照（[flagged] / 61 points / 25 comments） — https://news.ycombinator.com/item?id=49861047
- [W04] HN item 49861047 评论树逐条快照（含反方质疑） — https://news.ycombinator.com/item?id=49861047
- [W05] Claude Code 官方文档：Hooks reference（PreToolUse 与 decision 语义） — https://code.claude.com/docs/en/hooks
- [W06] OpenAI Agents SDK 官方文档：Guardrails（阻塞式与并行式的执行差异） — https://openai.github.io/openai-agents-python/guardrails/
- [W08] Claude Code 官方文档：CLI reference（--max-budget-usd） — https://code.claude.com/docs/en/cli-reference
- [W09] OpenAI API 官方文档：Spend limits（硬限额的语义与滞后） — https://developers.openai.com/api/docs/guides/spend-limits
- [W10] Anthropic 官方文档：Rate limits / Spend limits（tier 封顶与错误码） — https://platform.claude.com/docs/en/api/rate-limits
- [W11] OpenAI Help Center：Managing projects in the API platform — https://help.openai.com/en/articles/9186755-managing-projects-in-the-api-platform
- [W12] Lan Tian 博客：AI Agent Bankrupted Their Operator While Trying to Scan DN42 — https://lantian.pub/en/article/fun/ai-agent-bankrupted-their-operator-scan-dn42lantian.lantian/
- [W13] TechCrunch：The token bill comes due（2026-06-05） — https://techcrunch.com/2026/06/05/the-token-bill-comes-due-inside-the-industry-scramble-to-manage-ais-runaway-costs/
- [W14] npm 镜像（registry.npmmirror.com）@openai/codex 版本清单 — https://registry.npmmirror.com/@openai/codex
- [I26] Hermes 官方文档：Tool-Loop Guardrails > Per-turn runaway-loop caps（loop_caps 达上限即阻断该次工具调用）
- [I27] Hermes 官方文档：Tool-Loop Guardrails > Runtime anti-stall guards（identical_call_streak_halt）
