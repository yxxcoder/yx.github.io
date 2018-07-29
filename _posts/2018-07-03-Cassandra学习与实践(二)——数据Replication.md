---
layout: post
title:  "Cassandra学习与实践(二)——数据Replication"
date:   2018-07-03 17:00:00 +0800
categories: jekyll update
---

## Replication

Cassandra在多个节点存储replica来确保可靠性和容错性。Replication策略决定了replica保存到哪些节点。最长用的两种replication策略为SimpleStrategy和NetworkTopologyStrategy。



### SimpleStrategy

SimpleStrategy允许定义单个整数replication_factor，这确定了每行数据的副本节点数。 例如，如果replication_factor为3，则表示每行数据有三个副本，每个副本保存在不同的节点上