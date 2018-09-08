---
layout: post
title:  "Cassandra学习与实践(N)——常见问题"
date:   2018-08-10 21:30:00 +0800
categories: nosql
---

*本文译自Cassandra官方文档：http://cassandra.apache.org/doc/latest/faq/index.html*



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

### 为什么使用nodetool命令时集群列表(ring)只显示一个条目，即使在节点记录中看到各个节点彼此加入了集群？

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

### 在RHEL节点上无法加入集群(ring)

检查SELinux是否已打开; 如果是，请将其关闭

### 如何取消订阅电子邮件列表

发送电子邮件至`user-unsubscribe@cassandra.apache.org`

### 为什么top显示Cassandra使用的内存比Java heap max多得多

Cassandra在内部使用了内存映射文件（mmap，即[Memory Mapped Files](https://en.wikipedia.org/wiki/Memory-mapped_file)）。也就是说，我们使用操作系统的虚拟内存系统将大量磁盘上的文件映射到Cassandra进程的地址空间。这将“使用”虚拟内存; 如同top工具显示的那样，但在64位系统上，虚拟地址空间实际上是无限制的，所以不必担心

通常意义上的“内存使用”的角度来看，主要是在`brk()`或`mmap’d/dev/zero`上分配的数据量，它们代表所使用的实际内存。 关键问题是对于mmap的文件，永远不需要保留驻留在物理内存中的数据。 因此，无论你做什么，保持驻留在物理内存中基本上只是作为缓存，与普通I/O一样将以内核页（ kernel page）缓存保留您读/写的数据

mmap比IO的一个好处是只要内存里有，你里面就能读到了，不会碰到page fault(没有读系统page缓存内核做semi-context切换的开销)，详情请见[这里](http://www.varnish-cache.org/trac/wiki/ArchitectNotes)

### 什么是种子节点(seed)？

启动期间使用种子节点来发现集群

如果你配置了几个节点作为种子节点，那么集群中的节点倾向于向种子节点发送Gossip消息而不是非种子节点。换句话说，种子是Gossip网络的枢纽。通过种子节点，每个节点可以快速检测其他节点的状态变化

当新的节点加入集群时，种子节点也被用来帮助新节点发现集群中的其他节点。当您将新节点添加到集群时，您需要指定至少一个状态正常的节点作为种子节点。一旦节点加入了集群(ring)，它就会感知其他节点，因此在后续的运行中不再需要种子节点

您可以随时将某个节点当作种子节点。种子节点没有什么特别之处，如果在种子列表(`seed`)中列出的节点就是种子节点了

种子不会自动引导（即如果某个节点在其种子列表中有自己的位置，它将不会自动将数据传输到自身）如果您希望节点执行此操作，请先将其引导，然后再将其添加到种子中。 如果您没有数据（新安装），则根本不必担心引导过程

种子节点的推荐用法：

- 选择每个数据中心的两个 (或多个) 节点作为种子节点
- 所有节点配置相同的种子节点列表

### 单种子节点配置是否意味着会发生单点故障

集群可以在没有种子节点的情况下运行或启动; 但是将无法向群集添加新节点。建议在生产环境中配置多个种子节点

### 为什么我不能在jconsole上调用jmx的X方法

一些JMX操作使用数组参数，但由于jconsole不支持数组参数，因此无法使用jconsole调用这些操作（按钮对它们无效）。您需要编写JMX客户端来调用此类操作或需要支持数组的JMX监视工具

### 为什么会在日志中看到“... messages dropped ...”？

这是减载的现象，Cassandra保护自己不会接收到更多的请求

节点接收节点间的消息，但不会在超时后处理（ `read_request_timeout`, `write_request_timeout`, … 请见 [Cassandra Configuration File](http://cassandra.apache.org/doc/latest/configuration/cassandra_config_file.html#cassandra-yaml)），被删除而不是被处理（因为协调器节点将不再等待响应）

对于写入，这意味着该变动未应用于发送到的所有副本节点。不一致将通过读取修复，提示(hints)或手动修复来修复。写操作也有超时时间

对于读取，这意味着读取请求可能尚未完成

减载是Cassandra架构的一部分，如果这是一个持久性的问题，它通常是节点或集群过载的标志

### Cassandra出现java.lang.OutOfMemoryError: Map failed问题导致宕机

如果Cassandra节点宕机并且日志记有“Map failed”记录，则意味着操作系统拒绝java锁定更多内存。 在linux中，这通常意味着memlock是有限的。检查`/proc/<pid of cassandra>/limits`以验证并提高它（例如通过bash中的ulimit命令）。您可能还需要增加`vm.max_map_count`。 请注意，debian安装包会自动为您处理

### 如果使用相同的时间戳进行两次更新会发生什么

更新必须是可代替的，因为它们可能在不同的副本上以不同的顺序到达。只要Cassandra有一种确定性的方式来挑选获胜者（同样的时间戳），所选的就和其他的一样有效，而需要考虑的具体情况应该当作一个实现细节来看待。 也就是说，在时间戳绑定的情况下，Cassandra遵循两条规则：首先，删除优先于插入/更新。 其次，如果有两个更新，则选择具有字典项较大值的更新

### 为什么引导新节点失败并出现“Stream failed”错误

主要两种可能：

1. GC可能会造成长时间暂停，从而中断Stream传输
2. 在后台发生的压缩持有Stream流的时间过长，以使TCP连接失败

在第一种情况下，通常可通过调整GC参数解决该问题。在第二种情况下，需要将TCP keepalive设置为较低的值（Linux上的默认值非常高）。尝试运行以下命令

```shell
$ sudo /sbin/sysctl -w net.ipv4.tcp_keepalive_time=60 net.ipv4.tcp_keepalive_intvl=60 net.ipv4.tcp_keepalive_probes=5
```

若使这些设置持久化，需要添加到`/etc/sysctl.conf`文件中

注意：GCE的防火墙会中断非活跃状态超过10分钟的TCP连接。强烈建议在该环境中运行上述命令