# Harness 工程化

> 一句话总结：**Harness Engineering = 围绕 LLM 搭一整套"驾驭脚手架"**（上下文 + 工具 + 编排 + 记忆 + 观测 + 护栏），让模型在长链路真实任务里"持续做对"——区别于 Prompt Engineering（让模型听懂）和 Context Engineering（让模型知道该看什么），它解决的是"模型在执行中不跑偏、跑得稳、出错能爬起来"。

---

## 0. 这个新词从哪冒出来的

**Harness**（马具/缰绳）这个比喻很贴切：马本身力量大，但没缰绳就失控。LLM 同理——能力很强，但放进真实任务里没人约束就是失控。

- 词源：2026-02-05，Mitchell Hashimoto 在《My AI Adoption Journey》里正式命名。
- 他的核心定义："每次 Agent 犯一个错，就花点时间工程化一个方案，让它永远不再犯同样的错。"
- 关键动作：**把修复沉到环境里**（AGENTS.md 加规则、加 linter、补自动化测试、挂 Git Hook），**不是留人脑里**。
- 一周后 OpenAI 发《Harness engineering: leveraging Codex in an agent-first world》背书（3 人 / 5 个月 / 100 万行代码 / 1500 个 PR / 无人手写代码），Harness 正式出圈。

> **小结**：
> - Harness 的本质 = **复利式错误回收**（每次失败都让环境变强一点）。
> - 它不是又一个换皮概念，而是被"长链路 + 真实执行"逼出来的第三阶段。
> - 出身（基础设施老兵先喊 + 大厂几天内背书）决定了它不会昙花一现。

---

## 1. AI 工程的三次重心迁移

| 阶段 | 解决什么 | 核心动作 | 天花板 |
|---|---|---|---|
| **Prompt Engineering** | 模型「听懂」你 | 角色 / 示例 / 约束 / 任务 | 单轮短链路够用，复杂任务撑不住 |
| **Context Engineering** | 模型「知道」该用什么信息 | 召回 → 压缩 → 组装 | 信息送对了，长链路还是跑偏 |
| **Harness Engineering** | 模型「做对」一连串的事 | 上下文 + 工具 + 编排 + 记忆 + 观测 + 护栏 | 真正可生产交付 |

**递进关系（包含而非替代）**：
- Prompt ⊂ Context ⊂ Harness
- 短任务 Prompt 够；需要外部知识就上 Context；进入"长链路、可执行、低容错"场景，Harness 不可避免。

> **小结**：
> - 三者不是替代，是**层层包含**。
> - 一个等式记牢：**`Agent = Model + Harness`**（Harness = Agent − Model）。
> - 模型决定天花板，Harness 决定能不能落地——同样的模型效果天差地别，差的就是 Harness。

---

## 2. Harness 六大核心组件

OpenAI、Anthropic、LangChain 形态不同，但掀开 Harness 内部结构惊人相似——"让 Agent 在真实世界稳定工作"这个命题，会推着所有人往同一方向收敛。

| 分组 | 层级 | 解决什么核心问题 | 关键动作 |
|---|---|---|---|
| **输入侧** | ① 上下文精细化 | 这一轮发给模型的那坨 token 长啥样 | 钉角色目标 / 动态筛选 / 结构化分块 |
| **输入侧** | ④ 记忆与状态 | 上一轮发生的事怎么流到下一轮 | 状态外化到文件系统 / 分层生命周期 |
| **动作侧** | ② 工具系统 | 模型能用什么能力做对事 | 工具按需接 / 选对时机 / 结果二次提炼 |
| **动作侧** | ③ 任务编排 | 怎么把"一步接一步"串成对的事 | ReAct 循环 / Plan-and-Execute / Reflexion |
| **校验侧** | ⑤ 评估观测 | 怎么知道它每一步做对没 | Eval 集 / Trace / 指标 |
| **校验侧** | ⑥ 约束恢复 | 怎么让它失败时还能爬起来 | 硬约束 / 输出校验 / 失败重试 |

**贯穿全章的示例**：PR Review Agent——每天定时扫 GitHub 关注仓库的新 PR → 挑出值得关注的 → 写摘要 + 点评 → 发 Slack。

### 2.1 第一层：上下文精细化（取景框）

第一层管"空间"（这一轮上下文长啥样），第四层管"时间"（跨轮怎么流）——别搞混。

