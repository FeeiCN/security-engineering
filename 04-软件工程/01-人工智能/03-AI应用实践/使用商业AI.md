---
slug: /commercial-ai
icon: credit-card
description: 基于专业评测与官方价格比较商业 AI 的推理、写作、编程、Agent、速度、成本、生态和数据边界，并给出按任务选型方法。
last_reviewed: '2026-07-11'
---

# 使用商业 AI

商业 AI 选型需要同时比较产品、模型和 Agent。产品决定搜索、文件、办公套件与团队管理；模型决定单次推理的能力、速度和 API 成本；Agent 的脚手架决定它能否读写文件、执行命令、调用工具并完成长任务。

截至 2026 年 7 月 11 日，公开证据支持几个初步判断：

- **最高能力档**：Claude Fable 5 与 GPT-5.6 Sol 位于当前综合评测第一梯队。Fable 5 的综合分略高，GPT-5.6 Sol 的编码与 Agent 分项略高，且 API 单价更低。
- **写作与对话式编程**：Claude 系列在 LMArena 的创意写作和 Coding 人类偏好榜上占优。这个结果反映回答体验，不能替代真实代码库测试。
- **速度、多模态与 Google 生态**：Gemini 3.5 Flash 的输出速度、视觉理解和成本组合突出；Gemini 3.1 Pro 在科学代码与高难知识题上表现较强。
- **API 性价比**：GPT-5.6 Luna、GLM-5.2 和 Gemini 3.5 Flash 值得进入高吞吐任务的首轮测试。低单价只有在任务成功率和人工修正量可接受时才有意义。
- **编程 Agent**：Terminal-Bench 中 Codex CLI 与 Claude Code 的完整配置处于第一梯队，底层模型分数不能代替具体 Agent 产品测试。
- **办公与研发工作流**：核心资料位于 Google Workspace、Microsoft 365 或 GitHub 时，生态集成通常比模型榜单的几分差距更影响日常效率。

这些结论是带日期的公开资料快照。Fable、Sol、Terra 等名称在这里指 API 或评测配置，购买 ChatGPT、Claude、Gemini 订阅后不一定能精确选择相同模型和推理档位。模型、价格、配额和产品功能会变化，付款前仍要打开文中的官方链接复核。

## 先分清产品、模型和 Agent

| 购买对象 | 买到什么 | 需要单独核对什么 |
| --- | --- | --- |
| ChatGPT、Claude、Gemini 等订阅产品 | 对话、搜索、文件、研究、图片、项目空间和应用连接组成的现成工作台 | 月费、配额、可用模型、地区、导出、个人版数据规则 |
| OpenAI、Anthropic、Google 等 API | 可编程的模型调用能力 | 每百万 tokens 单价、缓存、长上下文、工具费、速率限制、数据保留 |
| Codex CLI、Claude Code、Gemini CLI、GitHub Copilot 等 Agent | 模型加上文件、终端、代码库和工具编排 | 使用的模型、推理档位、权限、脚手架、测试闭环和额外额度 |
| Team、Business、Enterprise 方案 | 账号生命周期、组织权限、共享资产、审计与合同 | SSO、SCIM、日志、保留期、数据驻留、分包商和退出迁移 |

订阅配额通常不能抵扣 API 费用。底层模型相同也不代表 Agent 产品表现相同，工具设计、上下文压缩、重试和验证流程都会改变结果。

实际选型按下面的顺序推进：

1. 先用服务地区、付款方式、数据分级、合同和合规要求排除不可采购的候选。
2. 再看资料所在的生态。Google Workspace、Microsoft 365、GitHub 或本地代码库会决定连接器与权限成本。
3. 按真实任务类型读取对应榜单，只保留两到三个候选，不用综合总榜替代专项证据。
4. 最后用相同输入、工具、预算和成功标准复测，计算成功任务的 API、重试与人工修正总成本。

## 专业评测怎么读

单一榜单只能回答一个窄问题。本文组合四类证据：

