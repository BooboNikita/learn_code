# Agent 记忆机制

> 一句话总结：**Claude Code 的 agent 记忆 = 静态层（CLAUDE.md 6 层级声明式指令）+ 动态层（结构化文件 + 小模型做选择题）**，用「土到反直觉」的磁盘 markdown，治了业界向量检索方案的四个病根。

---

## 0. 为什么 LLM 不需要"记忆"机制

LLM 本身**无状态**——每次调用都是「把 system prompt + 历史消息 + 当前问题」重新塞进上下文窗口。本质上客户端在「代它记忆」。

短期记忆 = 上下文窗口本身（满了就 compact）。

**agent 真正缺的是长期记忆**——能跨会话活下来、跟"对话历史"不是一回事的 4 类东西：

| 类别 | 示例 |
|---|---|
| **用户画像** | 十年 Go 后端，刚接触 React |
| **行为偏好** | 测试别用 mock，要打真实数据库 |
| **项目动态** | 移动端 3 月 5 号开始合并冻结 |
| **外部指针** | pipeline bug 都在 Linear 的 INGEST 项目里追踪 |

光把对话存起来 + RAG 检索，**根本解决不了这四类问题**。

> **小结**：
> - LLM 无状态，所谓"记忆"是客户端把历史塞回上下文。
> - agent 真正缺的是**长期记忆**：用户画像 / 行为偏好 / 项目动态 / 外部指针。
> - 这四类东西跟"对话历史"是两码事，RAG 也救不回来。

---

## 1. 业界 4 类方案为什么不够看

| 方案 | 思路 | 硬伤 |
|---|---|---|
| **滑动窗口** | 保留最近 N 轮，其余丢弃 | 关键信息与无关信息混丢（第 1 轮"我是后端"早被砍） |
| **对话摘要** | 旧对话用 LLM 总结，摘要塞回上下文 | 重要细节被压糊（"我们用 Kong" → "讨论了技术栈"） |
| **向量检索** | 转 embedding 存向量库，相似度召回 top-K | ① 相似≠相关 ② 召回不稳 ③ 工程量大 ④ 用户不可读 |
| **分层存储**（MemGPT/Letta） | core/recall/archival 三层，LLM 自主搬运 | 概念复杂、迁移成本高，搬数据仍靠 embedding |

**共同病根**（Claude Code 一一击破）：

1. **自由文本无约束** → 存什么没规则，记忆库变垃圾堆。
2. **不区分类型** → 用户画像/项目动态一锅炖，都查不准。
3. **没有老化机制** → 半年后"Kong 已换 nginx"，旧记忆是"权威的错误"。
4. **重检索、轻写入** → 都在琢磨"怎么查"，放任"该不该存、存什么"。

> **小结**：
> - 4 类方案的硬伤**互不重叠但病根相同**。
> - 向量检索不是"错"，但**人脑读不懂 + 召回不稳 + 维护成本高**让它在生产里很痛。

---

## 2. Claude Code 的两层架构

Claude Code 走了"土到反直觉"的路——**不用向量数据库、不用 embedding、不用任何复杂存储引擎，用磁盘上的 markdown 文件**。

| 层级 | 本质 | 来源 | 加载时机 |
|---|---|---|---|
| **静态层** | CLAUDE.md 体系（声明式指令） | 人写 | agent 启动时全量加载 |
| **动态层** | 自动记忆系统（学习式偏好） | agent 跟人互动中学 | 每轮结束后台抽取 + 按需检索注入 |

- 静态层 = 工位上的"公司员工手册"，新员工入职必看。
- 动态层 = 工位旁的"自己的工作笔记"，慢慢摸索出来的。

> **小结**：
> - 静态层管**确定性规则**（人写、启动加载）。
> - 动态层管**学到的事实**（agent 自动写、按需检索）。
> - 两层并行工作，互不覆盖。

---

## 3. 静态层：CLAUDE.md 的 6 个层级

一个 CLAUDE.md 不够——规则来源不同、可见范围不同、谁能改也不同。Claude Code 按加载顺序拆成 6 层：

| 层级 | 范围 | 谁维护 |
|---|---|---|
| **Managed Policy** | 组织级强制策略 | 公司下发 |
| **Local CLAUDE.md** | 个人偏好，跨项目通用 | 改自己机器 |
| **Project CLAUDE.md** | 项目根目录 | 团队共享，签入 git |
| **Auto MEMORY.md** | agent 自己学到的用户偏好 | agent 自动写 |
| **Team MEMORY.md** | 团队摸索出的协作经验 | 团队共享 |
| **Local MEMORY.md** | 本地调试约定 | 自己机器 |

