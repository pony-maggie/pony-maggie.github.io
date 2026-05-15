+++
date = 2026-05-15T22:02:11+08:00
draft = false
title = "OncoAgent 深度拆解：医疗 AI Agent 如何用 Corrective RAG、LangGraph 和 QLoRA 防幻觉"
categories = ["AI Agent", "架构设计"]
tags = ["AI Agent", "RAG", "LangGraph", "QLoRA", "医疗 AI", "多智能体"]
+++

## 前言

AI Agent 做 Demo 很容易，做生产系统很难；做普通生产系统已经难，做医疗场景更是地狱级难度。

因为医疗 AI 不能只追求「回答像不像专家」，它必须解决更硬的问题：**答案有没有指南依据？患者隐私能不能留在本地？复杂病例该不该升级给更强模型？模型胡说八道之前有没有安全闸门？**

Hugging Face 最近有一篇关于 **OncoAgent** 的技术预印本，给了一个很完整的参考架构：LangGraph 多节点编排、Corrective RAG、双层模型路由、QLoRA 微调、Reflexion 安全校验、Human-in-the-Loop 人类审查，全都放进同一个系统里。

今天我们不做资讯复述，直接拆它的工程设计，看看一个面向肿瘤临床决策支持的 Agent 系统，到底该怎么搭。

<!-- more -->

## 目录

