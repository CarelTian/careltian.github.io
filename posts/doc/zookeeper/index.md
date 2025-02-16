# 【论文阅读】ZooKeeper


论文链接 [ZooKeeper: Wait-free coordination for Internet-scale systems](https://pdos.csail.mit.edu/6.824/papers/zookeeper.pdf)

## 1 背景

大规模的分布式应用需要不同形式的协调机制，第一是最基本的基于配置的协调形式，第二是组成员关系和领导选举机制，第三是锁能实现原子性操作，控制临界区的访问。一种解决方案是将调度机制开发不同的服务，例如部分服务使用队列，部分服务使用领导者机制。ZooKeeper是一种提供API的方法为程序开发者提供原语，通过协调内核能够在不改变服务核心支持新原语。

Zookeeper API实现了一个类似于文件系统的零等待操作对象，除此之外还使用了FIFO客户端提供顺序执行。总之，ZooKeeper与使用阻塞锁原语的系统有很大差别。

## 2 Zookeeper 服务

### 2.1 服务概念

Zookeeper为客户端提供了一组抽象数据节点（znodes)，这些节点使用层次命名空间组织，类似于UNIX的文件系统，/A/B/C表示znode C的路径，除了临时znodes, 其他都可已有子节点。

常规节点：客户端能显示创建和删除

临时节点：客户端可以显示创建和删除，也可以通过会话系统自动创建或删除