六层之间是**叠加而非覆盖**，全部拼进 system prompt。

### 3.1 @include：让 CLAUDE.md 互相引用

```markdown
# 公司项目通用 CLAUDE.md
@~/company/security-rules.md
@~/company/coding-style.md

## 项目特定说明
...
```

加载时自动把 `@include` 文件内容读进来拼上——跟 C 的 `#include` 一模一样，背后还防了循环引用和路径遍历。

### 3.2 条件规则：编辑 .tsx 才加载前端规范

`.claude/rules/frontend.md`：

```markdown
---
name: 前端规范
description: React + Tailwind 项目规范
paths: ["**/*.tsx", "**/*.jsx"]
---
# 前端规范
...（规则正文）
```

`paths` 字段用 glob 匹配当前编辑文件，**匹配上才拼进 system prompt**——大项目几十条规则，token 几乎不浪费。

### 3.3 截断双保险：防"长行索引炸弹"

```typescript
export const MAX_ENTRYPOINT_LINES = 200
export const MAX_ENTRYPOINT_BYTES = 25_000
```

源码注释里记录的真实事故：197KB / 不到 200 行的 MEMORY.md——光限行数完全察觉不到。**行数 + 字节双限制**任意一个触发就截断，并主动追加"索引被截过"的警告。

> **小结**：
> - 6 层级解决了**多源规则**的可见范围问题。
> - `@include` 解耦公共内容，`paths:` 解耦**按需加载**。
> - 用户可控文本进 system prompt 都要"行 + 字节"双保险。

---

## 4. 动态层：自动记忆系统的完整闭环

### 4.1 四种类型 + 强制结构

只允许 4 种类型，其他一律不许写：

```typescript
export const MEMORY_TYPES = [
  'user',      // 用户画像
  'feedback',  // 行为偏好
  'project',   // 项目动态
  'reference', // 外部指针
] as const
```

`feedback` 和 `project` 强制三段结构：

```markdown
**Why:** 上季度 mock 测试通过但 prod 迁移挂了
**How to apply:** 所有标了"集成测试"的 case 都适用
```

`project` 还要把**相对日期转绝对日期**（"周四冻结" → "2026-03-05 冻结"），避免几天后过期。

### 4.2 存储设计：单文件 + 索引

每条记忆一个独立 `.md`，头部 YAML frontmatter 是"身份证"：

```markdown
---
name: 不要用 mock 数据库
description: 集成测试必须连真实数据库
type: feedback
---
集成测试必须连真实数据库，不要用 mock。
**Why:** 上季度 mock 测试通过但 prod 迁移挂了
**How to apply:** 所有标了"集成测试"的 case 都适用
```

`MEMORY.md` 是所有记忆的索引清单，**始终加载进 system prompt**；独立记忆文件**按需加载**——目录常驻、正文按需。

### 4.3 写入：Extract Memories 代理

主对话不分心，**每轮 query loop 结束后**后台跑一个 `extractMemories` 代理，通过 stopHook 触发。

伪代码：

```typescript
// 完美 fork 主对话，复用 prompt cache
const memoryAgent = runForkedAgent(parent, { name: 'extractMemories' })

// 给代理的指令（精简）
const prompt = `
分析这次对话历史，判断有没有值得长期记住的信息。
- 只允许 4 种类型：user / feedback / project / reference
- 跟现有记忆比对，过滤掉 hasMemoryWritesSince 里刚写过的
- 不确定就不写
`

// 代理输出要写入的文件清单 + YAML + 正文
memoryAgent.run(prompt).then(writeMemoryFiles)
```

复用父对话的 prompt cache → 多花的钱**极少**。这是治"重检索轻写入"病根的关键。

### 4.4 检索：用 Sonnet 选 top-5

**反直觉**：不用向量检索，让小模型当"选择题选择器"。

```typescript
async function findRelevantMemories(query, allMemories, alreadySurfaced) {
  // 1. 只读每个文件前 30 行，提取 frontmatter（name + description）
  const headers = allMemories.map(readFirst30Lines)

  // 2. 拼成清单发给 Sonnet
  const prompt = `
    Query: ${query}
    Available memories:
    ${headers.join('\n')}

    Only include memories that you are certain will be helpful
    based on their name and description. Be selective and discerning.
    返回 JSON: { files: ["feedback_no_mock.md", ...] }
  `

  // 3. 两个过滤：alreadySurfaced（去重）+ recentTools（去重复噪音）
  return sonnetCall(prompt).filter(notIn(alreadySurfaced))
}
```

为什么是 Sonnet 不是 Haiku？**记忆相关性判错的代价远大于多花的那点钱**——错的记忆塞进上下文会污染整个回复。

