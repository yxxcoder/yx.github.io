---
layout: post
title:  "Cassandra学习与实践(四)——架构组建"
date:   2018-08-22 22:00:00 +0800
categories: nosql
---

### 什么是Gossip协议

Cassandra使用Gossip协议去获得集群中其他节点的位置和状态信息。Gossip是一个点对点的通信协议，在这个协议中，节点之间定期交换状态信息。Gossip协议每隔一秒运行一次，节点和不超过三个节点交换信息。在交换的信息中，旧的信息会被新的信息覆盖。在同一个cluster集群中应该使用相同的seed节点列表

当一个节点启动的时候，它会去读取配置文件`cassandra.yaml`从而决定它属于哪个集群，从那个节点获得集群中其他节点的信息以及一些其他的的参数：

- `cluster_name`：当前节点所属的Cassandra集群的名称，集群中每一个节点的集群名应该是一样的
- `listen_address`：当前节点的ip地址或主机名，使得其他节点能够联系到这个节点。这个地址应该是公共地址，而不是localhost
- `seed_provider`：通过逗号分隔的seed节点的主机名或ip地址。每个节点的此选项应该是一致的。在包含多个数据中心的集群中，每个数据中心都应该有一个或多个seed节点
- `storage_port`：节点间的通信端口（默认为7000）。集群中每个节点的配置应该是一样的
- `initial_token`：这个参数用于手动指定节点的token（哈希）值，一般不用配置
- `num_tokens`：虚拟节点中定义随机分配到该节点的token的数量。原来情况下，一个节点只有一个token，很容易造成数据都分布在一个节点上的情况，采用虚拟节点，每个节点有多个token，可以平均存放数据，可以避免造成某个节点数据量过大。如果某个节点的机器性能好一些，可以把这个数值设置的大一些，越大存储的数据越多。如果所有的机器配置都一样，建议所有机器节点配置一样的num_token

<br>

### 数据分发与复制

#### 一致性哈希

一致性哈希决定数据存储在哪个节点上，它基于数据的主键，Cassandra为每个主键计算一个hash值，为每个数据节点指定一个其负责的hash值范围，然后根据主键hash值和节点负责的hash值范围将不同的行存储到不同的节点

#### 虚拟节点

由于这种方式会造成数据分布不均的问题，在Cassandra1.2以后采用了虚拟节点的思想：不需要为每个节点分配token，把圆环分成更多部分，让每个节点负责多个部分的数据，这样一个节点移除后，它所负责的多个token会托管给多个节点处理，这种思想解决了数据分布不均的问题

要使用虚拟机节点，配置cassandra.yaml文件中的`num_token`大于1即可（默认值为256）

#### 数据复制

在集群的副本总数被称为复制因子，它决定的是一行数据被存储到多少台机器上面。复制因子为1意味着每一行在一个节点只有一个副本。为2意味着每一行有两个副本，其中每个副本是在不同的节点。所有的副本都同等重要，没有主副之分

- `SimpleStrategy`：单数据中心使用，`SimpleStrategy`根据主键的哈希值，决定每一个副本应该复制到哪个节点，然后不考虑节点之间的拓扑网络结构，其他的副本放置在环中顺时针方向的下一个节点

- `NetworkTopologyStrategy`：当你的集群使用或者计划使用多数据中心时，请使用`NetworkTopologyStrategy`。这种策略指定了每个数据中心有多少个副本


##### 配置多个数据中心的集群最常见的两种方法：

- 每个数据中心两个复制：这种配置允许每个副本组出现单点故障，同时仍然保证本地读取的一致性水平为ONE
- 每个数据中心三个副本：这种配置允许每个副本组出现单点故障，同时保证本地读取的一致性水平为LOCAL_QUORUM。或者多个副本组出现多节点故障，同时保证本地读取的一致性水平为ONE


<br>

### 分区器的概念

我们存储的数据是如何对应到相应的虚拟节点的呢？分区器决定了数据如何在集群内被分发。简单来说，一个分区器就是一个用来计算partition key哈希值的一个哈希函数。每一行数据由partition key的值唯一标识，并且根据partition key的哈希值决定如何在集群内被分发

#### Cassandra提供了以下这些分区器

- Murmur3Partitioner（默认）：基于MurmurHash哈希算法
- RandomPartitioner：基于MD5哈希算法
- ByteOrderedPartitioner：根据partition key的bytes进行有序分区

#### Murmur3Partitioner：基于MurmurHash哈希算法

Murmur3Partitioner和RandomPartitioner都使用哈希值来平均分配一个column family的数据到集群上的所有节点。Murmur3Partitioner使用MurmurHash函数，MurmurHash哈希函数为每个行键值创建了64byte的哈希值。哈希数值的范围从 -2<sup>63</sup> 到 2<sup>63</sup> 

#### RandomPartitioner：基于MD5哈希算法

RandomPartitioner是Cassandra 1.2之前的默认分页器。RandomPartitioner分发数据使用MD5哈希函数计算行键值，数值范围从 0 到 2<sup>127</sup> -1

#### ByteOrderedPartitioner：根据partition key的bytes进行有序分区

ByteOrderedPartitioner用于基于partition key的bytes进行有序分区。它允许按照partition key进行条件范围查询。也就是说，可以像关系型数据库的主键那样，通过游标，有序遍历所有的partition key。例如，如果使用用户名作为partition key，可以范围查询所有名字在Jake和Jeo之间的用户。这样的条件范围查询，在使用随机分区器时，是不允许的

尽管范围查询听上去很有吸引力，**并不推荐使用有序的分区器，因为：**

- 负载均衡更困难
- 顺序的分区容易造成热点（hotspot）
- 多表时负载不均衡


<br>

### 告密者（Snitch）

Snitch决定应当从哪个数据中心和机架写入和读取数据




