- [为什么医疗 Agent 不能只靠一个大模型](#为什么医疗-agent-不能只靠一个大模型)
- [OncoAgent 的整体架构](#oncoagent-的整体架构)
- [复杂度路由：简单病例快答，复杂病例深推理](#复杂度路由简单病例快答复杂病例深推理)
- [Corrective RAG：先证明资料靠谱，再让模型回答](#corrective-rag先证明资料靠谱再让模型回答)
- [Reflexion 安全循环：把幻觉挡在出口前](#reflexion-安全循环把幻觉挡在出口前)
- [QLoRA 微调：为什么一张 MI300X 就能训完整套模型](#qlora-微调为什么一张-mi300x-就能训完整套模型)
- [隐私与记忆隔离：医疗场景的底线工程](#隐私与记忆隔离医疗场景的底线工程)
- [实战建议：普通团队能抄哪些作业](#实战建议普通团队能抄哪些作业)

---

## 为什么医疗 Agent 不能只靠一个大模型

很多人做 Agent 的第一反应是：找一个最强模型，塞一段系统提示词，再接上知识库。

在医疗场景里，这个方案风险太大。

原因很简单：**医疗问答不是开放式闲聊，而是带责任边界的证据推理**。模型输出的每句话，都应该能追溯到指南、病历、检索材料或明确的安全拒答策略。

OncoAgent 针对的是肿瘤临床决策支持。它要面对的问题包括：

1. 指南资料多且更新快，比如 NCCN、ESMO 等专业指南。
2. 医学术语高度精确，「TKI」和「tyrosine kinase inhibitor」这种同义表达必须能匹配。
3. 患者数据涉及 PHI（Protected Health Information，受保护健康信息），不能随便丢给云端 API。
4. 复杂病例不能让模型自由发挥，必须有医生审核。
5. 检索不到可靠依据时，宁愿拒答，也不能编一个看似合理的方案。

所以 OncoAgent 的核心思路不是「让 LLM 更会说」，而是把系统拆成多个可控环节：**路由、检索、生成、校验、审查、兜底**。

这才是医疗 Agent 和普通聊天机器人的分水岭。

---

## OncoAgent 的整体架构

OncoAgent 使用 LangGraph 实现一个有状态的有向图。你可以把它理解成一条临床推理流水线：每个节点只负责一件事，并把结果写入共享的 AgentState。

整体拓扑如下：

~~~mermaid
graph LR
    A[Router<br/>复杂度路由] --> B[Ingestion<br/>输入清洗与 PHI 脱敏]
    B --> C[Corrective RAG<br/>指南检索与文档评分]
    C --> D[Specialist<br/>专科推理]
    D --> E[Critic<br/>安全与证据校验]
    E -->|通过| F[HITL Gate<br/>医生审查闸门]
    E -->|失败重试| D
    F --> G[Formatter<br/>格式化输出]
    C -->|检索失败| H[Fallback<br/>安全拒答]
    E -->|多次失败| H
    H --> I[END]
    G --> I
~~~

这套设计最值得注意的点，不是「用了很多 Agent」，而是每个 Agent 的边界非常清楚：

| 节点 | 作用 | 关键控制点 |
|------|------|------------|
| Router | 判断病例复杂度，选择快模型或深推理模型 | 复杂度评分 |
| Ingestion | 清洗输入，移除患者隐私信息 | Zero-PHI |
| Corrective RAG | 检索指南并过滤无关文档 | 文档评分、重写查询 |
| Specialist | 基于上下文生成临床建议 | 只基于检索材料 |
| Critic | 检查格式、安全、证据支持 | 最多 2 次反馈重试 |
| HITL Gate | 高风险场景交给医生确认 | 人类审查 |
| Fallback | 无法确认时安全拒答 | 不编造答案 |

这其实是一个很通用的生产 Agent 设计原则：**把 LLM 放在需要推理的位置，把确定性规则放在需要约束的位置。**

---

## 复杂度路由：简单病例快答，复杂病例深推理

OncoAgent 没有让所有请求都走同一个模型，而是设计了双层模型架构：

- **Tier 1：9B 速度模型**，负责简单分诊和低复杂度问题。
- **Tier 2：27B 深推理模型**，负责复杂病例、多突变、多阶段、多治疗史问题。

路由不是靠模型拍脑袋，而是用一个加权评分公式：

~~~text
S = w_cancer + w_stage + w_mutations + w_treatment
~~~

例如：

| 因素 | 条件 | 权重 |
|------|------|------|
| 癌种 | 罕见癌种 | +0.40 |
| 癌种 | 原发灶未知 | +0.30 |
| 分期 | IV 期 | +0.25 |
| 分期 | III 期 | +0.15 |
| 突变 | 2 个及以上突变 | +0.30 |
| 突变 | 单个突变 | +0.15 |
| 既往治疗 | 出现治疗史关键词 | +0.10 |

当 S >= 0.5，请求进入 Tier 2；否则走 Tier 1。

这套机制很朴素，但工程价值很高：

1. **成本可控**：简单问题不用每次都调用大模型。
2. **延迟可控**：常见场景优先快速响应。
3. **风险可控**：高复杂度病例自动升级，并触发更严格审查。

实战里很多 Agent 系统也可以这么做。不要一上来就「所有请求最强模型走起」，先定义业务复杂度，再按风险分层调度模型。

---

## Corrective RAG：先证明资料靠谱，再让模型回答

普通 RAG 的经典流程是：用户提问 → 向量检索 → 拼进上下文 → LLM 回答。

问题是，检索结果「看起来相关」不等于「真的能支撑答案」。在医疗领域，这个差距很致命。

OncoAgent 用的是 **Corrective RAG（CRAG，纠错式检索增强生成）**。它在检索和生成之间多加了几层闸门：

~~~mermaid
flowchart TD
    Q[临床问题] --> R1[PubMedBERT Bi-Encoder<br/>召回 top-15]
    R1 --> R2[Cosine Distance Gate<br/>距离阈值过滤]
    R2 -->|通过| R3[Cross-Encoder Re-rank<br/>重排 top-5]
    R2 -->|不通过| F[安全拒答]
    R3 --> R4[Context Trimming<br/>控制上下文长度]
    R4 --> G[文档相关性评分]
    G -->|相关| S[Specialist 生成]
    G -->|不相关| Rewrite[查询重写<br/>最多重试 1 次]
    Rewrite --> R1
~~~

它的核心思想是：**检索不是把资料捞上来就完事，而是要证明资料和问题真的有关。**

OncoAgent 的四阶段检索包括：

| 阶段 | 组件 | 作用 |
|------|------|------|
| 1. Recall | PubMedBERT Bi-Encoder | 大范围召回候选指南 |
| 2. Distance Gate | 余弦距离阈值 | 拦截离题内容 |
| 3. Re-ranking | Cross-Encoder | 更精细地判断 query-document 相关性 |
| 4. Context Trimming | 字符预算控制 | 避免上下文塞爆 |

它还可选使用 HyDE（Hypothetical Document Embeddings，假设文档嵌入）：先让模型生成一段「理想指南段落」，再用这段文本去检索，解决医学同义词匹配问题。

比如用户说「肺部肿瘤」，指南里可能写的是「lung carcinoma」或「NSCLC」。直接向量检索可能漏掉，HyDE 可以把自然语言问题投影到更接近指南文本的表达空间。

这对开发者的启发很直接：**RAG 的质量瓶颈往往不在生成，而在检索验证。**

---

## Reflexion 安全循环：把幻觉挡在出口前

OncoAgent 的 Specialist 节点生成建议后，不会直接把答案给医生，而是先交给 Critic 节点。

Critic 做三层校验：

1. **格式检查**：输出是否符合结构化 schema。
2. **安全检查**：是否出现禁止模式，比如没有指南依据的绝对剂量建议。
3. **证据蕴含检查**：生成内容是否真的被 RAG 上下文支持。

如果失败，Critic 会把具体反馈重新注入 Specialist，让它重试，最多 2 次。

~~~mermaid
sequenceDiagram
    participant S as Specialist
    participant C as Critic
    participant H as HITL Gate
    participant F as Fallback

    S->>C: 生成临床建议
    C->>C: 格式校验
    C->>C: 安全规则扫描
    C->>C: 证据支持检查
    alt 校验通过
        C->>H: 进入医生审查
    else 校验失败且可重试
        C->>S: 返回具体问题，要求修正
    else 多次失败
        C->>F: 安全拒答
    end
~~~

这里有一个很关键的工程选择：**安全校验逻辑不是完全交给 LLM 控制，而是由确定性代码执行。**

这能避免一种常见风险：攻击者通过 Prompt Injection 让模型「忽略安全规则」。如果安全规则本身也是模型自由解释，那就很容易被绕过；如果规则在代码层执行，模型最多只能生成文本，不能改闸门。

这也是生产 Agent 的基本功：**LLM 负责推理，系统负责约束。**

---

## QLoRA 微调：为什么一张 MI300X 就能训完整套模型

OncoAgent 的训练部分也很有意思。

它构建了一个名为 OncoCoT 的训练集，总量约 **266,854 条样本**，来源包括真实临床病例、医学 QA 数据集，以及用大模型合成的肿瘤病例。

两个模型都用 QLoRA 做参数高效微调：

| 参数 | Tier 1 9B | Tier 2 27B |
|------|-----------|------------|
| 量化 | NF4 4-bit | NF4 4-bit |
| LoRA rank | 16 | 32 |
| Effective batch size | 16 | 16 |
| Sequence packing | 2048 tokens | 2048 tokens |
| Early stopping | patience = 3 | patience = 3 |

QLoRA 的价值在于：不用全量更新模型参数，而是把主模型压到 4-bit，再训练少量 LoRA adapter。这样显存压力大幅下降，训练成本也更可控。

原文提到，他们在 AMD MI300X 上使用 Unsloth 和 sequence packing，把完整数据集微调时间压到约 50 分钟。

Sequence packing 这个优化非常实用。医疗病例通常长短不一，如果每条样本都 padding 到 2048 tokens，会浪费大量计算。Packing 会把多个短样本拼进同一个上下文窗口，减少空 token：

~~~text
低效做法：
[case A + padding padding padding]
[case B + padding padding padding]
[case C + padding padding padding]

高效做法：
[case A][case B][case C][少量 padding]
~~~

这不是模型结构创新，但往往就是这种工程细节，决定了训练任务是「一天跑完」还是「一小时跑完」。

---

## 隐私与记忆隔离：医疗场景的底线工程

OncoAgent 把隐私设计放在系统入口，而不是事后补丁。

第一步就是 Zero-PHI：在任何文本进入 LLM 前，先识别并替换患者姓名、出生日期、病历号、地址、机构标识等敏感信息。脱敏后的文本进入 AgentState，原始文本不继续向下游传递。

这件事很重要，因为很多团队会犯一个错误：先把用户输入交给模型，再要求模型「请不要泄露隐私」。这在合规场景里是不够的。

正确做法是：

~~~mermaid
flowchart LR
    Raw[原始病历文本] --> Redact[PHI 脱敏节点]
    Redact --> State[AgentState<br/>只保存脱敏文本]
    State --> RAG[RAG]
    State --> LLM[LLM 推理]
    Raw -.不进入下游.-> Drop[丢弃或隔离保存]
~~~

另外，OncoAgent 还通过 thread_id 做 per-patient memory isolation（按患者隔离记忆）。每个患者会话都有独立 checkpoint，避免 A 患者的上下文污染 B 患者。

这点不只适用于医疗。金融、法律、企业知识库、多租户 SaaS Agent 都一样：**记忆系统必须从第一天就按租户、用户、任务边界隔离。**

---

## 实战建议：普通团队能抄哪些作业

OncoAgent 是医疗场景，门槛很高，但它的很多架构思想可以直接复用到普通 Agent 项目里。

### 1. 不要让 LLM 做所有事

数据库查询、阈值判断、格式校验、权限检查、敏感信息脱敏，能用确定性代码做的，就不要塞给 LLM。

LLM 最适合做的是：理解、归纳、推理、生成、解释。

### 2. 给 RAG 加「证据闸门」

不要只看 top-k 检索结果。至少加三类检查：

- 相似度阈值：低于阈值直接拒答。
- 文档重排：用 Cross-Encoder 或 reranker 提升相关性。
- 答案引用校验：生成内容必须能被上下文支持。

### 3. 用复杂度路由控制成本

不是所有请求都值得上最贵模型。可以按问题长度、实体数量、风险等级、工具调用数量、历史失败次数来打分。

低风险请求走小模型，高风险请求走强模型，极高风险请求进入人工审核。

### 4. 把失败路径设计成一等公民

生产 Agent 必须能优雅失败。检索不到资料、模型输出不合规、校验失败、工具超时，都应该有明确 fallback。

最差的失败方式不是「答不上来」，而是「一本正经地胡说」。

### 5. 日志和状态要可审计

OncoAgent 用 AgentState 保存节点输出，天然形成审计轨迹。普通业务系统也建议记录：

- 路由为什么选这个模型？
- RAG 检索到了哪些文档？
- 哪些安全规则触发过？
- 最终答案引用了哪些证据？
- 人工审核是否介入？

没有审计链路，Agent 出问题时你只能猜。

---

## 总结

OncoAgent 最值得学习的地方，不是用了多少热门技术名词，而是它把医疗 AI 的核心风险拆成了工程问题：

- 用复杂度路由解决成本和风险分层。
- 用 Corrective RAG 解决检索不可靠。
- 用 Reflexion + deterministic checks 解决输出安全。
- 用 HITL Gate 解决高风险决策责任。
- 用 Zero-PHI 和本地部署解决隐私合规。
- 用 QLoRA、Unsloth、sequence packing 降低领域微调成本。

一句话总结：**真正能落地的 Agent，不是一个会聊天的大模型，而是一套围绕模型构建的证据、约束、审计和兜底系统。**

这也是未来 AI Agent 工程化最重要的方向。

参考资料：[OncoAgent: A Dual-Tier Multi-Agent Framework for Privacy-Preserving Oncology Clinical Decision Support](https://huggingface.co/blog/lablab-ai-amd-developer-hackathon/oncoagent-official-paper)

欢迎关注收藏我，获取更多硬核技术干货
