# DeepSeek Harness 解析：从技术架构看"一切皆插件"

> 来源：DeepSeek 官方文档、Cordis 论文、CSDN 架构拆解等公开资料（2026-08）
> 一句话总结：**DeepSeek Harness 是基于 Cordis 微内核的开源 Agent 运行框架**，技术上的核心创新是"一切皆插件"（连 Agent 主循环本身都是可替换插件）+ 只追加可追溯日志——没有需要 patch 的特权核心，九类能力平面全部可配置替换，这是它与 Claude Code、Codex 等"单体核心 + 扩展"架构的本质区别。

## 一、技术定位：Model + Harness = Agent

先澄清一个易混点：**Harness 不是模型，是套在模型外面的执行层**，负责工具调度、会话管理、记忆、任务闭环、错误重试。官方公式：

> **`Model + Harness = Agent`** — 模型负责思考，Harness 负责执行。[$TRAE_REF](https://www.deepseek.com/harness/en/)

这与本仓[《Harness 工程化》](../大语言模型/Harness工程化.md)的等式 `Agent = Model + Harness`（Harness = Agent − Model）一致——模型决定天花板，Harness 决定能不能落地。区别在于：那篇讲的是 Harness 的**通用六层理论**，本篇讲的是 DeepSeek 把这套理论**工程化成一个具体开源框架**的技术实现。

| 维度 | 本仓《Harness 工程化》 | DeepSeek Harness（本篇） |
|------|----------------------|--------------------------|
| 性质 | 理论框架（六层模型） | 工程实现（开源框架） |
| 核心 | ①上下文 ②工具 ③编排 ④记忆 ⑤观测 ⑥护栏 | Cordis 内核 + 八层可插拔平面 |
| 落脚 | 设计原则与口诀 | 插件装配、运行模式、可追溯机制 |

---

## 二、Cordis 微内核：插件元框架

DeepSeek Harness 的底层不是自研框架，而是基于 [Cordis](https://github.com/cordiverse/cordis) 插件系统构建。Cordis 是一个"插件元框架"（meta-framework），配套论文《Cordis: A Meta-Framework of Spatiotemporal Composability》。[$TRAE_REF](https://www.deepseek.com/harness/en/)[$TRAE_REF](https://rohitraj.tech/en/notes/deepseek-harness-vs-claude-code-codex-cli-2026)

### 2.1 内核只做三件事

Cordis 内核**不含任何模型 / 工具 / UI 逻辑**，只做三件事：[$TRAE_REF](https://deepseek.csdn.net/6a7e815110ee7a33f29acc81.html)

| 职责 | 说明 |
|------|------|
| **插件生命周期管理** | 加载 / 启动 / 停止 / 卸载 |
| **插件间依赖注入** | 声明依赖关系，内核负责装配顺序 |
| **全局事件总线** | 插件间通信的统一通道 |

### 2.2 插件协作三件套：service + typed event + reversible effect

Cordis v4 提供三种协作原语，所有 Harness 业务组件都通过它们协作：[$TRAE_REF](https://deepseek.csdn.net/6a7e815110ee7a33f29acc81.html)[$TRAE_REF](https://rohitraj.tech/en/notes/deepseek-harness-vs-claude-code-codex-cli-2026)

| 原语 | 作用 | 类比 |
|------|------|------|
| **Service（服务）** | 插件向内核注册能力，其他插件按需调用 | 微服务注册中心 |
| **Typed event（类型化事件）** | 插件间收发结构化消息 | 强类型事件总线 |
| **Reversible effect（可逆副作用）** | 插件卸载时，所有注册被"反向撤销"，不留孤儿状态 | 数据库事务回滚 |

> **可逆副作用是热插拔的底层保证**：传统插件系统卸载时常残留注册项（事件监听、定时器、文件句柄），污染运行时；Cordis 保证卸载即完全回滚，这正是"换掉任意能力不留痕迹"的技术前提。

### 2.3 没有特权核心

最关键的设计结论：**Harness 没有需要 patch 的特权核心**。连"决定下一步做什么"的 Agent 主循环（loop）本身都是可替换插件，所有改动在配置层完成，不碰源码。[$TRAE_REF](https://deepseek.csdn.net/6a7e815110ee7a33f29acc81.html)

> 这正是对比 Claude Code / Codex 的核心差异：后两者把扩展点锁死在核心里，想换模型、换工具、加审计往往只能 fork；Harness 的扩展路径是"挂插件，不改源码"。[$TRAE_REF](https://deepseek.csdn.net/6a7e815110ee7a33f29acc81.html)

---

## 三、HPA 模型：八层可插拔业务平面

CSDN 架构拆解把 Harness 抽象成 **HPA 模型（Harness Pluggable Architecture）**，便于理解它和单体 Agent 的本质区别：[$TRAE_REF](https://deepseek.csdn.net/6a7e815110ee7a33f29acc81.html)

```
┌─────────────────────────────────────────┐
│  Cordis 内核（生命周期 / 依赖注入 / 事件总线）│  ← 无业务逻辑
├─────────────────────────────────────────┤
│ ① Model    ② Tool     ③ Skill           │
│ ④ Session  ⑤ Sandbox  ⑥ Storage          │  ← 八层业务平面，全可替换
│ ⑦ Loop     ⑧ Scheduling                   │
├─────────────────────────────────────────┤
│ UI（呈现层，独立挂载）                       │
└─────────────────────────────────────────┘
```

八层对应本仓[《Harness 工程化》](../大语言模型/Harness工程化.md)的六层理论，映射关系：

| HPA 平面 | 对应通用六层 | 说明 |
|----------|------------|------|
| ① Model | — | 模型适配器（接入近 40 家，含本地 vLLM）[$TRAE_REF](https://deepseek.csdn.net/6a7e815110ee7a33f29acc81.html) |
| ② Tool | ②工具系统 | 工具注册表，按需装配 |
| ③ Skill | ②工具系统 | 技能库（封装好的多步能力） |
| ④ Session | ④记忆状态 | 会话管理 + 只追加日志 |
| ⑤ Sandbox | ⑥约束恢复 | 沙箱隔离，限制副作用范围 |
| ⑥ Storage | ④记忆状态 | 存储后端（SQLite 等） |
| ⑦ Loop | ③任务编排 | **Agent 主循环本身**，可替换 |
| ⑧ Scheduling | ③任务编排 | 子 Agent 调度、任务排队 |
| UI | — | 呈现层，独立挂载 |

官方仓库已含 **130+ 可互换插件**，覆盖模型接入、沙箱终端、文件编辑器、技能系统、会话存储、调度循环、Web UI。[$TRAE_REF](https://deepseek.csdn.net/6a7e815110ee7a33f29acc81.html)

> **小结**：HPA 的本质 = 把单体 Agent 的"焊死核心"拆成"可插拔平面"。换模型、换工具、换编排策略、换存储后端，都是配置层操作，不 fork、不 patch。

---

## 四、可追溯性：只追加日志与替换事件

这是和"一切皆插件"并列的第二条底层设计哲学。官方一句话概括：

> **"模型可见，即已记录"**（Model-visible means logged）——任何到达模型的东西，都必须能从日志重建。[$TRAE_REF](https://www.deepseek.com/harness/en/)[$TRAE_REF](https://rohitraj.tech/en/notes/deepseek-harness-vs-claude-code-codex-cli-2026)

### 4.1 记什么

只追加（append-only）会话日志完整记录：系统提示词、思维链、工具调用与结果、子 Agent 调度、**每一次上下文注入**。[$TRAE_REF](https://www.deepseek.com/harness/en/)[$TRAE_REF](https://m.baike.com/wiki/DeepSeek%20Harness/7673137627810267151)

### 4.2 压缩不删历史，用"替换事件"改表象

这是最精妙的技术点：上下文压缩**不删除原始历史**，而是用**替换事件（replacement event）**改变模型此后看到的表象。[$TRAE_REF](https://m.baike.com/wiki/DeepSeek%20Harness/7673137627810267151)

| 维度 | 传统 Agent | DeepSeek Harness |
|------|-----------|------------------|
| 会话日志 | 可覆盖/丢弃 | 只追加（append-only） |
| 上下文压缩 | 删历史腾空间 | 不删，用替换事件改表象 |
| 追溯能力 | 弱 | Trajectory 视图，可恢复/分叉/检索/回放 |

### 4.3 Trajectory 视图

开发者可在 **Trajectory 视图**中按来源（source）检视记录，恢复、分叉、搜索、回放都基于同一个事件流。[$TRAE_REF](https://www.deepseek.com/harness/en/)

> **小结**：可追溯性回答的是"出了问题能不能查"。传统 Agent 长链路出错后，上下文已被压缩覆盖，无法重建当时模型到底看到了什么；Harness 的 append-only + 替换事件机制保证了**完整因果链可回放**——这对调试和审计是刚需。

---

## 五、四种运行模式（preset，非四套 Agent）

Harness 预置四套 preset，本质是"为当前会话装配不同工具 / 提示词 / 运行时"，**不是四套独立 Agent**，同一宿主下一键切换。[$TRAE_REF](https://www.deepseek.com/harness/en/)[$TRAE_REF](https://deepseek.csdn.net/6a7e815110ee7a33f29acc81.html)

| 模式 | 默认插件集 | 适用场景 | 技术要点 |
|------|-----------|---------|---------|
| **Standard 标准** | 全套（文件编辑 / Shell / 检索 / Skills / 规划 / 子 Agent / 工作流） | 日常编码、重构 | 最常用，默认入口 |
| **Code（PTC）** | 标准 + CodeMode SDK | 长链路多步任务（DB 迁移、多文件重构） | 模型写一段 TypeScript 程序组合多轮工具调用，**减少往返开销** |
| **Minimal 极简** | 仅持久 Bash + str_replace_editor | 模型基准测试 | 工具少、上下文负担小，干净测模型能力 |
| **Creator 创造** | 标准 + 运行时检查 + 内存插件实验 + 预设创作指导 | 二开插件、自定义 preset | 高信任模式，会跑模型写的插件代码 |

### 5.1 Code 模式的技术价值

Code 模式（PTC）值得单独讲：传统 Agent 调工具是"一问一答"——模型决策一次、调一次工具、拿结果、再决策。长链路任务（如多文件重构）这样往返几十轮，延迟和 token 成本都高。

Code 模式让模型**直接生成一段 TypeScript 程序，在程序里编排多轮工具调用**，一次输出里完成多步操作。这本质是把 ReAct 的"逐轮交互"升级成"批量编排"，减少模型-工具往返次数。[$TRAE_REF](https://www.deepseek.com/harness/en/)

> 关联本仓[《Harness 工程化》](../大语言模型/Harness工程化.md)第三层"任务编排"：ReAct（逐轮）/ Plan-and-Execute（先规划再执行）/ Code 模式（代码批量编排）是编排层的三种实现，对应不同链路长度。

---

## 六、子代理编排机制

这是 rc.8（2026-08-19）引入的能力，技术上涉及多 Agent 协作的调度协议。

### 6.1 Profile Bundle：竞品当子代理

Claude Code 和 Codex 可作为 **Profile Bundle（可选组件包）** 按需安装，被 Harness 主任务当作子代理调度执行。[$TRAE_REF](https://36kr.com/p/3947852851664512)

Codex 的支持更深：

| 能力 | 说明 |
|------|------|
| **非交互权限模式** | 无需人逐次确认即可自动执行[$TRAE_REF](http://m.toutiao.com/group/7676085169291330048/) |
| **多个命名实例** | 同一任务挂多个 Codex 子代理承担不同职责[$TRAE_REF](https://36kr.com/p/3947852851664512) |
| **reportDelivery 机制** | 子任务完成后回传结果并唤醒等待中的父任务，多 Agent 协作不用干等[$TRAE_REF](https://36kr.com/p/3947852851664512) |

### 6.2 演进脉络

- rc.7（08-17）：Codex/Claude Code 子代理任务接入 **Job Panel** 统一管理[$TRAE_REF](https://36kr.com/p/3947852851664512)
- rc.8（08-19）：从"接入"升级为"按需安装、多实例并行"[$TRAE_REF](https://36kr.com/p/3947852851664512)

> 关联本仓[《父子 Agent 通信》](../大语言模型/父子Agent通信.md)：reportDelivery 的"回传 + 唤醒父任务"是父子 Agent 通信协议的一个具体实现——子 Agent 完成后不阻塞父 Agent，通过事件唤醒而非轮询。

---

## 七、多模态回退管道

rc.8 补齐了图片输入链路，技术上分两条路径。[$TRAE_REF](http://m.toutiao.com/group/7676085169291330048/)

### 7.1 原生视觉路径

模型适配器可通过配置启用**原生图片请求**，`/goal`、`/plan` 等命令支持图文混合输入，`@`菜单新增文件和历史会话引用。需声明 `inputModalities: [text, image]` 才会按多模态处理。[$TRAE_REF](http://m.toutiao.com/group/7676085169291330048/)

### 7.2 "伪视觉"回退路径

底层模型不具备视觉能力时，Harness 兜底调用 **OCR、颜色统计、像素扫描**等工具，把图片拆成结构化信息再喂给文本模型——社区称之为"伪视觉"。[$TRAE_REF](https://36kr.com/p/3947852851664512)[$TRAE_REF](http://m.toutiao.com/group/7676085169291330048/)

这本质是**用工具编排给纯文本模型拼出一套视觉能力**，而非依赖模型自身多模态——和"一切皆插件"哲学一致：能力不够就挂工具补，不绑定特定模型。

### 7.3 载荷约束

| 约束 | 限制 |
|------|------|
| 单张本地附件 | 默认 3.5 MiB |
| 单条消息最多 | 20 张 |
| 原始图片合计 | ≤ 100 MiB |
| 适配器后单次请求图片载荷 | 默认 ≤ 20 MiB |
| 超限处理 | 从最旧图片开始用占位文字替代 |

[$TRAE_REF](http://m.toutiao.com/group/7676085169291330048/)

---

## 八、配置驱动的扩展模型

"一切皆插件"最终落地为"配置驱动"——开发者通过配置文件选/换/扩能力，不改源码。[$TRAE_REF](https://www.deepseek.com/harness/en/)

### 8.1 模型接入（OpenAI 兼容端点）

模型无关，支持近 40 家。接内网网关或本地 vLLM，用 `settings.yaml` 配置（适合 CI/CD）：[$TRAE_REF](https://deepseek.csdn.net/6a7e815110ee7a33f29acc81.html)

```yaml
# $DSH_HOME/settings.yaml
llm-pi-ai:
  providers:
    my-gateway:                 # Provider ID，创建后不可改名
      apiKeyEnv: GATEWAY_API_KEY   # 从环境变量读 Key，避免明文落盘
      api: openai-completions       # 走 OpenAI Chat Completions 协议
      baseURL: https://api.example.com/v1
      models:
        - id: model-name-here     # 模型路由免重启：改完保存即生效
```

### 8.2 关键运维命令

```bash
# 一行拉起 Web UI（npm 临时拉取，无需 clone）
npx @deepseek-ai/dsh web

# 导出本机实际启动的插件装配表（调试 / 写 patch 前必看）
dsh --dump-config

# Headless 单次任务：跑完打印结果自动退出，嵌入流水线
dsh --profile headless "run the tests and summarize failures"

# 收紧权限：写操作限定在会话工作区
export DSH_PERMISSION_MODE=workspace-write
```

[$TRAE_REF](https://deepseek.csdn.net/6a7e815110ee7a33f29acc81.html)

> **小结**：配置驱动的精髓 = 模型路由免重启（改完保存即生效）、插件装配可导出（dump-config）、权限可收紧（workspace-write / danger-full-access 三档）、可 headless 嵌入 CI/CD。这让 Harness 既是交互工具也是自动化基础设施。

---

## 九、与 Claude Code / Codex 的架构对比

从技术架构维度对比（非产品功能维度，功能对比见[《Claude Code 与 Codex 对比》](ClaudeCode与Codex对比.md)）：

| 维度 | DeepSeek Harness | Claude Code | Codex |
|------|------------------|-------------|-------|
| **架构** | 一切皆插件（Cordis 微内核） | 单体核心 + 扩展 | 单体核心 + 扩展 |
| **扩展路径** | 挂插件，不改源码 | 多需 fork / patch 核心 | 多需 fork / patch 核心 |
| **Agent 主循环** | 可替换插件 | 核心焊死 | 核心焊死 |
| **可观测性** | 只追加日志 + Trajectory 视图 | 部分（常只记工具调用） | 部分 |
| **运行模式** | 标准 / Code / 极简 / 创造（四套） | 通常单模式 | 通常单模式 |
| **工具调用** | 经典 + Code 模式（代码组合） | 经典为主 | 经典为主 |
| **模型无关** | 是（~40 家 + 自定义端点 + 本地） | 偏 Anthropic 生态 | 偏 OpenAI 生态 |
| **License** | MIT 开源 | 商业产品 | 商业产品 |

[$TRAE_REF](https://deepseek.csdn.net/6a7e815110ee7a33f29acc81.html)

> Harness 最关键的差异是"插件缝"：没有需要 patch 的特权核心，agent loop 本身也可从配置替换。[$TRAE_REF](https://deepseek.csdn.net/6a7e815110ee7a33f29acc81.html)

---

## 十、技术边界与工程注意

| 维度 | 现状 | 工程注意 |
|------|------|---------|
| 成熟度 | v0.1 开发者预览，核心插件与接口快速迭代 | pin 精确版本（如 rc.8）跑 PoC，勿承载生产主力[$TRAE_REF](https://deepseek.csdn.net/6a7e815110ee7a33f29acc81.html) |
| 破坏性变更 | 官方全大写警告会有不兼容变更 | 升级前备份数据库；rc.8 SQLite 结构已不兼容旧版[$TRAE_REF](http://m.toutiao.com/group/7676085169291330048/) |
| 权限治理 | 三档权限模式 | 切勿对不可信任务开 `danger-full-access`[$TRAE_REF](https://deepseek.csdn.net/6a7e815110ee7a33f29acc81.html) |
| 插件副作用 | 可逆效应保证卸载回滚 | 引第三方插件前审查其副作用可逆性，避免孤儿状态污染运行时[$TRAE_REF](https://deepseek.csdn.net/6a7e815110ee7a33f29acc81.html) |
| Creator 模式 | 插件实验在内存中，重启即失 | 持久化需打包成 bundle（package.json 声明 `dsh.bundle`）[$TRAE_REF](https://deepseek.csdn.net/6a7e815110ee7a33f29acc81.html) |

---

## 复习清单

1. **Harness 的技术公式？** `Model + Harness = Agent`，模型负责思考，Harness 负责执行（工具调度、会话、记忆、任务闭环）。
2. **Cordis 内核做哪三件事？** 插件生命周期管理、依赖注入、全局事件总线——内核不含任何业务逻辑。
3. **插件协作的三种原语？** service（服务注册调用）+ typed event（类型化事件通信）+ reversible effect（可逆副作用，卸载即回滚，不留孤儿状态）。
4. **"没有特权核心"的含义？** 连 Agent 主循环（决定下一步做什么）都是可替换插件，所有改动在配置层完成，不 fork、不 patch。
5. **HPA 八层业务平面？** Model / Tool / Skill / Session / Sandbox / Storage / Loop / Scheduling，UI 独立挂载；官方已含 130+ 可互换插件。
6. **可追溯性的两机制？** append-only 日志（完整记录模型所见的一切）+ 替换事件（压缩不删历史，只改模型此后的表象）。
7. **Trajectory 视图能做什么？** 按来源检视记录，恢复 / 分叉 / 搜索 / 回放都基于同一事件流。
8. **四种运行模式？** Standard（全套）/ Code（模型写 TS 编排多步，减少往返）/ Minimal（仅 Bash+编辑器，测模型）/ Creator（内存插件实验+预设创作）——是 preset 不是四套 Agent。
9. **子代理编排机制？** Profile Bundle 按需装 Claude Code/Codex 当子代理；Codex 支持非交互权限模式 + 多命名实例；reportDelivery 回传结果并唤醒父任务。
10. **多模态回退管道？** 原生视觉路径（适配器配 inputModalities）+ 伪视觉回退（OCR/像素扫描/颜色统计拆图喂文本模型）。
11. **配置驱动的四个要点？** 模型路由免重启、插件装配可导出（dump-config）、权限三档可收紧、可 headless 嵌入 CI/CD。
12. **与 Claude Code/Codex 的架构本质区别？** 后两者"单体核心+扩展"（扩展需 fork 核心），Harness"一切皆插件"（agent loop 本身可替换）。
