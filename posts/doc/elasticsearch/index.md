# Elasticsearch 笔记


## 基础介绍

Elasticsearch 是一个分布式架构的开源搜索引擎，使用全文检索引擎作为底层技术实现，支持搜索、数据存储和分析的功能，适用于海量数据的实时搜索和分析场景。

数据模型：文档模型（JSON格式）

查询语言：基于RESTful API和DSL

使用场景：全文搜索，大量数据的实时分析，高速读写

索引机制：倒排索引

扩展性：易于水平扩展



#### 节点角色

Master：管理集群状态，执行分片分配，节点加入和退出

Data: 存储数据，负责CRUD

Coordinating：作为请求的协调节点，将请求分发到其他节点并汇总结果

Ingest：负责数据预处理

Machine learning： 负责运行机器学习任务，数学建模等

Remote Cluster client：允许连接和检索远程集群数据


---

> 作者: ZergFlood  
> URL: https://careltian.github.io/posts/doc/elasticsearch/  

