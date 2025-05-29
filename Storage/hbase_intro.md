# [HBase](https://hbase.apache.org/book.html)

- [HBase架构模型hbase](https://www.cnblogs.com/simon-1024/p/12161970.html)

## HBase Data Model Terminology

**Table**

An HBase table consists of multiple rows.

**Row**

A row in HBase consists of a row key and one or more columns with values associated with them. Rows are sorted alphabetically by the row key as they are stored. For this reason, the design of the row key is very important. The goal is to store data in such a way that related rows are near each other. A common row key pattern is a website domain. If your row keys are domains, you should probably store them in reverse (org.apache.www, org.apache.mail, org.apache.jira). This way, all of the Apache domains are near each other in the table, rather than being spread out based on the first letter of the subdomain.

**Column**

A column in HBase consists of a column family and a column qualifier, which are delimited by a : (colon) character.

**Column Family**

Column families physically colocate a set of columns and their values, often for performance reasons. Each column family has a set of storage properties, such as whether its values should be cached in memory, how its data is compressed or its row keys are encoded, and others. Each row in a table has the same column families, though a given row might not store anything in a given column family.

**Column Qualifier**

A column qualifier is added to a column family to provide the index for a given piece of data. Given a column family content, a column qualifier might be content:html, and another might be content:pdf. Though column families are fixed at table creation, column qualifiers are mutable and may differ greatly between rows.

**Cell**

A cell is a combination of row, column family, and column qualifier, and contains a value and a timestamp, which represents the value’s version.

**Timestamp**

A timestamp is written alongside each value, and is the identifier for a given version of a value. By default, the timestamp represents the time on the RegionServer when the data was written, but you can specify a different timestamp value when you put data into the cell.

## 逻辑视图和物理视图

![hbase_logical_view](https://raw.githubusercontent.com/TDAkory/ImageResources/master/img/AppFrameThoughts/hbase_logic_store.jpg)

* Table（表）：一个表由一个或者多个列族构成。数据的属性。比如：name、age、TTL(超时时间)等等都在列族里边定义。定义完列族的表是个空表，只有添加了数据行以后，表才有数据。
* Column Family（列族）：在HBase里，可以将多个列组合成一个列族。建表的时候不用创建列，因为列是可增减变化的，非常灵活。唯一需要确定的就是列族，也就是说一个表有几个列族是一开始就定好的。此外表的很多属性，比如数据过期时间、数据块缓存以及是否使用压缩等都是定义在列族上的，而不是定义在表上或者列上。这一点与以往的关系型数据库有很大的差别。列族存在的意义是：HBase会把相同列族的列尽量放在同一台机器上，所以说想把某几个列放在一台服务器上，只需要给他们定义相同的列族。
* Row（行）：一个行包含多个列，这些列通过列族来分类。行中的数据所属的列族从该表所定义的列族中选取，不能选择这个表中不存在的列族。由于HBase是一个面向列存储的数据库，所以一个行中的数据可以分布在不同的服务器上。
* RowKey（行键）：rowkey和MySQL数据库的主键比起来简单很多，rowkey必须要有，如果用户不指定的话，会有默认的。rowkey完全由是用户指定的一串不重复的字符串，另外，rowkey按照字典序排序。一个rowkey对应的是一行数据！！！
* Region：Region就是一段数据的集合。之前提到过高表的概念，把高表进行水平切分，假设分成两部分，那么这就形成了两个Region。注意一下Region的几个特性：
  * Region不能跨服务器，一个RegionServer可以有多个Region。
  * 数据量小的时候，一个Region可以存储所有的数据；但是当数据量大的时候，HBase会拆分Region。
  * 当HBase在进行负载均衡的时候，也有可能从一台RegionServer上把Region移动到另一服务器的RegionServer上。
  * Region是基于HDFS的，它的所有数据存取操作都是调用HDFS客户端完成的。
* RegionServer：RegionServer就是存放Region的容器，直观上说就是服务器上的一个服务。

![hbase_physical_view](https://raw.githubusercontent.com/TDAkory/ImageResources/master/img/AppFrameThoughts/hbase_physical_store.jpg)

* NameSpace：命名空间，类似于关系型数据库的 DatabBase 概念，每个命名空间下有多个表。HBase有两个自带的命名空间，分别是 hbase 和 default，hbase 中存放的是 HBase 内置的表，default 表是用户默认使用的命名空间。
* Row：HBase 表中的每行数据都由一个 RowKey 和多个 Column（列）组成，数据是按照 RowKey 的字典顺序存储的，并且查询数据时只能根据 RowKey 进行检索，所以 RowKey 的设计十分重要。
* Column：列，HBase 中的每个列都由 Column Family(列族)和 Column Qualifier（列限定符）进行限 定，例如 info：name，info：age。建表时，只需指明列族，而列限定符无需预先定义。
* TimeStamp：时间戳，用于标识数据的不同版本（version），每条数据写入时，如果不指定时间戳，系统会自动为其加上该字段，其值为写入 HBase 的时间。
* Cell：单元格，由{rowkey, column Family：column Qualifier, time Stamp} 唯一确定的单元。cell 中的数据是没有类型的，全部是字节码形式存贮。

## Hbase Region Server

## HFile文件格式

### flush过程