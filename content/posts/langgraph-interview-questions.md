---
title: "LangGraph 项目面试：面试官真正会追问的 12 个问题与答题骨架"
date: 2026-09-16
description: "把简历里的 LangGraph 项目讲到面试官点头：checkpointer 存什么、循环怎么保证停、中断恢复怎么调用、写操作幂等放哪一层、评测口径怎么报。12 个追问逐题给出答题骨架与露馅说法，附 langgraph 1.2.11 实测。"
tags: [langgraph, agent, interview]
author: 彭梓坚
---

很多人在简历里写「基于 LangGraph 实现多智能体 Agent」，但真到了追问环节，第一轮就露底。露底的标志往往不是「没答上来」，而是「答的方向不对」——面试官问「checkpointer 存的是什么」，你答「存的是聊天记录」；问「你的路由函数为什么这么写」，你答「因为 LangGraph 就是这么要求的」。方向错了，比不会更致命，因为它暴露了你从没真正设计过这张图。

这篇文章不教你「LangGraph 是什么」，也不罗列框架优势。它做一件更具体的事：把「在简历里写了 LangGraph 项目」之后最可能被追问的 12 个问题摊开，每题讲清三件事——面试官在验什么、答题骨架长什么样、以及哪些说法一出口就露底。

题目的来源我先交代清楚：这 12 问来自我自己做 LangGraph 项目的经历、复盘时被问穿的经历，以及公开面经里反复出现的追问；它不是任何机构的题库，更不是行业统计结论。官方文档站上没有任何面试题页面，这一点我放在文末说明。有一点可以放心说：LangGraph 确实被写进过大陆 AI Agent 岗的 JD——腾讯混元大模型 Agent 开发工程师岗的 JD 原文点名了 LangGraph 等框架（该岗位原链接在抓取日已返回 404，我引用的是标注来源为腾讯官网的转载页，所以只讲「被点名」，不讲「当前在招」）。但「被 JD 点名」不等于「这 12 题必考」，两者要分开。

为了把结构讲清，全文用一个抽象出来的典型场景当载体——「面向套餐咨询与办理的有状态问答 Agent」：政策类问题走检索，价表查询与超套试算走结构化工具，办理动作要二次确认。这是我为讲清结构抽象出来的示例，不含任何真实业务数据、真实用户或评测数字。全文也不出现任何性能、准确率、成本的具体数字——原因在第五节。

## 1. 面试官问 LangGraph 时，到底在验什么

面试官对 LangGraph 的追问可以分成三层。

第一层验「你有没有把图讲成状态机」。状态 schema 里放什么、转移条件写在哪、副作用放在哪个节点——这是「我用过 API」和「我设计过状态机」的分界线。第二层验「你踩过运行时的坑吗」。怎么恢复、并发会不会打架、循环会不会失控、慢在哪一跳，这一层只靠背概念答不上来，必须真的跑过、撞过墙。第三层验「你有没有工程闭环」。评测怎么做、回归怎么守、观测看什么、上线后怎么兜底。多数候选人只准备了第一层，所以在第二、三层被问穿。

最典型的开局三连问是：「为什么不直接用 while 循环？」「为什么不用 LangChain 自带的 Agent？」「为什么不用厂商托管的 Agent？」注意，这三个问题的正确答案从来不是「LangGraph 更先进」，而是要落在三个可验证的工程理由上：显式控制流（下一步走哪，由你的图决定，而不是由模型自由决定）、能不能中断与恢复（人审批到一半停下来、进程崩了能续上）、能不能观测每一步（每个节点、每次模型调用都有痕迹）。把框架当能力介绍，是这一轮最常见的出局方式。

后面每一问都可以套同一个骨架，我把它固定成四步：边界（这个方案不适合什么，先划清能力圈）→ 机制（我的图具体怎么保证这件事）→ 证据（我怎么知道它成立，靠什么实验或日志）→ 代价（我为此付了什么，延迟、复杂度还是存储）。哪怕遇到没准备过的追问，套这个骨架也能答出结构，而不是当场编。

## 2. 状态与图结构——面试的第一道分水岭

