---
layout: post
title: "Claude Code API 模式的三笔隐藏开销"
date: 2026-11-02
excerpt: "Claude Code 的 API 账单里还有不进 transcript 的后台调用。本文拆解权限分类器、subagent 进度摘要和 recap 的成本，并给出三种省钱办法。"
tags: [llm, programming]
comments: true
---


* toc
{:toc}

用 API 计费跑 Claude Code，账单一般被理解为调用主模型的 token 费用：主线程和 subagent 每轮对话各算一次，transcript 里记多少就付多少。我对了一天账，发现这个理解漏掉了三块，都不进 transcript。第一块是权限分类器的调用，可以换成很便宜的模型。第二块是后台 subagent 的进度摘要，这一天它吃掉了 Opus 账单的 16%–20%，而 Claude Code 没有显式提供关闭它的设置。第三块是对话的 recap，金额不大，但它同样用主模型读一遍整个上下文。下面先把这一天的账拆清楚，再逐项说怎么省。

## 案例：一天的账单拆到每一笔

样本取自 11 月 1 日 10:00 到次日 02:00，共 16 小时，是一天典型的多 agent 编码工作：2 个主会话、9 个后台 subagent，共 1036 次 Opus 5.5 调用，其中 899 次来自 subagent。主模型是 Opus 5.5，权限 auto mode 打开，Sonnet 槽位已经接到 DeepSeek V4.1 Flash。API 计费是官方美元价乘 7，下文的人民币数字都按这个口径计算。

当天账单 ¥1041。用本地 transcript 按 `message.id` 去重求和，Opus 只有 ¥811，比账单上的 Opus 少 ¥218。这 ¥218 能在每个会话的 `cost-state` 记录里找到：这是 Claude Code 自己的累计记账，它比 transcript 多出来的输入、缓存读取、输出，与账单差额吻合。所以账单没有多算，确实有一批调用由 Claude Code 发出，却没有写进 transcript。这批调用的特征很统一：缓存读取约 1.07 亿 token，输出只有 4.5 万，即把上下文几乎原样读一遍，只回很短的一句话。

按调用来源把整张账单拆开：

| 来源 | 模型 | 费用 | 是否进 transcript |
|---|---|---|---|
| 主线程对话 | Opus 5.5 | ¥248 | 是 |
| subagent 对话 | Opus 5.5 | ¥563 | 是 |
| 后台 subagent 进度摘要 | Opus 5.5 | ¥170–210 | 否 |
| 自动压缩（4 次） | Opus 5.5 | ¥7–46 | 否 |
| recap（7 次） | Opus 5.5 | ¥4 | 否 |
| auto mode 权限分类器 | DeepSeek Flash（Sonnet 槽位） | ¥4.2 | 否 |
| 后台会话状态分类器等小任务 | Haiku 4.5 | ¥7.8 | 否 |

隐藏开销中进度摘要和压缩共用 ¥218 的余量，压缩时是否命中缓存决定了两者之间怎么分，所以都写成区间。下面省钱三招分别对应表里的分类器、进度摘要和 recap。

## 第一招：auto 权限判别器换 DeepSeek Flash

auto mode 下，Claude Code 每次执行有风险的工具调用（主要是 Bash）前，会先请一个分类器判断能不能放行。这个分类器用的是 Sonnet 槽位，也就是环境变量 `ANTHROPIC_DEFAULT_SONNET_MODEL` 指向的模型。

当天分类器的用量：未命中缓存的输入 128 万，命中缓存 3650 万，输出 2 万。平均每次判断约 4 万输入、20 来个输出 token。表里的 ¥4.2 是换成 DeepSeek 之后的实际花费；如果保持默认的 Sonnet 5.5，按两家的官方价格算：

| 模型 | 输入（未命中） | 缓存命中 | 输出 | 当天分类器费用 |
|---|---|---|---|---|
| Claude Sonnet 5.5 | $2.00 / M（写缓存 $2.50） | $0.20 / M | $10 / M | 约 ¥70–75 |
| DeepSeek V4.1 Flash 高峰 | ¥2 / M | ¥0.04 / M | ¥8 / M | ¥4.2 |
| DeepSeek V4.1 Flash 空闲 | ¥1 / M | ¥0.02 / M | ¥4 / M | ¥2.1 |

