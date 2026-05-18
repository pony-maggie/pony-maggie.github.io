+++
date = 2026-05-18T22:05:39+08:00
draft = false
title = "连续批处理异步化：Hugging Face 如何把 LLM 推理吞吐率再抬一截"
+++

在大模型推理里，吞吐率不是只看 GPU 算得快不快，还要看 CPU 和 GPU 有没有抢跑。Hugging Face 这篇文章讲得很直白：传统 continuous batching 虽然减少了 padding 浪费，但 CPU 负责组 batch、更新状态，GPU 负责前向计算，这两者还是同步轮流干活，空档一多，GPU 就会闲着。

这篇文章的核心价值，不是再讲一遍 continuous batching，而是把它进一步改造成异步流水线：CPU 在准备 batch N+1 时，GPU 已经在算 batch N。最后的效果也很实在：同样是 8K token、batch size 32、8B 模型，GPU 活跃时间从 76.0% 提到 99.4%，总耗时从 300.6 秒降到 234.5 秒，约 22% 的加速。

## 先看结论

如果你只想记住一句话：

> continuous batching 解决了别浪费 token，async batching 解决了别浪费时间。

前者优化的是 batch 内部效率，后者优化的是 CPU 与 GPU 的并行度。两者叠加，才是真正把推理吞吐率往上拱。

~~~mermaid
flowchart LR
  subgraph S1[同步批处理]
    A1[CPU 准备 batch N] --> B1[GPU 计算 batch N]
    B1 --> C1[CPU 采样和更新状态]
    C1 --> D1[CPU 准备 batch N+1]
  end

  subgraph S2[异步批处理]
    A2[CPU 准备 batch N+1] --- B2[GPU 计算 batch N]
    A2 --> C2[提交 H2D / compute / D2H]
    B2 --> D2[事件同步后读取结果]
  end
~~~

## 为什么同步批处理还不够

很多人看到 continuous batching，会自然以为已经把 GPU 利用率拉满了。其实没有。

它减少了无效 padding，但流程还是这样的：

1. CPU 挑选请求、更新 KV cache 表、剔除完成请求、补入新请求。
2. CPU 把输入搬到 GPU。
3. GPU 做 forward，采样下一个 token。
4. 结果回到 CPU。
5. CPU 再开始下一轮。

问题在于，这个循环里 CPU 和 GPU 是串行接力，不是并行协作。GPU 算的时候 CPU 在等，CPU 整理的时候 GPU 也在等。对单轮推理看似损失不大，但在每秒几百次的调度循环里，这些空档会迅速变成吞吐率损耗。

Hugging Face 统计的结果很有代表性：在 8K token 的生成任务里，GPU 有 24% 的时间在等 CPU。也就是说，光把等人这件事干掉，就能白捡一大截性能。

## 第一步：把 GPU 任务拆成三条流水线

作者的思路不是上来就做复杂优化，而是先把一次 batch 的 GPU 工作拆开：

1. Host-to-Device，简称 H2D，输入从 CPU 拷到 GPU。
2. Compute，GPU 前向计算。
3. Device-to-Host，简称 D2H，输出从 GPU 拿回 CPU。

这三段天然不是一个东西，没必要放在同一条串行队列里。于是就有了三个 CUDA stream：

- h2d_stream
- compute_stream
- d2h_stream

问题来了。只把它们拆开还不够，因为 stream 之间是独立的。如果你不加约束，它们会同时跑，结果就是 compute 可能在 H2D 还没完成时就开算，D2H 可能在结果还没生成时就开始搬数据，直接出错。

~~~mermaid
sequenceDiagram
  participant CPU
  participant H2D as H2D Stream
  participant CMP as Compute Stream
  participant D2H as D2H Stream

  CPU->>H2D: enqueue input copy
  CPU->>CMP: enqueue forward pass
  CPU->>D2H: enqueue output copy
  Note over H2D,CMP,D2H: 没有依赖时会并发乱跑
~~~

## 第二步：用 CUDA event 把依赖关系钉死

这篇文章最关键的工程手法，就是 CUDA event。

