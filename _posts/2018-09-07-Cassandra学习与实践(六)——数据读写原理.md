---
layout: post
title:  "Cassandra学习与实践(六)——数据读写原理"
date:   2018-09-07 12:00:00 +0800
categories: nosql
---

#### *参考资料*

- *极客学院 [http://www.jikexueyuan.com](http://www.jikexueyuan.com)*
- *Cassandra官方文档 [http://cassandra.apache.org/doc/latest/faq/index.html](http://cassandra.apache.org/doc/latest/faq/index.html)*
- *学习Cassandra [https://teddyma.gitbooks.io/learncassandra_cn/content/](https://teddyma.gitbooks.io/learncassandra_cn/content/)*
- *Cassandra读写过程 [https://zhuanlan.zhihu.com/p/30498475](https://zhuanlan.zhihu.com/p/30498475)*


<br>

## 如何连接到一个数据节点

一个客户端请求会连接集群上的哪个节点由客户端driver的配置决定

如果使用[Datastax drivers](http://www.datastax.com/download)，主要由下面这两个客户端配置决定连接到哪个节点：

- Contact points
- Load balancing policies

### Contact Points

Contact points设置的值是一个或者多个节点IP的列表，当driver在客户端创建Cluster实例时，driver首先会按照指定的顺序尝试连接contact points中的节点地址。如果第一个节点连接失败，则尝试连接第二个。只要任何一个节点连接成功，则停止尝试连接后面的节点地址

为什么此时不需要连接所有的节点，是因为集群中的任何一个节点都保存了集群上所有节点的元数据，也就是说，只要连上一个节点，driver就能得到所有其他节点的配置信息。接下来，driver会使用获得到所有节点的配置信息构造连接池（构造连接池的时候，driver会连接所有的节点，但是，不是根据contact points中指定的节点地址，而是根据从服务端获得的所有节点的元数据）。这也意味着，contact points设置中，不必要指定集群中所有节点的地址

最佳实践是，在contact points中设置相对于客户端响应最快的所有节点的地址

```java
Cluster cluster = Cluster
	.builder()
  	.addContactPoints("127.0.0.1", "127.0.0.2")
  	.withloadBalancingPolicy(Policies.defaultLoadBalancingPolicy())
  	.build();
 Session session = cluster.connect();
```

<br>

### Load Balancing Policies

默认情况下，一个客户端的Cluster实例管理了集群中所有节点的连接，并且，将客户端请求**随机连接到任意节点**。不过，在某些情况下，尤其是多数据中心时，默认行为的性能可能不是最优的

例如，如果集群包含两个数据中心，一个在中国，一个在美国。如果客户端在中国，那么，应该避免直接连接美国的节点，不然太慢

Cluster实例的Load balancing policies设置，决定了，客户端请求如何分配节点的连接

Datastax driver中最常用的内置Load balancing policies是`DCAwareLoadBalancePolicy`，它可以指定客户端请求只连接到指定的数据中心的节点

当然，也可以实现自定义的load balanacing policies，实现自定义的客户端连接分配

<br>

### 协调者（Coodinator）

当一个客户端连接一个节点并发出读写请求时，这个节点就是协调者。协调者的作用就像客户端和集群节点之间的代理一样。协调者在基于[分区器配置](https://yxxcoder.github.io/nosql/2018/08/22/Cassandra%E5%AD%A6%E4%B9%A0%E4%B8%8E%E5%AE%9E%E8%B7%B5(%E4%B8%89)-%E6%9E%B6%E6%9E%84%E7%BB%84%E5%BB%BA.html)和[副本策略](https://yxxcoder.github.io/nosql/2018/07/03/Cassandra%E5%AD%A6%E4%B9%A0%E4%B8%8E%E5%AE%9E%E8%B7%B5(%E5%9B%9B)-%E6%95%B0%E6%8D%AEReplication.html)的情况下决定环中的哪个节点会得到请求

<br>

## 数据如何写入到一个数据节点

如果所有的节点在同一个数据中心，coordinator会将写请求发到所有的replica。如果所有的replica节点可用，无论客户端指定的[consistency level](https://yxxcoder.github.io/nosql/2018/07/03/Cassandra%E5%AD%A6%E4%B9%A0%E4%B8%8E%E5%AE%9E%E8%B7%B5(%E5%9B%9B)-%E6%95%B0%E6%8D%AEReplication.html)是什么，每一个replica都会处理写请求。写操作的consistency level只决定了coordinator在通知客户端写操作成功之前，多少个replica节点必须返回成功

例如，一个单数据中心集群有10个节点，replication factor值为3，一个写请求，会被发给所有的三个replica节点。如果，客户端指定的consistency level是ONE，那么，只要任何一个replica节点通知coordinator写成功，coordinator就会返回客户端写成功。一个节点返回写成功，表示，写数据已经保存到这个节点的内存中的memtable和磁盘上的commit log

![column](https://yxxcoder.github.io/images/单数据中心.png)

如果是多数据中心部署，Cassandra为了优化写性能，对于远程数据中心，会在每个远程数据中心的节点中，选择一个作为远程数据中心中的coordinator。因此，和远程数据中心里的replica nodes的通信，本地数据中心的coordinator，只需要和每个远程数据中心的这一个coordinator节点通信

如果使用consistency level ONE或者LOCAL_QUORUM，coordinator只需要和本地数据中心的节点通信。因此，可以避免，跨数据中心的的通信影响写操作性能

![column](https://yxxcoder.github.io/images/双数据中心.png)

<br>

### 写失败时，Hinted Handoff

有时，因为硬件，网络，或因为经历长时间的垃圾回收暂停导致的超载，节点无法回应。Hinted handoff可以允许在集群容量减小的情况下继续执行写请求

在检测到节点down，错过的写请求会在一段时间内存到协调者中，如果在cassandra.yaml文件中开启了hinted handoff。在DataStax5.0后面的版本中，hint存在一个hints的目录中。Hint包含了，down节点的target ID，一个Hint ID（即是一个time UUID），一个message ID（DataStax Enterprise的版本），和数据（blob存储）

Hints每10秒写一次磁盘，避免hints过时。当gossip发现节点恢复时，协调者把保存的写请求线索里的数据写进去，然后删除hint文件。如果一个节点down掉3个小时以上（默认值），协调者不再写入新的hints

协调者会每10分钟检查写hints，可能由于短时间的停电超时以至于gossip没有监测到。如果一个副本超载或者不可用，failure detector还不会把节点标记成down，直到大多数或全部写到那个节点时超时触发失败（默认10秒）。协调者会返回一个TimeOutException异常，写失败了但是hint会被存下来。如果几个节点同时断电，会造成协调者大量的内存压力。协调者追踪有多少hints，如果实在太多，协调者拒绝写，并抛出OverloadedException
<br>
### coordinator挂掉怎么办？

一致性级别会影响hints是否会被写入和后续写请求会不会失败。一个集群有两个节点A和B，RF=1，CL=ONE，每一个行都只存在一个节点中。假设A是协调者，但是在row K写入前down，这种情况，不会满足一致性，因为A是协调者，它不能存hint了。B也不能写数据，因为没有接受到协调者发来的数据也没有hint存下来。协调者检查在线副本数量，如果客户端指定的一致性不能满足不会尝试写hint。一个hinted handoff 错误发生，返回UnavailableException异常。写请求失败，hint没有被写入

通常来说，建议有集群有足够的节点，和足够的RF避免写请求失败。比如一个集群有3个节点A,B,C，RF=2。当row K写入协调者A时，即使down，CL=ONE或者CL=QUORUM也是可以满足的。因为A和B都会接受数据，所以一致性满足。A节点会存储C的hint当C恢复时写入

在非分布式架构中，客户端请求并得到回复如图：

![column](https://yxxcoder.github.io/images/req_and_resp.jpg)

如果在服务器回复前挂掉了，客户端不知道发生了什么，只有当服务器重新上线后重发请求

![column](https://yxxcoder.github.io/images/request.jpg)

客户端可能会连接集群里的任一节点，不管它是不是要读数据的那个节点，叫协调者。它负责路由客户端请求到合适的副本集

如果协调者在途中挂了，和非分布式架构一样，客户端也只能重试。不同的是客户端可以**立马连接集群里别的节点**

![column](https://yxxcoder.github.io/images/request_coordinator.jpg)

如果是副本挂了，协调者没有。有两种情况，第一种是协调者failure detector在请求之前知道副本挂了，所以它都不会再把请求路由到那个节点上面。相反，它立刻回复客户端一个UnavilableException。这是Cassandra唯一会写失败的时候（当太少的副本集在线时，协调者收到了请求，这是唯一会写失败的时候）

![column](https://yxxcoder.github.io/images/request_and_replica_fail.jpg)

如果协调者收到了请求，在发送给副本的时候挂了，协调者返回一个TimedOutException。读写更新这些超时时间可以配置。超时不是失败。协调者把这个操作更新在本地，然后记录在hint里，当副本重新上线时再发给它。这是强制更新后处理（post-update）

![column](https://yxxcoder.github.io/images/request_and_replica_timeout.jpg)

<br>

## 如何从多个数据节点读取数据

客户端请求需要读的replica数量，由客户端指定的consistency level决定。coordinator会将读请求发给响应最快的replica。如果多个节点返回了数据，coordinator会在内存中比较每一列的timestamp，返回合并后的最新的数据

为了确保所有的replica对于经常访问的数据的一致性，在每一次读操作返回之后，coordinator会在后台同步所有其他replica上的该行数据，确保每个replica上拥有该行数据的最新版本

### 单数据中心，consistency level QUORUM

如果是单数据中心，replication factor值为3，读操作consistency level为QUORUM，那么，coordinator必须等待3个replica中的2个返回数据。 如果返回的数据版本不一致，合并后的最新的数据被返回。在后台，第三个replica的数据也会被检查，确保该行数据的最新版本在所有replica的一致性

![column](https://yxxcoder.github.io/images/singleDCConQuorum.svg)

### 单数据中心，consistency level ONE

如果是单数据中心，replication factor值为3，读操作consistency level为ONE，coordinator访问并返回最近的replica返回的数据。在后台，其他两台replica的数据也会被检查，确保该行数据的最新版本在所有replica的一致性

![column](https://yxxcoder.github.io/images/singleDCConOne.svg)

### 双数据中心，consistency level QUORUM

如果是双数据中心，replication factor值为3，读操作consistency level为QUORUM，coordinator必须等待4个replica返回数据。4个replica可以来自任意数据中心。在后台，其他所有数据中心的replica的数据也会被检查，确保该行数据的最新版本在所有replica的一致性

![column](https://yxxcoder.github.io/images/multipleDCConQuorum.svg)

### 双数据中心，consistency level LOCAL_QUORUM

如果是双数据中心，replication factor值为3，读操作consistency level为LOCAL_QUORUM, coordinator必须等待本地数据中心的2个replica返回数据。在后台，其他所有数据中心的replica的数据也会被检查，确保该行数据的最新版本在所有replica的一致性

![column](https://yxxcoder.github.io/images/multipleDCConLocalQuorum.svg)

### 双数据中心，consistency level ONE

如果是双数据中心，replication factor值为3，读操作consistency level为ONE, coordinator访问并返回最近的replica返回的数据，无论该replica是本地数据中心的还是远程数据中心的。在后台，其他所有数据中心的replica的数据也会被检查，确保该行数据的最新版本在所有replica的一致性

![column](https://yxxcoder.github.io/images/multipleDCConOne.svg)

### 双数据中心，consistency level LOCAL_ONE

如果是双数据中心，replication factor值为3，读操作consistency level为LOCAL_ONE，coordinator访问并返回本地数据中心最近的replica返回的数据。在后台，其他所有数据中心的replica的数据也会被检查，确保该行数据的最新版本在所有replica的一致性

![column](https://yxxcoder.github.io/images/multipleDCConLocalOne.svg)

### 使用speculative_retry做快速读保护（Rapid read protection）

快速读保护允许coordinator最开始选择的replica节点down了或者超时了，依然能返回数据。如果一个表配置了speculative_retry参数，假如coordinator最先选择的replica读取超时，coordinator会尝试读取其他可用的replica代替

![column](https://yxxcoder.github.io/images/rapidReadProtection.svg)