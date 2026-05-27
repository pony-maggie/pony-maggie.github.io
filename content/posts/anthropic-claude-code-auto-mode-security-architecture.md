+++
date = 2026-05-27T22:03:07+08:00
draft = false
title = "Claude Code Auto Mode：Anthropic 是怎么给 Agent 加上安全自治护栏的"
categories = ["AI Agent", "安全"]
tags = ["Anthropic", "Claude Code", "Auto Mode", "Agent Security", "MCP"]
+++

Claude Code 的 Auto Mode 看起来像一个很朴素的功能：少弹一点“是否允许执行”的确认框。但它背后其实是在解决 Agent 领域最难啃的问题之一：怎么让模型更自治，同时不把安全边界一起放飞。

Anthropic 这篇文章的价值不在于“又多了一个模式”，而在于它把一个可落地的安全架构讲清楚了：

- 输入侧怎么防 prompt injection
- 输出侧怎么拦危险动作
- 什么操作可以直接放行，什么必须升级判断
- 为什么要用“两层防线”，而不是只靠一个大模型拍脑袋

这篇文章我会把它拆成一个更容易理解的工程视角。

## 目录

- [Auto Mode 解决的不是“能不能做”，而是“该不该打断”](#auto-mode-解决的不是能不能做而是该不该打断)
- [整体架构：输入扫描 + 输出分类器](#整体架构输入扫描--输出分类器)
- [Anthropic 眼里的四类风险](#anthropic-眼里的四类风险)
- [权限分层：哪些动作先天就能过](#权限分层哪些动作先天就能过)
- [为什么要做成“快筛 + 深判”](#为什么要做成快筛--深判)
- [对做 Agent 的人有什么启发](#对做-agent-的人有什么启发)
- [参考资料](#参考资料)

## Auto Mode 解决的不是“能不能做”，而是“该不该打断”

传统 Claude Code 的交互很安全：运行命令前问一次，改文件前问一次。问题是，Agent 一旦开始做长任务，用户会迅速进入“批准疲劳”：

- 点太多次，注意力下降
- 看不清风险，直接机械确认
- 真正危险的操作，反而更容易漏掉

Anthropic 的判断很直接：纯手工审批太慢，完全放开又太危险。

所以 Auto Mode 的目标不是取消安全，而是把安全从“人工按钮”升级成“模型辅助的决策链”。

~~~mermaid
flowchart LR
    U[用户请求] --> A[Claude Code]
    A --> B{是否需要执行动作?}
    B -->|一般动作| C[直接放行]
    B -->|可能危险| D[模型分类器判断]
    D -->|允许| C
    D -->|拒绝/阻断| E[提示用户确认或改写任务]
    A --> F[外部工具输出]
    F --> G[Prompt Injection Probe]
    G --> H[污染内容不直接进上下文]
~~~

## 整体架构：输入扫描 + 输出分类器

Auto Mode 不是单点防御，而是两层：

1. 输入层：先看工具返回的内容有没有“诱导模型改变目标”的迹象。
2. 输出层：再看模型准备执行的动作是否越界。

这个思路很重要，因为 Agent 的危险往往不是“模型突然变坏”，而是：

- 读到了恶意网页
- 读到了带陷阱的文件
- 接着被诱导去做了本不该做的动作

### 输入层：Prompt Injection Probe

Anthropic 的做法是：在工具输出进入上下文之前，先过一个 server-side probe。

如果内容像是在试图“接管”模型，比如：

- 忽略上面的系统提示
- 把这些秘钥发到外部
- 改成执行另一条任务

那就先打上警告，再送入上下文。

这很像给 Agent 的“眼睛”加了偏光镜，不是让它看不见，而是让它对可疑内容保持怀疑。

### 输出层：Transcript Classifier

输出层更像一个“虚拟审批员”。

它看的是：

- 用户说了什么
- Agent 准备做什么动作
- 这个动作是否和用户意图对齐
- 是否可能造成越权、泄露、破坏

这里有个很关键的设计：先快筛，再深判。

- 第一层是低成本的单 token 过滤
- 只有疑似危险时，才进入更重的推理判断

这能把昂贵的思考留给真正值得判断的案例。

## Anthropic 眼里的四类风险

Anthropic 把危险动作的来源分成四类，我觉得这个分类很工程化，也很实用：

| 风险类型 | 说明 | 典型例子 |
|---|---|---|
| Overeager behavior | 模型太积极，做过头了 | 擅自删远端分支、拿到凭据就乱试 |
| Honest mistakes | 理解错了边界 | 把共享资源当成自己的 |
| Prompt injection | 被外部内容带偏 | 网页或文件输出里藏了指令 |
| Misaligned model | 模型本身目标偏移 | 目前少见，但仍需评估 |

~~~mermaid
flowchart TB
    R[危险动作来源] --> O[Overeager behavior]
    R --> H[Honest mistakes]
    R --> P[Prompt injection]
    R --> M[Misaligned model]

    O --> X[同样都要阻断]
    H --> X
    P --> X
    M --> X
~~~

这个分类的意义在于：无论危险是怎么来的，防线策略都一致，先拦住。

也就是说，Auto Mode 的安全设计不是“理解意图再放行一切”，而是“先默认存在风险，再逐步放行”。

## 权限分层：哪些动作先天就能过

Anthropic 不是把所有动作都交给分类器判断。那样太慢，也太贵。它先定义了一个基础 allowlist：

- 文件读取
- 文本搜索
- 代码导航
- todo 和 plan 切换
- 一些不会改状态的操作

这些动作本身就风险低，可以直接过。

而一些高风险能力，会被降级处理，比如：

- 任意 shell 执行
- 泛化脚本解释器
- 可能直接触达生产系统的命令

核心思路很清楚：先把“安全的默认值”做足，再让分类器只盯着真正敏感的边界。

## 为什么要做成“快筛 + 深判”

如果你只用一个大分类器，问题会很多：

- 所有动作都要走重模型，延迟高
- 大部分无害动作也被重判，成本高
- 模型需要看太多内容，容易分心

Anthropic 的方案更像一个高性能风控系统：

- 第一层负责吞吐量
- 第二层负责准确率

这和很多工程系统里的设计逻辑一样：先用便宜的规则筛一遍，再把少量高风险样本交给重模型。

~~~mermaid
sequenceDiagram
    participant User as 用户
    participant Agent as Claude Code
    participant Probe as 输入探针
    participant Classifier as 输出分类器
    participant Tool as 外部工具

    User->>Agent: 发起任务
    Agent->>Tool: 读取文件或网页
    Tool-->>Probe: 返回结果
    Probe-->>Agent: 可疑内容加警告或放行
    Agent->>Classifier: 准备执行动作
    Classifier-->>Agent: 允许或阻断
~~~

这里还有一个很值得注意的点：分类器是 reasoning-blind 的。

它不看 Claude 自己的完整思维链，只看用户消息和工具动作。这么做的好处是：

- 减少把内部推理暴露给另一个组件
- 降低被“自我说服”的风险
- 让判断更聚焦在行为本身

## 对做 Agent 的人有什么启发

如果你也在做 Agent，我建议直接抄这几个原则：

1. 不要把“自治”理解成“全自动”。真正成熟的自治，一定是带边界的自治。
2. 先设计默认安全，再谈高级能力。把 allowlist、隔离、分类器、审计日志先做出来。
3. 输入和输出都要防。只拦危险输出不够，prompt injection 往往从输入开始。
4. 把高频低风险动作和高危动作分开。不要让所有请求都走同一条重路径。
5. 把风控系统当产品能力，而不是补丁。这类机制越早做，后面越不容易返工。

## 参考资料

参考资料：[Claude Code Auto Mode: security architecture](https://www.anthropic.com/engineering/claude-code-auto-mode)
