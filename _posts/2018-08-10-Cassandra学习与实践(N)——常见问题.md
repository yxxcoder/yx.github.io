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

### 可以在Cassandra中存储BLOB类型数据吗

Cassandra未针对大型文件或BLOB存储进行优化，并且始终会读取单个blob值并将其完全发送到客户端。因此，存储小的blob（小于1MB）应该不是问题，但建议手动将大的blob字段拆分成较小的块

请特别注意，默认情况下，由于Cassandra配置文件文件的`max_mutation_size_in_kb`配置（默认为`commitlog_segment_size_in_mb`的一半，而`commitlog_segment_size_in_mb`默认为32MB），Cassandra将拒绝任何大于16MB的值

### 执行Nodetool命令返回“拒绝连接主机：127.0.1.1”。 这是为什么？

Nodetool依赖于JMX，而JMX又依赖于RMI，而RMI又在交换的每一端根据需要设置自己的监听器和连接器。 通常所有这些都是在后台自动执行的，但是主机连接或连接到的主机连接的名称解析不正确可能导致交叉连线并出现异常

如果您没有使用DNS，请确保您的`/etc/hosts`文件在两端都是争取的。 如果失败，请尝试将`cassandra-env.sh`底部附近的`-Djava.rmi.server.hostname = <public name>` JVM选项设置为可从远程计算机访问

### 批量处理操作可以加速我的大容量载入吗

不会。使用批量加载数据通常只会增加延迟的“尖峰”。 请改用异步的INSERT，或使用争取的Bulk Loading

一个例外是将更新批处理到单个分区，这可能是一个好的做法（只要单个批处理的大小保持合理）。 但永远不要盲目地批量处理一切

### 在RHEL节点上无法加入ring

检查SELinux是否已打开; 如果是，请将其关闭

### 如何取消订阅电子邮件列表

发送电子邮件至`user-unsubscribe@cassandra.apache.org`

### 为什么top显示Cassandra使用的内存比Java heap max多得多

Cassandra在内部使用了内存映射文件（mmap，即[Memory Mapped Files](https://en.wikipedia.org/wiki/Memory-mapped_file)）。也就是说，我们使用操作系统的虚拟内存系统将大量磁盘上的文件映射到Cassandra进程的地址空间。这将“使用”虚拟内存; 如同top工具显示的那样，但在64位系统上，虚拟地址空间实际上是无限制的，所以不必担心

通常意义上的“内存使用”的角度来看，主要是在`brk()`或`mmap’d/dev/zero`上分配的数据量，它们代表所使用的实际内存。 关键问题是对于mmap的文件，永远不需要保留驻留在物理内存中的数据。 因此，无论你做什么，保持驻留在物理内存中基本上只是作为缓存，与普通I/O一样将以内核页（ kernel page）缓存保留您读/写的数据

mmap比IO的一个好处是只要内存里有，你里面就能读到了，不会碰到page fault(没有读系统page缓存内核做semi-context切换的开销)，详情请见[这里](http://www.varnish-cache.org/trac/wiki/ArchitectNotes)

### 什么是种子节点(seed)？

启动期间使用种子节点来发现集群

如果你配置了你的节点指向几个节点作为种子节点，那么你集群中的节点会发送相比非种子节点较多的gossip

信息到种子节点。换句话说，种子是Gossip网络的枢纽。通过种子，每个节点可以快速检测其他节点的状态变化