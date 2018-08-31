---
layout: post
title:  "Cassandra学习与实践(五)——存储引擎"
date:   2018-08-01 12:00:00 +0800
categories: nosql
---

*本文译自Cassandra官方文档：http://cassandra.apache.org/doc/latest/architecture/storage_engine.html*

## CommitLog

CommitLog是记录Cassandra节点本地所有变动（mutation）的日志。 写入Cassandra的任何数据在写入memtable之前都将首先写入CommitLog日志。 这在意外关闭的情况下提供了耐久性。 启动时，提交日志中的任何变动（mutation）都将应用到memtables

所有数据写入操作都通过存储在commitlog段中进行优化，从而减少了写入磁盘所需的查找次数。Commitlog Segments受`commitlog_segment_size_in_mb`选项的限制，一旦达到大小，就会创建一个新的commitlog段。当所有数据刷新到SSTable后，Commitlog中的数据可以归档，删除或回收。当Cassandra当前写的数据的时间早于SSTables中的数据时，Commitlog会被截断。 在停止Cassandra之前运行`nodetool drain`会将memtables中的所有内容写入SSTables，并且无需在启动时与commitlogs同步



- `commitlog_segment_size_in_mb`：默认大小为32，这在大多数场景下是没问题的，但是如果要存档commitlog（请参阅commitlog_archiving.properties），那么你可能需要更精细的归档粒度；8 or 16 MB 是合理的。最大的变动（mutation）大小也可通过cassandra.yaml中的`max_mutation_size_in_kb`设置进行配置。默认值为`commitlog_segment_size_in_mb * 1024`的一半

  **注意：如果显式设置了`max_mutation_size_in_kb`，那么`commitlog_segment_size_in_mb`必须设置为`max_mutation_size_in_kb / 1024`大小的至少两倍**


  *Default Value:* `32`

<br/>

- `commitlog_sync`：可以是“定期的”或“批量的”，默认为batch

  - `batch`：在`batch`模式下，Cassandra不会应答（ack）写操作直到commitlog写到（fsync）磁盘上。它会在fsyncs之间等待`commitlog_sync_batch_window_in_ms`毫秒。这个窗口期应保持短，因为writer线程在等待时将无法完成额外的工作。出于同样的原因，您可能需要增加`concurrent_writes`
    - `commitlog_sync_batch_window_in_ms`：在“batch”fsyncs之间等待的时间。默认值为2
  - `periodic`：在`periodic`模式下，写操作会被立即应答（ack），每`commitlog_sync_period_in_ms`毫秒都会简单地同步CommitLog
    - `commitlog_sync_period_in_ms`：在“periodic”fsyncs之间等待的时间。默认值为10000

  **注意：如果发生意外关机, 则如果同步延迟, 则Cassandra可能会失去同步期间或更多的数据。如果使用 "batch" 模式, 建议将 commitlogs 存储在单独的存储设备中**


  *Default Value:* `batch`

<br/>

- `commitlog_directory`：在磁性机械硬盘（magnetic HDD）上运行时， 默认情况下此选项被注释掉，这个目录应该和data目录分开存放到不同的物理磁盘。如果未设置, 则默认目录为 `$CASSANDRA _home/data/commitlog`

  *Default Value:* `/var/lib/cassandra/commitlog`

  <br/>

- `commitlog_compression`：压缩以应用于commitlog。 如果省略，则提交日志将被解压缩。 支持LZ4，Snappy和Deflate压缩器

  *Default Value:* 

  ```shell
  #   - class_name: LZ4Compressor
  #     parameters:
  #         -
  ```


<br/>

- `commitlog_total_space_in_mb`：commitlog占用的总空间。如果使用的空间超过了这个值，Cassandra转到下一个最近的段，刷新那些最旧的commitlog 段对应的memtables到磁盘中，删除这些log段。这减少了启动时重播的数据量，防止不经常更新的表不定期的保留commitlog段。一个小的commitlog总空间会导致不活跃的表产生更频繁的刷新活动

  默认值是 8192与1/4 的commitlog 目录磁盘中的较小值

  *Default Value:* `8192`


<br/>  



## Memtables

Memtables是内存结构，作为Cassandra的写入缓冲。通常，每个表有一个活跃的memtable。最终，memtables 被刷新到磁盘上，成为不可变的 SSTables。这可以通过多种方式触发：

- memtables的内存使用量超过配置的阈值（ `memtable_cleanup_threshold`）
- CommitLog接近其最大大小，并强制执行memtable刷新以允许释放commitlog段

Memtables可以完全存储在堆上或部分堆外存储，具体取决于`memtable_allocation_type`

<br/>  



## SSTables

SSTables是Cassandra用于在磁盘上保存数据的不可变数据文件。由于SSTables从Memtables刷新到磁盘或从其他节点流式传输，Cassandra会触发将多个SSTable组合成一个的压缩

由于SSTables从Memtables刷新到磁盘或从其他节点流式传输，因此Cassandra会触发压缩将多个SSTable组合成一个。 一旦编写了新的SSTable，就可以删除旧的SSTable

<br/>

#### 每个 SSTable 由存储在不同文件中的多个组件组成：

#### `Data.db`

​	实际数据, 即行的内容

#### `Index.db`

​	从分区键（partition keys）到Data.db文件中的位置的索引。对于宽分区，这可能还包括分区内行的索引

#### `Summary.db`

​	Index.db文件中每128个条目（默认情况下）的采样

#### `Filter.db`

​	SSTable中分区键（partition keys）的布隆过滤器（Bloom Filter）

#### `CompressionInfo.db`

​	Data.db文件中的压缩块的偏移量和长度的元数据

#### `Statistics.db`

​	存储SSTable相关的元数据，包括时间戳，逻辑删除（tombstones），集群key，压缩，修复，压缩，TTL等信息

#### `Digest.db`

​	Data.db文件的CRC-32摘要

#### `TOC.txt`

​	SSTable的组件文件的纯文本列表

<br/>

在Data.db文件中，行按分区组织。这些分区按token顺序排序（即当使用默认分区器Murmur3Partition时，通过分区键的hash）。在分区内，行按其集群key的顺序存储

可以使用基于块的压缩（block-based compression）选择性地压缩SSTable

