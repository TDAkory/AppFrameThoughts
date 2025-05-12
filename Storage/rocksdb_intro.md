# [RocksDB](https://rocksdb.org/)

> [github page](https://github.com/facebook/rocksdb)
> [RocksDB Overview](https://github.com/facebook/rocksdb/wiki/RocksDB-Overview)

![rocksdb](https://raw.githubusercontent.com/TDAkory/ImageResources/master/img/AppFrameThoughts/rocksdb.jpg)

## [memtable](https://github.com/facebook/rocksdb/wiki/MemTable)

![rocksdb memtable data structure comparsion](https://raw.githubusercontent.com/TDAkory/ImageResources/master/img/AppFrameThoughts/rocksdb_memtable_struct_compare.png)

## [wal](https://github.com/facebook/rocksdb/wiki/Write-Ahead-Log-%28WAL%29)

- 每次对RocksDB的更新都被写到两个地方:
  - 内存：Memtable
  - 磁盘：WAL
- WAL文件持续增长，直到超过单个文件最大配置，此时会创建新WAL文件
- 失败恢复时，用WAL恢复内存的Memtable、Immutable Memtable即可
- 当Immutable Memtable flush到磁盘以后，会更新列族的最大序列号
  - 所有列族共用一个WAL
  - 因此所有列族的最大序列号都超过一个WAL文件时，该WAL文件才会被删除
  
日志文件包含一系列变长记录，变长记录按kBlockSize(32k)聚合

有两种格式`Legacy record format`：

```shell
+---------+-----------+-----------+--- ... ---+
|CRC (4B) | Size (2B) | Type (1B) | Payload   |
+---------+-----------+-----------+--- ... ---+

CRC = 32bit hash computed over the payload using CRC
Size = Length of the payload data
Type = The type is used to group a bunch of records together to represent
       blocks that are larger than kBlockSize
  kZeroType = 0,
  kFullType = 1,
  kFirstType = 2,
  kMiddleType = 3,
  kLastType = 4,
Payload = Byte stream as long as specified by the payload size
```

`Recyclable record format`:

```shell
+---------+-----------+-----------+----------------+--- ... ---+
|CRC (4B) | Size (2B) | Type (1B) | Log number (4B)| Payload   |
+---------+-----------+-----------+----------------+--- ... ---+
Same as above, with the addition of
Log number = 32bit log file number, so that we can distinguish between
records written by the most recent log writer vs a previous one.

Types:
  kRecyclableFullType = 5,
  kRecyclableFirstType = 6,
  kRecyclableMiddleType = 7,
  kRecyclableLastType = 8,
```

## [sst file](https://github.com/facebook/rocksdb/wiki/Rocksdb-BlockBasedTable-Format)

```shell
<beginning_of_file>
[data block 1]
[data block 2]
...
[data block N]
[meta block 1: filter block]                  (see section: "filter" Meta Block)
[meta block 2: index block]
[meta block 3: compression dictionary block]  (see section: "compression dictionary" Meta Block)
[meta block 4: range deletion block]          (see section: "range deletion" Meta Block)
[meta block 5: stats block]                   (see section: "properties" Meta Block)
...
[meta block K: future extended block]  (we may add more meta blocks in the future)
[metaindex block]
[Footer]                               (fixed size; starts at file_size - sizeof(Footer))
<end_of_file>
```

Data Block内存放KV的基本单位是Entry，采用前缀压缩存放。

```shell
// 单个Entry格式如下
shared_bytes: varint32：表示与前一个 Key 共享的前缀字节数
unshared_bytes: varint32：表示与前一个 Key 不共享的字节数
value_length: varint32
key_delta: char[unshared_bytes]：表示与前一个 Key 不共享的 Key 部分
value: char[value_length]

// Data Block尾部格式如下
restarts: uint32[num_restarts]
num_restarts: uint32
```

- Data Block内部的查找流程：
  - 在restarts数组上二分，定位key属于哪一个restart组
  - 从定位的restart组中，从前往后遍历

## [Compaction](https://github.com/facebook/rocksdb/wiki/Compaction)

[Leveled Compaction](https://github.com/facebook/rocksdb/wiki/Leveled-Compaction)

Leveled Compaction（分层压实）和Tiered Compaction（分块压实）是数据库系统中用于管理数据存储和优化读写性能的两种常见压实策略，它们的异同点如下：

目的相同，都是为了将磁盘的随机写入转换为顺序写，提高写入性能，并降低空间放大和读放大的影响；<br> 均应用于基于LSM树的数据存储系统，优化数据在磁盘上的存储布局

| 不同点 | Leveled Compaction | Tiered Compaction |
| --- | --- | --- |
| 数据合并方式 | 数据从一层到下一层的合并是通过读取并重写目标层已有的数据来实现的，每层通常只有一个有序的数据段（Sorted - Run），也有扩展版本允许每层有N个Sorted - Run，但数据在每层是完全有序的 | Ln层所有的数据向下合并到Ln + 1层新的Sorted - Run，不需要读取Ln + 1层原有的数据，每层可以包含多个Sorted - Run，每层的数据不是完全有序的 |
| 写放大因子 | 相对较高，每次合并都需要读取和重写目标层的数据，处理大量小文件合并时会产生较多额外写入操作 | 能够最大程度地减少写放大，提升写入性能，合并数据时不会读取和重写目标层已有的数据，写放大因子通常接近1 |
| 读性能影响 | 数据在每层是有序的，读取时通常只需要访问少数几个层，对频繁更新的数据，能将数据集中在较少文件中，减少读取时需扫描的文件数量，读放大较低 | 每层存在多个Sorted - Run且数据不完全有序，读取数据时可能需要扫描更多文件或数据块，读放大相对较高，但如果数据访问模式是顺序的或对读取性能要求不高，影响可能不明显 |
| 空间放大 | 若每层只允许一个Sorted - Run，空间放大较小；若允许每层有多个Sorted - Run，空间放大可能会增加，但合理配置可控制在一定范围内 | 每层可以有多个Sorted - Run，数据不断写入和合并时，可能导致空间放大因子相对较高，每层的多个Sorted - Run会占用更多磁盘空间 |
| 适用场景 | 适用于读操作频繁、对读延迟要求较高、数据更新频繁的场景，如在线事务处理（OLTP）系统，以及需要快速随机访问数据的应用程序 | 适用于写操作频繁、对写入性能要求较高、数据访问模式以顺序读取为主或者对读性能要求相对较低的场景，如大规模数据的批量写入、数据仓库等 | 

## [MANIFEST](https://github.com/facebook/rocksdb/wiki/MANIFEST)

* `MANIFEST` refers to the system that keeps track of RocksDB state changes in a transactional log
* `Manifest` log refers to an individual log file that contains RocksDB state snapshot/edits
* `CURRENT` refers to the latest manifest log

[How we keep track of live SST files](https://github.com/facebook/rocksdb/wiki/How-we-keep-track-of-live-SST-files)

## 事务

- ReadUncommited 读取未提交内容，所有事务都可以看到其他未提交事务的执行结果，存在脏读
- ReadCommited 读取已提交内容，事务只能看到其他已提交事务的更新内容，多次读的时候可能读到其他事务更新的内容
- RepeatableRead 可重复读，确保事务读取数据时，多次操作会看到同样的数据行(使用快照隔离来实现)。
- Serializability 可串行化，强制事务之间的执行是有序的，不会互相冲突。