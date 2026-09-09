# Claude Code 记忆机制

> 一句话总结：**Claude Code 的记忆 = CLAUDE.md（人写的静态指令）+ Auto Memory（Claude 自己记的动态笔记）+ Auto Dream（空闲时后台整理的"REM 睡眠"）**，全程用纯 markdown 文件 + 索引实现，没有一个向量数据库。

> 数据截止：2026 年 8 月。Dreams / Auto Dream 仍处于 Research Preview，特性细节以官方文档为准。

> 姊妹篇：《[Agent 记忆机制](../大语言模型/Agent记忆机制.md)》从源码架构角度拆解了同一套系统（为什么不用向量检索、写入/检索/注入的内部实现），本篇侧重官方机制、时间线与实战用法。

---

## 一、总体架构：四套记忆机制

LLM 本身无状态，每个 Claude Code 会话都从全新的上下文窗口开始 [cite:1]。跨会话传递知识靠四套互补机制 [cite:2]：

| 机制 | 谁来写 | 作用 | 运行时机 | 存储位置 |
|---|---|---|---|---|
| **CLAUDE.md** | 你 | 指令与规则 | 会话启动时全量加载 | `./CLAUDE.md` 等 |
| **Auto Memory** | Claude（会话中） | 项目模式与学到的经验 | 每个会话持续写 | `~/.claude/projects/<项目>/memory/` |
| **Session Memory** | Claude（后台） | 会话摘要，compact 后保持连续性 | 会话中后台维护 | 会话目录下 |
| **Auto Dream** | Claude（定期） | 记忆去重、修剪、重组 | 空闲时（24h + 5 会话） | 同 Auto Memory |

人类类比：**员工手册**（CLAUDE.md）、**白天随手记的工作笔记**（Auto Memory）、**短期回忆**（Session Memory）、**REM 睡眠整理记忆**（Auto Dream）[cite:2]。

再加上"上下文窗口 + auto-compact + context editing"这套**会话内短期记忆管理**（见第五节），构成完整图景。

### 演进时间线

| 时间 | 事件 |
|---|---|
| 2025-09-30 | 平台层发布 context editing + memory tool（配合 Sonnet 4.5，支撑长时 agent 任务）[cite:4] |
| 2026-02 | Claude Code 推出目录制 Auto Memory，默认开启（随 Opus 4.6 部署；第三方对版本号记录不一：v2.1.32 / v2.1.59）[cite:3][cite:7] |
| 2026-04 | Managed Agents API 的 Dreams 原语进入 Research Preview（beta 头 `dreaming-2026-04-21`）[cite:8] |
| 2026 上半年 | Claude Code 内置 Auto Dream（未正式官宣，`/memory` 面板中可见 "Auto-dream: on"）[cite:2] |
| 2026-08-26 | Claude 应用层打通 chat ↔ Cowork 记忆，"同一套记忆跨场景调用"（生态趋势）[cite:9] |

---

## 二、CLAUDE.md：你写给 Claude 的静态记忆

CLAUDE.md 是 markdown 文件，你手写、Claude 每个会话开始时读取。**它是上下文，不是强制配置**——写得越具体、越简洁，遵循率越高 [cite:1]。

### 2.1 三个官方作用域

| 作用域 | 位置 | 用途 | 共享对象 |
|---|---|---|---|
| **Managed policy** | macOS: `/Library/Application Support/ClaudeCode/CLAUDE.md`；Linux: `/etc/claude-code/CLAUDE.md` | 组织级强制下发（合规、安全规范） | 机器上所有用户，无法被个人设置排除 |
| **Project** | `./CLAUDE.md` 或 `./.claude/CLAUDE.md` | 团队共享的项目规范 | 通过版本控制共享给团队 |
| **User** | `~/.claude/CLAUDE.md` | 个人偏好，跨所有项目 | 仅自己 |

