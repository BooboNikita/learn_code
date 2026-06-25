# 父子 Agent 通信

> 一句话总结：**Claude Code 的父子 Agent = 父子型多 Agent 模式**——父 Agent 通过工具调用把任务派给**全新独立会话**的子 Agent（不继承父的 token / 工具状态 / 记忆），子 Agent 跑完把字符串结果以 XML 伪装成用户消息回传；**默认单向、团队模式才补齐双向消息驱动**。

---

## 0. 为什么一个 agent 不够用

单 agent = `LLM + 工具 + 循环`，真实项目三大瓶颈：

| 问题 | 后果 |
|---|---|
| **上下文爆炸** | 调研 / 实现 / 评审三阶段内容全塞一个上下文，token 蹭蹭涨 |
| **职责混乱** | 一个 agent 又当研究员又当程序员又当评审员，容易跑偏 |
| **没法并发** | 一个 agent 一次只能做一件事，干等着 |

**Multi-Agent 核心思路 = 老板带团队**：把任务拆给"专家"，每个专家上下文干净、职责清晰、能并行。

| 形态 | 特征 | Claude Code 对应 |
|---|---|---|
| **父子型** | 主 agent 派 subagent 拿结果回来 | 常规 Subagent |
| **平级协作型** | agent 之间通过共享状态互相协作 | — |
| **主从型**（Coordinator-Worker） | 协调者派 worker、收结果、做合成 | Coordinator 模式 |

> **小结**：
> - 单 agent 瓶颈 = **上下文污染 + 串行 + 职责重叠**。
> - 父子型 = 默认形态；Coordinator = 高并发拆解的旗舰形态。
> - Fork Subagent 是父子型的**缓存优化版本**，不算独立形态。

---

## 1. 父子 Agent 的隔离机制

> 隔离 = **工具隔离 + 上下文隔离**。**不是一刀切全隔离，是按字段粒度逐项决策。**

### 1.1 第一维度：工具隔离（三道门）

子 agent 不能继承父 agent 全部工具，走三道"准入"门：

| 门 | 规则 | 目的 |
|---|---|---|
| **通用黑名单** | 禁"派新 subagent / 主动问用户 / 切换规划模式 / 停止其他任务" | 防递归嵌套 + 防子抢主线程权力 |
| **自定义加严** | 用户自写 agent 再加一层黑名单 | 自定义没审核，多防一道更安全 |
| **异步白名单** | 后台 agent 只准用读文件 / 搜代码 / 执行命令 / 编辑 | 白名单比黑名单更保守 |

```typescript
// src/tools/AgentTool/agentToolUtils.ts:70
export function filterToolsForAgent({ tools, isBuiltIn, isAsync, permissionMode }): Tools {
  return tools.filter(tool => {
    if (tool.name.startsWith('mcp__')) return true            // MCP 工具全放行
    if (ALL_AGENT_DISALLOWED_TOOLS.has(tool.name)) return false
    if (!isBuiltIn && CUSTOM_AGENT_DISALLOWED_TOOLS.has(tool.name)) return false
    if (isAsync && !ASYNC_AGENT_ALLOWED_TOOLS.has(tool.name)) return false
    return true
  })
}
```

### 1.2 第二维度：上下文隔离（按字段决策）

直觉方案 A（完全共享）不行：父 agent 读过 file.ts 前 100 行，子接着读到 200 行 → 父的"读到哪"缓存被污染。
直觉方案 B（完全新建）也不行：用户 Ctrl+C 中止信号广播出去，子因新上下文收不到，自顾自继续跑。

**Claude Code 的折中：按字段粒度逐项决策**。

| 决策 | 字段 | 动作 | 原因 |
|---|---|---|---|
| **① 读文件缓存** | `readFileState` | 克隆一份给子 | 防子读改写父视图 |
| **② 写全局状态** | `setAppState` | 设成空操作 `() => {}` | 防双线并发改 UI 状态 |
| **③ 任务注册通路** | `setAppStateForTasks` | 例外保留 | 不然子起的后台 bash 变孤儿 |
| **④ 独立 ID + 深度 +1** | `agentId` / `depth` | 发新 ID、深度 +1 | 防失控嵌套（>5 层强制停） |

```typescript
// src/utils/forkedAgent.ts:345
export function createSubagentContext(parentContext, overrides): ToolUseContext {
  return {
    readFileState: cloneFileStateCache(parentContext.readFileState),   // ①
    setAppState: () => {},                                             // ② 空操作
    setAppStateForTasks: parentContext.setAppStateForTasks
                        ?? parentContext.setAppState,                  // ③ 例外保留
    agentId: overrides?.agentId ?? createAgentId(),                    // ④
    queryTracking: {
      chainId: randomUUID(),
      depth: (parentContext.queryTracking?.depth ?? -1) + 1,           // ④
    },
  }
}
```