第一问通常是：你的状态 schema 里放了什么？哪些字段是「覆盖」语义，哪些是「追加」语义？这一问背后验的是 reducer——同一个字段被多个节点、多轮写入时，是后写覆盖，还是按规则合并。答不出 reducer 的人，往往也没意识到「写错 reducer 会出现什么 bug」。

先用一段实测可跑的代码把两种语义钉死（这段代码在 langgraph 1.2.11 + Python 3.11 上实测跑通）：

```python
from operator import add
from typing import Annotated
from typing_extensions import TypedDict
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.graph import END, START, StateGraph

class State(TypedDict):
    query: str
    route: str
    history: Annotated[list, add]   # 追加：多轮写入用 operator.add 合并
    answer: str                      # 覆盖：后写覆盖先写

def classify(state: State):
    r = "policy" if "套餐" in state["query"] else "calc"
    return {"route": r, "history": [f"classify->{r}"]}

def retrieve(state: State):
    return {"answer": "policy-answer", "history": ["retrieve"]}

def calc(state: State):
    return {"answer": "calc-answer", "history": ["calc"]}

def route_fn(state: State):
    return state["route"]

builder = StateGraph(State)
builder.add_node("classify", classify)
builder.add_node("retrieve", retrieve)
builder.add_node("calc", calc)
builder.add_edge(START, "classify")
builder.add_conditional_edges("classify", route_fn, {"policy": "retrieve", "calc": "calc"})
builder.add_edge("retrieve", END)
builder.add_edge("calc", END)

graph = builder.compile(checkpointer=InMemorySaver())
config = {"configurable": {"thread_id": "t-1"}}
graph.invoke({"query": "流量套餐怎么选", "history": []}, config)
```

如果你在同一 thread 内再 invoke 一次（换一个价格类问题，让它走 calc 分支），就能看到两种语义最直观的差异：`history` 变成 `['classify->policy', 'retrieve', 'classify->calc', 'calc']`——追加，两轮的痕迹都在；而 `answer` 只剩最后一跳的值 `calc-answer`——覆盖，前一轮的 `policy-answer` 没了。把 `history` 写成裸字段是真实会发生的 bug：历史消息被最后一条覆盖，你以为是「多轮对话」，实际上模型只看到了最后一轮。反过来，把本应覆盖的字段误加 reducer，也会在循环里无限增长。能把这个症状讲出来，第一层追问就过了大半。

第二问是路由：路由函数应该是什么形状？答案是纯函数、只读状态、返回分支标识，分支标识与目标节点通过映射表对应。真正的隐患在于——不要把 LLM 的分类结果直接当路由 key。模型输出会漂移，一旦返回一个映射表里不存在的分支名，这一轮运行会直接崩在路由查表处（实测抛的是 `KeyError: 'unknown'`），而不是静静地走错分支。所以正确的做法是：让模型输出落进一个封闭的枚举，路由函数只做「枚举 → 节点」的查表，查不到就进兜底分支。

第三问是循环与终止：图是有环的，怎么保证一定会停？「如果模型一直要求再调一次工具呢？」这里有一个很反直觉的实测事实：默认的步数上限不是很多人以为的 25，而是 10007。实测一个无终止条件的自环，抛的是 `GraphRecursionError`，异常开头原文是 `Recursion limit of 10007 reached without hitting a stop condition.`；显式传 `recursion_limit: 5`，第 5 步就停。所以「靠默认上限兜底」在工程上太迟了。正规答法有三层：步数上限（`recursion_limit`）、显式终止节点（状态里放一个「已完成」标志，条件边判断到就进 END）、状态里的计数或预算字段（工具调了几次、额度还剩多少，超了就强制收口）。

第四问是子图，属于加分题：什么时候把一段逻辑包成子图？官方文档给了两种接法的选型表——父子共享状态键时，直接把编译好的子图当节点传给 `add_node`；不共享键时，在节点函数里手动 `subgraph.invoke(...)` 做显式映射。这条答不出不致命，答错很致命（比如声称「子图会自动隔离状态」）。

## 3. 持久化与恢复——checkpointer 到底是什么