社区资料还列出第四作用域 `./CLAUDE.local.md`（gitignore 的个人本地指令）[cite:3]，官方新文档已淡化它，主推"项目 CLAUDE.md + `@import` 指向个人文件"的组合 [cite:1]。源码视角的 6 层级拆解（含 Team/Local MEMORY.md）见《Agent 记忆机制》§3。

### 2.2 加载机制：向上遍历 + 按需加载 + 拼接不覆盖

- Claude 从当前工作目录**逐级向上**收集 CLAUDE.md，全部**拼接**进上下文——不是"高层覆盖低层"的优先级链 [cite:1]。
- 子目录中的 CLAUDE.md **不会**启动即加载，而是在 Claude 读到该目录的文件时才注入 [cite:1]。
- 块级 HTML 注释（`<!-- 维护者注记 -->`）在注入前被剥离，给维护者留话不耗 token；代码块内的注释保留 [cite:1]。
- monorepo 中可用 `claudeMdExcludes` 跳过其他团队无关的 CLAUDE.md [cite:1]。
- `--add-dir` 附加目录默认不加载其 CLAUDE.md，需设 `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=1` [cite:1]。

### 2.3 @import：模块化拆分

```markdown
See @README for project overview and @package.json for npm commands.

# 个人偏好（不入库）
- @~/.claude/my-project-instructions.md
```

- 相对路径相对**包含 import 的文件**解析，不是工作目录 [cite:1]。
- 递归导入**最多 5 跳** [cite:1]。
- 首次遇到项目内外部 import 会弹审批对话框，拒绝后不再询问 [cite:1]。
- **AGENTS.md 互操作**：Claude Code 只读 CLAUDE.md；仓库若已有 AGENTS.md（其他 agent 生态通用），写一个 `@AGENTS.md` 导入即可，两边共享同一份指令 [cite:1]。

### 2.4 `.claude/rules/`：按路径条件加载

```
your-project/.claude/rules/
├── code-style.md      # 无 paths → 启动即加载
├── testing.md
└── api-design.md      # 有 paths → 匹配到文件才加载
```

带 `paths` frontmatter 的规则只在 Claude 触碰匹配文件时加载 [cite:1]：

```markdown
---
paths:
  - "src/api/**/*.{ts,tsx}"
---
# API 开发规范
- 所有端点必须做输入校验
```

- 支持 symlink，可把一套公共规则链接进多个项目；循环 symlink 会被优雅处理 [cite:1]。
- 用户级 `~/.claude/rules/` 对全机项目生效，先于项目规则加载（项目规则优先级更高）[cite:1]。
- **规则 vs Skills**：需要常驻上下文的用 rules；任务型、只在用到时才需要的指令用 Skills（按需加载）[cite:1]。

### 2.5 写好 CLAUDE.md 的官方建议 [cite:1]

1. **尺寸**：每个文件目标 200 行以内，超长消耗上下文且降低遵循率。
2. **具体可验证**："用 2 空格缩进" 而非 "格式规范"；"提交前跑 `npm test`" 而非 "做好测试"。
3. **结构化**：markdown 标题 + 列表分组，Claude 和人一样按结构扫描。
4. **一致性**：两条规则冲突时 Claude 会随机选一条，定期清理过时/冲突规则。
5. `/init` 可自动生成初始 CLAUDE.md（分析代码库产出构建命令、测试指令、项目约定）；设 `CLAUDE_CODE_NEW_INIT=true` 启用交互式多阶段流程（选产物 → 子代理探索 → 补问 → 出提案再写盘）[cite:1]。

### 2.6 大团队治理：settings 管硬约束，CLAUDE.md 管行为

| 关注点 | 配置在哪 |
|---|---|
| 封禁工具/命令/路径 | Managed settings: `permissions.deny` |
| 强制沙箱隔离 | Managed settings: `sandbox.enabled` |
| 代码风格、合规提醒、行为指令 | Managed CLAUDE.md |