> **小结**：
> - 工具隔离 = **三道门**（通用黑名单 / 自定义加严 / 异步白名单）。
> - 上下文隔离 = **按字段粒度决策**（克隆 / 关闭 / 保留 / 计数）。
> - 精髓不是"全隔离"或"不隔离"，而是**对每个状态的语义单独判断**。

---

## 2. 父子 Agent 是怎么通信的

> 默认形态 = **单向通知**（子→父完成通知）；团队模式才升级成**完整双向消息驱动**。**面试必须讲清这个分水岭**。

### 2.1 通信方式对比

| 形态 | 父→子 | 子→父 | 底座 |
|---|---|---|---|
| **默认形态** | ✗（子跑时父不能插话） | ✓（task-notification 伪装成用户消息） | 同步派发 + 长任务自动转后台 |
| **团队（agent-teams）模式** | ✓（SendMessage 往信箱扔字条） | ✓（复用 task-notification） | 异步消息驱动 |

### 2.2 默认形态：派出去，跑完把结果交回来

子跑时父**只能等**，没法中途塞新指令。看起来像一次重型工具调用。

**长任务自动转后台（auto-background）**：子 ≤ 2 分钟前台阻塞等；> 2 分钟自动转后台，父继续干别的；完成时发通知。

```typescript
// src/tools/AgentTool/AgentTool.tsx:72
function getAutoBackgroundMs(): number {
  if (isEnvTruthy(process.env.CLAUDE_AUTO_BACKGROUND_TASKS)
      || getFeatureValue_CACHED_MAY_BE_STALE('tengu_auto_background_agents', false)) {
    return 120_000  // 2 分钟阈值
  }
  return 0
}
```

**完成通知长这样（XML 伪装用户消息）**：

```xml
<task-notification>
  <task-id>agent-a1b</task-id>
  <output-file>/tmp/xxx.txt</output-file>
  <status>completed</status>
  <summary>Agent "Investigate auth bug" completed</summary>
  <result>Found null pointer in src/auth/validate.ts:42...</result>
  <usage>
    <total_tokens>12345</total_tokens>
    <tool_uses>8</tool_uses>
    <duration_ms>34567</duration_ms>
  </usage>
</task-notification>
```

为啥用 XML 不用结构化对象？

1. **LLM 对 XML 友好**——Anthropic 训练时强调 XML 结构化表达
2. **XML 是纯文本**——可直接塞进对话历史
3. **伪装成用户消息**——天然复用 agentic loop 处理逻辑

```typescript
// src/tasks/LocalAgentTask/LocalAgentTask.tsx:197
const message = `<${TASK_NOTIFICATION_TAG}><${TASK_ID_TAG}>${taskId}</${TASK_ID_TAG}>
<${OUTPUT_FILE_TAG}>${outputPath}</${OUTPUT_FILE_TAG}>
<${STATUS_TAG}>${status}</${STATUS_TAG}>
<${SUMMARY_TAG}>${summary}</${SUMMARY_TAG}>
${resultSection}${usageSection}</${TASK_NOTIFICATION_TAG}>`;
enqueuePendingNotification({ value: message, mode: 'task-notification' });
```

### 2.3 团队（agent-teams）模式：父子真正双向对讲

"调函数等返回"是同步思路，无法支持"一边跑一边对讲"（5 分钟代码评审中临时改要求 / 同时指挥 5 个 subagent 怎么办？）。**信箱 + 字条模型**：每个 subagent 有"员工档案"（`LocalAgentTaskState`），里面 `pendingMessages: string[]` 数组就是信箱，子 agent 每轮工具调用结束瞄一眼信箱，把新字条作为"用户消息"注入对话历史。

**父→子：SendMessage 工具（仅团队模式启用）**

```typescript
// src/tools/SendMessageTool/SendMessageTool.ts:800
const task = appState.tasks[agentId]
if (isLocalAgentTask(task) && !isMainSessionTask(task)) {
  if (task.status === 'running') {
    // 正在跑：扔信箱，纯数组追加，4 行实现
    queuePendingMessage(agentId, input.message, context.setAppStateForTasks)
    return { data: { success: true, message: 'Message queued...' } }
  }
  // 已停止：自动从 transcript 唤醒继续干
  const result = await resumeAgentBackground({ agentId, prompt: input.message, ... })
}
```

子 agent 即使已完成，父 agent 仍可 SendMessage 把它**从磁盘 transcript 唤醒**继续干。

