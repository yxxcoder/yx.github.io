---
layout: post
title:  "Cassandra学习与实践(四)——架构组建"
date:   2018-08-22 22:00:00 +0800
categories: nosql
---

### 什么是Gossip协议

Cassandra使用Gossip协议去获得集群中其他节点的位置和状态信息。Gossip是一个点对点的通信协议，在这个协议中，节点之间定期交换状态信息。Gossip协议每隔一秒运行一次，节点和不超过三个节点交换信息。在交换的信息中，旧的信息会被新的信息覆盖。在同一个cluster集群中应该使用相同的seed节点列表

当一个节点启动的时候，它会去读取配置文件`cassandra.yaml`从而决定它属于哪个集群，从那个节点获得集群中其他节点的信息以及一些其他的的参数：

- `cluster_name`
- `listen_address`
- `seed_provider`
- `storage_port`
- `initial_token`
- `num_tokens`