相比默认配置，一天省 ¥66–73，约为全天账单的 6%–7%。Sonnet 一栏是估算，因为两家的分词器不同，缓存命中率也未必一样。

配置如下：

```bash
export ANTHROPIC_DEFAULT_SONNET_MODEL='DeepSeek-V4.1-Flash-1017[1m]'   # 名字以你的网关为准
export ANTHROPIC_DEFAULT_SONNET_MODEL_SUPPORTED_CAPABILITIES=effort,thinking,adaptive_thinking,interleaved_thinking
```

第二个变量声明这个模型支持哪些请求参数。Claude Code 认不出的模型名，在 `ANTHROPIC_BASE_URL` 这类第一方接入方式下，默认按支持 adaptive thinking 和 effort 处理。DeepSeek Flash 本身支持这些，所以对它来说这一行只是把默认值写明。替换成别的模型时要按它的实际能力填：不支持 adaptive thinking 却收到这个参数，网关会直接返回 400。

前提是网关在同一个 `ANTHROPIC_BASE_URL` 下同时提供 Claude 和 DeepSeek，因为 Claude Code 只认一个 base URL。另外要清楚这样换的代价：分类器是 auto mode 唯一的安全闸门，换成更弱的模型，误放行危险命令的概率会轻微上升。

Haiku 槽位（`ANTHROPIC_DEFAULT_HAIKU_MODEL`）也可以照此处理。Claude Code 把一批不需要强模型的小任务交给它：

- 给会话起标题。
- 给后台任务起两三个词的名字。
- 把一组工具调用概括成 "Searched in auth/" 这样的短标签。
- 后台会话的状态分类器。

用得最多的是最后一项。用 `claude --bg` 或 `/bg` 放到后台的会话，在每轮结束时都会把 transcript 末尾一段交给 Haiku，一轮进行中也大约每分钟交一次。Haiku 据此判断会话是在进行中、在等人回复，还是已经结束，结果显示在 agent 面板上。这些调用同样不进 transcript，连 `cost-state` 也不记。案例这一天它们共花了 ¥7.8：账单上有 Haiku 费用的 11 个小时，恰好就是那个后台会话有状态记录的 11 个小时。

## 第二招：关掉后台 subagent 的进度摘要

### 它在做什么

用 Agent 工具启动的后台 subagent，在 agent 面板里有一行状态文字，比如"正在跑第三轮测试"。这行字由模型实时生成。Claude Code 给每个运行中的后台 subagent 挂一个定时器，默认每 30 秒触发一次。触发时如果 subagent 的 transcript 有变化，就把它的完整上下文 fork 一份，追加一句"用几个词概括进度"，交给 subagent 自己的模型调用一次。在 Claude Code 二进制代码里，这些调用的标签是 `querySource: "agent_summary"`，并带有 `skipTranscript: true`，所以不进 transcript。

我的 subagent 上下文常在 20 万到 37 万 token 之间，模型是 Opus 5.5。每次摘要要从缓存读一遍几十万 token（约 ¥0.3），产出不到 10 个 token。代码里其实还有一个便宜的 `short` 模式：先把过程压成 2000 token 以内的笔记，再交给小模型。这个模式由 Anthropic 远程下发的 flag 控制，走第三方网关并设了 `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1` 时拿不到，只能用默认的 `full` 模式。

这一天它花了 ¥170–210，相当于在 subagent 对话本身 ¥563 的花费上又加了三成。按小时看，每小时多出来的缓存读取都稳定在 subagent 自身缓存读取的 0.5–0.8 倍。主线程最忙的那个小时几乎没有额外开销，而主线程完全空闲、只有 subagent 在跑的两个小时，额外开销照样不少，可见它跟着 subagent 走。

### 怎么关

Claude Code 没有关闭这个功能的设置。`CLAUDE_CODE_AGENT_PROGRESS_SUMMARIES` 这个环境变量只能打开它，设成 `0` 也算打开。启动后台 subagent 的代码里，决定是否挂定时器的一行去掉混淆后是：

```js
enableSummarization: SDK开关() || (coordinator模式 || fork开关) && !非交互模式
```

