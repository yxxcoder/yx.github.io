---
layout: post
title:  "Cassandra学习与实践(四)——数据Replication"
date:   2018-07-03 17:00:00 +0800
categories: nosql
---

*本文译自Cassandra官方文档：http://cassandra.apache.org/doc/latest/architecture/dynamo.html#replication*

## Replication

Cassandra在多个节点存储副本来确保可靠性和容错性。Replication策略决定了副本保存到哪些节点。最长用的两种replication策略为SimpleStrategy和NetworkTopologyStrategy。

<br/>

### SimpleStrategy

SimpleStrategy允许定义单个整数`replication_factor`，这确定了每行数据的副本节点数。 例如，如果replication_factor为3，则表示每行数据有三个副本，每个副本保存在不同的节点上

SimpleStrategy以同等方式对待所有节点，忽略每个节点的数据中心`dc`和机架`rack`配置，将Token按照从小到大的顺序，从第一个Token位置处依次取N个节点作为replica节点

<br/>

### NetworkTopologyStrategy

NetworkTopologyStrategy允许为群集中的每个数据中心指定复制因子。即使您的群集仅使用单个数据中心，NetworkTopologyStrategy也应优先于SimpleStrategy，以便以后更轻松地向群集添加新的物理或虚拟数据中心

除了允许按DC指定复制因子之外，NetworkTopologyStrategy还尝试从不同机架中选择数据中心内的副本。如果机架的数量大于或等于DC的复制因子，则将从不同的机架中选择每个副本。否则，每个机架将至少保留一个副本，但某些机架可能容纳多个副本。但要注意，这种机架感知行为有一些潜在的特殊含义。例如，如果每个机架中没有偶数个节点，则最小机架上的数据负载可能会高得多。同样，如果将单个节点引导到新机架中，它将被视为整个环的副本。因此，许多开发者选择在单个机架上配置所有节点

<br/>



## 可调节一致性

Cassandra通过一致性级别支持一致性和可用性之间的每次操作权衡。一次操作的一致性级别需要指定多少副本节点响应，以认为操作成功

可以使用以下一致性级别：

| ONE          | 任意一个replica节点响应                          |
| :----------- | ---------------------------------------- |
| TWO          | 任意两个replica节点必须响应                        |
| THREE        | 任意三个replica节点必须响应                        |
| QUORUM       | 大多数（n / 2 + 1）replica节点必须响应              |
| ALL          | 所有replica节点必须响应                          |
| LOCAL_QUORUM | 本地数据中心的大多数replica节点（coordinator所在的数据中心）都必须响应 |
| EACH_QUORUM  | 每个数据中心的大多数replica节点必须响应                  |
| LOCAL_ONE    | 只有一个replica节点必须响应。 在多数据中心集群中，还可以保证读取请求不会发送到远程数据中心的replica节点 |
| ANY          | 任意一个节点操作成功。如果所有的replica节点都挂了，写操作还是可以在记录一个[hinted handoff](http://www.datastax.com/documentation/cassandra/2.0/cassandra/dml/dml_about_hh_c.html#concept_ds_ifg_jqx_zj)事件之后，返回成功。如果所有的replica节点都挂了，写入的数据，在挂掉的replica节点恢复之前，读不到 |

无论一致性级别如何，写入操作始终发送到所有副本。 一致性级别仅控制coordinator在响应客户端之前等待的响应时间

对于读取操作，coordinator通常仅向足够的副本发出读取命令以满足一致性级别。有几个例外：

- 如果副本节点在指定的时间窗口内没有响应，则可能会向额外的副本节点发出冗余读取请求
- 基于`read_repair_chance`和`read_repair_chance`（表的模式的一部分），可以将读取请求随机发送到所有副本，以便修复可能不一致的数据

<br/>

## 选择一致性级别

通常选择高到足以重叠的读写一致性级别，从而产生“强”的一致性。 这通常表示为W + R> RF，其中W是写一致性级别，R是读一致性级别，RF是复制因子。 例如，如果RF = 3，则QUORUM请求将需要来自三个副本中的至少两个的响应。 如果QUORUM用于写入和读取，则至少有一个副本保证参与写入和读取请求，这反过来保证将读取最新的写入。 在多数据中心环境中，LOCAL_QUORUM可用于提供较弱但仍然有用的保证：保证读取可在同一数据中心内查看最新的写入

如果不需要这种强一致性，则可以使用较低的一致性级别（如ONE）来提高吞吐量，延迟和可用性

