# [FastKV](https://microsoft.github.io/FASTER/)

#DB_engine

> https://github.com/microsoft/FASTER

FasterKV的整体架构可分为3个部分：

1. 多个并发的工作线程
   1. 基于Epoch Protection的思想，结合原子操作完成无锁的线程间同步
2. Hash索引，每个HashEntry指向一批Records，基于链地址法解决Hash冲突
   1. Record即数据的实际存储格式，包含Key & Value
   2. Index中不包含Key & Value，仅存储由Allocator分配的逻辑地址，将Index和Allocator彻底解耦
3. Record Allocators，Record的实际分配器，包含三种不同的实现方式：
   1. In-Memory，即全内存存储
   2. Append-Only-Log，基于Append-Only的大数据量解决方案
   3. Hybrid-Log，融合上面两种思想，支持就地更新(in-place updates)的大数据量解决方案
