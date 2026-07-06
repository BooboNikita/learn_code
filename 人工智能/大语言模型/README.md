# 大语言模型 LLM

> Large Language Model：参数量亿级以上的预训练语言模型。

## 学习要点
- 基础：Tokenizer、Embedding、Attention、Transformer 架构。
- 训练：预训练、SFT（监督微调）、RLHF / DPO。
- 推理：KV-Cache、量化（INT4/INT8）、PagedAttention、Speculative Decoding。
- 应用：RAG、Agent、Function Calling、Prompt Engineering。

## 推荐资料
- 《动手学大模型》系列教程
- Hugging Face 文档

## 文章列表
- [Agent 记忆机制](Agent记忆机制.md) — Claude Code 静态层 + 动态层架构，治业界 4 类方案病根。
- [父子 Agent 通信](父子Agent通信.md) — Claude Code 父子型多 Agent：工具/上下文隔离、默认单向 vs 团队模式双向通信、Fork 与 Coordinator 模式。
- [Harness 工程化](Harness工程化.md) — 围绕 LLM 的驾驭脚手架：六层组件（上下文/工具/编排/记忆/观测/护栏）+ 五大工程难题 + 五条口诀。
- [Loop 工程化](Loop工程化.md) — 从 Prompt 到 Loop 的范式转变：五大件（自动化/Worktree/Skill/Connector/Sub-Agent）+ 真实 Loop 实例 + 三盆冷水（验证/理解债/认知投降）。
- [什么是大语言模型](什么是大语言模型.md) — LLM 本质 = 海量语料预训练 + 百亿千亿参数 + CLM 自回归预测下一个 token；与传统 NLP 的三层区别（任务方式/范式/能力来源）+ 涌现与 Scaling Law。
- [大模型量化](大模型量化.md) — FP16 → INT4 把 7B 模型从 14GB 压到 3.5GB；三大算法 GPTQ（误差补偿）/ AWQ（激活感知）/ QLoRA NF4（正态非均匀）的横向对比与场景选型。
- [DeepSeek 系列创新](DeepSeek系列创新.md) — MLA 压缩 KV Cache、DeepSeekMoE 稀疏激活、Auxiliary-Loss-Free 负载均衡、MTP、GRPO 纯 RL 推理，以及 V4 公开信息梳理。
