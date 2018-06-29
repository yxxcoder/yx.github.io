---
layout: post
title:  "Cassandra实践——安装与配置"
date:   2018-06-25 17:00:00 +0800
categories: jekyll update
---

## Overview

Apache Cassandra是一个超高扩展性的开源NoSQL数据库

Cassandra是当同时需要高扩展性、高可用性、高性能、跨多数据中心和在云端管理超大量结构化、半结构化和非架构化数据时的理想解决方案

Cassandra具有持续可用性， 线性扩展性，易于管理大量服务器和不会有单点故障等特性

Cassandra的数据模型支持非常方便的列索引，高性能的类日志结构（log-structured）数据更新, 强大的非规格化（denormalization）和物化（materialized）视图和缓存功能



## 安装

### 准备

最新版本的Java 8（Oracle Java Standard Edition 8或OpenJDK 8）。可使用java -version命令验证是否安装了正确版本的JDK

cqlsh依赖Python 2.7。可使用python -version命令验证是否安装了正确版本的Python



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

- `cluster_name`:  集群的名称

- `seed`:  集群种子节点IP列表，通过逗号分隔，负责处理现有集群中新加入的节点，建议每个数据中心三个种子节点

- `storage_port`:  不一定需要更改此设置，但需确保没有防火墙阻止此端口

- `listen_address`:  节点的ip地址，以允许其他节点与此节点进行通信，或者可以设置listen_interface来告诉Cassandra使用哪个接口、哪个地址。listen_address与listen_interface只能设置一个

- `rpc_address`:  配置为节点的ip地址，提供RPC服务

- `native_transport_port `:  不需要更改此设置，确保此端口未被防火墙阻止，客户端将在此端口上与Cassandra进行通信

  ​

#### 多数据中心配置

1. **cassandra.yaml文件**

   - `seed`:  集群种子节点IP列表，通过逗号分隔，负责处理现有集群中新加入的节点，建议每个数据中心三个种子节点
   - `endpoint_snitch`:  单数据中心默认为`simpleSnitch`，多数据中心时官方推荐在生产环境下使用`GossipingPropertyFileSnitch`，节点的rack和dc名字保存在`cassandra-rackdc.properties`，并且会通过`gossip`这个`p2p`协议传播到所有节点上去。如果`cassandra-topology.properties`文件存在，`cassandra`会把两个`properties`文件的结果合并，如果两个`properties`文件里面有有同一个节点的配置，以`cassandra-rackdc.properties`的配置为准。本教程配置为`GossipingPropertyFileSnitch`

2. **cassandra-rackdc.properties文件**

   - `dc=DC1`:   数据中心名称
   - `rack=RAC1`:  机架名称，防止整个机框掉点，数据丢失

3. **创建keyspace，制定拓扑策略**

   在多数据中心环境建keyspace的时候需要指定拓扑策略：

   ```s
   CREATE KEYSPACE "test_keyspace" WITH REPLICATION = {'class' : 'NetworkTopologyStrategy', 'dc1' : 3, 'dc2' : 2};
   ```


   意思是采用网络多数据中心策略，有两个数据中心，dc1的replica factor为3，dc2的replica factor为2



#### 目录路径

通过配置cassandra.yaml文件中的以下变量改变目录的路径: 

- `data_file_directories`:  数据文件存放目录

- `commitlog_directory`:  commitlog文件所在的目录

- `saved_caches_directory`:  缓存存放目录

- `hints_directory`:  hints文件存放目录

  ​

#### 环境变量

JVM级别的设置（如堆大小）可以在`cassandra-env.sh`中设置，可以将任何其他JVM命令行参数添加到`JVM_OPTS`环境变量；当Cassandra启动时，这些参数将被传递给JVM



####日志

Cassandra使用的日志框架是logback，可以通过编辑`logback.xml`来更改日志记录属性。默认情况下，将`INFO`级别日志写入`system.log`文件，将`DEBUG`级别日志写入`debug.log`文件中。在前台运行时，还会将`INFO`级别日志输出到控制台