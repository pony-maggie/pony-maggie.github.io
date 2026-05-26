+++
title = "Gemini Managed Agents 开发指南：一条 API 搭出可执行、可持久的 AI Agent"
date = 2026-05-26T22:04:02+08:00
draft = false
categories = ["AI Agent", "Gemini", "Google"]
tags = ["Gemini", "Managed Agents", "AI Agent", "Sandbox", "Python"]
+++

如果你已经做过一阵子 Agent，八成会遇到这几个老问题：

- 模型会调用工具，但不会自己把流程跑完
- 你得自己维护运行环境、依赖安装、文件读写和中间状态
- 多轮任务一长，上下文就开始膨胀，最后“记性不好”

Gemini 的 Managed Agents 这篇指南，讲的就是怎么把这些脏活累活交给平台。

一句话概括：你不是在给模型加几个 function call，而是在云端直接开一个能跑代码、能装包、能读写文件、能接数据的 Agent 工作间。

<!-- more -->

## 先看结论：Managed Agent 到底解决什么

传统 Agent 的问题，不是“不能推理”，而是“周边系统太碎”。

你往往要自己拼这些东西：

- 运行沙箱
- 依赖安装
- 多轮状态保存
- 文件上传和下载
- 流式输出
- 网络访问控制
- 凭据注入

Managed Agents 把这些统一封装进 Interactions API 和 Agents API：

- interactions 负责一次次交互
- environment 负责运行时沙箱
- agent 负责行为定义和可复用配置

可以把它理解成下面这条链路：

```mermaid
flowchart TD
    U[用户任务] --> A[Interactions API]
    A --> E[远程 Linux Sandbox]
    E --> T[代码执行 / 文件操作 / 浏览器 / 工具调用]
    T --> O[输出结果]
    A --> C[自动压缩上下文]
    C --> A
```

核心变化只有一个：Agent 的执行层被平台托管了。

## 1. 快速上手：一条调用开一个工作环境

最小化使用方式很直接。

    from google import genai

    client = genai.Client()

    interaction = client.interactions.create(
        agent="antigravity-preview-05-2026",
        input="Research the top 10 AI stories today and create a PDF briefing with summaries",
        environment="remote",
    )

    print(interaction.output_text)

这段代码做了三件事：

1. 创建一个隔离的 Linux sandbox
2. 在 sandbox 里执行 Agent 任务
3. 返回结果以及后续复用要用的 interaction.id 和 environment_id

这里最值钱的点不是“能跑”，而是状态和环境可以被继续接上。

换句话说，第一次跑完不是结束，而是拿到了一个可复用的工作区。

## 2. 多轮对话：状态和环境不是一回事

这篇文章里最容易被忽略的设计点，是把“对话历史”和“环境状态”分开了。

- previous_interaction_id：延续对话历史
- environment_id：延续文件、依赖、工作目录和执行状态

    interaction_2 = client.interactions.create(
        agent="antigravity-preview-05-2026",
        environment=interaction.environment_id,
        previous_interaction_id=interaction.id,
        input="Now create a landing page using JavaScript and HTML",
    )

这意味着什么？

- 你可以保持同一个沙箱，继续往里装包、写文件、生成中间产物
- 你也可以重新开一轮对话，但仍然共享同一份工作目录

这比每轮都重新初始化强太多了。
对于真实任务来说，Agent 最贵的不是一次回答，而是上下文里不断重复的准备成本。

## 3. 流式输出：别等最后一秒才看结果

长任务如果只等最终结果，调试体验会很差。

Managed Agents 支持流式返回，这样你能实时看到：

- 生成中的文本
- 推理过程中的事件
- 工具调用更新

    stream = client.interactions.create(
        agent="antigravity-preview-05-2026",
        input="Scrape Hacker News, summarize the top 5 stories, and save the results as a PDF.",
        environment="remote",
        stream=True,
    )

    for event in stream:
        print(event)

这对长任务特别有用。

比如：

- 爬取数据
- 跑分析脚本
- 生成报告
- 写文件到 sandbox

你可以一边看事件，一边判断任务是否偏航，而不是等它失败了才知道。

## 4. 往 sandbox 塞数据：仓库、云存储、内联文件都行

Managed Agents 不是只能空跑。

它支持在环境初始化时，把外部数据直接挂进来：

- repository：拉 Git 仓库
- gcs：挂 Google Cloud Storage
- inline：直接写入文本文件

    interaction = client.interactions.create(
        agent="antigravity-preview-05-2026",
        input="Analyze the data and summarize the project.",
        environment={
            "type": "remote",
            "sources": [
                {
                    "type": "repository",
                    "source": "https://github.com/my-org/data-templates.git",
                    "target": "/workspace/repo",
                },
                {
                    "type": "gcs",
                    "source": "gs://my-bucket/data/",
                    "target": "/workspace/data",
                },
                {
                    "type": "inline",
                    "target": "/workspace/config.yaml",
                    "content": "mode: analysis\noutput: pdf",
                },
            ],
        },
    )

这一步的价值非常大。

很多 Agent 项目做不好，不是模型不行，而是上下游数据准备太麻烦。
把数据源直接挂进环境里，Agent 才能真正“拿到上下文就开干”。

## 5. 下载 sandbox：把结果变成可交付物

Agent 做完事，结果不该只停留在文本输出。

你可以把整个环境打包下载成 tar，再在本地解压。

这类设计特别适合：

- 分析报告
- 代码修复
- 数据处理
- 页面生成
- 文档产物导出

```mermaid
flowchart LR
    A[Sandbox 内生成文件] --> B[下载整个环境]
    B --> C[tar 包]
    C --> D[本地解压]
    D --> E[报告 / 代码 / 资产 / 图表]
```