三件核心工作：
1. **钉死角色和目标**——"你是 PR 审查助手，当前任务是挑出值得关注的 PR 并生成摘要，成功标准 = 我挑出来的真的都是该被关注的"。
2. **动态筛选而非一次塞满**——只把当前 PR 相关的信息拉进来，其余留文件系统。
3. **结构化组织**——固定规则 / 动态证据 / 中间结论**分三处放**，否则模型会"自我污染"（用前面错的中间结论影响后面）。

> Anthropic 把"塞太满导致注意力涣散"命名为 **context rot**（上下文腐化），解法叫 **just-in-time retrieval**——边干活边按需抓，不一上来塞满。

### 2.2 第二层：工具系统（接什么、什么时候用、结果怎么喂回）

工具不是接得越多越好。OpenAI Codex 早期接一堆工具，Agent 频繁用错；砍掉一大半效果反而上去了。

三道必答题：

| 问答 | 决定 |
|---|---|
| **给哪些工具** | 只给 PR 审查必需的（命令行 / 读文件 / 代码搜索 / Slack 发送），其他先别加 |
| **什么时候用** | 该查时查，不该查时不查——diff 已在上下文就别再重拉 PR |
| **结果怎么喂回** | 30 条匹配先提炼再喂，否则 30 条原文一进来又污染上下文 |

> **MCP（Model Context Protocol）** = 工具层标准协议，让任何工具用同一种方式接到任何 Agent，类似工具领域的"USB-C"。

### 2.3 第三层：任务编排（for 循环的魔鬼）

Agent 本质 = `for { 思考 → 行动 → 观察结果 → 再思考 }`，这个循环叫 **ReAct**（Reasoning + Acting）。

朴素但魔鬼在细节：每一步 Agent 都会做，但**所有步骤串起来就乱**。第三层职责 = 给模型一条明确工作轨道。

| 编排模式 | 思路 | 适用 |
|---|---|---|
| **ReAct** | 思考-行动-观察循环 | 短链工具调用 |
| **Plan-and-Execute** | 先规划完整计划再执行 | 长链路任务 |
| **Reflexion** | 每次失败反思再重试 | 反复出错的任务 |
| **Tree of Thoughts** | 同时探索多条思路选最优 | 复杂推理 |

### 2.4 第四层：记忆与状态（胶卷）

没有状态管理，Agent 每轮失忆——PR Review Agent 今天审过的 PR 明天又审一遍，Slack 变重复消息坟场。

**核心洞察**（Mitchell Hashimoto / Anthropic）：Agent 的状态**不应放在上下文窗口里**，而应**外化到文件系统**。

PR Review Agent 的三类状态必须分层存：

| 类型 | 生命周期 | 注入方式 |
|---|---|---|
| **任务状态** | 活到任务结束 | 当天任务跑完归档 |
| **会话中间结果** | 活到当轮结束 | 随会话结束丢弃 |
| **长期记忆** | 跨所有任务 | 每次调用都注入（如 CLAUDE.md / .cursorrules） |

> Claude Code / Cursor / Trae 的规则文件就是"长期记忆"层的典型实现——每次调用自动注入，Agent 永远记得项目核心约束。

### 2.5 第五层：评估观测（尺子 + 量）

这一层最易被跳过，但跳过就进退两难。

**尺子 = Eval 集（灵魂）**：
- 手写一批典型任务，每个标注"正确答案长啥样"。
- 改 Harness 任何一处（CLAUDE.md / 新工具 / 调整编排），都跑一遍 Eval 集对比成功率。
- 没有 Eval 集，你对 Agent 好坏的判断永远停留在"我感觉这次变好了"的玄学阶段。

**量 = Trace + 日志 + 指标**：
- LangSmith / Langfuse 这类 trace 系统，让你看到 Agent 每一步决策、调了哪个工具、返回啥、花了多少 token。
- 能看 trace 才能定位失败那一步发生了什么。

> 案例：LangChain 靠 Eval 集 + trace 回放迭代，把 Terminal Bench 从 52.8 推到 66.5、榜位从 30+ 冲进前 5。

### 2.6 第六层：约束校验与恢复（生产环境必需品）

真实环境里失败不是例外是常态：GitHub 限流、diff 太大冲爆上下文、Slack webhook 过期、token 耗光……

