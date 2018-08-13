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

### 为什么nodetool ring只显示一个条目，即使在节点记录中看到各个节点彼此加入了ring？

当您为每个节点分配相同的令牌(token)时会发生这种情况。不要那样做。通常是因为在VM上安装Cassandra，然后将该VM克隆到其他节点造成的（特别是在使用Debian软件包时，安装后自动启动Cassandra，从而生成并保存令牌），最简单的解决方法是擦除数据和commitlog目录，从而确保每个节点在下次重启时生成随机令牌(token)

### 可以更改正在运行中的Cassandra群集的副本数量（keyspace）吗

可以的，但需要运行full repair（或cleanup）才能更改现有数据的副本数量：

1. 更改keyspace的复制因子（ [ALTER KEYSPACE](http://cassandra.apache.org/doc/latest/cql/ddl.html#alter-keyspace-statement)）
2. 如果要减少副本数量，请在群集上执行nodetool cleanup`以删除多余的副本数据。 需要在每一个节点执行此命令
3. 如果要增加副本因子，请运行`nodetool repair -full`以确保根据新配置复制数据。以每个副本集为单位运行此命令。这是一个密集的过程，可能会导致不良的集群性能。 强烈建议进行滚动修复，因为试图立即修复整个集群很可能会出问题。 注意，您需要运行完整修复（`-full`）以确保不会跳过已修复的sstables