---
layout: post
title:  "Cassandra学习与实践(N)——常见问题"
date:   2018-08-10 21:30:00 +0800
categories: nosql
---

### 为什么不能设置`listen_address`为0.0.0.0（监听所有地址）？

​	Cassandra是一个gossip-based的分布式系统，`listen_address`是告诉其它节点访问地址，告诉别的节点说“连接我任何地址都可以”是一个糟糕的想法，如果一个集群中不同的节点使用了不同方式的地址，悲剧将会发生

​	如果您不想为群集中的每个节点手动配置IP，请将其留空，Cassandra将使用`InetAddress.getLocalHost()`来选择地址。 但是要保证这个是正确的地址(/etc/hosts/, dns 等要配置对)

​	JMX是个例外，默认情况下绑定到0.0.0.0 (Java bug 6425769)

### Cassandra用到了哪些端口

​	默认情况下，Cassandra使用7000进行集群间的通信（如果启用了SSL，则为7001），对于本地协议客户端使用9042，JMX使用7199。节点间通信和本机协议端口可在Cassandra配置文件`cassandra.yaml`中进行配置。 JMX端口可在`cassandra-env.sh`中配置（通过JVM选项）。所有端口使用的都是TCP协议

### 添加新节点时，集群中的现有数据会发生什么变化

当新节点加入群集时，它将自动与集群中的其他节点建立连接，并将正确的数据复制到自身节点。同样的增加、替换、移动、删除节点都是这样的。详情请参阅 [Adding, replacing, moving and removing nodes](http://cassandra.apache.org/doc/latest/operating/topo_changes.html#topology-changes)

### 当从Cassandra删除数据时，磁盘使用率保持不变，这是为什么？

写给Cassandra的数据会被持久化到SSTables。由于SSTable是不可变的，因此在执行删除操作时实际上不能删除数据，而是写入标记（也称为“逻辑删除”）以指示值的新状态。逻辑删除后执行压缩操作时，数据将被完全清除并恢复相应的磁盘空间。详情请参阅 [Compaction](http://cassandra.apache.org/doc/latest/operating/compaction.html#compaction)



