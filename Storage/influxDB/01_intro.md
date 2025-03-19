# Intro

- [InfluxDB中文文档](https://jasper-zhang1.gitbooks.io/influxdb/content/)
- [influxdata](https://www.influxdata.com/)
- 

> 时序数据库 InfluxDB®版是一款专门处理高写入和查询负载的时序数据库，用于存储大规模的时序数据并进行实时分析，包括来自DevOps监控、应用指标和IoT传感器上的数据。

## 主要特点

* InfluxDB®是您处理时序数据的一个绝佳选择，目前有以下特点：

* 专为时间序列数据量身打造的高性能数据存储。TSM引擎提供数据高速读写和压缩等功能。
  
* 简单高效的HTTP API写入和查询接口。
  
* 针对时序数据，量身打造类似SQL的查询语言，轻松查询聚合数据。
  
* 允许对tag建索引，实现快速有效的查询。
  
* 数据保留策略（Retention policies）能够有效地使旧数据自动失效。

## 关键概念

- database: 数据库名，在 InfluxDB 中可以创建多个数据库，不同数据库中的数据文件是隔离存放的，存放在磁盘上的不同目录。
  
- retention policy: 存储策略，用于设置数据保留的时间，每个数据库刚开始会自动创建一个默认的存储策略 autogen，数据保留时间为永久，之后用户可以自己设置，例如保留最近2小时的数据。插入和查询数据时如果不指定存储策略，则使用默认存储策略，且默认存储策略可以修改。InfluxDB 会定期清除过期的数据。
  
- measurement: 测量指标名，例如 cpu_usage 表示 cpu 的使用率。

- tag sets: tags 在 InfluxDB 中会按照字典序排序，不管是 tagk 还是 tagv，只要不一致就分别属于两个 key，例如 host=server01,region=us-west 和 host=server02,region=us-west 就是两个不同的 tag set。

- tag: 标签，在InfluxDB中，tag是一个非常重要的部分，表名+tag一起作为数据库的索引，是“key-value”的形式。

- field name: 例如上面数据中的 value 就是 fieldName，InfluxDB 中支持一条数据中插入多个 fieldName，这其实是一个语法上的优化，在实际的底层存储中，是当作多条数据来存储。

- timestamp: 每一条数据都需要指定一个时间戳，在 TSM 存储引擎中会特殊对待，以为了优化后续的查询操作。

### 行协议

```text
<measurement>[,<tag_key>=<tag_value>[,<tag_key>=<tag_value>...]] <field_key>=<field_value>[,<field_key>=<field_value>...] [<timestamp>]
```