这是全文最重要的一问，也是最容易被问穿的一问。一句话的标准答案：checkpointer 存的是图在每个 super-step 边界的状态快照，不是聊天记录，更不是向量库。强调「不是向量库」不是矫情——实测 SQLite checkpointer 落的两张表，列是 `thread_id / checkpoint_ns / checkpoint_id / parent_checkpoint_id / type / checkpoint(BLOB) / metadata(BLOB)` 和 `writes(... value(BLOB))`，没有任何向量列，也没有 embedding 字段。

这里顺带有一个真实的「陷阱题」素材：`langgraph-checkpoint-sqlite` 这个发行版的依赖里确实声明了 `sqlite-vec`。如果看到依赖就断言「checkpoint 是向量库」，就掉进坑里了。实测结论是：`sqlite_vec` 只出现在 `langgraph/store/sqlite/`（长期记忆那条路径），在 `langgraph/checkpoint/sqlite/` 的源码里出现 0 次。向量能力属于 store 和 cache，不属于 checkpoint。面试官若有意埋这个坑，多半是在验「你有没有真读过依赖、读过源码」。

第二问是 `thread_id` 的语义。官方原话可以直接背：选定的 `thread_id` 就是你的持久游标——复用它会恢复同一个 checkpoint，换新值就开一条全新的空 thread。checkpointer 用 `thread_id` 作主键，没有它就既不能存也不能恢复。所以「换一个 thread_id 去恢复」不会报错，但也恢复不了东西，你只会得到一个新的、空状态的中断。

第三问是档位：内存、文件、数据库怎么选？准确类名和模块路径要记准：`InMemorySaver` 在 `langgraph.checkpoint.memory`，`MemorySaver` 是它的同一对象别名（实测 `MemorySaver is InMemorySaver` 为 True）；`SqliteSaver` 在 `langgraph.checkpoint.sqlite`；`PostgresSaver` 在 `langgraph.checkpoint.postgres`。内存实现进程重启即丢，官方明确说只能用于开发。

两个「用过 vs 设计过」的分界线，都藏在建表细节里：`SqliteSaver.setup()` 的 docstring 说「自动调用、用户不应直接调用」，而 `PostgresSaver.setup()` 的 docstring 说「首次必须由用户直接调用」。SQLite 落 2 张表，Postgres 落 4 张表（多出的表之一是把 channel 值按版本拆出来的 `checkpoint_blobs`）。还有一个实测才能发现的坑：`AsyncPostgresSaver` 不能从包根 `langgraph.checkpoint.postgres` 导入，只能从 `langgraph.checkpoint.postgres.aio` 导入，写错直接 ImportError。

最后把两个概念一刀切开：checkpointer 是线程内的短期记忆（按 thread 存，用于对话延续、HITL、时间旅行、故障容错），store 是跨线程的长期记忆（存应用自定义的键值，独立于 thread，用于用户偏好和事实）。面试官最爱把这两个混着问，答混了直接掉档。

## 4. 人在环路（HITL）与写操作的幂等

人在环路的机制核心是 `interrupt()`。先说清楚它是什么：中断不是异常，是图的一种正常状态。当前版本的主推方式是函数式中断 `interrupt()`，写在节点内部任意位置，与编译期静态断点 `interrupt_before / interrupt_after` 不同——官方明确不推荐用静态断点做 HITL 工作流，只用于调试。

调用方看到中断的方式很具体：默认 `invoke()` 的返回值里多一个 `__interrupt__` 键；`get_state(config).next` 会显示「下一步待执行哪个节点」。恢复要满足两件事：同一个 `thread_id`，加上 `Command(resume=值)`，这个值会成为 `interrupt()` 调用的返回值。下面是实测可跑的最小版本：

```python
from typing_extensions import TypedDict
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.graph import END, START, StateGraph
from langgraph.types import Command, interrupt

class State(TypedDict):
    action: str
    approved: bool

def approval_node(state: State):
    approved = interrupt({"question": "确认开通该套餐？", "action": state.get("action")})
    return {"approved": bool(approved)}

b = StateGraph(State)
b.add_node("approval_node", approval_node)
b.add_edge(START, "approval_node")
b.add_edge("approval_node", END)
g = b.compile(checkpointer=InMemorySaver())
cfg = {"configurable": {"thread_id": "hitl-1"}}

r1 = g.invoke({"action": "open_plan_A"}, cfg)   # 停在这里，r1 里带 __interrupt__
r2 = g.invoke(Command(resume=True), cfg)         # 恢复，返回 {'action': ..., 'approved': True}
```

