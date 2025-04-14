# Redis

- [13 Years Later – Does Redis Need a New Architecture?](https://redis.io/blog/redis-architecture-13-years-later/)

## 概要

KV全内存
单线程（IO多线程+单Worker）
无序 + 不可变 + 无缓冲
主从 + 异步一致
提供最终一致性
通过fork实现快照
提供有限的ACID支持

作为「存储」（不是缓存）来看，它也是个五脏俱全的数据库，并且在字节内部也广泛使用，可见其稳定性、性能、成本、故障丢数据风险是能接受的

## IO流程

### Redis 线程模型与流程

- **线程模型**：默认Redis数据面由主线程完成，后台有线程执行写文件等操作，也可开启开关使网络IO线程独立，经典高性能模式是单线程Run - to - Complete。
- **单线程IO流程**：基于IO多路复用，封装各平台高效工具（如Linux epoll等）。从网络IO读取数据存到client读buffer，执行命令后回复内容写入client buffer，等待写入socket。命令多处理内存数据结构，O(1)时间完成，O(N)操作需谨慎，有Lazy Free和渐进Rehash特性应对可能的阻塞。

### Redis 的 WAL（AOF）与 Data（RDB）文件

- **AOF（WAL文件）Append Only File**
  - 保存执行命令，语义类似WAL，实际存全量数据。命令执行后写入aof_buf，异步任务写入文件。落盘时机取决于保存模式，默认1s fsync一次。
  - AOF重写：文件过大且有重复命令时进行，基于fork子进程，父进程处理增量，完成后删除旧文件。Redis 7.0后采用AOF Manifest机制，效率更高。
- **RDB（Data文件）Redis DataBase File**：内存数据序列化后排布，数据小、加载快，是数据库快照。生成成本高，一般周期性（如900s）生成，采用fork机制，写QPS高时可能因COW致内存暴涨、CPU紧张，也可遍历数据库生成。

### 垃圾回收机制

- **AOF重写**：非改写命令，是精简文件，根据数据库情况重写，基于fork子进程实现。
- **内存碎片整理**：Redis全内存保存KV易产生碎片，4以下靠重启处理，4以上引入defrag模块，依赖Jemalloc，可以通过接口询问jemalloc将目标指针移动到新地址空间能否减少碎片，遍历部分key替换指针，替换时新申请的内存也需要和Jemalloc谨慎交互，要在「当前线程的非缓冲内存」区域申请内存（直接到更底层），否则碎片就更碎片了。

### 小结

Redis存储引擎不标准，适用于数据量小场景，方便做数据库Dump和快照（通过fork）。

## Redis索引

- **索引依赖**：Redis 基于全内存存储，对索引依赖不强。实例重启时，通过加载 RDB 或 AOF 获取全量数据，在内存中构建 hash 结构作为“主索引”，建立 key 到内存中 object 的映射，无需关联硬盘地址。

```c
typedef struct dictEntry {
    void *key;                //键
    union {
        void *val;            //值
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next; //指向下一个节点，形成链表
} dictEntry;
typedef struct dictht {
    dictEntry **table; //两个数组，实现渐进式rehash的关键。
    unsigned long size;
    unsigned long sizemask;
    unsigned long used;
} dictht;
```

- **hash结构实现**：采用普通开散列方式，发生冲突时通过 `dictEntry` 的 `next` 指针串联节点。`dictEntry` 结构体包含键（`key`）、值（`v`，以联合类型存储不同类型值 ）以及指向下一节点的指针（`next` ）；`dictht` 结构体包含哈希表数组（`table` ，是渐进式 rehash 关键 ）、大小（`size` ）、掩码（`sizemask` ）和已使用数量（`used` ）。
- **渐进式rehash**：用于解决哈希表扩容时 O(N) 操作可能阻塞服务器的问题。扩容时先申请新内存，然后在每次增删改查操作时逐步迁移原 `dict` 成员到新 `dict`。查询时依次查原表（`table[0]` ）和新表（`table[1]` ）；添加只在新表；删除依次查原表和新表。

## 主从复制

- **快照**：通过 `fork` 实现，利用操作系统命令，无需复杂存储结构设计。
- **全量复制**：新从库向主库发送 `FULLRESYNC` 命令请求全量数据，本质类似 `RDB` 生成，由 `fork` 产生快照，在子进程生成 `RDB` 并直接写入目标 `socket`。 
- **增量复制**：已有数据的从库在断链或切主后，发送 `PSYNC` 命令同步增量数据，即同步 `aof_buf` 中的 `WAL` 。 
- **主从级联**：主库生成 `RDB` 开销大，采用从库挂载从库模式可分担压力，但会导致从库数据一致性延迟增加。 

## 事务与Pipeline

- **Pipeline**：保证若干命令通过一次 `TCP` 连接传递到服务端并顺序执行，期间可能插入其他客户端命令。 
- **事务**
    - **原子性**：Redis 不支持事务回滚，若事务有错误不影响其他正确命令执行。编写无错误事务可实现原子性，但存在限制。 
    - **一致性**：在多种出错情况下能满足一致性要求，因其约束较少。 
    - **隔离性**：事务总是串行执行，满足隔离性要求。 
    - **持久性**：仅在支持 `AOF/RDB` 持久化且 `appendsync` 配置为 `always` 时可保证，

主从复制异步模式下切主可能丢数据，存在持久化风险。总体在原子性、持久性上有取舍，合理配置可保障一致性。 