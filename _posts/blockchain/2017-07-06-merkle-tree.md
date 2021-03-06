---
layout: post
title: Merkle Tree与区块链
category: blockchain
tags: [blockchain,比特币,Merkle Tree]
---


## 什么是merkle tree

假设你已经知道了什么是哈希算法以及哈希是用来干啥的。

网络传输数据的时候，A收到B的传过来的文件，需要确认收到的文件有没有损坏。如何解决？

有一种方法是B在传文件之前先把文件的hash结果给A，A收到文件再计算一次哈希然后和收到的哈希比较就知道文件有无损坏。

但是当文件很大的时候，往往需要把文件拆分很多的数据块各自传输，这个时候就需要知道每个数据块的哈希值。怎么办呢？

这种情况，可以在下载数据之前先下载一份哈希列表(hash list)，这个列表每一项对应一个数据块的哈希值。对这个hash list拼接后可以计算一个根hash。实际应用中，我们只要确保从一个可信的渠道获取正确的根hash，就可以确保下载正确的文件。


**似乎很完美了。但是还不够好！**


上面基于hash list的方案这样一个问题：

**有些时候我们获取(遍历)所有数据块的hash list代价比较大，只能获取部分节点的哈希。**


有没有一种方法可以通过部分hash就能校验整个文件的完整性呢？


答案是肯定的，merkle tree能做到。它长这样子:

![这里写图片描述](http://static.open-open.com/lib/uploadImg/20140403/20140403101532_513.jpg)


特点如下:

1、数据结构是一个树，可以是二叉树，也可以是多叉树（本BLOG以二叉树来分析）

2、Merkle Tree的叶子节点的value是数据集合的单元数据或者单元数据HASH。

3、Merke Tree非叶子节点value是其所有子节点value的HASH值。

很明显，这种结构跟hash list相比较，根哈希不是用所有的数据块哈希拼接起来计算的，而是通过一个层级的关系计算出来的。

**在上图中，叶子节点node7的value（v7) = hash(f1),是f1文件的HASH;而其父亲节点node3的value = hash(v7, v8)，也就是其子节点node7 node8的值得HASH**

## 其它应用场景

### 文件下载

假设我有两台机器，A和B，有一个文件从A传输到B。B首先获取可信的文件merkle tree，当文件下载完毕后，B通过自己构建的merkle tree根节点和获取的根节点比较，如果不一致,通过这种二叉树的结构可以在log(N)的复杂度快速定位到出错的数据块。


### 副本同步

一个集群里的所有机器，需要保持数据的同步，如果数据不一致，需要快速的定位到不一致的节点。

可以在每台机器上针对每个区间里的数据构造一棵Merkle Tree，这样，在两台机器间进行数据比对时，从Merkle Tree的根节点开始进行比对，如果根节点一样，则表示两个副本目前是一致的，不再需要任何处理；如果不一样，则遍历Merkle Tree，定位到不一致的节点也非常快速

## merkle tree应用在区块链上

下面是本文的重点。

比特币系统的区块链中，每个区块都有一个merkle tree。



![这里写图片描述](http://img.blog.csdn.net/20170122205337108?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd281NDEwNzU3NTQ=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)



可以看到merkle root哈希值存在每一个区块的头部，通过这个root值连接着区块体，而区块体内就是包含着大量的交易。每个交易就相当于前面提到的数据块，交易本身有都有自己的哈希值来唯一标识自己。



### 什么是SPV

为了更好的解释，这里先插播一个概念，SPV。也就是中本聪描述到的“简化支付验证” 正是有了SPV，才让区块链和比特币更加广泛的被传播。

我们大部分人在电脑安装的比特币钱包都是轻量级(非全节点)的，也就是本地并没有所有的区块数据(上百G)。理论上来说，要验证一笔交易，钱包需要遍历所有的区块找到和该笔交易相关的所有交易进行逐个验证才是可靠的。

**但是轻量级的钱包没有完整的数据，如何验证一笔支付的合法性呢?**

merkle tree就起到了关键的作用。

当然SPV有它的局限性(这个有空再写文章细讲)。


### 这里是分割点

----------

比特币系统是如何验证一笔交易的合法性呢？

首先区块链中每个区块中的merkle tree的根哈希都是公开可信的。假设现在alice转账给bob一个比特币，比特币钱包需要确认这笔交易是否被包含在了区块链中。



![这里写图片描述](http://note.youdao.com/yws/public/resource/f3ec62377075710c7b1977cdd33eac94/xmlnote/046CA744D80D4720A500611F898E47B3/2287)


入上图所示，
假设我们要判断HK代表的交易是否存在，只需要生成一个仅有 4 个哈希值长度的 Merkle 路径，来证明区块中存该笔交易。该路径有 4 个哈希值（在图中由蓝色标注）HL、HIJ、HMNOP 和 HABCDEFGH。

由这 4 个
哈希值产生的认证路径， 再通过计算另外四对哈希值 HKL、 HIJKL、 HIJKLMNOP
和 Merkle 树根（在图中由虚线标注），任何节点都能证明 HK（在图中由绿色
标注）包含在 Merkle 根中。


**具体认证路径的生成，主要是通过遍历。有专门的算法，有兴趣的可以自行搜索相关的论文**



**参考**

[1] http://www.cnblogs.com/fengzhiwu/p/5524324.html

[2] <<精通比特币>>

