---
title: "同一个模型换个壳，分数能差多少？——21 组 model × harness 实测该怎么读"
date: 2026-09-17
tags: [harness, ai-agent, coding-agent, evaluation]
author: 彭梓坚
description: "拆一份 21 组 model × harness 对照研究的读法：harness 对成功率的影响常落在噪声里，对成本的影响却实打实；附一次同任务换壳小自测，讲清换壳前要锁死哪些变量。"
---

你大概经历过这个时刻：刷到一条帖子说「同一个模型，换到 XX 工具分数立刻上去了」，或者「某某 CLI 比另一个贵一倍，别用了」。你信了一半，然后去试，结果发现——分数没怎么变，账单倒是真变了。

这感觉没错。2026 年 9 月，一份挂在 harnesstax.github.io 的研究（本机抓取于 2026-09-17）把 7 个模型 × 3 个壳（Claude Code、Codex CLI、Pi）配成 21 组，在 SWE-bench Lite 和 Terminal-Bench 2.0 两个开源 benchmark 上做了对照 [W01]。它给出三个值得先记下来的结论：

- 成功率：同一模型换壳，成功率差异很小——原文称「平均 harness 效应」落在 SWE-bench Lite 约 ±2%、Terminal-Bench 2.0 约 ±5% 以内 [W05]。
- 成本：差距是实打实的。跨共享模型，Claude Code 的成本约为 Pi 的 2.0 倍、约为 Codex 的 1.6 倍（SWE-bench Lite），约为 Pi 的 1.5 倍（Terminal-Bench 2.0），比值口径是「成本比的几何均值（geometric means of cost ratios）」[W05]。
- 配对：一个只提供四个工具的极简壳 Pi 进了两个 benchmark 的 Pareto 前沿 [W10]；12 组「Anthropic/OpenAI 模型 × 两个 benchmark」的对照里，有 9 组是「别家壳」拿到了该模型的最高成功率 [W07]。

但在你把它转发成「换壳没用」之前，先把它的适用范围读进去：这是两个开源 benchmark，每个只随机抽了 30 个任务（SWE-bench Lite 全量是 300 个，抽样 10% [W20]），每任务跑 3 次、每次上限 100 个 agent turn，用各 benchmark 官方评测器判分 [W02]；作者自己写明，这两个 benchmark 是开源的，模型可能在训练时见过它们 [W08]。

换句话说：这份研究告诉你「在这个试验台上如此」，而不是「放之四海皆准」。读它的正确姿势，是先回答五个问题——benchmark 是什么、样本和重复次数是多少、成本怎么算、差值是否落在置信区间内、这个任务分布跟我的活儿像不像。这篇文章就按这五问展开，最后我会用自己本机跑的一次同任务换壳小对照，示范怎么把「换壳」从传闻变成自己项目里可复现的取舍。

顺带交代一处命名，避免无意的洗稿：本文标题里的「换壳」指的就是 harness；而「harness tax」这个词本身并非这份研究首创——它早于这份研究，出现在 Portkey 2026 年 4 月的一篇博文里，其定义是「agent 在为你做任务之前，花在它自己身上的每一个 token」[W19]；这份研究在提出该概念处也明确引用了它 [W09]。本文解读的对象，是 2026 年 9 月这份 21 组对照研究。

## 1. harness 是什么——模型之外的那一层

先定义词。harness 在这里不是框架（framework）的同义词，而是一个有出处的术语：源研究把它定义为「a software system that manages a model's tools, context, and task execution」——一套管理模型的工具、上下文与任务执行的软件系统 [W22]。这不是本文自创的词：Anthropic 官方文档把 Claude Code 称作「围绕 Claude 的 agentic harness」，说它提供工具、上下文管理和执行环境 [W14]；Anthropic 工程博客和 OpenAI 的 Codex agent loop 文章也用同一个词指同一层东西 [W11][W12]。

为什么这个定义重要？因为一旦你承认「我用的模型 X」其实是「某个壳里的模型 X」，后面所有对照才有意义。壳之间至少有五处差异，其中只有一部分体现在分数上，大部分体现在账单上：

- 工具集规模：Pi 默认只给 `read` / `write` / `edit` / `bash` 四个工具（可用 skills、extensions、packages 等扩展）[W10][W13]；Claude Code 的内置工具分五类（File operations / Search / Execution / Web / Code intelligence）[W14]。
- 首轮上下文：7 个模型平均下来，Claude Code 的首轮上下文是 Pi 的 10 倍以上（instructions 更长、tool schema 更大）[W06]。
- turn 的定义与上限：各壳对「一轮」的定义不同，effort 设置也按各壳自己的口径 [W02]。
- 权限与沙箱策略、是否记忆或压缩上下文：这些直接影响稳定性，但很少直接进分数。

