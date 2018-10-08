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
- concurrent_compactors
- memtable_flush_writers



