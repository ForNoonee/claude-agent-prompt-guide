# Claude Agent 提示词教程：省 token、提效果、强记忆

> 根据 Anthropic 官方文档与工程博客整理的中文教程，**非 Anthropic 官方出品**。内容核对日期：2026-10-10，数字和参数以当日官方页面为准，模型更新后可能变化。每节末尾都标了出处，完整清单见文末「来源」。

写 Agent 提示词，关键不是写得多，而是让模型每一步只看到最少、最有用的信息。Anthropic 把这件事叫作「上下文工程」：系统提示词、工具、历史消息、记忆，都是要精打细算的上下文。

这份教程按三个问题组织，每条做法都标了出自哪份官方文档：

- **怎么省 token**：第 4 节
- **怎么让效果更好**：第 2、3、5 节
- **怎么让它记得住**：第 6 节

最后一节是可以直接改着用的系统提示词模板。

## 1. Agent 看到的全部信息，就是它的上下文

Agent 每一步做决定时，只能依据上下文窗口里的东西。Anthropic 说上下文是「边际收益递减的有限资源」：东西塞得越多，模型从里面准确找回信息的能力会下降，官方称为「上下文腐烂」（context rot）。所以目标是：**用最少的、信息量最高的 token，换来想要的结果**。

| 组成部分 | 作用 | 最常见的浪费 |
| --- | --- | --- |
| 系统提示词 | 定义角色、目标、行事规则 | 堆一长串边角规则、硬编码 if-else |
| 工具定义 | 告诉模型能做什么、怎么调 | 工具太多、功能重叠、描述含糊 |
| 消息历史 | 对话与每次工具调用的结果 | 旧的工具返回一直留在里面 |
| 外部数据 | 文件、网页、数据库查询结果 | 一次性全部预加载 |
| 记忆 | 跨步骤、跨会话保留的状态 | 什么都往里记、从不整理 |

官方给的大方向有三条：

1. **按需加载，而不是预先塞满**：只给 Agent 文件路径、查询语句、链接这类轻量「索引」，让它用工具在用到时再去取。Claude Code 的做法是混合的：CLAUDE.md 一开始就加载，其余文件用 glob、grep 等到要用时再找。
2. **先极简，再按失败补**：用最强的模型配一个最简的提示词跑起来，观察它在哪里出错，再针对性地加指令或示例。
3. **长任务要主动管理上下文**：压缩、笔记、子代理三种办法，第 4、6 节展开。

出处：[Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)（2025-09-29）

## 2. 系统提示词：写清楚、讲原因、给示例、分好段

官方的黄金法则：**把提示词拿给一个不了解这个任务的同事看，他看不懂，Claude 也会糊涂。** 把 Claude 当成一个很聪明、但刚入职、不了解你们习惯的新同事。

开始写之前，先读你所用模型的专门页面（Opus 5.5、Sonnet 5.5、Haiku 5.5 等都有），里面写了该模型和上一代的行为差异。下面是所有当前模型通用的八条。

1. **要什么就明说。** 想要「超出基本要求」的效果要直接说，别指望它自己猜。顺序或完整性重要时，用编号步骤。
    - 弱：做一个数据看板
    - 强：做一个数据看板，尽可能多加相关的功能和交互，做成功能完整的版本
2. **讲清楚为什么。** 给出原因，模型会自己举一反三。
    - 弱：绝不要用省略号
    - 强：你的回答会被语音引擎朗读，它不知道省略号怎么读，所以不要用省略号