这段代码背后有两个必须讲清的点。第一，恢复时节点会从开头整体重跑，不是从 `interrupt()` 那行接着跑。实测把节点内 `interrupt()` 之前的副作用清空后恢复，那一轮副作用又被执行了一次。官方文档的措辞是：恢复时节点从调用 `interrupt()` 的那个节点的开头重新开始。工程含义一句话：有副作用的代码不能放在 `interrupt()` 之前。

第二，不给 checkpointer 会怎样？很多人以为会报错。实测：不挂 checkpointer 时 `interrupt()` 不报错，仍然返回 `__interrupt__`——图确实中断了，但没有线程可续。正确的表述是「中断能触发但无法恢复，checkpointer 是恢复能力的前提，不是中断触发的前提」。

这一节还有一道「事故题」：为什么有副作用的动作必须二次确认？要答成风险语言，而不是满分语言——「未确认就落单」「重复开通」「同一请求重放两次」这三类事故，分别靠什么机制拦住。而且要补齐四件事：确认发生在第几步、谁批准、超时或拒绝怎么走、审批记录落在哪。只说「我加了人工确认」就结束，等于没答。

幂等键放哪一层是更深的追问：模型给的参数不可信，幂等键必须由服务端从状态里取（比如从 thread_id 或业务侧生成），不能把模型吐出来的 id 当唯一键。工具本身应当被设计成「有副作用但可安全重试」：恢复重跑时，同一幂等键的第二次调用不会重复落单。

第三个追问是工具返回的数值如何进入上下文。核心原则：让模型复述金额是危险的，因为模型可能改写数字。正确做法是结构化结果直接用于下游计算，复述只做展示。价表查询工具返回一个结构体，试算工具消费这个结构体，最后展示层的数字来自工具结果而不是模型生成。

## 5. 评测口径——面试最爱追问、也最容易翻车的一段

这一节我不给任何具体指标数字，只给方法和话术，因为「你这个指标是怎么来的」这一问，恰恰是绝大多数候选人翻车的地方，而翻车的根因，是口径不清。

先看检索侧。很多项目说「我们用 Hit@K 和 MRR」。注意：Hit@K 这个叫法我没有找到权威定义来源——教材用的词是 Precision@k / Recall@k，Hit@K 是工程口语里的别名，本质上更接近 Recall@K 的一个特例：看正确的那条有没有出现在前 K 条里。所以不要声称它是标准术语。更关键的是适用条件：Precision@k 不需要你知道「相关问题总数」，代价是它被教材明确评为「最常用评测里最不稳定、且最不能直接平均」的一个——因为一个 query 的相关文档总数会强烈影响它。教材给的替代方案是 R-precision（先取「已知相关文档数」，再看 top 里的召回），它对相关文档数做了归一，跨 query 求平均才有意义。

样本量是更大的坑。小样本报一位小数就是自曝：比如 16 条样本里命中数差 1 条，就是 6.25 个百分点（这是算术，不是任何项目的成绩）。统计上有一条硬依据——Brown、Cai 与 DasGupta 2001 年的论文证明了标准 Wald 区间（比例 ± 1.96×标准误）在比例接近 0 或 1 时覆盖率「持续地差」，并明确建议小样本（n ≤ 40）用 Wilson 或 Jeffreys 区间。Anthropic 的评测统计研究（对应论文 *Adding Error Bars to Evals*）也建议报告 95% 置信区间并做功效分析。MRR 还有一个常被忽略的细节：无答案的问题不是被丢掉，而是以 0 计入平均，同时要单独报「没找到的条数」——TREC-8 的结果表里 MRR 和「# not found」就是并列两列。这是 TREC 一家的做法，我没有找到把它写成行业统一口径的来源，所以别把它说成「规矩就是这样」。

