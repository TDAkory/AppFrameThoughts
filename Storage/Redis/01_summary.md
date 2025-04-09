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