3. **写在合适的「抽象高度」。** 不要写死一堆 if-else，也不要只给空泛口号。写清楚原则和关键的判断标准，让模型有依据去自己判断。
4. **用示例代替长规则清单。** 官方建议 3 到 5 个，要贴近真实用例、彼此不同，用 `<example>` 标签包起来，和指令区分开。
5. **用 XML 标签或 Markdown 标题分区。** 比如 `<instructions>`、`<context>`、`<input>`，标签名前后一致。长文档（约 2 万 token 以上）放在最前面，问题放在最后：官方测试里，复杂的多文档输入这样排，回答质量最多能提升 30%。
6. **一句话角色设定就有用。** 例如「你是一个擅长 Python 的编程助手」。
7. **说「要做什么」，不说「别做什么」。** 与其写「不要用 Markdown」，不如写「用流畅的段落作答」。
8. **别用吓人的全大写。** 新模型对系统提示词更敏感，写「CRITICAL: You MUST use this tool」反而会让它乱用工具。平常地写「遇到 X 时用这个工具」就行。

**要它动手，就别只让它建议。** 你问「能建议一下怎么改吗」，它就只会给建议；要它直接改，就说「改这个函数，提升性能」。也可以在系统提示词里定下默认倾向（下面是官方示例的中文译法）：

```text
<default_to_action>
默认直接动手改，而不是只提建议。用户意图不明确时，推断最可能有用的行动并执行；
缺少的细节用工具去查，不要猜。
</default_to_action>
```

想让它保守一点，就反过来写一段 `<do_not_act_before_instructions>`：意图不明确时只调研、给建议，用户明确要求了才改文件。

出处：[Prompting best practices](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices)、[Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)

## 3. 工具：少而精，描述当成提示词来写

工具描述也会进上下文，每轮都占 token。官方的结论是「工具不是越多越好」：如果一个人都分不清该用哪个工具，Agent 也分不清。

| 原则 | 怎么做 | 例子 |
| --- | --- | --- |
| 按任务设计，不照搬 API | 把常一起用的几步合成一个工具 | 一个 `schedule_event`，而不是 `list_users` + `list_events` + `create_event`；用 `search_logs` 代替 `read_logs` |
| 加前缀分组 | 同类工具用同一前缀，边界清楚 | `asana_search`、`jira_search` |
| 返回人能看懂的内容 | 用名字代替一串乱码 ID；给一个 `response_format` 参数选简洁/详细 | 官方示例：同一结果，简洁版 72 token，详细版 206 token |
| 控制返回量 | 分页、按范围取、过滤、截断，并设合理默认值 | Claude Code 默认把单个工具返回限制在 25,000 token |
| 报错要教它怎么改 | 说清楚哪里错、下一步该怎么改，而不是甩一个错误码 | 示意：「日期格式应为 YYYY-MM-DD，你传的是 10/8」 |
| 描述写给新同事看 | 参数名不含糊，用途、什么时候该用都写明白 | 用 `user_id` 而不是 `user` |

官方说，工具描述的小修改往往能带来大提升。改工具的办法是跑真实任务的评估，把 Agent 的运行记录整理好交给 Claude，让它分析工具在哪里被用错。留一份没参与修改的测试集，防止改过头。