由此得出本文的核心判断：换壳首先换的是成本与稳定性，其次才是分数。这不是让壳「背锅」——更复杂的壳在更难、更开放的任务上可能确实值那个钱，只是源研究对此只给了让步式的表述，没有下结论 [W06]。这一点留到第 3 节再拆。

## 2. 那份 21 组对照是怎么做出来的

一份评测能不能信，先看它怎么控制变量。这份研究的实验台是这样搭的：

7 个模型 × 3 个壳 = 21 组对照 [W01]。每个 benchmark 用同一批随机抽出的 30 个任务，每组「模型 × 壳」在每个任务上跑 3 次，起点是各壳的原生配置并选 high effort 设置，每次尝试上限 100 个 agent turn，最后用 benchmark 官方评测器判分 [W02]。其中 Terminal-Bench 2.0 有正式论文 [W21]。

成本口径是全篇可信的关键。研究算的是 token 成本，且用 2026 年 9 月 1 日的一张固定 direct-API 价目表换算，同一模型在所有壳上套用同一份价格 [W03]。为什么要这样？因为如果不固定价格，「壳更贵」完全可能只是「这个壳默认调用了更贵的模型」——同一价目表把模型的价格变量消掉，剩下的差值才能归给壳。

统计口径同样值得读：先对每个任务内的 3 次取平均，再跨 30 个任务取平均；95% 置信区间用 10,000 次 bootstrap 重采样估计，每次有放回抽 30 个任务均值 [W03]。控制变量方面，SWE-bench Lite 上屏蔽了任务容器的外网、关掉 Claude Code 和 Codex 的默认联网工具、并在 API 请求层拒绝托管工具声明；给 Pi 额外加了两个包，Kimi K3 统一走 Fireworks AI 的单一 thinking 模式 [W04]。

这几句之所以重要，是因为它回答了一个更本质的问题：这份评测把「壳」和「模型」做了分离。多数人复现不出同样结论，恰恰是忽略了这一层控制。

## 3. 三个发现，逐条拆开看

发现一：成本敏感，分数不敏感。拿其中一个模型 Claude Fable 5 举例：它在 SWE-bench Lite 上，Claude Code 解出 97.8%，Codex 和 Pi 都是 96.7%，但 Claude Code 的成本约是 Pi 的两倍（$1.33 vs $0.67）[W05]。原文把这概括为「平均 harness 效应」落在 SWE-bench Lite 约 ±2%、Terminal-Bench 2.0 约 ±5% 以内 [W05]。

这里必须停下来较一次真：原文没有给出「平均 harness 效应」这个统计量的计算公式——正文、图表定义、数据集里都没有。我穷举了 8 种自然口径用图表原值复算，没有一种能同时对上 ±2% 和 ±5% 两个数 [W05]。作为替代，我用一个定义明确的口径自算：每模型跨壳极差的一半，平均之后，SWE-bench Lite 是 2.22 个百分点、Terminal-Bench 2.0 是 3.81 个百分点——这是本文自算，不是原作者口径 [W05]。所以，±2% 只能按原句转述，不能替作者补定义。

发现二：极简壳能打。Pi 只提供四个工具就进了两个 benchmark 的 Pareto 前沿 [W10]。它的钱省在哪？Fable 5 在 SWE-bench Lite 上，Pi 平均 15.4 turn/次，Claude Code 平均 15.3 turn/次，几乎一样，但 Claude Code 贵约一倍、成功率只高 1.1% [W06]。差在单位 turn 的花费，而根子在上游的首轮上下文：Claude Code 的首轮上下文是 Pi 的 10 倍以上 [W06]。据该研究 Figure 3 的图表数据，三个壳首轮上下文 tokens 的均值如下：

| 壳 | 版本 | 工具数 | 首轮上下文 tokens（mean） |
|---|---|---|---|
| Pi | 0.85.1 | 4.0 | 1,972 |
| Codex | 0.146.0 | 7.43 | 11,308 |
| Claude Code | 2.1.224 | 23.0 | 27,011 |

（数据来源：该研究公开的图表 JSON，n=630 [W06]。）源研究同时保留了一句让步：更丰富的壳体功能在别的模型、工作负载或交互场景下仍可能有收益，壳体复杂度应当被当作经验性的取舍 [W06]。

发现三：模型不必待在自家壳里。6 个 Anthropic/OpenAI 模型 × 两个 benchmark 的 12 组对照里，有 9 组是别家壳拿到最高成功率 [W07]。例如 Sonnet 4.6 在 SWE-bench Lite 上 Codex 68.9% vs Claude Code 66.7%（成本相近）[W07]。但这句话要加一个很容易被忽略的限定：9/12 是「最高观测值」的计数，不是「显著更好」的计数——据该研究公开的图表数据，SWE-bench Lite 的 14 对配对检验里只有 1 对的成功率差异通过了 Holm 校正，而成本差异有 8 对通过 [W05]。也就是说，作者自己的数据里，成功率差多半不显著、成本差显著，只是这一点没写进正文。

