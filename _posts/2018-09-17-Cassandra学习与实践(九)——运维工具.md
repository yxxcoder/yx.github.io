---
layout: post
title:  "Cassandra学习与实践(九)——运维工具"
date:   2018-10-02 12:00:00 +0800
categories: nosql
---

#### *参考资料*

- *极客学院 [http://www.jikexueyuan.com](http://www.jikexueyuan.com)*

- *Cassandra官方文档 [http://cassandra.apache.org/doc/latest/faq/index.html](http://cassandra.apache.org/doc/latest/faq/index.html)*



<br>



- nodetool version

  显示当前Cassandra的版本信息

- nodetool status

  显示当前机器节点信息（UN正常，DN宕机）、数据中心、机架信息

- nodetool netstats

  获取节点的网络连接信息，可以指定参数 -h 查看具体节点信息