**settings 规则由客户端强制执行，无论 Claude 怎么决定；CLAUDE.md 只是塑造行为，不是硬约束层** [cite:1]。托管 CLAUDE.md 用 MDM / Group Policy / Ansible 下发 [cite:1]。

---

## 三、Auto Memory：Claude 自己记的动态笔记

### 3.1 是什么

- 2026 年 2 月起随版本默认开启 [cite:3][cite:7]。
- 存储在 `~/.claude/projects/<项目>/memory/`，**按 git 仓库键控**：同仓库的所有 worktree、子目录共享同一份记忆；非 git 目录用项目根 [cite:1][cite:3]。
- **机器本地**：不跨机器、不跨云环境共享，除非自己同步目录 [cite:1]。

### 3.2 目录结构：索引常驻 + 正文按需

```
~/.claude/projects/<project>/memory/
├── MEMORY.md          # 索引：每个会话加载前 200 行或 25KB（先到为准）
├── debugging.md       # 主题文件：启动不加载，按需读取
├── api-conventions.md
└── ...
```

- **MEMORY.md 是索引不是正文**：只有前 200 行 / 25KB 注入每个会话，Claude 靠它知道"存了什么、在哪" [cite:1][cite:3]。
- 主题文件不占启动上下文，Claude 用普通文件工具按需读取 [cite:1][cite:3]。
- 记什么由 Claude 判断"未来对话是否有用"：构建命令、调试洞察、架构笔记、代码风格偏好、工作流习惯；也可以直接说 "remember: 这个项目用 Bun 不用 Node" 让它当场存 [cite:3]。

### 3.3 管理与审计

- **`/memory` 命令**是统一入口：查看当前会话加载了哪些 CLAUDE.md / CLAUDE.local.md / rules 文件；开关 Auto Memory；打开本项目记忆目录；直接编辑任意记忆文件 [cite:1][cite:3]。
- 设置项 `autoMemoryEnabled` 控制开关 [cite:3]。
- 全部是纯 markdown，随时可读、可改、可删 [cite:1]。
- 子代理（subagent）也可维护自己独立的 auto memory [cite:1]。

> 架构内部实现（谁来触发写入、如何检索、如何注入）：每轮 query loop 结束后由后台 Extract 代理提取记忆，检索时让 Sonnet 从索引清单里选 top-5，注入时用 system-reminder 包裹并附带老化警告——详见《[Agent 记忆机制](../大语言模型/Agent记忆机制.md)》§4。

---

## 四、Auto Dream：给记忆做"REM 睡眠"

### 4.1 解决什么问题

Auto Memory 用久了必然**腐化**：矛盾条目堆积、相对日期（"昨天"）失去意义、引用已删除文件的调试笔记还在——本该帮 Claude 记忆的笔记反而变成噪音 [cite:2]。

### 4.2 平台原语：Dreams（Research Preview）[cite:8]

Managed Agents API 官方文档对 Dreams 的定义：

- 输入：一个已有 **memory store** + **1~100 个**历史会话转录。
- 输出：**新的** memory store——重复项合并、过期/被推翻的条目替换为最新值、从转录中提炼新洞察。
- **输入 store 永不被修改**：可以先审查输出、不满意就丢弃。
- 异步 job：`pending → running → completed / failed / canceled`，通常几分钟到几十分钟。
- `instructions` 字段（≤4096 字符）可定制整理方向。

### 4.3 Claude Code 里的 Auto Dream

Auto Dream 是 Dreams 管线在 CLI 侧的自动集成，四阶段流程 [cite:2]：

1. **Orient**：`ls` 记忆目录、读 MEMORY.md、浏览已有主题文件（避免建重复文件）。
2. **Gather**：按优先级收集新信号——日志 > 已漂移的旧记忆 > grep 会话转录（**明确禁止全文通读**，只查已经怀疑重要的窄词）。
3. **Consolidate**：合并新信息进主题文件；相对日期转绝对日期（"昨天决定用 Redis" → "2026-03-15 决定用 Redis"）；删除被推翻的事实；修剪过期记忆；合并重复条目。
4. **Prune & Index**：MEMORY.md 收回 200 行以内——它是索引不是垃圾桶，只放"一行描述 + 指针"。