| 来源 | 主要测量 | 使用边界 |
| --- | --- | --- |
| [Artificial Analysis Intelligence Index v4.1](https://artificialanalysis.ai/methodology/intelligence-benchmarking) | Agent 34%、编码 24%、科学推理 24%、通用能力 18%；同时测价格、速度和延迟 | 英文、纯文本综合评测；不同模型的最大推理预算并不相同 |
| [LMArena Text Arena](https://arena.ai/leaderboard/text)、[排序实现](https://github.com/lmarena/arena-rank)与[技术报告](https://arxiv.org/abs/2403.04132) | 用户对匿名模型回答做成对投票，可看总体、创意写作、Coding 等分类 | 衡量人类偏好，容易受表达风格影响；分数不是正确率 |
| [SWE-bench](https://www.swebench.com/)、[SWE-Bench Pro](https://scale.com/research/swe_bench_pro) 与 [Terminal-Bench](https://www.tbench.ai/leaderboard/terminal-bench/2.1) | 代码补丁能否通过测试、终端任务能否真正完成 | 成绩同时受模型、推理档位、Agent 脚手架和重复次数影响 |
| [METR Time Horizon 1.1](https://metr.org/time-horizons/) | Agent 对不同人类工时任务的 50% 和 80% 成功时间跨度 | 主要是软件、机器学习和安全任务；不是 Agent 实际连续运行时长 |

Artificial Analysis 基于部分模型的重复实验，估计综合指数的 95% 置信区间小于 ±1%；它没有公开每个模型的具体区间，单项测试的不确定性也可能更大。本文读取 LMArena 默认开启 Style Control 时的分数；它会控制回答长度和部分 Markdown 特征，仍无法消除全部风格偏差。本文所说的“点估计”是榜单观测到的中心值，只有结合 `±` 范围或 95% 区间才能判断差异是否稳定。新模型也需要等待更多投票。

## 旗舰模型能力对比

下表来自 [Artificial Analysis 模型榜](https://artificialanalysis.ai/leaderboards/models)，取 2026 年 7 月 11 日可见结果并四舍五入到一位小数。综合、编码和 Agent 指数均为 0～100；表中保留了实际测试配置，避免把普通模式和最大推理模式混在一起。

| 模型与测试配置 | 综合 | 编码 / Agent | 据分项缩小的候选方向 |
| --- | ---: | ---: | --- |
| Claude Fable 5，最大推理并带 Opus 4.8 回退 | 59.9 | 76.5 / 52.8 | 复杂知识工作和高难推理；专家级跨学科难题 HLE 为 53.3% |
| GPT-5.6 Sol，max | 58.9 | 77.4 / 54.0 | 复杂推理、编码和多工具 Agent；表内编码与 Agent 分项最高 |
| Claude Opus 4.8，最大自适应推理 | 55.7 | 74.3 / 47.2 | 复杂写作、代码与知识工作，质量优先 |
| GPT-5.6 Terra，max | 55.0 | 76.7 / 47.4 | 接近旗舰能力，同时控制 API 成本 |
| Claude Sonnet 5，最大自适应推理 | 53.4 | 71.5 / 46.7 | 代码、知识工作与价格的平衡档 |
| GPT-5.6 Luna，max | 51.2 | 71.4 / 45.6 | 高吞吐 Agent；综合分接近更高价模型 |
| GLM-5.2，max | 51.1 | 68.8 / 43.1 | 价格敏感的 Agent 与代码任务；需另测中文和渠道差异 |
| Gemini 3.5 Flash，high | 50.2 | 70.1 / 37.4 | 高速、多模态和搜索增强任务；高难多模态 MMMU Pro 为 84.3% |
| Gemini 3.1 Pro Preview | 46.5 | 68.8 / 21.4 | 科学代码与知识推理；科学代码 SciCode 58.9%、研究生级问答 GPQA Diamond 94.1% |

前三列是 Artificial Analysis 的测量结果。最后一列是结合分项、产品能力和后文专项榜单形成的试测建议，不是该机构给出的官方用途排名。

**综合能力。** Fable 5 与 GPT-5.6 Sol 相差约 1 分，与 Artificial Analysis 对整套指数给出的总体不确定性估计接近。由于没有逐模型区间，不宜宣布稳定先后，可以把它们视为同一领先梯队。Fable 5 是带 Opus 回退的系统配置，单次请求价格也更高；GPT-5.6 Sol 在这组测试中的编码与 Agent 分项更高。

**写作体验。** 截至 2026 年 7 月 10 日，LMArena 总榜有 727 万次投票，Fable 5 为 1509±9；Claude Opus 4.6 Thinking 为 1504±4。创意写作分类超过 108 万次投票，Opus 4.6 Thinking 为 1500±7。Claude 系列因此值得作为长文、改稿和对话式创作的优先候选。事实准确性仍需来源核验。

细分榜比总榜更接近实际选型。下表列出同一快照中各方向的点估计第一名，以及本文重点厂商中当前可购的候选；它不是完整前三名列表，被省略的模型不代表能力更弱。名次相邻或区间重叠时，不能据此宣布稳定胜者。

| 方向 | 点估计第一 | 本文重点厂商的可购候选 |
| --- | --- | --- |
| 专业问题 | Opus 4.6 Thinking，1545±10 | GPT-5.4 High 第 7、Sonnet 5 High 第 10、Gemini 3.1 Pro 第 12 |
| 数学 | Fable 5，1521±37 | Gemini 3.5 Flash High 第 3、GPT-5.5 第 7、Gemini 3.1 Pro 第 10 |
| 多轮对话 | Fable 5，1525±23 | Gemini 3 Pro 第 9、GPT-5.2 Chat 第 10 |
| 创意写作 | Opus 4.6 Thinking，1500±7 | GPT-5.6 Sol 第 4、Gemini 3 Pro 第 5 |
| 文本编程 | Fable 5，1564±18 | Sonnet 5 第 16、Gemini 3.1 Pro 第 17、GPT-5.4 High 第 19 |
| 中文 | Opus 4.6 Thinking，1554±12 | Gemini 3.5 Flash Medium 第 5、GPT-5.5 第 6、Gemini 3 Pro 第 9 |

**多模态与科学任务。** Gemini 3.5 Flash 在视觉评测和速度上更均衡，Gemini 3.1 Pro 在 GPQA 与 SciCode 上较强。它们的 Agent 综合分项低于当前 GPT 与 Claude 旗舰，涉及多轮工具操作时需要更严格的任务实测。

**速度与吞吐。** Artificial Analysis 在该快照中记录到 GPT-5.6 Luna 约 204 tokens/s、Gemini 3.5 Flash High 约 158 tokens/s 的生成速度，适合进入长输出和批量处理的首轮。这个指标不包含开始生成前的等待时间；实时交互还要比较首字延迟和端到端耗时。速度也会随服务商、地区、负载、推理档位和输出长度变化。

**中文任务。** Artificial Analysis 综合指数是英文评测。GLM、Qwen、Kimi 或国际模型在中文写作、中文检索和本地业务术语上的表现，都不能从这张表直接推出。

## 编程和 Agent 需要单独看

LMArena 的 Coding 分类有 146 万次偏好投票。Fable 5 为 1564±18，Opus 4.7 Thinking 为 1553±7，Claude 系列整体靠前；刚发布的 GPT-5.6 Sol xHigh 为 1528±29，但当时只有 449 票，区间较宽。这说明 Claude 的代码解释和对话体验当前更受 Arena 用户偏好，不能证明它在真实仓库中会解决更多 Issue。

**真实仓库。** SWE-Bench Pro 的 [Public 731 题榜](https://scale.com/leaderboard/swe_bench_pro_public)包含更长的多文件任务，[Private 276 题榜](https://scale.com/leaderboard/swe_bench_pro_private)还能降低公开题污染：

| 模型与设置 | Public 731 题 | Private 276 题 |
| --- | ---: | ---: |
| GPT-5.4，xHigh | 59.1%±3.6 | 43.4%±6.0 |
| Claude Opus 4.6，thinking | 51.9%±3.6 | 47.1%±6.1 |
| Gemini 3.1 Pro，thinking | 46.1%±3.6 | 32.2%±5.7 |

三组结果都使用 mini-SWE-agent，但推理档位并不等预算；Public 的这些新结果采用不封顶成本和最多 250 轮。GPT-5.4 在公开集的点估计领先，Opus 4.6 在私有集的点估计领先。私有集上 GPT 与 Opus 的置信区间大幅重叠，无法宣布稳定胜者。仓库分布变化带来的差异，往往大于榜单上几分的差距。

**同脚手架的成功率与成本。** [SWE-bench Verified 官方榜](https://www.swebench.com/)用同一 [mini-SWE-agent 2.0.0](https://github.com/SWE-agent/mini-swe-agent/tree/v2.0.0) 测试 500 道人工筛选题。下表是 2026 年 7 月 11 日的资料快照，各次提交日期为 2026 年 2 月 17～19 日：

| 模型与推理配置 | 解决率 | 平均 API 成本 / 题 |
| --- | ---: | ---: |
| Claude 4.5 Opus，high | 76.8% | \$0.75 |
| Gemini 3 Flash，high | 75.8% | \$0.36 |
| MiniMax M2.5，high | 75.8% | \$0.07 |
| Claude Opus 4.6 | 75.6% | \$0.55 |
| GPT-5.2 Codex | 72.8% | \$0.45 |

这些结果采用 pass@1，也就是每题只运行一次并保留一条轨迹；官方没有提供重复采样或置信区间。约 1 个百分点的差异不足以支持稳定排序；成本相差超过十倍时，候选顺序会明显变化。榜单的 `Avg. USD` 是该次评测的平均 API 成本，不是每个成功任务成本、订阅价或包含人工复核与失败重试的总成本。

**CLI Agent 产品。** Terminal-Bench 2.1 使用 89 个软件、机器学习、安全和数据任务，每题运行 5 次，并统一超时与资源限制。下表比较厂商 Agent 完整配置与同模型的 Terminus 2 脚手架：

| 模型 | 厂商 Agent | Terminus 2 | 点估计差（百分点） |
| --- | ---: | ---: | ---: |
| GPT-5.5 | Codex CLI 83.4%±2.2 | 78.2%±2.4 | +5.2 |
| Claude Fable 5 | Claude Code 83.1%±2.0 | 80.4%±2.3 | +2.7 |
| Claude Opus 4.8 | Claude Code 78.9%±2.5 | 74.6%±2.4 | +4.3 |
| Gemini 3.1 Pro | Gemini CLI 70.7%±2.9 | 70.3%±2.9 | +0.4 |

四组厂商 Agent 配置的点估计都高于 Terminus 2，说明选型时不能只看底层模型。两边还可能使用不同的系统提示、工具接口、上下文管理和 token 预算，部分区间也有重叠；这些结果无法单独测出产品编排层贡献了多少。

**长任务可靠性。** METR 页面更新于 2026 年 5 月 8 日。它把题目换算为低上下文人类专家需要的完成时间。下表只摘录与本文选型直接相关的四个已测系统；当时尚未覆盖 Fable 5、Sonnet 5、GPT-5.5 和 GPT-5.6：

| Agent 系统 / 脚手架 | 50% 成功时间跨度（95% 区间） | 80% 成功时间跨度（95% 区间） |
| --- | ---: | ---: |
| Claude Opus 4.6 / ReAct | 11 小时 59 分（5.3～60.6 小时） | 1 小时 10 分（27 分～2 小时 50 分） |
| Gemini 3.1 Pro / Triframe | 6 小时 24 分（3.89～11.58 小时） | 1 小时 30 分（52 分～2 小时 39 分） |
| GPT-5.3 Codex / Triframe | 5 小时 50 分（3.25～13.61 小时） | 55 分钟（22 分～2 小时 2 分） |
| GPT-5.4 / ReAct | 5 小时 42 分（3.11～12.81 小时） | 54 分钟（24 分～1 小时 49 分） |

每个任务运行 6 次，并针对各模型调整提示和调用方式。Opus 4.6 的 50% 点估计最高，Gemini 3.1 Pro 的 80% 点估计最高，宽区间和不同脚手架使两种口径都没有稳定胜者。按 80% 口径，表内四个系统的点估计为 54～90 分钟；它是这组任务的拟合成功概率，不是业务服务等级。METR 还明确提示，超过 16 小时的估计在现有任务集上不可靠。

## 价格与实际成本

### 当前旗舰与代表型号 API 价

下表是厂商官方页面在 2026 年 7 月 11 日的当前有效公开价，单位是美元 / 百万 tokens；促销条件会单独注明。来源：[OpenAI](https://developers.openai.com/api/docs/pricing)、[Anthropic](https://platform.claude.com/docs/en/about-claude/pricing)、[Google](https://ai.google.dev/gemini-api/docs/pricing)、[Z.AI](https://docs.z.ai/guides/overview/pricing)。缓存列只写命中价，不含首次写入与存储。

| 模型 | 输入 | 缓存命中 | 输出 |
| --- | ---: | ---: | ---: |
| Claude Fable 5 | \$10 | \$1 | \$50 |
| GPT-5.6 Sol | \$5 | \$0.50 | \$30 |
| Claude Opus 4.8 | \$5 | \$0.50 | \$25 |
| GPT-5.6 Terra | \$2.50 | \$0.25 | \$15 |
| Claude Sonnet 5，引导价至 2026-08-31 | \$2 | \$0.20 | \$10 |
| Gemini 3.1 Pro Preview，输入不超过 200K | \$2 | \$0.20 | \$12 |
| Gemini 3.5 Flash | \$1.50 | \$0.15 | \$9 |
| GLM-5.2，Z.AI 国际站 | \$1.40 | \$0.26 | \$4.40 |
| GPT-5.6 Luna | \$1 | \$0.10 | \$6 |
| Claude Haiku 4.5 | \$1 | \$0.10 | \$5 |
| Gemini 3.1 Flash-Lite | \$0.25 | \$0.025 | \$1.50 |

### 仍在售的成熟与低成本 API 型号

新一代发布后，成熟型号仍可能因为稳定性、兼容性或更低价格适合生产环境。下表同样取 2026 年 7 月 11 日官方标准价，单位为美元 / 百万 tokens：

| 模型 | 输入 | 缓存命中 | 输出 |
| --- | ---: | ---: | ---: |
| [GPT-5.5](https://developers.openai.com/api/docs/pricing) | \$5 | \$0.50 | \$30 |
| [Claude Sonnet 4.6](https://platform.claude.com/docs/en/about-claude/pricing) | \$3 | \$0.30 | \$15 |
| [GPT-5.4](https://developers.openai.com/api/docs/pricing) | \$2.50 | \$0.25 | \$15 |
| [GPT-5.4 mini](https://developers.openai.com/api/docs/pricing) | \$0.75 | \$0.075 | \$4.50 |
| [GPT-5.4 nano](https://developers.openai.com/api/docs/pricing) | \$0.20 | \$0.02 | \$1.25 |

价格表还需要结合这些规则：

- GPT-5.6、GPT-5.5 和 GPT-5.4 的输入超过 272K 时，整次请求按 2 倍输入价和 1.5 倍输出价计费；上下文窗口为 1,050,000 tokens。GPT-5.4 mini 和 nano 没有这一长上下文价格档。
- Gemini 3.1 Pro 的输入超过 200K 时，输入、缓存命中和输出分别升至 \$4、\$0.40 和 \$18；缓存还按时间收存储费。
- Claude Fable 5、Opus 4.8、Sonnet 5 和 Sonnet 4.6 的 1M 上下文仍按表内 token 单价，没有长上下文加价。
- Claude Sonnet 5 的 \$2 / \$10 是截至 2026 年 8 月 31 日的引导价，9 月 1 日起变为 \$3 / \$15。
- Anthropic 表示 Fable 5、Opus 4.7 及后续型号和 Sonnet 5 的新 tokenizer 对同一文本约产生多 30% tokens，实际增幅取决于内容。
- 缓存命中价只适用于重复读取。GPT-5.6 在 272K 内的首次写入是输入价的 1.25 倍，长上下文再按对应倍率计费；Anthropic 的 5 分钟和 1 小时写入分别为 1.25 倍和 2 倍，命中为 0.1 倍；Gemini 3.1 Pro 存储为 \$4.50 / 百万 tokens / 小时，3.5 Flash 与 Flash-Lite 为 \$1 / 百万 tokens / 小时。
- 本文所列 OpenAI、Anthropic 与 Google 模型的 Batch 输入和输出是标准价的 50%；缓存、存储和功能支持另按厂商规则，OpenAI 最长 24 小时返回。
- OpenAI 对符合条件的区域处理端点加收 10%；Anthropic 的 US-only inference 对 Opus 4.6、Sonnet 4.6 及后续型号按 1.1 倍计费。搜索、地图和代码执行也可能另收费。

假设一次请求使用 100K 输入和 10K 输出，无缓存、无工具调用，按表中单价计算：Fable 5 约 \$1.50、GPT-5.6 Sol \$0.80、Opus 4.8 \$0.75、Terra \$0.40、Sonnet 5 \$0.30、Gemini 3.5 Flash \$0.24、GLM-5.2 \$0.184、Luna \$0.16、Flash-Lite \$0.04。推理 tokens 也会计入输出；厂商 tokenizer 不同，这只是同 token 数的单价演算，不代表处理同一份材料的成本。

### 国内官方 API 价

单位为人民币 / 百万 tokens，渠道和地区与上面的国际站价格分开：

| 模型 | 输入 | 缓存命中 | 输出 |
| --- | ---: | ---: | ---: |
| [GLM-5.2](https://bigmodel.cn/pricing) | ¥8 | ¥2 | ¥28 |
| [GLM-5.1，输入不足 32K](https://bigmodel.cn/pricing) | ¥6 | ¥1.30 | ¥24 |
| [GLM-5.1，输入至少 32K](https://bigmodel.cn/pricing) | ¥8 | ¥2 | ¥28 |
| [GLM-5-Turbo，输入不足 32K](https://bigmodel.cn/pricing) | ¥5 | ¥1.20 | ¥22 |
| [GLM-5-Turbo，输入至少 32K](https://bigmodel.cn/pricing) | ¥7 | ¥1.80 | ¥26 |
| [GLM-5，输入不足 32K](https://bigmodel.cn/pricing) | ¥4 | ¥1 | ¥18 |
| [GLM-5，输入至少 32K](https://bigmodel.cn/pricing) | ¥6 | ¥1.50 | ¥22 |
| [Qwen3.7-Max](https://help.aliyun.com/zh/model-studio/model-pricing) | ¥12（活动 ¥6） | [显式 ¥1.2 / 隐式 ¥2.4](https://help.aliyun.com/zh/model-studio/context-cache) | ¥36（活动 ¥18） |
| [Kimi K2.6](https://platform.kimi.com/docs/pricing/chat-k26) | ¥6.50 | ¥1.10 | ¥27 |
| [MiniMax-M2.7-highspeed](https://platform.minimaxi.com/docs/guides/pricing-paygo) | ¥4.20 | ¥0.42 | ¥16.80 |
| [MiniMax-M2.7](https://platform.minimaxi.com/docs/guides/pricing-paygo) | ¥2.10 | ¥0.42 | ¥8.40 |
| [MiniMax-M3，输入 ≤512K](https://platform.minimaxi.com/docs/guides/pricing-paygo) | ¥2.10 | ¥0.42 | ¥8.40 |
| [MiniMax-M3，输入 >512K](https://platform.minimaxi.com/docs/guides/pricing-paygo) | ¥4.20 | ¥0.84 | ¥16.80 |

Qwen3.7-Max 的 Batch 输入输出半价，不能与上下文缓存折扣同时使用；表中活动价来自当前页面的限时五折标记，结束时间仍要在控制台确认。显式缓存首次写入为 ¥15 / 百万 tokens、默认保存 5 分钟，命中为 ¥1.2，隐式缓存命中为 ¥2.4 且不保证识别成功。

GLM 表内型号的缓存存储当前限时免费。MiniMax-M3 表中是永久五折后的实付价，目录价为表中两倍；M2.7 与 M2.7-highspeed 的缓存写入均为 ¥2.625 / 百万 tokens，M3 官方未列缓存写入价。GLM-5.2、Qwen3.7-Max 和 MiniMax-M3 支持约 1M 上下文，Kimi K2.6 为 262,144 tokens。

### 国际订阅与办公产品价

| 产品 | 个人或基础档 | 高使用量或组织档 |
| --- | --- | --- |
| [ChatGPT](https://developers.openai.com/codex/pricing/) | Free \$0、Go \$8/月、Plus \$20/月；Pro 提供 5 倍 / 20 倍额度，从 \$100/月起 | Business 年付 \$20/人/月、月付 \$25；Enterprise 与 Edu 按销售或 credits 方案核对 |
| [Claude](https://claude.com/pricing) | Pro \$20/月，年付折合 \$17/月；Max 5x 为 \$100/月、Max 20x 为 \$200/月 | Team 标准席位年付 \$20/月、月付 \$25；Premium 年付 \$100/席位/月、月付 \$125/席位/月；Enterprise 为 \$20/席位加 API 用量 |
| [Google AI](https://one.google.com/intl/en_us/about/google-ai-plans/) | 美国区 Plus \$4.99/月、Pro \$19.99/月 | Ultra 从 \$99.99/月起；公开页仅列月付，企业版另询价 |
| [GitHub Copilot](https://github.com/features/copilot/plans) | Pro \$10/月、Pro+ \$39/月、Max \$100/月；每月美元计价 AI Credits 分别为 \$15 / \$70 / \$200 | Business \$19/人/月；Enterprise \$39/人/月，组织额度池化并可另购 |
| [Microsoft Copilot](https://www.microsoft.com/en-us/microsoft-365-copilot/pricing/individuals) | Personal \$9.99/月或 \$99.99/年；Premium \$19.99/月或 \$199.99/年 | [Copilot Chat](https://www.microsoft.com/en-us/microsoft-365-copilot/pricing/enterprise) 对合资格组织账号不另收费；Microsoft 365 Copilot \$30/人/月，年付 |

表内个人方案使用美国公开价，不含可能适用的税费；移动端、地区定价、汇率和权益可能不同。组织价还要核对最低席位、年付条件和现有基础许可证。订阅产品通常不包含同厂商 API 用量。

### 国内订阅与编程套餐

| 产品 | 入门档 | 高使用量档 | 额度口径 |
| --- | --- | --- | --- |
| [阿里云百炼 Token Plan 团队版](https://help.aliyun.com/zh/model-studio/token-plan-overview) | 标准坐席 ¥198/席位/月，25,000 Credits | 高级坐席 ¥698/席位/月，100,000 Credits；尊享坐席 ¥1,398/席位/月，250,000 Credits | 按席位计费，多模型共享 Credits，可另购团队加油包 |
| [Kimi Code](https://www.kimi.com/code) | Andante ¥49/月；Moderato ¥99/月 | Allegretto ¥199/月；Allegro ¥699/月 | 使用量按周刷新；Allegretto 起支持消耗 3 倍额度的高速模式 |
| [GLM Coding Plan](https://bigmodel.cn/claude-code) | Lite ¥49/月 | Pro ¥149/月；Max ¥469/月 | Pro 为 5 倍 Lite、Max 为 20 倍 Lite，支持 20 多种编程工具 |
| [MiniMax Token Plan](https://platform.minimaxi.com/subscribe/token-plan) | Plus ¥49/月 | Max ¥119/月；Ultra ¥469/月 | 受 5 小时窗口和周窗口约束，约支持 3～4 / 4～5 / 6～7 个 Agent |

Kimi Code 年付折合 Andante / Moderato / Allegretto / Allegro 每月 ¥39 / ¥79 / ¥159 / ¥559。MiniMax 年付 Plus / Max / Ultra 为 ¥490 / ¥1,190 / ¥4,690，官方估算每月约含 6 亿 / 18 亿 / 71 亿以上 M3 tokens；Max 和 Ultra 另含每日 3 / 5 条视频生成。GLM 另有连续包季和包年折扣。

这些套餐仍受动态限流、固定窗口和周窗口约束，页面给出的 token 数是估算值。Credits、消息数、Agent 数、token 数和相对倍数不是同一种额度，不能用表面数字直接比较；还要用同一任务记录实际完成量。

### 编程 Agent 的套餐与额度

Terminal-Bench 比较的是具体 Agent 产品，不能把上面的模型 API 单价直接当成 CLI 成本。截至 2026 年 7 月 11 日，官方公开入口如下：

| Agent | 可用入口 | 公开额度与超额方式 |
| --- | --- | --- |
| [Codex CLI](https://developers.openai.com/codex/pricing/) | ChatGPT Free、Go、Plus、Pro、Business、Edu、Enterprise；或 API Key | Plus / Business 每 5 小时约 15～90 条 Sol、20～110 条 Terra、50～280 条 Luna 本地消息；Pro 为 5 倍或 20 倍。可购买 ChatGPT / workspace credits，API Key 按 token 计费 |
| [Claude Code](https://code.claude.com/docs/en/costs) | Claude Pro、Max、Team、Enterprise；或 Console、Bedrock、Google Cloud、Microsoft Foundry | 与 Claude 聊天共享订阅容量，Max 提供相对 Pro 的 5 倍或 20 倍用量，没有稳定的固定消息数。达到额度后可启用 usage credits 或按 API 用量计费 |
| [Gemini CLI](https://geminicli.com/docs/quota-and-pricing/) | Google 个人账号、Google AI Pro / Ultra、Code Assist、Workspace AI Ultra；或 API Key、Vertex AI | 个人账号、AI Pro、AI Ultra 公开上限分别为每日 1,000 / 1,500 / 2,000 次请求；固定套餐没有统一超额单价，API Key 与 Vertex AI 按量计费 |

这些数字无法直接横向排名。Codex 的消息按模型、上下文和任务动态消耗；Claude 的额度还会被网页和桌面端共享；Gemini 一次提示可能触发多个模型请求。Gemini 官方页同时提示部分个人入口正在迁移到 Antigravity CLI，个人套餐当前可用性需要在安装和付款前复核。Terminal-Bench 也没有统一披露各提交的 token 成本，因此只能用自有任务记录每个成功任务的实际花费。

## 按任务作出选择

**高价值复杂任务。** 将 GPT-5.6 Sol 与 Claude Fable 5 或 Opus 4.8 放入首轮。GPT-5.6 Sol 在当前综合表的编码和 Agent 分项略高，API 价格低于 Fable 5；Claude 当前在写作与 Coding 偏好分类中整体更靠前，部分成熟型号的票数也更多。高推理模式的延迟和输出 token 都会明显上升。

**日常知识工作和 API 代码任务。** GPT-5.6 Terra、Claude Sonnet 5 与 GPT-5.6 Luna 是更合理的成本起点。它们与旗舰的综合分差小于价格差距，但本文没有三者同口径的真实仓库成绩，不能直接推断 CLI Agent 表现。真实任务的成功率和返工时间决定最终选择。

**已有生产集成。** GPT-5.5、GPT-5.4 与 Claude Sonnet 4.6 仍在官方价目表中。已有提示模板、评测基线或兼容性认证时，不需要因为新一代发布立即迁移；先比较新旧型号在同一任务上的成功率、延迟、token 数和迁移成本。GPT-5.4 mini 与 nano 可以作为简单分类、抽取和路由任务的低价候选。

**写作、改稿和创意表达。** Claude 优先进入试用，尤其适合需要多轮修改、语气保持和长篇结构的工作。研究文章还要单独检查引用、日期和事实，LMArena 偏好分不能证明准确性。

**科学、视觉与 Google 搜索。** Gemini 3.1 Pro 适合复杂科学推理和多模态材料，Gemini 3.5 Flash 更适合速度、图像理解和搜索增强任务。大量简单分类、翻译或抽取可从 Flash-Lite 开始。

**终端和代码库 Agent。** Codex CLI 与 Claude Code 是当前 Terminal-Bench 第一梯队。仓库语言、测试速度、私有依赖、权限和现有开发工具会改变结果，最终按 [AI Coding](/ai-coding) 的补丁闭环验收。

**价格敏感和中文场景。** GLM-5.2 在当前英文综合评测中呈现较强价格性能；MiniMax M2.5 在同脚手架 SWE-bench Verified 中显示出较低的平均成本，但这个结果不能直接外推到 M2.7 或 M3。Qwen、Kimi、GLM 与 MiniMax 都有国内官方 API 或订阅渠道。中文质量、搜索来源、服务地区和数据条款仍要使用自己的中文任务测试。

**办公套件。** 文件、邮件和会议主要在 Google Workspace 时先测试 Gemini；主要在 Word、Excel、PowerPoint、Outlook、SharePoint 和 Teams 时先测试 Microsoft 365 Copilot。GitHub 仓库、Pull Request 和组织策略是研发主线时，再单独评估 GitHub Copilot。

## 数据与组织治理

“不用于训练”只回答了一个问题，还要核对保留期限、删除、存储地区、分包商、连接器、管理员权限和数据保护协议。

| 商业产品 | 官方公开口径 |
| --- | --- |
| [OpenAI 商业产品](https://openai.com/enterprise-privacy/) | OpenAI 表示商业数据默认不用于训练模型；具体方案继续核对保留、连接器、数据驻留和管理员控制 |
| [Claude for Work 与 Anthropic API](https://privacy.claude.com/en/articles/7996868-is-my-data-used-for-model-training) | 商业产品的输入输出默认不用于训练；明确提交反馈、Bug 或主动授权属于例外 |
| [Google Workspace with Gemini](https://workspace.google.com/intl/en_us/security/ai-privacy/) | 未经许可，Workspace 数据不会由人工审查，也不会在域外用于生成式 AI 模型训练 |
| [Microsoft 365 Copilot](https://learn.microsoft.com/en-us/copilot/microsoft-365/microsoft-365-copilot-privacy) | 提示、响应和通过 Microsoft Graph 访问的数据不用于训练基础大模型；仍遵循用户已有查看权限 |
| [GitHub Copilot Business 与 Enterprise](https://copilot.github.trust.page) | GitHub 表示不会使用 Business 或 Enterprise 客户数据训练 AI 模型 |

同一厂商的个人版、工作区、API、连接器和主动反馈可能采用不同规则。本文对 Qwen、Kimi、GLM 与 MiniMax 只核对了公开价格和额度，不能据此推断训练使用、保留期限、存储地区或组织控制已经满足要求。组织采购应让安全、法务和数据负责人核对具体方案的实际合同，不能只依赖产品宣传页。

## 用固定任务集做最终验证

从真实工作中选 20～50 个任务，覆盖日常、高难、长上下文、信息不足和失败恢复场景。例如：

- 根据已知材料起草可以直接发送的客户回复。
- 对多个原始来源做研究，将证据与结论一一对应。
- 在私有代码库修复一个 Bug，运行测试并检查差异。
- 处理长文档，提取约束、冲突和待确认事项。
- 输入不足时先追问，遇到权限或工具失败时正确停止。

使用同一组输入、附件、工具权限和验收标准比较候选。关键任务至少重复三次，并尽量隐藏模型名称后再评分。每次记录模型版本、推理档位、日期与设置。

```text
任务：
产品 / 模型 / Agent / 推理档位：
输入数据级别与工具权限：
成功标准：
多次运行结果：
人工修正内容与时间：
首个答案延迟与总耗时：
API、工具与重试成本：
主要失败模式：
```

真正需要优化的是**单个成功任务的总成本**：API、工具调用、失败重试和人工修正时间都要计算。一个请求便宜但需要运行三次，可能比一次成功的高价模型更贵。

榜单用于缩小候选范围。固定任务集用于决定当前工作流的最终选择；任务分布、模型版本或 Agent 脚手架变化后，需要重新测试。

## 上线前的治理与退出方案

注册、付款、API 密钥和账号使用应符合厂商当前公开的服务地区、付款和服务条款。第三方接码、共享账号、未授权代付和规避地区限制会带来账号归属、隐私、付款与服务终止风险。

不同环境和服务使用独立密钥，设置预算、速率和动作权限，定期轮换。删除、发布、数据库和生产操作按动作授权，不为了方便关闭全部确认。

原始文档、任务契约、提示模板、验收标准和最终制品应保存在可导出、可版本化的位置。迁移测试可以选择一个常用工作流，只凭导出的文档、模板和规则在另一个模型上恢复。关键上下文只存在于产品对话历史时，迁移成本会持续增加。

第二个模型适合解决明确问题：某类关键任务在固定评测中明显更好、单一服务故障时仍需工作，或特定数据只能进入指定环境。新增模型也会增加账号、计费、模板、测试和故障排查成本。

## 做出选择前的检查表

- 产品订阅、底层模型、API 和 Agent 已分层比较。
- 专业榜单的任务、脚手架、推理档位、样本和置信区间已经核对。
- 代表性任务有明确成功标准，并完成多次运行。
- 费用包含缓存、长上下文、工具调用、失败重试和人工修正。
- 数据保留、训练使用、删除、地区和组织权限已从官方条款核实。
- 原始资料、模板、规则和结果可以导出并迁移。
- 账号、付款、密钥和自动化方式符合官方服务条款。

这些条件满足后，选型结论才有资格回答“哪个服务更适合当前任务”。