三件事必须做：

```python
# 伪代码：约束 + 校验 + 恢复
class PRReviewGuardrail:
    def __init__(self, agent):
        self.agent = agent
        self.max_prs = 20
        self.max_tokens = 100_000
        self.slack_whitelist = {"#pr-review", "#alerts"}

    async def step_constraints(self, ctx):
        # 1. 硬约束：token / 数量 / 行为边界
        if ctx.token_used > self.max_tokens:
            raise StopAndResume()           # 立即停下，保存进度
        if ctx.prs_analyzed >= self.max_prs:
            return "BATCH_DONE"
        if ctx.pr.status == "closed":
            return "SKIP_CLOSED"            # 不对 closed PR 评论

    def validate_output(self, summary, pr):
        # 2. 校验：每步输出前后做硬规则检查
        assert summary.format == "markdown"
        assert 3 <= summary.paragraphs <= 5
        return summary

    async def send_to_slack(self, msg, channel):
        # 3. 恢复：典型失败都有明确恢复路径
        if channel not in self.slack_whitelist:
            raise PermissionError("channel not whitelisted")
        for attempt in range(3):
            try:
                return await self.slack.post(channel, msg)
            except RateLimitError:
                await asyncio.sleep(60 * (attempt + 1))   # 指数退避
            except WebhookExpiredError:
                self.queue.enqueue(msg)                   # 落本地队列下次重试
```

> **OpenAI 的极致做法**：把资深工程师经验固化成"Golden Principles"（黄金原则），沉到仓库里 + 后台 Agent 定期扫描对比、自动开修复 PR。

### 2.7 六层不是清单是路标

Agent 犯错时，修复该落到哪一层？

| 现象 | 修哪层 |
|---|---|
| 总是漏掉某个上下文 | ① 上下文精细化 |
| 总是用错工具 | ② 工具系统 |
| 步骤乱 | ③ 任务编排 |
| 跨天记不住进度 | ④ 记忆状态 |
| 没法判断做得好不好 | ⑤ 评估观测 |
| 一失败就崩溃 | ⑥ 约束恢复 |

> Harness 是**路标不是清单**——一次搭不完，犯错一次就加固一层，时间一长它就长出来了。

> **小结**：
> - 六层分三组：**输入侧（①④）+ 动作侧（②③）+ 校验侧（⑤⑥）**。
> - 核心工作流：**取景框 → 工具 → 轨道 → 胶卷 → 尺子 → 护栏**。
> - Mitchell 的"复利" = 每次错误都沉到六层的某一层，永远不留在人脑里。

---

## 3. Harness 的五大工程难点

概念清晰是一回事，落地是另一回事。下面 5 个真实工程难题，看 OpenAI / Anthropic / Cognition 是怎么反常识地解的。

### 3.1 难题一：跑久了越走越偏（Context Anxiety）

**现象**：Agent 一开始表现挺好，跑着跑着开始"忘"——忘目标、忘做过什么、重复劳动、偏离主线。

**反常识观察**（Cognition 用 Claude Sonnet 4.5 重做 Devin 时发现）——模型自己好像也感觉到"快撑不住了"，他们命名为 **Context Anxiety**（上下文焦虑）：
- 模型开始着急收尾、突然简化方案、跳过验证、急匆匆宣布"完成"。
- 而且模型对自己"还剩多少上下文"的估计**非常不准**。

**解法**：**Context Reset > Context Compaction**。
- 朴素思路 = 压缩历史摘要腾空间，Anthropic 博客验证：光压缩不够，Sonnet 4.5 那种"已经累了"的负担感模型会带着。
- 真正解法 = **直接换掉整个上下文窗口**。
- 具体做法：整个系统只有**一个 Agent**，每轮用"增量推进"prompt 启动，状态全部外化到**文件系统**（启动脚本 + git commit + 交接文档）。
- 关键：每轮开始 Agent 面对一个完全干净的窗口，靠读文件系统恢复"我现在在哪一步"。

> 比喻：内存泄漏时的解法不是优化内存，是**重启进程 + 从磁盘恢复状态**。

**原则一**：**重启胜过修补，状态沉到文件里**。

### 3.2 难题二：自我打分总偏乐观