**触发条件（双门槛，缺一不可）** [cite:2]：

- 距上次整理 **≥ 24 小时**；
- 期间新增 **≥ 5 个会话**。

单日一个长会话不触发（会话不够），两小时内跑 10 个短会话也不触发（时间不够）。实测一次整理 913 个会话约花 8~9 分钟，后台运行无感 [cite:2]。

**安全保证** [cite:2]：

| 保证 | 说明 |
|---|---|
| 项目代码只读 | 整理期间只能写记忆目录，不能碰源码/配置/测试 |
| 锁文件防并发 | 同项目两个实例只有能一个跑整理，防合并冲突 |
| 后台执行 | 不阻塞当前会话，无需等待 |

**手动触发**：直接对 Claude 说 "dream" / "consolidate my memory files"；`/dream` 命令存在但尚未全量推送。大重构之后建议手动触发一次 [cite:2]。

### 4.4 Auto Memory vs Auto Dream [cite:2]

| | Auto Memory | Auto Dream |
|---|---|---|
| 角色 | **采集**新信息 | **维护**已有信息 |
| 时机 | 每个会话中 | 定期（24h + 5 会话） |
| 跳过的代价 | 漏记有用信息 | 记忆质量随时间腐化 |

只开 Auto Memory 不开 Auto Dream = 只记笔记从不整理的记笔记人。两个都要开。

---

## 五、会话内记忆：上下文窗口的管理

跨会话的四套机制之外，**会话内**还有一层短期记忆管理：

- **上下文窗口本身**就是短期记忆，满了触发 auto-compact 压缩。
- **Session Memory**（第三方源码分析 [cite:6]）：后台子代理自动维护一个 markdown 会话笔记，记录当前对话关键信息，主要用途是 **auto-compact 时保持上下文连续性**；社区资料称其约每 5K tokens 后台运行一次，下个会话启动时可加载相关的历史会话摘要 [cite:2]。
- **平台层组合拳**（2025-09-30 发布 [cite:4]，机制详见官方 memory tool 文档 [cite:5]）：
  - **context editing**：上下文逼近阈值时自动清理旧的工具调用结果；清理前 Claude 会收到警告提示，**先把重要信息保存进记忆文件再清理**。
  - **memory tool**：文件式记忆库工具，Claude 可在专记忆目录里创建/读取/更新/删除文件，跨对话持久，开发者自管存储后端。
  - 两者组合让 agent 能跑原本会超出上下文极限的长任务。
- `/compact` 后感觉"指令丢了"？CLAUDE.md 每个会话都会重新加载；先检查规则是否互相冲突、文件是否过长 [cite:1]。

---

## 六、真实限制（用之前必须知道）

| 限制 | 说明 | 缓解手段 |
|---|---|---|
| **MEMORY.md 200 行 / 25KB 上限** | 超出部分启动不加载；索引滚出加载窗口的主题文件，Claude 可能不知道它存在 [cite:3] | 定期修剪索引，保持精简 |
| **索引式检索，非语义检索** | 没有向量相似度兜底："docker-compose 端口映射" 的笔记能否被 "端口冲突" 的问题命中，全看索引措辞能否桥接 [cite:3] | 索引条目写同义词密集的标题（治标） |
| **按仓库隔离** | A 仓库的记忆 B 仓库不可见；官方 issue #36561（跨项目全局记忆）、#21854 已关闭为重复 [cite:3] | 自己同步目录，或外接记忆后端 |
| **机器本地** | 不跨机器、不进云环境 [cite:1] | 自己同步 `memory/` 目录 |
| **CLAUDE.md 是上下文不是配置** | 不被强制执行；写得越长遵循率越低 [cite:1] | 200 行以内、具体、无冲突；硬约束走 settings |

