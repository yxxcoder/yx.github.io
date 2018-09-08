---
layout: post
title:  "Cassandra学习与实践(二)——数据模型 & CQL"
date:   2018-08-31 17:00:00 +0800
categories: nosql
---

*参考资料：*

*极客学院：http://www.jikexueyuan.com*

*Cassandra官方文档：http://cassandra.apache.org/doc/latest/faq/index.html*

*学习Cassandra：https://teddyma.gitbooks.io/learncassandra_cn/content/*



## 数据模型

### 列（Column）

一组包含名称值对的数据叫做行（Row），而每一组名称值对（Name/Value Pair）被称为列（Column）

Cassandra最基本的数据结构是Column，它是一个3元的数据类型，包含：name，value和timestamp（客户端提供的时间戳，它常记录了最后一次变更的时间）。这里提到的name和value都是byte[]类型的，长度不限

![column](https://yxxcoder.github.io/images/column.jpeg)

### 超级列（Super Column）

![supercolumn](https://yxxcoder.github.io/images/supercolumn.png)

如果Column的value值不是单纯的数值，而是被分割成多个子Column，那么这个大的column就叫做Super Column

Column和Super Column都是name与value的组合，最大的不同在于Column的value是一个`string`，而Super Column的value是多个Columns组成的Map

Super Column本身是不包含timestamp的

![super column](https://yxxcoder.github.io/images/super column.png)

### 列族（Column Families）

多个列的键值对组成列族（Column Families）

Column Families的概念，用于逻辑上的分割，将同类的数据联系在一起。例如，我们可能会有User Column Family、Hotel Column Family、AddressBook Column Family等。这种方式下，一个Column Family多少类似于关系型数据库中的table

<br>

### 键空间（keyspaces）

列族的上一级容器就是键空间（keyspaces）。键空间是Cassandra的数据容器，可以理解为关系型数据库的中的数据库（Database）

<br>

### 复合键（Composite Keys）

我们有时会遇到不同省份可能有同样的城市名称，或不同的城市有重名的街道，这时使用单一的城市名称或街道名称来作为Key就会无法识别。Cassandra允许使用`Key1:Key2`的结构来存储一对值作为Key，一个常见的例子是使用`<userid:lastupdate>`这样的结构来存储用户ID及最后登陆时间

<br>

### 集群（Cluster）

Cassandra的节点实例，它可以包含多个Keyspace

<br>

### Cassandra的排序规则

Cassandra在定义列族时，可以包含一个名为CompareWith的元素，这个元素决定了此列族的排序规则。Cassandra提供的排序支持以下几种数据类型，包含了字符、字节、数字和日期时间：AsciiType，BytesType，LexjcalUUIDType，IntegerType，LongType，TimeUUIDType，or UTF8Type

![sort](https://yxxcoder.github.io/images/sort.png)


### Cassandra数据设计模式

#### 面向行（Row-Oriented)

在Cassandra中可以使用一个唯一识别号访问行，所以我们可以更好理解为，Cassandra是一个带索引的，面向行的存储

#### 无结构（SchemaFree)

根据你的需求场景，你可以只保存你需要的数据，而不必拘泥于早前定义的表结构

<br>

## CQL

CQL是Cassandra Query Language的简称，CQL由类似SQL的语句组成，包括修改，查询，保存，变更数据的存储方式等等功能。每一行语句由分号（;）结束，例如：

```CQL
SELECT * FROM MyTable;

UPDATE MyTable SET SomeColumn = 'SomeValue' WHERE columnName = 'Something Else';
```

CQL的关键字是忽略大小写的，除非用双引号括起来，才是大小写敏感的。如果不用双引号括起来，即使CQL写成大写，也会被保存为小写。定义Tables、Column、KeySpace等不可以直接使用Cassandra保留字，如果一定要用可以使用双引号引用

```CQL
CREATE TABLE test(

	Foo int PRIMARY KEY,

	"Bar" int

);
```

<br>

CQL数据定义语句主要是CREATE、DROP、ALTER，具体包括如下几点：

```CQL
CREATE KEYSPACE/TABLE/INDEX/TRIGGER/TYPE

USE KEYSPACE

ALTER KEYSPACE/TABLE/INDEX/TRIGGER/TYPE

DROP KEYSPACE/TABLE/INDEX/TRIGGER/TYPE
```

<br>






















