---
layout: post
title:  "Cassandra学习与实践(四)——架构组建"
date:   2018-08-22 22:00:00 +0800
categories: nosql
---

### 什么是Gossip协议

Cassandra 是无中心化的，每一个节点都可能担任临时协调者角色，也可能担任数据备份角色，这也意味着所有节点没有差异，也不会存在差异，因为所有行为都是按照规则约束的随机行为

为了支持无中心化和分区容错，Cassandra 使用 gossip 协议允许每个节点追踪集群里其他节点的状态信息

Gossip 协议（也叫八卦协议）通常假设在大型、无中心化的网络系统中容易出现网络故障，也被用于分布式数据库内的自动备份机制。本身它的名称就来源于人类的八卦行为，你可以和任何人交换互相感兴趣的信息。Gossip 比较适合在没有很高一致性要求的场景中用作信息的同步。信息达到同步的时间大概是 log(N)，这个 N 表示节点的数量。Gossip协议每隔一秒运行一次，每次和三个节点交换信息。Gossip 中的每个节点维护一组状态，状态可以用一个 key/value 对表示，还附带了一个版本号，版本号大的为更新的状态

在同一个cluster集群中应该使用相同的seed节点列表。当一个节点启动的时候，它会去读取配置文件`cassandra.yaml`从而决定它属于哪个集群，从那个节点获得集群中其他节点的信息以及一些其他的的参数：

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

Cassandra 采用一个叫做 snitches（告密者）的办法决定集群内部每个节点的相对主机距离，用来决定哪一个节点被用来读和写。Snitches 收集整个网络拓扑信息，这样 cassandra 可以有效地路由请求。注意：一个集群中的所有的节点都必须采用相同的Snitch策略

当 Cassandra 发起一个读请求，它需要通过设定的一致性级别与几个备份交互。为了提供读请求的最大速度，Cassandra 选择一个单一的副本用于整个对象的查询，并且要求额外的副本执行 hash 值，用于确保拿到请求数据的最新版本。Snitch 的角色是去帮助确认哪个副本会最快地返回，并且这个副本需要包含查询的所有数据

#### Dynamic snitching

默认所有的snitch都使用了一个动态snitch层，用来监控读请求，当发现有延迟的时候，把请求转发到其他性能比较好的节点

#### Snitch classes

cassandra.yaml中的`endpoint_snitch`参数应设置为实现IEndPointSnitch的类，该类将由dynamic snitch包装，并确定两个端点是在同一个数据中心还是在同一个机架上。 开箱即用，Cassandra提供了snitch实现：

- `SimpleSnitch`：SimpleSnitch是默认使用的snitch，主要用于一个数据中心的情况下。不考虑数据中心和机架的具体信息。 策略顺序为接近，当忽略读修复时可以提高缓存局部性
- `GossipingPropertyFileSnitch`：生产环境推荐使用。本地节点的机架和数据中心在`cassandra-rackdc.properties`中定义，通过gossip协议传播到其他节点。 如果存在`cassandra-topology.properties`，则将其用作后备，允许从PropertyFileSnitch迁移
- `PropertyFileSnitch`：可以在`cassandra-topology.properties`文件中显性配置数据中心和机架的位置信息，这种方式最灵活
- `Ec2Snitch`：将一个简单的集群部署在亚马逊的EC2可以使用Ec2Snitch。这种方式适用于单区域的EC2部署。通过EC2 API加载Region和Availability Zone的信息。Region可视为数据中心，Availability Zone可视为机架。仅适用私有IP，因此这不适用于多个区域
- `Ec2MultiRegionSnitch`：使用公共IP作为broadcast_address以允许跨区域连接（因此，您也应该将种子节点`seed`设置为公共IP）。您需要在公共IP的防火墙上打开`storage_port`或`ssl_storage_port`（对于区域内流量，Cassandra将在建立连接后切换到专用IP）
- `RackInferringSnitch`：接近度由机架和数据中心决定，每个节点的IP地址的第二段对应数据中心，第三段对应机架（如192.168.188.101和192.168.199.102认为是一个数据中心的节点，192.168.188.101和192.168.188.102认为是同一个机架的节点）。除非这种情况符合您的部署约定，否则这最好用作编写自定义Snitch类的示例
