在创建znode，客户端可以设置序列号，单调递增。子节点的序列号不小于其父节点。Zookeeper实现了监视机制，无需轮询就能接收变动通知，只表明数据发生变更，不显示具体内容。这个机制是使用session建立的，客户端连接zookeeper会立即建立session，当session关闭则监视也关闭。例如getData(&#34;/foo&#34;, true)后，/foo被修改了两次，那么就会触发一次监视事件。zookeeper 不仅可以做一般的数据存储，还可以在每个app节点下独立设置一个组员协议。

### 2.2 客户端API

- create(path, data, flags)：创建路径path的 znode，存储 data[]，并返回新 znode 的名称。flags设置常规还是临时。

- delete(path, version): 删除路径名为path的znode,并且可以指定版本号
- exists(path, watch)：返回bool值，可以指定是否监视
- getData(path, watch)：获取znode的数值，可以指定是否监视
- setData(path, data, version):设置数值，并且可以指定版本号
- getChildren(path, watch):返回集合，一个znode下的所有子节点
- sync(path): 等待所有客户端连接服务器中的挂起操作

所有操作可以同时满足同步或异步API，没有并发使用同步，用并发就异步，客户端能保证回调按顺序执行。每个更新方法都会附带版本号用来检查，设置-1表示不检查版本号。

### 2.3 ZooKeeper 保证

线性化写：所有的更新请求都是可序列化，按照优先级执行

FIFO客户端顺序：同一客户端请求先入先出

ZooKeeper的线性化定义是可以满足一个客户端有多个未完成的操作，也就是说可以选择未完成操作使用特定顺序，也可以使用先入先出。设想的场景当系统中一个节点被选为领导者，需要修改节点大量的配置和通知进程，领导者可以创建名为ready的znode，其他进程只有当这个znode存在时才能使用。如何znode在创建之前领导者宕机了，那么其他节点不会使用这个未完成的配置。

另一个问题可能出现在客户端除了 ZooKeeper 之外还有自己的通信渠道时。例如，假设有两个客户端 A 和 B，它们在 ZooKeeper 中共享一个配置，并通过一个共享的通信渠道进行通信。如果 A 修改了 ZooKeeper 中的共享配置，并通过共享通信渠道告知 B 这个更改，B 期望在重新读取配置时看到这个更改。但是，如果 B 所连接的 ZooKeeper 副本比 A 的稍微落后，它可能无法立即看到新配置。为了更高效地处理这种情况，ZooKeeper 提供了 sync：在读取操作之前执行 `sync`，相当于进行了一次“慢读取”。`sync` 会使服务器在处理读取请求之前应用所有待处理的写请求，但不会引入完整写操作的开销。这一原语在概念上类似于 ISIS 系统中的 `flush` 原语

### 2.4 原语例子

尽管zookeeper是零等待的，依然可以实现高效的阻塞原语。

配置管理：可以利用zookeeper为配置文件创建一个znode，程序启动后观察znode是否出现变动。

同步：有时需要按顺序启动一个主节点，多个工作节点，利用znode来存储主节点信息，包括ip和端口

组成员：成员启动时创建临时znode，出现故障会被自动删除

简单锁：利用锁文件的方法，锁表示为一个znode。客户端获取锁表现为创建一个指定名称的znode，释放锁表现为删除这个znode。但是这种方法会出现羊群效应，多个客户端共同等待一个锁，导致某些客户端一直获取不了。一个解决方案是在申请等待队列中使用递增序号，按顺序获取锁。

读写锁：同样是临时节点，使用递增序号避免线程饥饿。

双栅栏：一种同时启动，同时结束的同步操作。代码如下

```java
public class DoubleBarrier {
    private final ZooKeeper zk;
    private final String barrierPath = &#34;/double_barrier&#34;;
    private final int participantCount;

    public DoubleBarrier(ZooKeeper zk, int participantCount) {
        this.zk = zk;
        this.participantCount = participantCount;
    }

    public void enter() throws Exception {
        String nodePath = barrierPath &#43; &#34;/node_&#34;;
        zk.create(nodePath, new byte[0], ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL);
        
        while (true) {
            List&lt;String&gt; children = zk.getChildren(barrierPath, true);
            if (children.size() == participantCount) {
                break;
            }
            Thread.sleep(100);
        }
        System.out.println(&#34;All processes have entered. Start working!&#34;);
    }

    public void leave() throws Exception {
        String nodePath = barrierPath &#43; &#34;/node_&#34;;
        zk.delete(nodePath, -1);
        
        while (true) {
            List&lt;String&gt; children = zk.getChildren(barrierPath, true);
            if (children.isEmpty()) {
                break;
            }
            Thread.sleep(100);
        }
        System.out.println(&#34;All processes have finished. Barrier released!&#34;);
    }
}

```



## 3 Zookeeper 应用场景

主要谈及在雅虎的应用服务

## 4 Zookeeper 实现

 ZooKeeper 对请求进行预处理，如果请求涉及服务器间的协作，比如写操作，则会启动一个基于原子广播协议的共识机制。这种机制确保所有服务器最终将请求导致的变更同步至完全复制的数据库中，从而维护数据的一致性。如果只读请求，则可以直接从服务器本地的数据库副本中获取数据并形成响应，无需触发复杂的共识过程，这大大提升了读取操作的效率。

备份数据库是保存在内存中，每个znode默认最大保存1MB数据，可以允许改变。为了保证可恢复性，定期将写操作保存到硬盘生成快照。

### 4.1请求处理

由于消息层是原子性的，要保证本地副本不会出现分歧，尽管在某一时刻应用了多个事务。当领导者接收到写请求后，必须要要计算出未来状态，因为可能存在尚未保存在数据库的事务。比如客户端发出setData，版本号与未来的版本号相匹配，那么服务器才会生成对应的setDataTXN，否则只会返回errorTXN。

### 4.2 原子广播

所有更新Zookeeper的请求会转发给领导者，领导者会执行命令并通过Zab广播改变的状态。Zab使用简单的大多数节点一致协议，只要保证多数服务器正确就能正常工作。(2f&#43;1)

为了实现高吞吐量，使用了管道满载，并保证强顺序一致性。领导者的广播按照顺序交付，旧领导者的广播保证在新领导者发送之前交付。Zab的领导者同时也是Zookeeper的领导者，有时事务会在网络重转中执行两次，由于事务幂等性，这是可以接受的。

### 4.3 复制数据库

每个复制体都会在内存中有zookeeper状态的拷贝，当服务崩溃时，会从最近的状态恢复。这里提到了模糊快照的概念，使用DFS遍历znode，将原数据写入磁盘，读写并发可能会导致缺失某些更新。由于事务幂等性，重新回放最近的事务日志即可恢复到最后一致的状态。

### 4.4 客户端服务器交互

服务器在处理写请求会发送和清除相关更新的监视通知，保证通知是顺序的在本地处理。读请求也是在本地处理，携带一个zxid, 表示服务看到的最后一个事务。快速读取的缺点是不能保证读的顺序性，所以实现了Sync原语操作来确保读到是最新的数据。

客户端和服务器建立通信前会检查zxid,如果服务器发现客户端持有最新的zxid，那么就不会建立连接，客户端能保证连接到的是最新数据。

## 5 评价

这篇论文读起来没有其他分布式系统论文那么舒服，因为没有对系统的细节做太多说明。这部分内容需要继续在开源代码里研究。


---

> 作者: ZergFlood  
> URL: https://careltian.github.io/posts/doc/zookeeper/  

