# 【论文阅读】MapReduce




论文链接 [MapReduce: Simplified Data Processing on Large Clusters](https://pdos.csail.mit.edu/6.824/papers/mapreduce.pdf)

## 1 背景

2004年正处于互联网起步阶段，谷歌公司为了处理大量的元数据（文档、日志、摘要）需要成百上千台机器处理。这时需要设计一个程序，能够让分布在不同位置的机器并行处理分布式的数据，同时要有容错，简化计算的代码。受到Lisp语言中函数式编程的启发，创造了map和reduce两种操作来处理输入数据。
## 2 编程模型

整个模型的输入和输出都是Key/Value形式
Map: 一个函数由用户编写，输入为分割后的数据块，计算得到中间结果对key/value，然后分组合并发送给Reduce函数
Reduce：同样为用户编写，接受中间键值对，根据key来合并组成一个更小的集合，然后输出最终结果。

```pseudocode {data-open=true}
map(String key, String value):
    // key: document name
    // value: document contents
    for each word w in value:
        EmitIntermediate(w, “1″);
reduce(String key, Iterator values):
    // key: a word
    // values: a list of counts
    int result = 0;
    for each v in values:
        result &#43;= ParseInt(v);
    Emit(AsString(result));
```
下图以一个简单的例子展示词频统计的数据流动过程。
{{&lt; image src=&#34;/img/mapreduce-1.png&#34; alt=&#34;图一&#34; width=&#34;700&#34;&gt;}}
## 3 实现

### 3.1 执行概述

下图以不同职能划分的视角解释MapReduce的过程，
1. MapReduce库首先将输入文件拆分为 M 个部分，每个部分通常为 16 MB 到 64 MB（具体大小可通过可选参数由用户控制）。随后，它在 一组集群机器上启动多个程序副本进行处理
2. 其中一个程序主节点，其余的是工作节点，由主节点分配任务。任务包括 M 个 map 任务 和 R 个 reduce 任务。主节点会选择空闲的工作节点，并为其分配 map 或 reduce 任务。
3. 被分配到map任务的工作节点会读取对应输入分片的内容，对输入数据进行解析，将其拆分为 键值对并将每个键/值对传递给用户定义的 Map 函数。Map 函数生成的中间键值对会暂时缓存在内存中。
4. 缓冲的键值对会被定期写入本地磁盘，并通过分区函数划分为 R 个区域。这些缓冲键值对在本地磁盘上的位置会返回给主节点，主节点负责将这些位置信息转发给 reduce 工作节点。
5. 当主节点通知 reduce 工作节点这些位置后，reduce 节点通过远程过程调用从 map 工作节点的本地磁盘读取缓冲数据。当 reduce 节点读取完所有中间数据后，会根据中间键对数据进行排序，以便将相同键的所有数据分组到一起。由于通常许多不同的键会映射到同一个 reduce 任务，因此排序是必要的。如果中间数据量过大，无法全部加载到内存中，则使用外部排序。
6. reduce 工作节点遍历排序后的中间数据，对于每个遇到的唯一中间键，将该键及其对应的中间值集合传递给用户定义的 Reduce 函数。Reduce 函数的输出会被追加到该 reduce 分区的最终输出文件中。
7. 当所有的 map 任务和 reduce 任务都完成后，主节点唤醒用户程序。此时，用户程序中的 MapReduce 调用返回到用户代码。

{{&lt; image src=&#34;/img/mapreduce-2.png&#34; alt=&#34;图一&#34; width=&#34;700&#34;&gt;}}

### 3.2 master数据结构
它存储每一个map和reduce 任务的状态（空闲、工作中或完成)，以及 worker 机器 (非空闲任务的机器) 的标识。
主节点是中间文件区域位置从 map 任务传播到 reduce 任务的桥梁。因此，对于每个完成的 map 任务，主节点会存储该任务生成的 R 个中间文件区域的位置和大小信息。随着 map 任务的完成，主节点持续接收这些位置和大小信息的更新，并将信息逐步推送给正在执行 reduce 任务的工作节点。
### 3.3 容错机制
worker故障：master周期性ping每个worker，如果出现没有相应，则被认定为故障
出现故障后，master将故障worker正在执行或已产出的map标记为失效，重新调度分配给其他worker。如果产出的reduce已完成，则被视为有效产出

master故障：周期性将数据结构写入磁盘，以最小化损失。出现故障终止程序，等到新master运行接替此检查点继续运行。

出现故障的语义：由于MapReduce操作是原子性的，可以保证输出的正确性和顺序一致。大多数情况可以忽略这方面问题。

### 3.4 存储位置

GFS——多台机器组成的分布式文件系统。将每个文件分钟64MB的块，将多个副本(3份)保存在不同机器中。master在调度的时候会考虑避免网络传输开销，优先在含有map所需文件的机器
执行。	

### 3.5 支援任务

指的是某些机器由于网络差，磁盘读写故障等原因拖后腿，速度远远其他worker。当执行到最后阶段时，无论这个拖油瓶是否正在运行，直接将任务重新分配给其他worker，取最先完成的输出文件为准。

## 4 改进	
### 4.1 分区
使用hash(key) mod R分区，能够比较公平的把相同的Key分到同一个区，并且解决负载均衡问题。

### 4.2 排序
每个分区内保证是根据Key来排序的，这有利于在最终输出文件时支持随机访问。
### 4.3 Combiner 函数
一种提高I/O效率的方法，在本地将中间结果合并后再转发给Reduce，减小网络传输压力和磁盘读写压力。



---

> 作者: ZergFlood  
> URL: https://careltian.github.io/posts/doc/mapreduce/  

