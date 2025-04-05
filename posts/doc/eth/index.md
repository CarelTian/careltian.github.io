# ETH搭建联盟链的方法


## 1 背景

4年前的时候做区块链项目总喜欢自己造轮子，但是代码基本功不好，为了代码能跑起来删去了大量细节和简化了很多内容。最后只成了一个简单的玩具。我目标是在国内建立一个应用程度高的联盟链平台，在实现目标之前先站在巨人的肩膀上，利用以太坊做链端的部署。

## 2 步骤

使用git clone拉取eth源码仓库，按照文档步骤编译启动。截止到文档编辑日期，使用的版本为geth version 1.13.12

创建一个新目录，然后创建一个`genesis.json`文件，具体参数不细说，按照需要配置。

```json
{
  &#34;config&#34;: {
    &#34;chainId&#34;: 6452,
    &#34;homesteadBlock&#34;: 0,
    &#34;eip150Block&#34;: 0,
    &#34;eip155Block&#34;: 0,
    &#34;eip158Block&#34;: 0,
    &#34;byzantiumBlock&#34;: 0,
    &#34;constantinopleBlock&#34;: 0,
    &#34;petersburgBlock&#34;: 0,
    &#34;istanbulBlock&#34;: 0,
    &#34;berlinBlock&#34;: 0,
    &#34;londonBlock&#34;: 0,
    &#34;clique&#34;: {
      &#34;period&#34;: 15,
      &#34;epoch&#34;: 30000
    }
  },
  &#34;difficulty&#34;: &#34;0x1&#34;,
  &#34;gasLimit&#34;: &#34;0x989680&#34;,
  &#34;extradata&#34;: &#34;0x0000000000000000000000000000000000000000000000000000000000000000ee0b440ff5594029c8260fadccb712cdf021484e0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000&#34;,
  &#34;alloc&#34;: {
    &#34;0xeE0B440Ff5594029C8260FADCcb712CDF021484e&#34;: {
      &#34;balance&#34;: &#34;50000000000000000000&#34;
    }
  }
}
```

`geth --datadir ./data init genesis.json`     初始化节点





geth --datadir ./data --networkid 6452 --port 6452 --nodiscover --http --http.api &#34;eth,net,web3,personal,miner&#34; --mine --miner.etherbase &#34;0xeE0B440Ff5594029C8260FADCcb712CDF021484e&#34; --allow-insecure-unlock --unlock &#34;0xeE0B440Ff5594029C8260FADCcb712CDF021484e&#34; 


---

> 作者: ZergFlood  
> URL: https://careltian.github.io/posts/doc/eth/  

