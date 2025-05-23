# [WiredTiger](https://www.mongodb.com/zh-cn/docs/manual/core/wiredtiger/)

> https://source.wiredtiger.com/3.2.1/index.html

WiredTiger 存储引擎是 MongoDB 的默认存储引擎。

## 数据组织

WiredTiger作为一种存储引擎，在数据组织和管理上有诸多创新特性：
- **数据结构与组织**
  - 采用B+ Tree 形式组织数据，Leaf Page存实际数据，Internal Page存指针 。与传统BTree不同，它只有分裂且延迟到刷盘，还支持Fast - Truncate和Range Delete。
  - 内存中的In - Memory Page和磁盘上的Disk Extent结构不同。In - Memory Page松散，利于构建无锁多核并发结构；Disk Extent紧凑且支持压缩，提升磁盘存储和IO效率。
- **内存管理**
  - Internal Page通过WT_REF数组实现Pointer Swizzling，加速内存和磁盘间指针转换。BTree分裂采用COW机制，配合EBR进行内存回收，虽简单低开销，但存在延迟回收和内存占用问题。
  - Leaf Page通过pg_row、disk_image存储基本数据，借助modify（update list和insert list ）实现无锁读写，支持MVCC，满足多核并发需求。 
- **磁盘存储**：Disk Extent由Page Header、Block Header和Extent Data组成，变长结构。Page Header保存Page状态信息用于数据恢复等；Block Header存储数据大小、校验和等头信息；Extent Data保存实际数据，由可变长Cell组成，支持压缩，提升了磁盘存储和IO效率。

### Internal Page

WiredTiger存储引擎的Internal Page在数据组织和管理中起着关键作用，其结构和设计思路如下：

1. **WT_REF数组**：Internal Page主要存储WT_REF数组，每个WT_REF是一个结构体，用于指向子节点。
   - **指向内存中的子节点**：当指向内存中的子节点时，记录WT_PAGE裸指针，即`wt_shared WT_PAGE *page`，通过这个指针可以直接访问内存中的子节点。
   - **指向磁盘上的数据Extent**：当指向磁盘上的数据Extent时，记录磁盘地址，即`wt_shared void *addr`。

2. **Pointer Swizzling**：WT_REF实现了Pointer Swizzling技术，用于加速内存、磁盘间的指针转换。通过这种方式，在访问子节点时，可以根据节点状态快速确定是从内存还是磁盘获取数据，提高了访问效率。

3. **状态标识**：WT_REF结构体中还包含一个状态字段`wt_shared volatile WT_REF_STATE __state`，用于表示子节点的状态，其可能取值如下：
   - `WT_REF_DISK`：表示Page在磁盘上。
   - `WT_REF_DELETED`：表示Page在磁盘上，但已被删除。
   - `WT_REF_LOCKED`：表示Page被锁定以进行独占访问。
   - `WT_REF_MEM`：表示Page在缓存中且有效。
   - `WT_REF_SPLIT`：表示父Page已分裂（此时WT_REF无效）。

4. **提高访问效率**：通过WT_REF数组和Pointer Swizzling技术，使得在BTree下降过程中，能够快速定位和访问子节点。如果目标子节点在内存中，直接通过裸指针访问；如果不在内存，则从磁盘捞取Page，并修改状态记录内存中的指针。这种设计去除了BufferPool等中心化组件带来的同步开销，使得访问非常轻量。

5. **配合去中心化设计**：为了充分发挥多核CPU的算力，WiredTiger采用了去中心化的设计思路。Internal Page的这种结构设计，避免了传统存储引擎中读写锁以及中心化BufferPool对多核性能的限制。同时，相关链路也必须去中心化，以配合这种设计，虽然引入了一定的复杂度，但提高了系统的整体性能。

6. **应对分裂操作**：考虑到BTree分裂操作相对低频，而读操作高频，Internal Page在分裂时采用了Copy - On - Write（COW）机制。具体来说，分裂时会拷贝一份相同的WT_REF数组，在新数组上应用修改，然后通过指针原子地将新数组挂到父节点，而老数组则走EBR（Epoch - Based Reclamation）删除。这种方式既保证了分裂操作的正确性，又不会对高频的读操作产生较大影响。

以下是Internal Page的结构图示意：

```shell
+---------------------+
| Internal Page       |
|---------------------|
| WT_REF数组          |
|---------------------|
| WT_REF 1            |
|---------------------|
| - page: WT_PAGE *   |
| - addr: void *      |
| - __state: STATE    |
|---------------------|
| WT_REF 2            |
|---------------------|
| - page: WT_PAGE *   |
| - addr: void *      |
| - __state: STATE    |
|---------------------|
| ...                 |
|---------------------|
| WT_REF n            |
|---------------------|
| - page: WT_PAGE *   |
| - addr: void *      |
| - __state: STATE    |
|---------------------|
+---------------------+
```

在这个图中，Internal Page包含一个WT_REF数组，数组中的每个元素（即WT_REF）都包含指向子节点的指针（page或addr）以及子节点的状态信息（__state）。这种结构设计使得Internal Page能够有效地组织和管理BTree中的子节点，实现高效的数据访问和操作。

### Leaf page

WiredTiger的Leaf Page是存储实际数据的核心组件，其设计巧妙地解决了传统存储引擎的性能瓶颈，特别是在多核并发场景下。下面详细解析其结构、设计思路及工作原理。

Leaf Page 主要由三部分组成：