把这些串起来，标准答法是一句话模板：样本规模 → 跑几次（一次回归还是连跑多次取区间）→ 报告下界而不是最好值 → 增益要能解释是哪个环节带来的。

忠实度、幻觉怎么测？我的做法是规则校验而不是让另一个模型当裁判：把工具返回值与价表做成值池，反查回答里出现的数字、金额是否都在池内。这么做的理由是可复现、无裁判方差、失败可定位。注意我的措辞——这是「我选择规则校验的理由」，不是「业界共识」。我能找到的公开支撑是：RAGAS 官方文档把 Exact Match、字符串相似度这类非 LLM 指标单独列成一类；对 LLM 当裁判的偏差也有系统研究（位置偏差不是随机噪声，且在候选质量差距小时最严重）。但没有任何规范性文件说「不要用 LLM 当裁判」，所以别把话说满。

业务约束（不许虚假开通、不许未确认落单、不许重复开通）怎么证明？我的做法是写成可自动断言的回归用例，报告「断言数 + 违规数」而不是「准确率」。这一条我没有找到把它确立为行业标准口径的公开来源，它是软件回归测试的常规做法搬到 LLM 场景，所以要如实说成自己的做法。最后给一个可执行动作：把评测脚本、样本文件、输出日志三条路径记下来，面试官问「能给我看看吗」时拿不出来，等于没做。

## 6. 成本、延迟与可观测性——第三层追问的入口

一次用户请求会触发几次模型调用？这是被追问概率最高的一类结构题：路由判定一次、检索改写一次、生成一次，加上可能的工具循环。关键是能说清「哪一跳最贵、哪一跳最慢」，以及循环和重试如何把成本放大。

「线上用户说答得慢，你怎么定位到具体一跳」是这层追问的标准题。抓手是图级、节点级、模型调用级、工具级四层观测的差异：图级看整体，节点级看单节点耗时，模型调用级看 token 与延迟，工具级看入参与结果。日志里必须留住会话标识、节点名、耗时、工具名与入参摘要、是否命中缓存。LangGraph 在节点内能通过 config 里的 metadata 读到 `langgraph_step`、`langgraph_node`、`langgraph_triggers`、`langgraph_path`、`langgraph_checkpoint_ns` 这几个字段；事件流侧还有严格递增的 seq 用于排序（文档明确提醒：墙钟 timestamp 会漂移，不要靠它排序）。

缓存的落点要分清：路由判定、检索结果、静态政策问答可以缓存；一切写操作与带状态的确认绝对不能缓存。SaaS 依赖（模型服务、向量库）挂掉时的降级路径也要能说一条。最后，准备一个真实故障故事——这次请求为什么把额度烧穿、为什么同一个动作执行了两次、为什么循环没退出——面试官对故事的兴趣远大于概念。

## 7. 12 个高频追问速查表与自检清单

把第 2–6 节的 12 问压成一张速查表（开篇的三连问不在其中）：

| # | 问题 | 面试官在验什么 | 一句话骨架 | 别这么说 |
|---|---|---|---|---|
| 1 | 状态里放了什么？哪些覆盖、哪些追加？ | 是否理解 reducer | 覆盖字段 vs `Annotated[list, add]`，给出真实 bug | 「我就把消息存列表里」 |
| 2 | 路由函数为什么这么写？ | 控制流是否显式 | 纯函数 + 查表 + 兜底分支 | 把 LLM 输出直接当路由 key |
| 3 | 循环怎么保证停？ | 运行时兜底意识 | 步数上限 + 显式终止 + 预算字段 | 「默认会停」 |
| 4 | 子图怎么接？ | 状态隔离理解 | 共享键直接传，否则节点内 invoke | 「子图自动隔离状态」 |
| 5 | checkpointer 存什么？ | 是否被「向量库」说法带偏 | super-step 状态快照，非聊天/向量 | 「存对话记录 / 存向量」 |
| 6 | thread_id 是什么？ | 恢复前提是否清楚 | 持久游标，同 thread 才能恢复 | 「随便换个 id 也能续」 |
| 7 | 三档 checkpointer 怎么选？ | 是否真部署过 | 内存开发 / SQLite 本地 / Postgres 生产 | 分不清类名与包路径 |
| 8 | 中断恢复的调用顺序？ | HITL 机制是否踩过 | interrupt → `__interrupt__` → Command(resume) | 把静态断点当 HITL 主推 |
| 9 | 写操作怎么幂等？ | 事故防范意识 | 二次确认 + 幂等键服务端取 | 「我加了人工确认」就结束 |
| 10 | 数值怎么进上下文？ | 是否防数字幻觉 | 结构化结果直接用，复述仅展示 | 让模型复述金额 |
| 11 | 指标怎么算的？ | 口径是否诚实 | 样本规模 + 跑几次 + 下界 | 报一位小数的小样本 |
| 12 | 慢在哪一跳？ | 观测是否分层 | 四层观测 + 定位字段 | 「就是模型慢」 |

