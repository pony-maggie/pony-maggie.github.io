+++
date = 2026-06-08T22:01:37+08:00
draft = false
title = "LangChain 自定义 Agent Harness：为什么 Agent 不是只靠模型就够了"
categories = ["AI Agent", "LangChain"]
tags = ["LangChain", "Agent", "Harness", "Middleware", "AI Agent"]
+++

如果你最近在做 Agent，大概率已经碰过一个很现实的问题：模型本身很聪明，但一旦任务变长、工具变多、状态变复杂，它就开始“看起来会做，实际上做不稳”。

LangChain 这篇 [How to Build a Custom Agent Harness](https://www.langchain.com/blog/how-to-build-a-custom-agent-harness) 讲的不是又一个 Agent 炫技 demo，而是一个更底层的问题：**Agent 真正的工程难点，不在模型，而在 harness**。

你可以把它理解成一句特别好记的话：

> `agent = model + harness`

模型负责“想”，harness 负责“把想法变成可执行、可控制、可恢复的系统”。

## 目录

- [先说结论：为什么 harness 才是核心](#先说结论为什么-harness-才是核心)
- [什么是 Agent Harness](#什么是-agent-harness)
- [Middleware 为什么是关键抽象](#middleware-为什么是关键抽象)
- [任务和 Harness 必须匹配](#任务和-harness-必须匹配)
- [一个更实用的架构图](#一个更实用的架构图)
- [怎么自己搭一个可定制的 Harness](#怎么自己搭一个可定制的-harness)
- [实战建议](#实战建议)
- [总结](#总结)

## 先说结论：为什么 harness 才是核心

很多人做 Agent 的思路是反过来的：先挑最强模型，再往里面塞工具，最后祈祷它足够稳定。

这条路不是完全不行，但有三个典型问题：

- 任务一长，上下文很快爆掉
- 工具一多，调用顺序和状态管理就开始失控
- 一旦涉及审批、重试、隔离、记忆，纯 prompt 很难兜住

LangChain 这篇文章的核心态度很明确：**不要把 Agent 只看成一个模型调用循环，Agent 本质上是一套围绕模型的执行外壳**。

这个外壳要负责的事情，比你想象得多：

- 给模型喂对上下文
- 控制工具注册和生命周期
- 管理状态、记忆和中间结果
- 拦截异常和失败
- 在关键点插入策略、审批和审计

所以真正值得优化的，不是“模型再聪明一点”，而是“harness 能不能把模型放到正确的工作环境里”。

## 什么是 Agent Harness

原文里最重要的一句话是：**harness 是围绕模型的脚手架**。

它不是 prompt，也不是工具列表本身，而是把模型、工具、状态和外部环境连接起来的一层执行系统。

可以画成这样：

~~~mermaid
flowchart TD
    U[用户输入] --> H[Agent Harness]
    H --> P[模型 / Policy]
    H --> T[工具注册与调用]
    H --> S[状态 / 记忆 / 运行上下文]
    H --> E[外部环境: 文件 / 沙箱 / 网络]
    P --> H
    T --> H
    S --> H
    E --> H
    H --> O[最终输出]
~~~

这里最关键的是：**模型不是系统本身，模型只是系统里的决策器**。

真正让 Agent 从“能聊天”变成“能干活”的，是 harness 提供了这些能力：

- 让模型在每一步都拿到足够上下文
- 让工具调用有规则、有边界
- 让长任务可以被拆解、恢复、重试
- 让业务策略不必硬塞进 prompt

这也是为什么很多 Agent 项目最后都不是死在推理能力上，而是死在工程结构上。

## Middleware 为什么是关键抽象

LangChain 这篇文章里，我最认可的点是它把 harness 的能力拆成了 middleware。

middleware 的好处是：**每个问题只做一件事，但可以自由组合**。

比如：

- 一个 middleware 负责上下文压缩
- 一个 middleware 负责工具初始化和清理
- 一个 middleware 负责策略判断
- 一个 middleware 负责输出流处理
- 一个 middleware 负责子 Agent 调度

这比把所有逻辑写进一个巨大的 agent loop 里靠谱得多。

### 为什么这比“直接写 prompt”更稳

prompt 的问题是它天然偏“建议”，不偏“约束”。

但生产环境里的很多事情不是建议，而是规则：

- 什么时候必须拦截
- 什么时候必须人工确认
- 什么时候必须切换模型
- 什么时候必须记录审计日志

这些东西如果只靠 prompt，迟早会在复杂输入下失效。

middleware 的价值就在这里：它把这些事情从“语言层建议”变成“执行层约束”。

## 任务和 Harness 必须匹配

文章里提到一个特别重要的概念：`task-harness fit`。

意思很直接：**不同任务，要配不同 harness，不存在一个万能模板**。

例如：

- 客服 Agent 更看重流程稳定、审批和记忆
- 编码 Agent 更看重文件系统、沙箱和长任务恢复
- 研究 Agent 更看重检索、摘要和中间结果保存
- 工作流 Agent 更看重分支、重试和可观测性

你如果拿一个“聊天型 harness”去跑“长任务执行型 Agent”，大概率会遇到这些问题：

- 上下文越来越脏
- 工具越来越乱
- 状态越来越难回放
- 成本越来越高

换句话说，**harness 不是配角，它决定了 Agent 的上限和下限**。

### 一个简单的对照表

| 任务类型 | 主要风险 | 需要的 harness 能力 |
|---|---|---|
| 简单问答 | 低，但容易过度设计 | 基础工具调用、轻量状态 |
| 长任务执行 | 上下文膨胀、步骤丢失 | 记忆、压缩、恢复、重试 |
| 多工具编排 | 调用顺序混乱 | 工具生命周期管理、状态隔离 |
| 需要审批的操作 | 误操作风险 | Human-in-the-loop、策略拦截 |
| 多 Agent 协作 | 角色串线、输出噪音 | 子 Agent、命名空间、事件分层 |

如果你把这张表记住了，后面设计 Agent 架构时会少很多拍脑袋决策。

## 一个更实用的架构图

如果把 LangChain 的思路翻译成工程视角，我会这样画：

~~~mermaid
flowchart LR
    I[输入事件] --> R[路由 / 策略层]
    R --> M[Middleware 链]
    M --> C[上下文管理]
    M --> G[工具治理]
    M --> A[模型调用]
    M --> D[状态与记忆]
    G --> E[文件 / API / 沙箱]
    A --> O[输出事件]
    D --> O
    C --> O
    E --> O
~~~

这里的重点不是“层数越多越高级”，而是把职责拆开：

- 路由层决定走哪种执行路径
- Middleware 链负责策略、状态和工具治理
- 模型只负责根据当前上下文做决策
- 输出层把结果变成可观察、可订阅的事件

这类结构的好处是，后续你想加新能力时，不需要重写整个 Agent。

你只要加一个 middleware，或者替换一层策略，就能把行为调整到位。

## 怎么自己搭一个可定制的 Harness

原文的思路其实很适合落地成一个最小实现。

你可以先把 Agent 拆成四块：

1. `model`：负责推理
2. `middleware`：负责控制和扩展
3. `tools`：负责和外部世界交互
4. `state`：负责记住当前任务的上下文

下面这个示意代码能说明结构，不必纠结每个 API 是否和你当前版本完全一致，重点是设计方式：

~~~python
from langchain.agents import create_agent

agent = create_agent(
    model="anthropic:claude-sonnet-4-6",
    tools=tools,
    system_prompt="You are a helpful assistant."
)
~~~

如果你要把它改造成生产可用的 harness，通常会继续补上这些能力：

- 在调用模型前做上下文裁剪
- 在工具调用前做权限校验
- 在工具调用后做结果归档
- 在失败时做重试或降级
- 在关键动作前触发人工审批

这就是 middleware 真正的价值：**把“Agent 怎么做事”从 prompt 里抽出来，变成可维护的代码结构**。

## 实战建议

如果你正在做自己的 Agent 平台，我建议直接按下面三条来：

1. **先设计 harness，再设计 prompt**
   - Prompt 只能优化行为倾向，harness 才能定义执行边界。

2. **把高风险逻辑放到 middleware**
   - 比如审批、限流、权限、审计、重试，不要散落在业务代码里。

3. **按任务分类做 harness**
   - 不要指望一个通用 Agent 适配所有场景。
   - 长任务、短任务、审批任务、研究任务，至少要有不同的配置层。

如果你只做一个“会聊的 Agent”，模型能力可能已经够了。

但只要你开始做“能稳定交付结果的 Agent”，harness 就会从配角变成主角。

## 总结

LangChain 这篇文章真正讲透的，不是一个新 API，而是一个工程事实：

**Agent 的核心竞争力，不是模型有多强，而是 harness 能不能把模型放进一个可控、可扩展、可恢复的执行系统里。**

当你接受这个前提以后，很多问题都会变得清晰：

- 为什么要有 middleware
- 为什么要做上下文管理
- 为什么要区分不同任务的 harness
- 为什么生产级 Agent 不能只靠 prompt

一句话收尾：

> 做 Agent，别只盯着模型分数，真正值钱的是你给模型搭了什么样的脚手架。

参考资料：[How to Build a Custom Agent Harness](https://www.langchain.com/blog/how-to-build-a-custom-agent-harness)