三个发现合起来只说一件事：默认配对是需要被检验的默认值，而不是「壳不重要」。把这份研究读成「随便用哪个壳都一样」，是过度解读。

## 4. 这份研究不能证明什么

这一节可能是最该被记住的。

其一，训练污染。两个 benchmark 都是开源的，模型可能见过，作者在结尾自己承认了这一点 [W08]。看到任何公开 benchmark 数字，先问这一句。

其二，成本 ≠ 你掏的钱。研究算的是按固定价目表换算的 token 成本 [W03]，不是你的账单。我在本机实测时撞见一个很直观的例证：把 Claude Code 指向一个跑在本地、完全免费的 qwen3:8b 模型，它的结果 JSON 里依然给出 total_cost_usd = 0.046417（其中 qwen3:8b 占 0.04176）——这是按模型名查价目表算出来的，不是真实扣费。评测里的「成本」是标价换算，不是账单。

其三，跨壳的 turn 不是同一个东西。原文自己写明 turn 数和 effort 设置按各壳自己的定义 [W02]，发现二里又强调一次「turn definitions vary across harnesses」[W06]。所以「turn 数相近」只能在同壳内比，跨壳相减没有意义。

其四，差值是否显著。SWE-bench Lite 上 95% 置信区间的平均半宽是 6.91 个百分点（据该研究公开的图表数据，本机复算）——比作者用来描述「差异很小」的 ±2% 宽 3 倍以上 [W03][W05]。小样本下，±2% 的差很容易被区间吃掉，所以「谁第一」要带区间看。最后再加一条边界：这些结论只在两个 coding benchmark 上成立，换到长会话、多文件重构、需要人类反馈的活，没有证据 [W08]。

## 5. 自己动手：一次同任务换壳的小对照

读十遍不如自己跑一次。下面是「换壳对照」最该先定下的口径，顺序不能反：变量只有壳，模型、任务、判定方式全部固定——一次只改一个变量，否则涨了跌了都不知道是谁的功劳。其中「锁同一个模型」尤其要强调：不同壳默认用的是不同模型，不锁就等于一次改了两个变量。

先交代本机环境：codex-cli 0.149.1、claude 2.0.22、ollama 0.34.1，opencode 未安装（本机实测）。两壳在官方能力上都支持改端点——Codex 有自定义 model provider 的 base_url [W16]，Claude Code 有 ANTHROPIC_BASE_URL [W17]；但 Codex 的 provider ID 里 openai / ollama / lmstudio 是保留的，自定义 provider 不能复用 [W23]。本机的 ollama 只有 OpenAI 兼容端点，没有 Anthropic Messages 端点（本机实测：`POST /v1/chat/completions` 返回 200，`POST /v1/messages` 无响应）。所以「锁同一模型」在个人环境里，等价于自己维护一个双协议端点——我写了一个 Anthropic↔OpenAI 翻译代理，让 Claude Code 经代理访问 ollama，Codex 直接走 ollama（本机实测）。

任务用一个带客观判定的小件：`calc.py` 里植入两个缺陷，`test_calc.py` 里放了五条断言，判定只看 pytest 退出码（0 = 通过），不靠肉眼打分。判定脚本可以简单成这样：

```python
import subprocess
import sys

def judge(task_dir: str) -> str:
    r = subprocess.run(
        [sys.executable, "-m", "pytest", "-q"],
        cwd=task_dir, capture_output=True, text=True,
    )
    return "PASS" if r.returncode == 0 else "FAIL"
```

每次运行前从 `task1/` 复制一份字节一致的干净副本，两个壳拿到同一份代码、同一句逐字相同的 prompt（共用下面这个 `PROMPT` 变量），模型都用 qwen3:8b。两条非交互命令分别长这样：

```bash
PROMPT="The tests in this directory fail. Fix calc.py so that \`python -m pytest -q\` passes. Do not modify test_calc.py."

# Codex：直接走本地 ollama（OSS provider）
codex exec --oss --local-provider ollama -m qwen3:8b \
  -s workspace-write --skip-git-repo-check --json \
  -C <任务副本目录> "$PROMPT"

# Claude Code：ANTHROPIC_BASE_URL 指向自写翻译代理，再由代理转到同一个 ollama
ANTHROPIC_BASE_URL=http://127.0.0.1:8787 ANTHROPIC_API_KEY=dummy-key-for-local-proxy \
claude -p "$PROMPT" --model qwen3:8b \
  --output-format json --permission-mode bypassPermissions
```