三个白板题准备法：第一，30 秒画出主流程——入口 → 路由 → 检索或工具 → 生成 → 审核确认 → 落库，并标出状态键与失败分支；第二，画一次「中断—恢复」的时序，标出 interrupt、`__interrupt__`、`Command(resume)`、节点重跑；第三，画一次「评测」的数据流——样本 → 跑批 → 指标 → 回归断言。

自检清单（照着勾）：我能说清 reducer 的追加/覆盖语义，并给出一个真实 bug；我能现场写出中断恢复的调用顺序；我能解释两种记忆（checkpointer 与 store）的存储差异；我能说出评测的样本量与报告口径；我能复述一次真实故障与修法；我项目里没有我没读过的代码——AI 生成的代码要逐行过一遍，否则别写进简历。

最后收口一句：LangGraph 考的不是框架熟练度，是你有没有把不确定性关进显式的状态机与可回放的证据里。能把这句背后的三个动作——显式控制流、可恢复的持久化、可断言的回放——都变成你自己的话，这一轮面试你就不是在「背框架」，而是在「讲设计」。

---

## 说明与参考

版本口径：全文 API 写法在 langgraph 1.2.11 + Python 3.11 上实测验证（含本文两段代码片段）；「checkpointer 无向量列」「默认 recursion_limit=10007」「恢复时节点整体重跑」「不配 checkpointer 时 interrupt() 不报错」等均来自实测输出，非文档转述。

关于「必考」：本文未使用「必考」二字，因为官方文档站无任何面试题页面（site:docs.langchain.com interview questions certification 检索 0 结果），而所有「必考 N 题」的说法均来自备考内容产出方的编辑判断，无公开抽样可复核。

主要来源（均为 2026-09-16 抓取）：

- LangGraph 官方文档：https://docs.langchain.com/oss/python/langgraph/graph-api
- LangGraph 官方文档（持久化）：https://docs.langchain.com/oss/python/langgraph/persistence
- LangGraph 官方文档（checkpointers）：https://docs.langchain.com/oss/python/langgraph/checkpointers
- LangGraph 官方文档（中断）：https://docs.langchain.com/oss/python/langgraph/interrupts
- LangGraph 官方文档（流式）：https://docs.langchain.com/oss/python/langgraph/streaming
- LangGraph PyPI：https://pypi.org/project/langgraph/
- Manning/Raghavan/Schütze《Introduction to Information Retrieval》第 8 章（Precision@k 与 R-precision）
- Brown, Cai & DasGupta (2001), "Interval Estimation for a Binomial Proportion", Statistical Science 16(2), DOI 10.1214/ss/1009213286
- Voorhees (1999), The TREC-8 Question Answering Track Report
- Anthropic 研究页：https://www.anthropic.com/research/statistical-approach-to-model-evals
- RAGAS 文档（指标分类页）：https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/
- Shi, Ma, Liang, Diao, Ma & Vosoughi, "Judging the Judges: A Systematic Study of Position Bias in LLM-as-a-Judge", arXiv:2406.07791（ACL-IJCNLP 2025）
- 腾讯混元 Agent 岗 JD（转载页，原链接已 404）：https://jobs.niuqizp.com/job-vm85LaLa5.html
