# 【论文阅读】Spanner


论文链接 [Spanner: Google’s Globally-Distributed Database](https://pdos.csail.mit.edu/6.824/papers/spanner.pdf)

## 1 背景

Spanner 是一种可扩展，分布式数据库，本质上也是一种由多个Paxos状态机组成的分片式数据库。它能实现全局数据拷贝，数据迁移。Spanner的目的是为了管理跨数据中心的复制数据，相比较于Bigtable不能很好的完成复杂且变化的模式的应用。而且将原先key-value存储结构改为半关系型表中。

Spanner具备一些特性，第一能够动态控制管理数据中心，读写延迟，第二提供外部全局一致的读写，基于时间戳全局一致性。

## 2 实现

Spanner的部署被称为universe(宇宙？领域？)  。Spanner 被组织为一组名为zones的集合，每个zone可以类比为Bigtable中的服务器。zone是一个管理部署的单元，正如字面的意思，通常可以用地理位置划分。每个数据中心可以有一个或多个zone来操作添加和删除。

如下图所示，每个zone内部包含一个zonemaster和成百上千个spanserver，前者负责传递数据给spanserver, 而spanserver为客户端提供服务。Universemaster通常用来显示每个zone的信息和调试，placement driver负责自动管理跨zone通信，同时也能控制span server数据移动和更新。

{{&lt; image src=&#34;/img/spanner-1.png&#34; alt=&#34;图一&#34; width=&#34;400&#34;&gt;}}



---

> 作者: ZergFlood  
> URL: https://careltian.github.io/posts/doc/spinner/  