交互式 CLI 里 SDK 开关为假，coordinator 模式默认关闭，剩下的就是 fork 开关。它本来管另一个功能：`subagent_type: "fork"`，一种继承主对话全部上下文的 subagent。判断这个开关时，代码先读环境变量 `CLAUDE_CODE_FORK_SUBAGENT`，值为 `0/false/no/off` 时直接判定关闭，服务端 flag 不再起作用。摘要本身也是用 fork 机制实现的，两者大概因此共用了一个开关。

所以在 `~/.claude/settings.json` 里加一行即可：

```json
{ "env": { "CLAUDE_CODE_FORK_SUBAGENT": "0" } }
```

我用两个真实的交互式会话做了对照，都带 `--debug-file`。两个会话各起一个后台 subagent，依次执行 5 次 `sleep 15`。开关为 1 的会话出现 2 次 `[AgentSummary] Forking`，开关为 0 的会话 0 次，连定时器都没有创建。两边 subagent 的 transcript 都是 46 行，工作量一致。

### 这个办法很脆弱

这个开关是借来的，依赖的是混淆后代码里一个碰巧成立的条件，任何一次 Claude Code 升级都可能失效。我验证的是 2.1.310；GitHub issue #86879 和 #105981 里有人在 2.1.309 到 2.1.312 上报告了同样的结果。

关掉以后的代价：fork 类型的 subagent 不能用了，agent 面板里的状态行停在启动时的任务描述上。我只用全新上下文的 subagent，这些代价都无所谓。

每次升级后可以用同样的办法自查：`claude --debug-file /tmp/as.log` 起一个后台 subagent，跑完后执行 `grep -c "AgentSummary\] Forking" /tmp/as.log`，结果应为 0。另外建议升级到 2.1.311 以上，这个版本修复了摘要未命中缓存、按全价重发 transcript 的情况。

## 第三招：关掉 recap

离开终端一阵再回来，Claude Code 会在对话末尾显示一段灰色的 recap，用一两句话概括当前目标和下一步。生成方式和进度摘要相同：把主线程的完整上下文 fork 一份，追加一句"用户离开后回来了，用 40 词以内概括"，调用一次模型（`querySource: "away_summary"`，同样不进 transcript）。很多人以为这类杂活会交给 Haiku，其实用的是主模型。原因是复用主线程已有的缓存更便宜：几十万 token 走 Opus 的缓存读取，比交给 Haiku 从头读一遍还省。

这一天 recap 触发了 7 次，每次读约 33 万 token 的缓存、输出不到 100 个 token，单次约 ¥0.5，全天约 ¥4，在三笔里最小。费用随主线程上下文增长，上下文到 100 万 token 时单次约 ¥1.4；离开电脑越频繁，触发次数也越多。它的用处是帮人回忆刚才在做什么，而大多数时候往上翻一屏就能看到，所以我直接关掉了。这个功能有正式开关，在 `/config` 里关闭，或者在 `~/.claude/settings.json` 里写：

```json
{ "awaySummaryEnabled": false }
```

## 合计与常规招数

按这一天的工作量，三招合计每天省约 ¥245–285，约为当天账单的四分之一：分类器 ¥66–73（相对默认的 Sonnet 配置），进度摘要 ¥170–210，recap 约 ¥4。其中进度摘要那笔最大，也最容易被忽略：它不进 transcript，所以按 transcript 统计用量的工具都看不到它。只有拿账单和 `cost-state` 对照，才能发现它。

此外还有两条常规做法，原理都是同一件事：每次调用都要把整个上下文读一遍，费用约等于上下文长度乘以调用次数。一是把自动压缩的窗口设短，例如在 `/config` 里或用 `autoCompactWindow` 设到 30 万到 40 万 token，不要让对话一路涨到 100 万。压缩一次本身要花钱（表里 4 次压缩 ¥7–46），但之后每次调用读的上下文短了，长会话里很快能赚回来。二是让对话保持连续，避免 prompt cache 过期。API key 模式下缓存默认只保留 5 分钟，停顿超过这个时间，下一轮要按全价重新写入整个上下文，30 万 token 的上下文重新写入一次约 ¥8–10。
