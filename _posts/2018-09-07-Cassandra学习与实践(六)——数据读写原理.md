---
layout: post
title:  "Cassandra学习与实践(六)——数据读写原理"
date:   2018-09-07 12:00:00 +0800
categories: nosql
---

*参考资料：*

*极客学院：http://www.jikexueyuan.com*

*Cassandra官方文档：http://cassandra.apache.org/doc/latest/faq/index.html*

*学习Cassandra：https://teddyma.gitbooks.io/learncassandra_cn/content/*



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



### Load Balancing Policies



