出处：[Writing effective tools for agents — with agents](https://www.anthropic.com/engineering/writing-tools-for-agents)（2025-09-11）

## 4. 省 token：先做不用改架构的，再做结构性的

最划算的三件事是提示词缓存、给工具返回瘦身、调低 effort，几乎不用改代码结构。长任务再加上下文编辑或压缩；工具多、数据量大的再上子代理、工具搜索和程序化工具调用。

| 手段 | 省在哪 | 官方数字 | 适合什么情况 |
| --- | --- | --- | --- |
| 提示词缓存 | 重复的前缀：工具定义、系统提示词、示例 | 命中按基础输入价的 0.1 倍计（Opus 5.5、Sonnet 5.5 为 0.05 倍）；写入 5 分钟档 1.25 倍、1 小时档 2 倍 | 同一套提示词反复调用 |
| 精简提示词和示例 | 每轮都要付的固定开销 | — | 所有场景 |
| 工具返回瘦身 | 历史里的大块结果 | Claude Code 单次返回默认上限 25,000 token；简洁格式 72 token，详细格式 206 token | 搜索、日志、数据库类工具 |
| 调 effort | 输出的全部 token：思考、回答、工具调用 | `low` 最省；Opus 5.5、Haiku 5.5 默认 `medium`，其余默认 `high` | 简单任务、子代理、对延迟敏感 |
| 上下文编辑（清旧工具结果） | 过期的工具返回 | 默认输入超过 100,000 token 触发，保留最近 3 次调用 | 工具调用很多的长任务 |
| 压缩（compaction） | 整段历史换成摘要 | 官方示例：约 10 万 token 压到 2 千到 3 千 | 来回很多轮的长对话 |
| 子代理 | 把探索过程隔离在子代理里 | 子代理可用掉数万 token，只回传 1,000–2,000 token 的摘要 | 并行调研、大范围搜索 |
| 工具搜索（`defer_loading`） | 暂时用不到的工具定义 | 58 个工具本要占约 55K token；官方示例开局从约 77K 降到约 8.7K，省 85% | 接了很多 MCP 服务器 |
| 程序化工具调用 | 中间结果留在代码环境里，不进上下文 | 复杂调研任务平均从 43,588 降到 27,297 token，少 37% | 多步处理大量数据 |
| 批处理 API | 价格本身 | 便宜 50%，多数批次 1 小时内完成 | 不急的批量任务 |

**用好缓存的六个要点**

- 缓存按 工具 → 系统提示词 → 消息 的顺序匹配前缀：不变的内容放前面，每次都变的放后面
- 时间戳、本次请求才有的内容、用户新消息，都不要放在缓存断点之前，否则每次都在写缓存、从不命中
- 最多 4 个断点；也可以在请求顶层加 `cache_control`，让系统自动放断点
- 改工具定义会让整个缓存失效；中途改 effort 或思考参数会让消息部分失效。要中途换 effort，用「逐条消息 effort」（beta）保住缓存
- 太短不缓存：Opus 5.5、Sonnet 5.5、Haiku 5.5 等新模型至少 512 token，不够也不报错
- 看返回里的 `cache_read_input_tokens`：为 0 就是没命中

上下文编辑和缓存有冲突：每清一次工具结果，缓存就要重写。官方建议设 `clear_at_least`，保证每次清掉的量值得那次重写。

出处：[Prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)、[Effort](https://platform.claude.com/docs/en/build-with-claude/effort)、[Context editing](https://platform.claude.com/docs/en/build-with-claude/context-editing)、[Batch processing](https://platform.claude.com/docs/en/build-with-claude/batch-processing)、[Introducing advanced tool use](https://www.anthropic.com/engineering/advanced-tool-use)（2025-11-24）

## 5. 提效果：先定标准，用最简单的方案跑起来，再靠评估迭代

官方的原话是：成功的关键在于「衡量表现，并不断迭代」。没有一版提示词是一次写对的。

1. **先写清「什么算做对了」。** 调研类任务要写明成功标准，并要求它用多个来源交叉核实。
2. **从最简单的方案开始。** 能用固定流程（workflow）解决，就不要上完全自主的 Agent，确实需要时再加复杂度。官方列了五种常用流程：
    - 链式：拆成前后几步，步与步之间可以加程序检查
    - 路由：先分类，再交给对应的专门流程或模型
    - 并行：子任务同时跑，或同一任务跑多次投票，再汇总
    - 编排-执行：一个主模型拆任务、分派、汇总
    - 评审-优化：一个生成，另一个按标准评审并反馈，循环改进
3. **用评估驱动修改。** 准备一组真实任务当测试集，每改一次提示词或工具就跑一遍；另留一组不参与修改的任务做复核，防止改过头。
4. **让它自己检查。** 最常用的是「起稿 → 按标准审 → 按审稿改」三步，每步单独一次调用，方便记录和评估。任务越长越要给验证工具，比如浏览器、电脑操作、测试。
5. **长任务分阶段做。** 第一个上下文窗口只搭框架（写测试、写启动脚本如 `init.sh`），之后的窗口按待办清单一点点推进。如果你的系统会自动压缩上下文，要在提示词里告诉它，否则它快到上限时会草草收尾。

**常见的跑偏和官方给的纠正写法**

| 现象 | 在提示词里加的要点 |
| --- | --- |
| 只给建议不动手 | 用祈使句下指令，或放 `<default_to_action>`（第 2 节） |
| 过度工程：多建文件、多加抽象和防御代码 | 只做被要求的或明显必要的改动；一次性操作不封装；不为假想的未来需求设计 |
| 为了让测试通过而硬编码 | 写对所有合法输入都成立的通用解法；测试是用来验证的，不是用来定义答案的；测试本身有错就告诉我 |
| 没看代码就下结论 | 用户提到的文件，先读再答；没打开过的代码不推测（`<investigate_before_answering>`） |
| 滥用子代理 | 能并行、需要隔离上下文时才用；简单任务、单文件修改、需要连续上下文的就直接做 |
| 高风险操作不确认 | 可撤销的本地操作直接做；删除、强推、对外发消息这类先问用户 |
| 快到上下文上限就提前收工 | 告诉它上下文会自动压缩，快到上限时先把进度存进记忆，再继续做完 |
| 工具被乱用 | 去掉 CRITICAL、MUST 这类强硬措辞 |

出处：[Prompting best practices](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices)（Agentic systems 一节）、[Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)（2024-12-19）

## 6. 提升记忆：把要记住的东西写到上下文外面，用到时再读回来

模型本身不记得上一次对话，上下文一清空就全忘了。所谓「记忆」，就是让 Agent 把学到的东西写进文件，下次先读文件再干活。这样既记得住，又不会把所有东西一直塞在上下文里。

| 办法 | 怎么用 | 适合 |
| --- | --- | --- |
| 记忆工具 `memory_20250818` | 加进 `tools`，你的程序负责实际读写 `/memories` 目录。启用后 API 会自动在系统提示词里加一段：开工前先看记忆目录，并假设自己随时会被中断 | 跨会话的项目背景、用户偏好、踩过的坑 |
| 结构化状态文件 | `tests.json` 记结构化状态，`progress.txt` 记自由文字的进度，git 当存档点 | 跨多个上下文窗口的长任务 |
| CLAUDE.md 与自动记忆（Claude Code） | CLAUDE.md 由你来写，每次会话都加载；自动记忆由 Claude 自己记，每次加载前 200 行或 25KB | 用 Claude Code 做项目 |
| 压缩 + 记忆 | 压缩让上下文保持小，记忆保住压缩后必须留下的信息 | 超长时间运行的 Agent |
| 上下文编辑 + 记忆 | 快到清理阈值时，Claude 会自动收到提醒，先把重要信息存进记忆，再清旧工具结果 | 工具调用很多的长任务 |

**官方推荐的多会话做法**

1. **第一个会话只建记忆：** 动手前先建好进度日志、功能清单，写清按什么脚本启动
2. **之后每个会话先读记忆：** 靠这些文件恢复状态，不用重新翻一遍代码
3. **结束前更新进度：** 写清做完了什么、还剩什么

关键是一次只做一个功能，端到端验证过才标「完成」，而不是代码写完就标。否则下一个会话会被错误的进度带偏。

**记忆写得好不好，决定它用不用得上**

- **写具体、能检查的话：** 「用 2 个空格缩进」而不是「代码格式要规范」；「提交前跑 `npm test`」而不是「要测试」
- **什么时候补一条：** 同一个错犯了第二次；同一句纠正这次又说了一遍
- **控制长度：** 每个 CLAUDE.md 尽量在 200 行以内，越长越占上下文，也越容易不被遵守。只和部分代码有关的规则，放进按路径加载的规则文件或 skill
- **定期清理：** 两条指令互相矛盾时，Claude 可能随便选一条。Claude Code 里可以用 `/doctor prompt-audit` 找过时或冲突的内容
- **限定记什么：** 例如「只把和 <主题> 有关的信息写进记忆」；记忆文件乱了就提醒它合并、改名、删掉过时文件

**自己实现记忆工具时的安全门槛**：所有路径限制在 `/memories` 内，拦掉 `../` 这类路径穿越；限制单个文件大小；定期删掉很久没用的记忆；写入前过滤敏感信息。

出处：[Memory tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool)、[Context editing](https://platform.claude.com/docs/en/build-with-claude/context-editing)、[How Claude remembers your project](https://code.claude.com/docs/en/memory)、[Prompting best practices](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices)

## 7. 拿去就能改的模板和上线前检查清单

下面的模板把前面六节的做法都放进去了，尖括号里换成你自己的内容。它整体不变，适合放在缓存断点之前；用户输入、当前时间这类每次都变的内容，放在后面的消息里。没用到的分区直接删掉，先极简，再按失败补。

```text
<role>
你是 <产品/团队> 的 <岗位> 助手，服务对象是 <谁>。
</role>

<goal>
你的任务是 <一句话目标>。做到下面几点才算完成：
- <能检查的标准 1>
- <能检查的标准 2>
</goal>

<context>
<用户是谁、在什么场景下用、为什么要这样做>
需要更多资料时，用 <工具名> 按需去查，不要凭印象回答。
</context>

<guidelines>
- <原则，并写原因：例如「回答会被语音朗读，所以不用省略号」>
- 用户意图不明确时：<直接动手 / 先给建议>（二选一，写清楚）
- 可撤销的操作直接做；删除、对外发送、改共享数据之前先问用户
- 只做被要求的或明显必要的事，不自行扩展
- 没读过的资料不下结论；不确定就直说不确定
</guidelines>

<tools>
- <工具 A>：<什么情况下用>，<返回什么>
- <工具 B>：<什么情况下用>，<返回什么>
互不依赖的工具调用可以同时发出。
</tools>

<memory>
开工前先读 <记忆目录或进度文件>。每完成一步就更新进度：做完了什么、下一步是什么。
上下文快满时会自动压缩，不要提前收工；先存好进度再继续。
只记和 <主题> 有关、以后还用得上的信息。
</memory>

<output_format>
<正面描述要的格式，例如「用两三段完整的话回答」，而不是「不要用列表」>
</output_format>

<examples>
<example>
输入：<示例 1>
理想输出：<结果 1>
</example>
<!-- 共 3 到 5 个，彼此不同 -->
</examples>
```

**上线前逐条打勾**

- [ ] 拿给一个不了解任务的同事看，他能看懂要做什么
- [ ] 每条规则都写了原因或判断标准，没有一长串 if-else
- [ ] 有 3 到 5 个彼此不同的示例
- [ ] 没有 CRITICAL、MUST 这类强硬措辞
- [ ] 工具尽量少，名字和参数不含糊，返回内容有长度上限
- [ ] 不变的内容放在前面并开了缓存，`cache_read_input_tokens` 大于 0
- [ ] effort 按任务难度重新设过，没有照搬旧模型的设置
- [ ] 长任务开了上下文编辑或压缩，并配了记忆或进度文件
- [ ] 有一组真实任务当测试集，这次改动后跑过一遍

## 来源

以下都是 Anthropic 官方页面，2026-10-10 打开核对。文中数字和参数以当日页面为准，模型更新后可能变化。

- [Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)（2025-09-29）
- [Prompting best practices](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices)
- [Writing effective tools for agents — with agents](https://www.anthropic.com/engineering/writing-tools-for-agents)（2025-09-11）
- [Introducing advanced tool use](https://www.anthropic.com/engineering/advanced-tool-use)（2025-11-24）
- [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)（2024-12-19）
- [Prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- [Effort](https://platform.claude.com/docs/en/build-with-claude/effort)
- [Context editing](https://platform.claude.com/docs/en/build-with-claude/context-editing)
- [Batch processing](https://platform.claude.com/docs/en/build-with-claude/batch-processing)
- [Memory tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool)
- [How Claude remembers your project](https://code.claude.com/docs/en/memory)（Claude Code）