这意味着 Agent 不只是“说了什么”，而是真正交付了什么。

## 6. 自定义 Managed Agent：把行为固化成可复用资产

如果你发现同一类任务反复出现，就别每次重写 prompt 了。

应该把它固化成一个 Agent 配置。

文章里给的思路是：

- base_agent：选择底座能力
- system_instruction：定义角色和边界
- base_environment：定义默认数据、技能和代码仓库

    agent = client.agents.create(
        id="data-analyst",
        base_agent="antigravity-preview-05-2026",
        system_instruction="You are a data analyst. Always include visualizations and export results as PDF.",
        base_environment={
            "type": "remote",
            "sources": [
                {
                    "type": "inline",
                    "target": ".agents/AGENTS.md",
                    "content": "Always use matplotlib for charts.",
                },
                {
                    "type": "inline",
                    "target": ".agents/skills/slide-maker/SKILL.md",
                    "content": "---\nname: slide-maker\n---\n# Slide Maker\nCreate HTML slide decks from data analysis results.",
                },
            ],
        },
    )

你可以把它理解成一个 Agent 模板工程：

- 角色设定放在 AGENTS.md
- 专项能力放在 SKILL.md
- 运行所需的数据和代码放在 base_environment

这套结构的好处很明确：

- 复用性更强
- 行为更稳定
- 团队协作更容易

## 7. 从已有环境 fork：先试，再沉淀

不是所有 Agent 一开始都要写成模板。

更实际的做法是：

1. 先用交互式方式把环境搭好
2. 把效果验证清楚
3. 再 fork 成可复用的 Agent

    # Step 1: set up the environment
    interaction = client.interactions.create(
        agent="antigravity-preview-05-2026",
        input="Install pandas and matplotlib. Create an analysis template.",
        environment="remote",
    )

    # Step 2: fork into a reusable agent
    agent = client.agents.create(
        id="my-data-analyst",
        base_agent="antigravity-preview-05-2026",
        base_environment=interaction.environment_id,
    )

这个流程很像我们在工程里常见的“先跑通，再模板化”。

先别急着抽象，先确认它真能干活。

## 8. 网络和凭据：安全边界要前置设计

真正能上生产的 Agent，一定绕不开安全问题。

Managed Agents 的一个优势是：网络访问和密钥注入是平台能力，不是你自己手搓。

文章里提到两件事：

- 可以配置网络 allowlist
- 凭据可以通过代理层注入成 HTTP header

    agent = client.agents.create(
        id="issue-resolver",
        base_agent="antigravity-preview-05-2026",
        system_instruction="You resolve GitHub issues. Clone the repo, find the bug, write the fix, run the tests, and open a PR.",
        base_environment={
            "type": "remote",
            "sources": [
                {
                    "type": "repository",
                    "source": "https://github.com/my-org/backend",
                    "target": "/workspace/repo",
                },
            ],
            "network": {
                "allowlist": [
                    {
                        "domain": "api.github.com",
                        "transform": {
                            "Authorization": "Bearer ghp_your_github_token",
                        },
                    },
                    {
                        "domain": "pypi.org",
                    },
                ]
            },
        },
    )

这套机制的意义不只是“安全”。

它还把可访问范围变成了可控变量：

- 允许访问哪些域名
- 允许注入哪些凭据
- 允许 Agent 做哪些外部动作

这比把 secret 直接丢进 prompt 或本地环境变量里稳得多。

## 9. Gemini API CLI：给人和 Agent 都留一个终端入口

文章还有一个很实用的点：官方把 CLI 也做了。

这样做的好处是：

- 人可以直接在终端里试
- Agent 可以在脚本里自动化
- 从 prompt 到部署的路径更短

    # 跑一个 prompt
    gemini-api run "What is the capital of France?"

    # 生成图片
    gemini-api run "A cat in space" --model gemini-3.1-flash-image-preview --output cat.png

    # 文本转语音
    gemini-api run "Hello from Gemini" --model gemini-3.1-flash-tts-preview --voice Kore --output hello.wav

    # 搭建、测试并创建 managed agent
    gemini-api agents init my-agent
    gemini-api agents test --prompt "Analyze the Q1 revenue data"
    gemini-api agents create

对工程团队来说，CLI 的价值很朴素：

- 小任务直接在终端试
- 大任务再封成 API
- 最后才是集成到产品里

这条路径非常符合真实研发节奏。

## 10. 什么时候该用 Managed Agents

如果你的场景满足下面任意几条，就值得认真考虑：

- 需要代码执行，而不是只做文本生成
- 需要持久文件系统或中间产物
- 需要多轮任务接续
- 需要把数据、仓库、配置挂进执行环境
- 需要对网络和凭据做可控治理

反过来，如果只是一个简单问答、短 prompt 转换，没必要上这么重的架构。

Managed Agents 更适合的是：

- 研究助理
- 数据分析助理
- 代码修复助理
- 报告生成助理
- 可部署的自动化工作流

## 总结

这篇指南最重要的信号，不是“Gemini 又出了一个新 API”，而是：

Agent 正在从“会调用工具的模型”，进化成“平台托管的可执行工作单元”。

一旦你接受这个方向，很多以前自己硬扛的事都可以交给平台：

- 环境初始化
- 依赖安装
- 文件读写
- 多轮状态
- 流式事件
- 安全网络
- 可复用 Agent 模板

这类能力成熟后，做 Agent 的重点就不再是“怎么把模型接起来”，而是“怎么把工作流设计得真正可维护”。

参考资料：[Gemini Managed Agents: Developer Guide](https://www.philschmid.de/gemini-managed-agents-developer-guide)

欢迎关注收藏我，获取更多硬核技术干货