> **小结**：
> - 默认形态 = **单向通知**（子→父 task-notification），父不能插话。
> - 团队模式 = **双向消息驱动**（父→子 SendMessage 扔字条 + 子→父 复用通知）。
> - 完成通知伪装成 XML 用户消息是**复用 agentic loop**的关键设计。
> - 子 agent 已停止后被父 SendMessage 会**自动从 transcript 唤醒**。

---

## 3. Fork Subagent：缓存友好的隐藏优化

> **字节级完全相同**才能命中 Anthropic prompt cache（省 ~90% 成本、降低首 token 延迟）。Fork Subagent = 派一个"字节级相同"的分身，**复用父的缓存前缀**。

Claude Code 的 system prompt **上万 token**。每派一个 subagent，如果 system prompt 不一样，API 那边就要从头算一遍。

**Prompt cache 命中条件 = 字节级完全相同**。一个字符不一样、工具列表顺序不一样、空格位置不一样都直接 miss。

**必须字节级对齐 5 项**：① system prompt ② user context ③ system context ④ 工具池 / 模型 ⑤ 对话历史前缀。

**精妙细节：Fork 的 system prompt 生成函数直接返回空串**：

```typescript
// src/tools/AgentTool/forkSubagent.ts:60
export const FORK_AGENT = {
  tools: ['*'],                // 用父的完整工具池
  model: 'inherit',            // 继承父的模型
  getSystemPrompt: () => '',   // 返回空串！直接用父渲染好的字节
  // ...
} satisfies BuiltInAgentDefinition
```

为什么不重新生成？→ 重新生成可能让某个功能开关 / 动态字段值变一个字符，缓存就 miss。**最稳的办法是把父 agent 已渲染的 prompt 字节原样拿过来用**。

| 场景 | 用什么 |
|---|---|
| "Ctrl+F 生成 PR 描述" / "/btw 总结" 等需要父完整上下文 | **Fork** |
| "派 Explore 搜代码" / "派 Plan 规划" 等有明确专业分工 | **常规 subagent** |
| Coordinator 模式 | **不可用**（`isCoordinatorMode() return false`，互斥） |

> **小结**：
> - Fork 适用"**派个分身试试另一条路**"——子 agent 拥有父的全部上下文（对话历史 / system prompt / 工具池）。
> - 5 项必须**字节级一致**（system prompt / user context / system context / 工具池 / 消息前缀）。
> - `getSystemPrompt: () => ''` 是**防止重新生成引入差异**的关键设计。
> - 成本优化（~90% 节省）本身就是**能力边界**——原本不敢派的任务可以放心派了。

---

## 4. Coordinator 模式：真正的多 Agent 并行

> 主 agent **退化成纯协调者**（不读 / 不写 / 不测试，只派 worker、收结果、合成答案），通过**异步 + 消息队列**做到极限并行。

### 4.1 启用条件

**同时满足**编译时 feature 开关 + 运行时 `CLAUDE_CODE_COORDINATOR_MODE=1`：

```typescript
// src/coordinator/coordinatorMode.ts:36
export function isCoordinatorMode(): boolean {
  if (feature('COORDINATOR_MODE')) {                // 编译时开关
    return isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE)  // 运行时变量
  }
  return false
}
```

主 agent 的 system prompt 被强制改写为协调者身份（"You are a **coordinator**. Direct workers to research, implement and verify code changes. Synthesize results and communicate with the user."）。

### 4.2 团队管理 5 大工具

| 工具 | 作用 |
|---|---|
| **TEAM_CREATE** | 派 worker / 创建团队 |
| **TEAM_DELETE** | 解散团队 |
| **SendMessage** | 给已派出的 worker 发后续指令（续命省钱） |
| **SYNTHETIC_OUTPUT** | 合成最终输出给用户 |
| **STOP_WORKER** | 停止跑错方向的 worker |

关键细节：这组工具对 **worker 是黑名单**——worker 拿不到，不能再派人，形成"协调者 + worker"两层扁平结构，**杜绝递归**。

### 4.3 任务流水线 4 阶段

| 阶段 | 谁来做 | 目的 |
|---|---|---|
| **调研** | Workers（并行） | 调查代码库、找文件、理解问题 |
| **合成** | 协调者本人 | 读完 findings、写成具体实现规格 |
| **实现** | Workers（按规格） | 做具体修改、提交 |
| **验证** | Workers（fresh） | 测试改动是否真的工作 |

> **协调者必须"理解"而不能"转发"**——如果只是转发，它就没有存在价值，worker 直接跟用户对话就行。

### 4.4 Continue vs Spawn 决策

