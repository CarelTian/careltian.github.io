# 以太坊源码分析（上）


为了更系统的学习Golang和未来的创业项目打基础，本文将开始分析以太坊的源码。

https://github.com/ethereum/go-ethereum

下面是从网上找来的以太坊模块分类表

```Markdown {data-open=true}
accounts        实现了一个高等级的以太坊账户管理
bmt             二进制的默克尔树的实现
build           主要是编译和构建的一些脚本和配置
cmd             命令行工具，又分了很多的命令行工具
common          提供了一些公共的工具类
compression     Package rle implements the run-length encoding used for Ethereum data.
consensus       提供了以太坊的一些共识算法，比如ethhash, clique(proof-of-authority)
console         console类
contracts   
core            以太坊的核心数据结构和算法(虚拟机，状态，区块链，布隆过滤器)
crypto          加密和hash算法，
eth             实现了以太坊的协议
ethclient       提供了以太坊的RPC客户端
ethdb           eth的数据库(包括实际使用的leveldb和供测试使用的内存数据库)
ethstats        提供网络状态的报告
event           处理实时的事件
les             实现了以太坊的轻量级协议子集
light           实现为以太坊轻量级客户端提供按需检索的功能
log             提供对人机都友好的日志信息
metrics         提供磁盘计数器
miner           提供以太坊的区块创建和挖矿
mobile          移动端使用的一些warpper
node            以太坊的多种类型的节点
p2p             以太坊p2p网络协议
rlp             以太坊序列化处理
rpc             远程方法调用
swarm           swarm网络处理
tests           测试
trie            以太坊重要的数据结构Package trie implements Merkle Patricia Tries.
whisper         提供了whisper节点的协议
```

首先分析accounts下的abi目录，时间仿佛回到了3年起，当时对着源码一筹莫展，不了解每个文件的用途，网上参考资料也稀缺，最终不了了之。

abi全称为Application Binary Interface（应用二进制接口），用于智能合约的编码和解码。将solidity类型和golang类型互相转化。


---

> 作者: ZergFlood  
> URL: https://careltian.github.io/posts/doc/eth/  