### 4.5 注入：system-reminder 包裹 + 老化警告

```xml
<system-reminder>
This memory was saved 5 days ago. Verify it's still accurate before acting on it.

[记忆内容]
</system-reminder>
```

| 记忆时间 | 警告策略 |
|---|---|
| 今天 / 昨天 | 不警告 |
| **2 天以前** | 主动加"这是过去的快照" |

软件开发节奏快，2 天前的"项目正在做 X"可能今天已改。模型见到记忆**自动带着"这是历史"心态**去用，必要时 grep 一下当前文件验证——**治"权威的错误"病根**。

> **小结**：
> - 闭环 = 4 类型强约束 + 索引常驻 + Extract 后台代理 + Sonnet 选 top-5 + system-reminder 注入 + 老化警告。
> - 关键设计：**没有一行 embedding、没有一个向量数据库**，全是结构化文件 + 小模型当选择器。
> - 治的 4 个病根：① 类型化约束写入 ② 索引/正文分层 ③ 2 天警告老化 ④ Extract 代理治"重检索轻写入"。

---

## 5. 4 条可借鉴的设计原则

| 原则 | 对应设计 | 迁移建议 |
|---|---|---|
| **结构化优于自由文本** | 4 类型 + 强制 frontmatter | 给记忆定 schema，强制每条填齐 |
| **索引常驻 + 内容按需** | MEMORY.md 索引常驻，正文按需加载 | 长 RAG 文档 / tool 列表 / 历史 PR 都可套 |
| **廉价模型做选择题** | Sonnet 选 top-5，不用相似度计算 | 候选 < 几百时，小模型选择 > 向量检索 |
| **时间感知 + 主动验证** | 2 天前记忆加警告 + 提示词提醒"≠ 现在存在" | 记忆不是真理是 git log，模型要主动验证 |

**核心思维转换**：向量检索把检索当"数学题"（相似度 = 0.87 是相关吗？阈值难定）；Claude Code 把检索当"选择题"（自然语言判断，可解释、可调试）。

> **小结**：
> - 4 条原则都能直接抄进自己的 agent 项目。
> - 精髓 = **不堆复杂度**，把"文件系统 + LLM"组合出比向量检索更好用的东西。

---

## 6. 面试回答套路（3 段式）

1. **澄清问题 + 定性**：先说 LLM 无状态，长期记忆本质是"在工位上贴便签"，核心问题是"贴在哪、谁来贴、什么时候撕"。
2. **破题 + 反例**：点出 4 类方案的共同病根（自由文本无约束 / 不区分类型 / 无老化 / 重检索轻写入）；举 Claude Code 作反例——不用向量库，用"结构化文件 + LLM 选择"。
3. **原则 + 落地**：亮 4 条可迁移设计原则（结构化 / 索引常驻 / 廉价模型 / 时间感知），证明你能把思路用到自己项目里。

> **小结**：
> - 60 分答案：上向量数据库存 embedding 做相似度检索。
> - 95 分答案：3 段式 + 提到"权威的错误"和"重检索轻写入"两个病根的解法。

---

## 7. 复习清单

1. **LLM 为什么没记忆？** 无状态，每次调用都要把历史塞回上下文。
2. **agent 真正缺哪种记忆？** 长期记忆：用户画像 / 行为偏好 / 项目动态 / 外部指针。
3. **滑动窗口的硬伤？** 关键信息与无关信息混丢。
4. **对话摘要的硬伤？** 重要细节被压糊（如"Kong"被压成"讨论了技术栈"）。
5. **向量检索 4 大硬伤？** 相似≠相关、召回不稳、维护成本高、用户不可读。
6. **4 类方案共同病根？** 自由文本无约束、不区分类型、无老化机制、重检索轻写入。
7. **CLAUDE.md 为什么拆 6 层？** 规则来源不同、可见范围不同、谁能改不同。
8. **@include 作用？** 跨文件引用公共规则，避免复制粘贴。
9. **条件规则的妙处？** 按 `paths` glob 匹配，按需加载省 token。
10. **截断双保险？** 行数 + 字节双限制，防"长行索引炸弹"（197KB / 200 行的事故）。
11. **动态层 4 种记忆类型？** user / feedback / project / reference。
12. **为什么用 Sonnet 不用 Haiku 做检索？** 记忆选错的污染代价远大于多花的钱。
13. **2 天前记忆为什么加警告？** 防"权威的错误"，让模型主动验证。
14. **可借鉴的 4 条原则？** 结构化优于自由文本、索引常驻+内容按需、廉价模型做选择题、时间感知+主动验证。