结果要诚实地说：两壳各跑 1 次，都没把任务做出来——pytest 基线 exit 1，跑完后 exit 仍是 1，`calc.py` 与原始文件 diff 为空，也就是说两个壳都没改代码（本机实测）。差异不在分数，在行为：Codex 在没读文件的情况下就开始「猜测代码长什么样」，最后输出了一段 pytest 使用指南；Claude Code 则是 `num_turns = 0`、一次工具调用都没发出，直接以 `error_during_execution` 收尾（本机实测）。壳搭好了，瓶颈在模型——qwen3:8b 把「修代码」理解成了「讲 pytest」。

每壳的样本只有 1 次，所以只报方向与数量级，不报任何百分点。这恰恰说明：个人自测只能用来发现量级差，不能用来排名；跨壳对照在个人环境里为什么难做，本身也是一条结论。另外有一个读数疑点必须标出来：两壳在这条路线上的单次请求 input tokens 都读到 2050，数值可疑地一致，我无法判断是巧合还是 ollama 的统计方式——因此本机路线不宜拿 token 读数做跨壳上下文对比，更别用它去验证「首轮上下文差 10×」（本机实测）。

## 6. 决策清单与收口

把前面的东西压成一张可以照做的清单：

1. 先固定你真正的任务集和判定方式（有没有测试用例、有没有明确对错）。
2. 再用同一个模型在两个壳上各跑一遍，只看成本和稳定性有没有量级差。
3. 分数差异小于噪声，就按成本与体验选；只有难任务、开放式探索、需要结构化流程时，才考虑为更重的壳付钱。

反面清单四条：把某壳的领先当成普遍规律；把 token 成本当成自己的账单；拿不同壳的 turn 数直接相减；只看成功率不看成本——据该研究公开的图表数据，最贵的那组并不是分数最高的那组 [W05]。

最后收口：模型决定上限，壳决定你为这个上限付多少、以及这个上限在你的任务上有多少能兑现。选壳不是选潮牌，是做一次能复现的对照。

## Sources

- [W01] HarnessTax: How Much Does the Harness Matter for Coding Agents? — 实验台与规模 — https://harnesstax.github.io/
- [W02] HarnessTax — 测量口径（30 任务×3 次、100 turn、bootstrap、固定价目表） — https://harnesstax.github.io/
- [W03] HarnessTax — 统计与成本换算口径（bootstrap 与固定价目表） — https://harnesstax.github.io/
- [W04] HarnessTax — 控制变量（屏蔽外网、关默认联网工具、拒绝托管工具声明） — https://harnesstax.github.io/
- [W05] HarnessTax — 发现一：成本比值与 ±2%/±5% 原句 — https://harnesstax.github.io/
- [W06] HarnessTax — 首轮上下文 >10× 原句 + Figure 3 图表数据 — https://harnesstax.github.io/
- [W07] HarnessTax — 发现三：12 组中 9 组别家壳成功率更高 — https://harnesstax.github.io/
- [W08] HarnessTax — 作者自述适用范围（训练污染与两个 benchmark 的边界） — https://harnesstax.github.io/
- [W09] HarnessTax — 参考文献 [10] 指向 Portkey 的 "The Harness Tax" 一文 — https://harnesstax.github.io/
- [W10] HarnessTax — 发现二：Pi 四工具与 Pareto 前沿 — https://harnesstax.github.io/
- [W11] Effective harnesses for long-running agents (Anthropic Engineering, Justin Young, 2025-11-26) — https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents
- [W12] Unrolling the Codex agent loop (OpenAI Engineering, Michael Bolin, 2026-01-23) — https://openai.com/index/unrolling-the-codex-agent-loop/
- [W13] Pi coding agent README (earendil-works/pi, packages/coding-agent/README.md) — https://github.com/earendil-works/pi/blob/main/packages/coding-agent/README.md
- [W14] How Claude Code works — 官方对 "agentic harness" 的定义 — https://code.claude.com/docs/en/how-claude-code-works
- [W16] Codex Advanced Configuration — Custom model providers（base_url / wire API / env_key） — https://developers.openai.com/codex/config-advanced
- [W17] Claude Code Environment variables — ANTHROPIC_BASE_URL / ANTHROPIC_API_KEY — https://code.claude.com/docs/en/env-vars
- [W19] Portkey — "Harness Tax" 一词的定义句（早于 HarnessTax 研究） — https://portkey.ai/blog/the-harness-tax/
- [W20] SWE-bench Lite 官方页面（300 任务子集） — https://www.swebench.com/lite
- [W21] Terminal-Bench: Benchmarking Agents on Hard, Realistic Tasks in Command Line Interfaces (arXiv:2601.11868, 2026-01-17) — https://arxiv.org/abs/2601.11868
- [W22] HarnessTax — 本文使用的 harness 定义原句 — https://harnesstax.github.io/
- [W23] Codex 文档 — 保留 provider ID（openai / ollama / lmstudio） — https://developers.openai.com/codex/config-advanced