设计哲学一句话：**用"文件系统 + 索引 + LLM 自己判断"替代向量数据库**——为什么这条路成立，见《[Agent 记忆机制](../大语言模型/Agent记忆机制.md)》的四个病根分析。

---

## 七、复习清单

1. **四套记忆机制分别是什么、谁来写？** CLAUDE.md（你）/ Auto Memory（Claude 会话中）/ Session Memory（Claude 后台）/ Auto Dream（Claude 定期整理）。
2. **CLAUDE.md 三个官方作用域？** Managed policy（组织下发）、Project（团队共享）、User（个人全项目）。
3. **CLAUDE.md 怎么加载？** 从工作目录向上逐级收集 + 子目录按需加载 + 全部拼接不覆盖。
4. **@import 的规则？** 相对路径按包含文件解析；递归最多 5 跳；`@AGENTS.md` 一行兼容其他 agent 生态。
5. **rules 的 paths frontmatter 干嘛的？** glob 匹配到文件才加载，按需注入省 token。
6. **Auto Memory 存哪？** `~/.claude/projects/<项目>/memory/`，按 git 仓库键控，worktree 共享，机器本地。
7. **MEMORY.md 的加载上限？** 前 200 行或 25KB（先到为准）；主题文件按需读取。
8. **/memory 能干什么？** 查看加载清单、开关 Auto Memory、打开记忆目录、直接编辑任意记忆文件。
9. **Auto Dream 的触发条件？** 距上次整理 ≥24 小时**且** ≥5 个新会话，双门槛缺一不可。
10. **Auto Dream 四阶段？** Orient → Gather → Consolidate → Prune & Index。
11. **Auto Dream 三条安全保证？** 项目代码只读、锁文件防并发、后台执行不阻塞。
12. **Dreams 原语的关键性质？** 输入 store 永不被修改、吃 1~100 个会话、异步 job、Research Preview。
13. **三大真实限制？** MEMORY.md 200 行/25KB 上限、索引式非语义检索、按仓库隔离不跨项目。
14. **settings 和 CLAUDE.md 的分工？** 硬约束（封禁、沙箱）用 managed settings 强制执行；行为引导用 CLAUDE.md。
15. **context editing 和 memory tool 怎么配合？** 逼近阈值时先警告 Claude 把重要信息存进记忆文件，再清理旧工具结果，支撑长任务。

---

## 参考资料

[cite:1] [官方] How Claude remembers your project（Claude Code 官方记忆文档）
https://docs.anthropic.com/en/docs/claude-code/memory

[cite:2] [二级资料] Claude Code Dreams: Anthropic's New Memory Feature
https://claudefa.st/blog/guide/mechanics/auto-dream

[cite:3] [二级资料] Claude Code Memory: Complete Guide to Persistence（2026-06-17）
https://vectorize.io/articles/claude-code-memory

[cite:4] [官方] Managing context on the Claude Developer Platform（2025-09-30）
https://www.anthropic.com/news/context-management

[cite:5] [官方] Memory tool（API 文档）
https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/memory-tool

[cite:6] [二级资料] Memory & Persistence Systems（learn-claude-code 源码分析）
https://github.com/AsterZephyr/learn-claude-code/blob/main/12-memory-persistence.md

[cite:7] [行业报道] Claude Code v2.1.32 : Auto Memory stocke vos sessions（2026-02-05）
https://ops-imperium.com/news/aiops/claude-code-auto-memory-v2-1-32/

[cite:8] [官方] Dreams（Managed Agents API，Research Preview）
https://platform.claude.com/docs/en/managed-agents/dreams

[cite:9] [行业报道] 之前聊天说完的事，现在 Cowork 也能接上了——Claude 记忆系统升级全解读（2026-08-26）
https://blog.csdn.net/weixin_66526635/article/details/164094764
