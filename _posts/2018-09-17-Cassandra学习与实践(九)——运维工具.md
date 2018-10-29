---
layout: post
title:  "Cassandra学习与实践(九)——运维工具"
date:   2018-10-02 12:00:00 +0800
categories: nosql
---

#### *参考资料*

- *Cassandra官方文档 [http://cassandra.apache.org/doc/latest/faq/index.html](http://cassandra.apache.org/doc/latest/faq/index.html)*
- *datastax [https://docs.datastax.com/en/dse/6.0/dse-admin/datastax_enterprise/tools/nodetool/toolsNodetool.html](https://docs.datastax.com/en/dse/6.0/dse-admin/datastax_enterprise/tools/nodetool/toolsNodetool.html)*



<br>



- nodetool version

  显示当前Cassandra的版本信息

  ![column](https://yxxcoder.github.io/images/nodetool_version.png)

- nodetool describecluster

  用于描述集群信息

  ![column](https://yxxcoder.github.io/images/nodetool_describecluster.png)

- nodetool describering

   后面跟keyspace名字，显示圆环的节点信息

  ![column](https://yxxcoder.github.io/images/nodetool_describering.png)

- nodetool status

  显示当前机器节点信息（UN正常，DN宕机）、数据中心、机架信息

  ![column](https://yxxcoder.github.io/images/nodetool_status.png)

- nodetool netstats

  获取节点的网络连接信息，可以指定参数 -h 查看具体节点信息

  ![column](https://yxxcoder.github.io/images/nodetool_netstats.png)

- nodetool tablehistograms

  nodetool tablehistograms 显示过去十五分钟内的表性能统计信息，包括读/写延迟，分区大小，单元数和SSTable数。使用此工具分析性能并调整单个表，并确保百分比延迟级别满足表中存储的数据的SLA

  ![column](https://yxxcoder.github.io/images/nodetool_tablehistograms.png)

- nodetool tpstats

  列出Cassandra维护的线程池的信息，你可以直接看到每个阶段有多少操作，以及他们的状态是活动中、等待还是完成

  ![column](https://yxxcoder.github.io/images/nodetool_tpstats.png)

- nodetool cfstats 

  查看表的一些信息，包括读的次数，写的次数，sstable数量，Memtable信息，压缩信息，bloomfilter信息。使用-H 则文件的信息会以可读的方式

  ![column](https://yxxcoder.github.io/images/nodetool_cfstats.png)

- nodetool compactionstats 

  显示当前压缩信息

  ![column](https://yxxcoder.github.io/images/nodetool_compactionstats.png)