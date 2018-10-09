---
layout: post
title:  "Cassandra学习与实践(八)——参数调优"
date:   2018-09-17 12:00:00 +0800
categories: nosql
---

#### *参考资料*

- *Cassandra官方文档 [http://cassandra.apache.org/doc/latest/faq/index.html](http://cassandra.apache.org/doc/latest/faq/index.html)*



<br>

## Cassandra性能调优

- memtable_allocation_type

指定Cassandra分配和管理memtable内存的方式，可选参数有heap_buffers（堆nio缓冲区）、offheap_buffers（非堆nio缓冲区）、offheap_objects（非堆对象）

- concurrent_reads


- concurrent_writes

对于具有比内存可容纳数据更多的数据的工作负载，Cassandra的瓶颈将是需要从磁盘提取数据的读取。`concurrent_reads`应该设置为（16 * number_of_drives），以便降低堆栈中排队的操作，操作系统和驱动器可以对它们重新排序。这同样适用于`concurrent_counter_writes`，因为计数器写入在递增和写回它们之前读取当前值。 另一方面，由于写入几乎没有IO限制，`concurrent_writes`的理想数量取决于系统中的核心数;（8 * number_of_cores）是一个好的经验法则

- concurrent_compactors
- memtable_flush_writers



