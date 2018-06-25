---
layout: post
title:  "Cassandra实践——安装与配置"
date:   2018-06-25 17:00:00 +0800
categories: jekyll update
---

## Overview

Apache Cassandra是一个超高扩展性的开源NoSQL数据库。

Cassandra是当同时需要高扩展性、高可用性、高性能、跨多数据中心和在云端管理超大量结构化、半结构化和非架构化数据时的理想解决方案。

Cassandra具有持续可用性， 线性扩展性，易于管理大量服务器和不会有单点故障等特性。

Cassandra的数据模型支持非常方便的列索引，高性能的类日志结构（log-structured）数据更新, 强大的非规格化（denormalization）和物化（materialized）视图和缓存功能。





## 安装

### 准备

最新版本的Java 8（Oracle Java Standard Edition 8或OpenJDK 8）。可使用java -version命令验证是否安装了正确版本的JDK。

cqlsh依赖Python 2.7。可使用python -version命令验证是否安装了正确版本的Python。



### 二进制压缩包安装

- 下载最新版本的[Cassandra安装文件](http://cassandra.apache.org/download/)。

- 解压压缩包

  `tar -zxvf apache-cassandra-3.6-bin.tar.gz`

  ​

### Docker 安装

- 执行以下命令:

   `docker run --name some-cassandra -d cassandra:tag`

   ​

### 启停操作

- 进入Cassandra安装目录，通过从命令行调用`bin/cassandra -f`在前台启动Cassandra，通过“Control-C”停止

- 通过从命令行调用`bin/cassandra`在后台启动Cassandra，调用`kill pid`或`pkill -f CassandraDaemon`停止Cassandra，其中pid是Cassandra进程id，可以通过调用pgrep -f CassandraDaemon找到

- 通过从命令行调用`bin/nodetool status`来验证Cassandra是否正在运行

  ​


### 配置Cassandra

要在单个节点上运行Cassandra，并不需要更改任何配置。但是，当部署节点群集或使用不在同一主机上的客户端时，则必须更改一些参数

可以在压缩包的`conf`目录中找到Cassandra的配置文件。对于软件安装包，配置文件将位于`/etc/cassandra`中

#### 基本配置

Cassandra中的大部分配置都是可以在cassandra.yaml文件中设置的

- `cluster_name`:集群的名称
- `seed`: 集群种子节点IP列表，通过逗号分隔
- `storage_port`: 不一定需要更改此设置，但需确保没有防火墙阻止此端口
- `listen_address`: 
- `listen_interface`: 
- `native_transport_port: `