**现象**：让 Agent 自己评估自己，它永远觉得自己干得不错——没标准答案的任务（设计 UI / 写文案 / 评代码可读性）偏差最明显。

**解法**：**生产和验收必须分离**。Anthropic 的三角分工：

| 角色 | 职责 |
|---|---|
| **Planner** | 把模糊需求扩展成完整规格 |
| **Generator** | 一步一步去实现 |
| **Evaluator** | 像 QA 一样**真实操作页面、检查实际运行结果**（不是抽象 review） |

> 别让 Agent 既当运动员又当裁判，也别让它既是厨师又是食客。

**原则二**：**生产验收分家，验收方必须能摸到真实世界**。

### 3.3 难题三：Agent 反复失败，工程师到底该干啥

**反常识**：本能反应"再调提示词 / 换更强模型"——OpenAI 在百万行代码项目里证明**这两个方向都错**。

**真正的工作重心**（Codex 实践）：
1. 把产品目标拆解成 Agent 能力边界内的小任务。
2. Agent 失败时不催它更努力，而是看**环境里缺什么能力**，把能力补进环境。
3. 建立反馈链路，让 Agent **看见自己工作的结果**（lint / 单测 / 真实运行），不是两眼一抹黑瞎跑。

> 思维转换：以前写代码的价值 = "我一天能写多少行"；未来 = "我能为 Agent 设计多好的运行环境"。

**原则三**：**与其催模型不如改环境**。

### 3.4 难题四：规范文件越写越长，Agent 反而更糊涂

**OpenAI 亲自踩的坑**：早期把 AGENTS.md 做成"百科全书"，结果模型注意力被严重稀释。

**解法**：AGENTS.md 从百科全书改造成**目录页**：
- 主文件只保留 **~100 行核心索引**（OpenAI 原文数字）。
- 详细内容拆到子文档（架构 / 设计原则 / 产品规格 / 执行计划 / 质量评分），各主题独立。
- Agent 平时只看目录，需要哪部分才钻进对应子文档。

这就是 **Progressive Disclosure**（渐进式披露）——和软件设计里"按需加载 / 懒加载"一脉相承。

**原则四**：**规则宁缺毋滥，给模型看的东西少即是多**。

### 3.5 难题五：AI slop 越堆越烂，技术债怎么还

**AI slop**（AI 代码泔水）= Agent 疯狂模仿仓库已有模式，好的坏的都复制。早期某段代码写歪，整个代码库开始"腐烂"。

**失败的尝试**：靠人工清理——OpenAI 团队每周五花一整天擦屁股。但 Agent 产出速度 >> 人类清理速度，周五清一天周一又堆满。

**解法**（Harness 思路）：

| 步骤 | 做法 |
|---|---|
| ① 经验显式化 | 把"什么是好代码"的隐性知识写成 **Golden Principles** 沉进仓库（"优先用共享工具包 / 不瞎猜数据格式 / 必须校验边界"） |
| ② Agent 对付 Agent | 后台 Agent 按固定节奏扫描仓库，对比 Golden Principles，找偏离 + 自动开修复 PR，1 分钟内人工审完 auto merge |

> **金句**："技术债像高利息贷款，几乎永远应该每天小额还一点，而不是攒着等某天集中还。"

**原则五**：**技术债天天还，后台 Agent 自动偿还**。

### 3.6 反直觉发现：Agent 用"老技术"反而更稳

OpenAI 在 Codex 项目里发现：Agent 对那些被叫 "boring" 的老技术掌握得最好。三个原因：**组合性好 + API 稳定 + 训练数据多**。

极端例子：有时宁愿让 Agent 自己写小工具函数，也不引入流行 npm 包——自己写的代码 Agent 能 100% 理解和控制，第三方包藏着 Agent 看不懂的黑盒。

> 选型启发：**越老、文档越齐、社区沉淀越久的技术，Agent 越容易做对**。

### 3.7 五条原则口诀

> **重启胜过修补，生产验收分家，与其催模型不如改环境，规则宁缺毋滥，技术债天天还。**

> **小结**：
> - 五大难点的解法共同点：**不在调模型，而在设计模型外面那套环境**——这就是 Harness 的灵魂。
> - Context Anxiety 揭示的本质：长链路任务里**重启 > 修补**。
> - "催模型努力"和"换更强模型"是反方向的两条死路，**改环境**才是正道。

---

