---
layout: post
title:  "Cassandra学习与实践(二)——数据Replication"
date:   2018-07-03 17:00:00 +0800
categories: jekyll update
---

## Replication

Cassandra在多个节点存储replica来确保可靠性和容错性。Replication策略决定了replica保存到哪些节点。最长用的两种replication策略为SimpleStrategy和NetworkTopologyStrategy。



### SimpleStrategy

SimpleStrategy允许定义单个整数`replication_factor`，这确定了每行数据的副本节点数。 例如，如果replication_factor为3，则表示每行数据有三个副本，每个副本保存在不同的节点上

SimpleStrategy以同等方式对待所有节点，忽略每个节点的数据中心`dc`和机架`rack`配置，将Token按照从小到大的顺序，从第一个Token位置处依次取N个节点作为副本



### NetworkTopologyStrategy

NetworkTopologyStrategy允许为群集中的每个数据中心指定复制因子。即使您的群集仅使用单个数据中心，NetworkTopologyStrategy也应优先于SimpleStrategy，以便以后更轻松地向群集添加新的物理或虚拟数据中心

除了允许按DC指定复制因子之外，NetworkTopologyStrategy还尝试从不同机架中选择数据中心内的副本。如果机架的数量大于或等于DC的复制因子，则将从不同的机架中选择每个副本。否则，每个机架将至少保留一个副本，但某些机架可能容纳多个副本。但要注意，这种机架感知行为有一些潜在的特殊含义。例如，如果每个机架中没有偶数个节点，则最小机架上的数据负载可能会高得多。同样，如果将单个节点引导到新机架中，它将被视为整个环的副本。因此，许多开发者选择在单个机架上配置所有节点





## 可调节一致性

