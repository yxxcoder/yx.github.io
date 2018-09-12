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

默认情况下，一个客户端的Cluster实例管理了集群中所有节点的连接，并且，将客户端请求随机连接到任意节点。不过，在某些情况下，尤其是多数据中心时，默认行为的性能可能不是最优的

例如，如果集群包含两个数据中心，一个在中国，一个在美国。如果客户端在中国，那么，应该避免直接连接美国的节点，不然太慢

Cluster实例的Load balancing policies设置，决定了，客户端请求如何分配节点的连接

Datastax driver中最常用的内置Load balancing policies是`DCAwareLoadBalancePolicy`，它可以指定客户端请求只连接到指定的数据中心的节点

当然，也可以实现自定义的load balanacing policies，实现自定义的客户端连接分配

<br>

## 数据如何写入到一个数据节点

如果所有的节点在同一个数据中心，coordinator会将写请求发到所有的replica。如果所有的replica节点可用，无论客户端指定的[consistency level](https://yxxcoder.github.io/nosql/2018/07/03/Cassandra%E5%AD%A6%E4%B9%A0%E4%B8%8E%E5%AE%9E%E8%B7%B5(%E5%9B%9B)-%E6%95%B0%E6%8D%AEReplication.html)是什么，每一个replica都会处理写请求。写操作的consistency level只决定了coordinator在通知客户端写操作成功之前，多少个replica节点必须返回成功

例如，一个单数据中心集群有10个节点，replication factor值为3，一个写请求，会被发给所有的三个replica节点。如果，客户端指定的consistency level是ONE，那么，只要任何一个replica节点通知coordinator写成功，coordinator就会返回客户端写成功。一个节点返回写成功，表示，写数据已经保存到这个节点的内存中的memtable和磁盘上的commit log

![column](https://yxxcoder.github.io/images/单数据中心.png)

如果是多数据中心部署，Cassandra为了优化写性能，对于远程数据中心，会在每个远程数据中心的节点中，选择一个作为远程数据中心中的coordinator。因此，和远程数据中心里的replica nodes的通信，本地数据中心的coordinator，只需要和每个远程数据中心的这一个coordinator节点通信。

如果使用consistency level ONE或者LOCAL_QUORUM，coordinator只需要和本地数据中心的节点通信。因此，可以避免，跨数据中心的的通信影响写操作性能

![column](https://yxxcoder.github.io/images/双数据中心.png)