## 4. 可迁移设计原则

把整套 Harness 思想浓缩成 7 条可抄进自己项目里的原则：

| 原则 | 落点 |
|---|---|
| **① 错误回收是复利** | Agent 每次失败，把修复沉到环境里（rules / linter / test / hook），不留人脑 |
| **② 状态外化到文件系统** | 不在上下文窗口里塞状态，靠"启动脚本 + 交接文档 + git"恢复进度 |
| **③ 上下文腐化要预防** | 规则文件用目录页（~100 行）+ 子文档，不要做百科全书 |
| **④ 工具按需接 + 结果二次提炼** | 30 条匹配先提炼再喂回，避免上下文污染 |
| **⑤ 评估要靠 Eval 集** | 改 Harness 任何一处都跑一遍 Eval 集，靠数据不靠感觉 |
| **⑥ 生产验收必须分家** | Planner / Generator / Evaluator 三角分工，验收方要能摸真实环境 |
| **⑦ 技术债天天还** | 经验显式化成 Golden Principles，后台 Agent 自动跑、自动开 PR |

> **小结**：
> - 7 条原则都能直接抄进自己的 agent 项目。
> - 精髓 = **不堆复杂度**，把"环境设计"做到位。
> - 未来程序员的主战场：**写规则 / 写流程 / 设计环境**，不是亲手写代码。

---

## 5. 面试回答套路（4 段式）

1. **澄清 + 定性**：LLM 能力到顶了，落地难——核心是"如何让模型在长链路里持续做对"，引出 Harness = Agent − Model。
2. **脉络 + 分类**：三次重心迁移（Prompt → Context → Harness），三者是包含关系（Prompt ⊂ Context ⊂ Harness）。
3. **架构 + 拆解**：六层 = ①上下文 ②工具 ③编排 ④记忆 ⑤观测 ⑥护栏；提一两个核心组件（如工具 MCP 协议、状态外化文件系统、Eval 集）。
4. **难点 + 原则**：五大难题 + 五条原则口诀（重启胜过修补 / 验收分家 / 改环境 / 宁缺毋滥 / 天天还债）。

**加分项**：提到 Context Anxiety、Context Reset vs Compaction、Progressive Disclosure、Golden Principles、Eval-driven iteration。

> **小结**：
> - 60 分答案：Harness = 提示词 + 上下文。
> - 95 分答案：4 段式 + 六层架构 + 五大难题 + 五条口诀 + 提到 `Agent = Model + Harness` 等式。

---

## 6. 复习清单

1. **Harness 词源和核心定义？** Mitchell Hashimoto 2026-02 提出；"每次 Agent 犯错都把修复沉到环境里"。
2. **`Agent = Model + Harness` 等式的含义？** Harness = Agent − Model，除模型外决定能否稳定交付的所有东西。
3. **AI 工程三次重心迁移？** Prompt（让模型听懂）→ Context（让模型知道）→ Harness（让模型做对），层层包含。
4. **Harness 六大核心组件？** ①上下文精细化 ②工具系统 ③任务编排 ④记忆状态 ⑤评估观测 ⑥约束恢复。
5. **Context rot（上下文腐化）是什么？** 上下文塞太满导致注意力涣散，模型开始前后矛盾、忽略规则。
6. **Just-in-time retrieval / Progressive Disclosure？** 不一上来塞满，目录常驻 + 详细文档按需加载。
7. **MCP 是什么？** Model Context Protocol，工具调用的标准协议（"USB-C for tools"）。
8. **ReAct 循环是什么？** 思考→行动→观察→再思考的 for 循环，是 Agent 的基本骨架。
9. **状态为什么外化到文件系统？** 避开 Context Anxiety，干净的上下文窗口靠读文件恢复"到哪一步"。
10. **Context Reset vs Context Compaction？** Reset = 整个换掉上下文窗口；Compaction = 只压缩历史。Anthropic 验证 Reset 才能解 Sonnet 4.5 的"上下文焦虑"。
11. **Planner / Generator / Evaluator 三角分工？** 规划 / 生成 / 验收分家，Evaluator 必须能操作真实环境而非抽象 review。
12. **OpenAI 怎么治 AI slop？** Golden Principles 显式化经验 + 后台 Agent 自动扫描对比 + 自动开 PR；技术债天天还而非攒着集中还。