- **`pg_row`**：裸指针数组，每个元素指向 `disk_image` 中的偏移量（offset），用于快速定位磁盘数据。
- **`disk_image`**：存储从磁盘读取并反序列化后的原始数据（KV对），是数据的基础版本。
- **`modify`**：  
  记录数据变更的结构，包含两个关键组件：
  - **`update list`**：记录对已有Key的修改，每个Slot对应一个Key，内部维护版本链（MVCC）。
    - **结构**：数组形式，每个Slot对应一个Key，存储该Key的所有修改版本。
    - **版本链**：每个Slot内部是一个无锁链表，链上每个节点是一个 `WT_UPDATE` 对象，包含：
      - 修改数据（完整KV）。
      - 时间戳（TimeStamp）。
      - 事务ID（Transaction ID）。
    - **删除标记**：若修改类型为删除，`WT_UPDATE` 会被标记为 `WT_UPDATE_TOMBSTONE`。
  - **`insert list`**：记录新增的Key，按Key Range分区，每个分区使用无锁SkipList实现。
    - **结构**：按Key Range分区的无锁SkipList，每个分区对应磁盘上的一个Row Range。
    - **节点组成**：每个节点是一个 `WT_UPDATE` 对象，同样包含版本链，用于记录新增Key的所有修改。

#### **无锁并发设计**

- **读写分离**：  
  - 读操作：同时访问 `disk_image`（基础数据）和 `modify`（增量数据），通过MVCC规则合并结果。
  - 写操作：直接在 `modify` 中追加新版本，不修改 `disk_image`，避免锁冲突。
- **无锁数据结构**：  
  - Update List：通过原子操作维护版本链。
  - Insert List：使用无锁SkipList，支持高并发插入。

#### **MVCC（多版本并发控制）**

- **版本链管理**：  
  每个Key的所有修改按时间顺序形成链表，读操作通过事务ID和时间戳筛选可见版本。
- **读写互不阻塞**：  
  写操作只追加新版本，不影响读操作访问旧版本，避免读写锁竞争。

#### **数据合并策略**

- **延迟合并**：  
  基础数据（`disk_image`）与增量数据（`modify`）的合并延迟到刷盘时进行，减少内存中的数据移动。
- **按需合并**：  
  读操作通过计算 `disk_image + modify` 动态生成当前版本，避免实时合并开销。

#### **读流程**

1. **检查 `modify`**：  
   - 遍历 `insert list`，查找是否存在新增的Key。
   - 遍历 `update list`，查找是否存在对已有Key的修改。
   - 根据MVCC规则（如事务隔离级别）选择可见版本。
2. **检查 `pg_row`**：  
   若 `modify` 中未命中，直接从 `disk_image` 读取基础数据。
3. **合并结果**：  
   将 `disk_image` 与 `modify` 中的增量合并，生成最终结果。

#### **写流程**

1. **修改已有Key**：  
   - 在 `update list` 中找到对应Slot。
   - 原子地在版本链头部插入新的 `WT_UPDATE` 节点。
2. **插入新Key**：  
   - 在 `insert list` 中找到对应的Key Range分区。
   - 在SkipList中插入新节点，节点内维护版本链。

```shell
+---------------------+
| Leaf Page           |
|---------------------|
| pg_row              |
|---------------------|
| | ptr1 | ptr2 | ... |
|---------------------|
| disk_image          |
|---------------------|
| | K1 | V1 | K2 | V2 |
|---------------------|
| modify              |
|---------------------|
| update list         |
|---------------------|
| | Slot1 | Slot2 | ...|
|---------------------|
|   |                   |
|   v                   |
| +----------------+    |
| | Version Chain  |    |
| | -------------- |    |
| | WT_UPDATE_1    |    |
| | -> WT_UPDATE_2 |    |
| | -> ...          |    |
| +----------------+    |
|---------------------|
| insert list         |
|---------------------|
| | Range1 | Range2 |...|
|---------------------|
|   |                   |
|   v                   |
| +----------------+    |
| | SkipList       |    |
| | -------------- |    |
| | Node1 | Node2 |...| |
| +----------------+    |
|---------------------|
```

**优势**：

1. **多核性能**：  
   无锁设计充分利用多核CPU，读写操作可并行执行。
2. **内存效率**：  
   增量数据（`modify`）体积小，减少内存占用。
3. **读写性能**：  
   读操作无需加锁，写操作只需追加，避免内存拷贝。

**挑战**：

1. **版本链管理**：  
   长时间运行可能导致版本链过长，需定期GC（垃圾回收）。
2. **读性能波动**：  
   若 `modify` 数据量大，读操作需合并大量增量，可能影响性能。
3. **实现复杂度**：  
   无锁数据结构和MVCC逻辑复杂，调试和维护难度高。

### Disk extent

磁盘文件结构

```shell
+---------------------+
| Disk Extent         |
|---------------------|
| Page Header         |
|---------------------|
| recno               |
| write_gen           |
| mem_size            |
| entries/datalen     |
| type                |
| flags               |
| version             |
|---------------------|
| Block Header        |
|---------------------|
| data_size           |
| checksum            |
| flags               |
| unused              |
|---------------------|
| Extent Data         |
|---------------------|
| Cell 1              |
|---------------------|
| | Metadata | Data | |
|---------------------|
| Cell 2              |
|---------------------|
| | Metadata | Data | |
|---------------------|
| ...                 |
|---------------------|
```

cell结构

```shell
+-------------------+
| Cell Metadata (98B)|
|-------------------|
| Prefix Compress   |
| Timestamp         |
| Transaction ID    |
| Data Length       |
| ... (Reserved)    |
|-------------------|
| Compressed Data   |
|-------------------|
```

Disk Extent 的设计通过变长存储、多级校验和灵活压缩，在存储密度、IO 效率和数据恢复之间取得了平衡。其核心思想是将内存中的松散数据（In-Memory Page）转换为磁盘上的紧凑结构，同时保留足够的元数据支持快速定位和验证。这种设计为现代存储引擎提供了高效持久化的解决方案。