| 决策条件 | 选择 | 原因 |
|---|---|---|
| 新任务跟 worker 现有上下文高度相关 | **Continue** | 它已经"知道"那些文件，沟通成本低 |
| 新任务跟 worker 现有上下文无关 / 老 worker 走偏 | **Spawn** | 避免旧上下文干扰判断 |
| **验证**类工作 | **永远 Spawn 新 worker** | 不能让刚写完代码的 worker 验自己 |

### 4.5 Coordinator vs 常规 Subagent

| 维度 | 常规 Subagent | Coordinator 模式 |
|---|---|---|
| 主 agent 角色 | 全能选手 | 纯协调者 |
| subagent 执行 | 同步（2 分钟后才转后台） | 默认异步 |
| 并发程度 | 偶尔并发 | 最大化并发 |
| 适合场景 | 单个任务 + 临时帮手 | 大任务 + 高并发拆解 |
| 系统形态 | 父子树 | 协调者 + worker 扁平层 |

> **小结**：
> - Coordinator = 主 agent 强制退化为协调者，**不读 / 不写 / 不测试**。
> - 团队管理 5 大工具对 worker 是**黑名单**（防递归扁平化）。
> - 协调者必须**亲自合成**，不能当传话筒。
> - 验证类工作**永远派新 worker**，避免自验偏差。

---

## 5. 5 条可迁移设计原则

| 原则 | 落点 |
|---|---|
| **① 上下文隔离按字段粒度做** | 读文件缓存克隆 / 写全局状态关掉 / 任务注册通路保留 / 深度计数 +1 |
| **② 通信走消息，不走函数调用** | 父→子写消息队列 / 子→父 XML 伪装用户消息；天然异步 + 并发 + 持久化 |
| **③ 工具权限分级管控** | 通用黑名单 / 自定义加严 / 异步白名单 |
| **④ 缓存友好是一种架构能力** | Fork Subagent 字节级复用父前缀，省 80-90% 成本 |
| **⑤ 并行优先 + 协调者合成** | 异步 + 消息队列做基础，协调者亲自合成而非转发 |

> **小结**：
> - 5 条原则都能直接抄进自己的 agent 项目。
> - 精髓 = **不堆复杂度**，每个维度做精细设计。
> - 面试时拿这 5 条去对照任意 Multi-Agent 系统，能迅速看出深浅。

---

## 6. 面试回答套路（4 段式）

1. **澄清 + 定性**：单 agent 三大瓶颈（上下文污染 / 串行 / 职责重叠）→ Multi-Agent = 老板带团队。
2. **破题 + 分类**：三种 Multi-Agent 形态（父子 / 平级 / 主从）→ 对应到 Claude Code 三套机制（常规 / Fork / Coordinator）。
3. **机制 + 代码**：讲清两条通信分水岭——**默认单向、团队模式才补齐双向**。配以 `filterToolsForAgent` / `createSubagentContext` / `queuePendingMessage` 三段伪代码佐证。
4. **原则 + 落地**：亮 5 条可迁移设计原则，证明能用到自己项目里。

> **小结**：
> - 60 分答案：Multi-Agent = 主 agent 派子 agent，串行等结果。
> - 95 分答案：4 段式 + 提到"**默认单向 vs 团队双向**"分水岭 + "**按字段粒度做隔离**" + "**协调者必须合成**"。

---

## 7. 复习清单

1. **单 agent 三大瓶颈？** 上下文爆炸、职责混乱、没法并发。
2. **工具隔离的三道门？** 通用黑名单 / 自定义 agent 加严 / 异步 agent 白名单。
3. **createSubagentContext 四个关键决策？** 读文件缓存克隆 / 写全局状态关掉 / 任务注册通路保留 / 深度 +1。
4. **默认通信形态？** 单向（子→父 task-notification 伪装用户消息）。
5. **auto-background 阈值？** 2 分钟（feature 门控 + 环境变量）。
6. **完成通知为啥用 XML？** LLM 对 XML 友好 / 纯文本可塞对话历史 / 伪装用户消息复用 agentic loop。
7. **团队模式怎么升级到双向？** 父→子用 SendMessage 工具往 `pendingMessages` 信箱扔字条。
8. **子 agent 已停止后父 SendMessage 怎么办？** 从磁盘 transcript 恢复完整对话历史，唤醒继续干。
9. **Fork Subagent 核心思路？** 派"字节级相同"的分身，复用父的 prompt cache（5 项必须字节一致）。
10. **Fork 的 system prompt 生成函数为啥返回空串？** 防止重新生成引入差异导致缓存 miss。
11. **Coordinator 模式主 agent 做什么？** 只派 worker、收结果、合成答案（不读 / 不写 / 不测试）。
12. **5 条可迁移设计原则？** 按字段隔离 / 消息驱动 / 工具分级 / 缓存友好 / 并行优先 + 协调者合成。
