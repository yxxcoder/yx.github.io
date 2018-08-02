---
layout: post
title:  "Cassandra学习与实践(三)——存储引擎"
date:   2018-08-01 12:00:00 +0800
categories: nosql
---

## CommitLog

CommitLog是记录Cassandra节点本地所有变动的日志。 写入Cassandra的任何数据在写入memtable之前都将首先写入CommitLog日志。 这在意外关闭的情况下提供了耐久性。 启动时，提交日志中的任何变动都将应用到memtables

所有数据写入操作都通过存储在commitlog段中进行优化，从而减少了写入磁盘所需的查找次数。Commitlog Segments受`commitlog_segment_size_in_mb`选项的限制，一旦达到大小，就会创建一个新的commitlog段。当所有数据刷新到SSTable后，Commitlog中的数据可以归档，删除或回收。当Cassandra当前写的数据的时间早于SSTables中的数据时，Commitlog会被截断。 在停止Cassandra之前运行`nodetool drain`会将memtables中的所有内容写入SSTables，并且无需在启动时与commitlogs同步

- `commitlog_segment_size_in_mb`：默认大小为32