event 的作用很简单：在某个 stream 上插一个完成标记，然后让另一个 stream 等这个标记。这样依赖就从 CPU 手里挪到 GPU 侧了，CPU 只负责发命令，不再负责傻等。

在这套设计里：

- H2D 完成后，记录 h2d_done
- Compute 开始前，等待 h2d_done
- Compute 完成后，记录 compute_done
- D2H 开始前，等待 compute_done

最终就形成了一条明确的 GPU pipeline：

~~~mermaid
flowchart LR
  H2D[H2D Stream] --> E1[(h2d_done)]
  E1 --> CMP[Compute Stream]
  CMP --> E2[(compute_done)]
  E2 --> D2H[D2H Stream]
  D2H --> CPU[CPU 读取结果]
~~~

这里有个容易忽略的点：wait 卡的是 stream，不是 CPU。也就是说，CPU 发完命令就能继续做下一轮 batch 的准备工作，这才是异步化真正的意义。

## 第三步：双缓冲，避免数据互相踩踏

把 CPU 和 GPU 并行之后，马上会遇到新问题：batch N 还在 GPU 上跑，CPU 已经开始准备 batch N+1 了。那输入缓存和输出缓存如果共用，很容易把正在使用的数据覆盖掉。

解决办法很朴素，也很有效：双缓冲。

- slot A 给 batch N
- slot B 给 batch N+1
- 两边轮换使用

这样 CPU 可以放心写下一批输入，GPU 也可以放心读上一批数据。代价是多占一份 RAM 和 VRAM，但这笔账通常很划算，尤其是配合 FlashAttention 这类不需要大 attention mask 的实现时。

不过双缓冲又引出一个新问题：CUDA graph 是绑定内存地址的。slot A 和 slot B 不能共用同一份图，所以需要两份 graph。作者的解法是再往前走一步，做一个 shared memory pool，让不同 graph 从同一池子里分配内存。只要同一时刻不会并发执行，这样就能把 VRAM 开销压下来。

## 第四步：carry-over，把上一轮 token 接到下一轮

continuous batching 里还有一个很实际的问题：同一个请求，batch N 的输出 token 会变成 batch N+1 的输入 token。

但 batch N+1 在准备输入时，batch N 还没算完，所以这一步不能硬等。作者的做法是：

1. 先用占位 token 把 batch N+1 组出来。
2. batch N 结束后，再把真正的新 token carry over 到 batch N+1 的输入里。

这一步通过一个 carry-over mask 来完成。它告诉系统哪些位置需要把 batch N 的输出复制过去，哪些位置保持不变。这样，CPU 在准备 batch N+1 的时候可以提前完成大部分工作，GPU 只在最后补上这几个新 token。

~~~mermaid
flowchart TD
  N[batch N 正在计算] --> O[产生新 token]
  P[batch N+1 先用占位 token 组装] --> M[carry-over mask]
  O --> M
  M --> Q[把新 token 写回 batch N+1]
~~~

## 结果为什么会这么明显

因为这套方案没有改模型，也没有发明新 kernel，核心只是把谁先谁后改成了谁能先跑就先跑。

作者的实测结果很漂亮：

- GPU active time：76.0% -> 99.4%
- 总耗时：300.6s -> 234.5s
- 提升幅度：22%

这类收益有个特点：它不是靠单点技巧堆出来的，而是靠调度模型本身的结构优化出来的。换句话说，性能不是算得更快，而是别让硬件闲着。

## 对工程实现的启发

如果你在做推理服务、Agent runtime 或者长上下文系统，这篇文章给了三个很实用的启发：

1. 不要默认 CPU 和 GPU 必须同步。
2. 先拆任务，再谈优化。
3. 所有跨阶段依赖，都尽量显式化，别藏在调用栈里。

真正高吞吐的系统，往往不是某个算子特别快，而是调度、缓冲、同步、缓存这些看起来不性感的部分做对了。

如果你只盯着模型本身，很容易忽略推理管线里的空转时间。可一旦请求量上来，吞吐率的大头，常常就藏在这些细节里。

## 参考资料

参考资料：[Hugging Face Blog - Unlocking asynchronicity in continuous batching](https://huggingface.co/blog/continuous_async)
