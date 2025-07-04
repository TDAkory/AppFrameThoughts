### **什么是存储引擎**：

* *提供把数据存到盘上的能力*：盘上空间建模（三变量：有序性、可变性、缓冲）
* *能根据某个主键重新读取数据*：内存、盘上索引结构
* *需要整理空间保障空间效率*：垃圾回收

**存储中有序性、可变性、缓冲三个变量**：

* **有序性**：存到盘上的数据有序有优势也有劣势，有序并非指盘上空间物理有序，而是存储引擎管理空间逻辑有序，也可部分有序。
  * **优势**：查找数据索引需求低，可减少索引规模；便于扫描和范围查询；可利用前缀压缩减少存储空间和匹配开销。
  * **劣势**：维护有序需额外计算开销，与写入量正相关；写性能稍差。
  * **典型案例**：有序的有B+树、LSM；无序的有Bw树、Bitcask（Hash引擎）。
* **可变性**：指写入盘上数据是否会被二次update，不可变不等于Append Only。
  * **优势**：顺序写或接近顺序写，性能高；并发易实现，无需复杂锁机制；写放大可能更小；数据一致性易保障。
  * **劣势**：部分场景读性能差；空间放大，垃圾多。
  * **典型案例**：可变的有B+树；不可变的有LSM、COW B+树、Bw树。在分布式架构、SSD发展、多核CPU时代，不可变结构更适配。
* **缓冲**：这里指与存储引擎结构深度结合的缓冲区，如LSM的memtable、MongoDB的WiredTiger的缓冲。
  * **优势**：将随机小IO聚合成大IO，提高写性能。
  * **劣势**：增加内存开销；结构更复杂。
  * **典型案例**：有缓冲的如LSM、WiredTiger；无缓冲的如B+树、COW B+树、Bw树（无缓冲并非不好，而是没必要）

一些典型的例子：

1. **B+树**：出现早，常用于SQL数据库（如InnoDB）。
    - **磁盘结构**：数据文件拆成若干16K的“页”，对应B+树节点，以“页”为读写单位，节点父子关系通过物理地址关联（仅父->子方向），头节点地址放文件头部。通过分槽页技术维持节点主键顺序，数据超长时用溢出页。采用WAL+double write机制应对刷盘非原子操作导致的问题。
    - **内存结构**：构建与磁盘相同的B+树结构，保存物理地址，可补充冗余信息，为结构变更映射到磁盘空间管理做准备。
    - **索引**：非叶子结点本身是索引，一般存内存，规模小、命中率高。增删改B+树即增删改索引，持久化后索引持久化。
    - **垃圾回收**：未详细提及。
2. **LSM树**：磁盘上是“有序”且“不可变”的SST文件，存储有序结构有多种选择，SSTable是通用格式。
    - **磁盘结构**：数据按格式存储在SST文件中。
    - **内存结构**：根据查找主键加载Data Block，形式灵活；用支持并发的SkipList作为memtable缓冲已写入WAL的更新，并维护有序性后下刷到硬盘。
    - **索引**：查询先查memtable，再依次查每层SST。索引信息杂，包括元数据（最小最大Key）、布隆过滤、Index Block等，不一定常驻内存，存于Table Cache，内存索引数据稀疏。
    - **垃圾回收**：通过SST之间的Compaction（层级压实或大小压实）清理垃圾，在后台执行。
3. **Bitcask（Hash引擎）**：是无序的Hash引擎，类似LSM但无需缓冲。
    - **磁盘结构**：写入Append Only到文件，垃圾回收面临挑战。
    - **内存结构**：在内存大Hash中更新数据最新位置，索引全内存。
    - **索引**：每个文件固定后在末尾生成每条数据的索引信息，内存索引是所有文件索引信息聚合。
    - **垃圾回收**：未详细提及具体方式，但强调无序情况下垃圾回收在内存上面临的挑战。
4. **Bw树**：介于有序无序之间、“不可变”、“无缓冲”的B+树，有关键特性latch-free（无锁）、delta update（增量）、log structured（日志）。
    - **磁盘结构**：逻辑上像B+树，节点关系通过“页ID”关联，通过Mapping Table映射页ID到物理地址或内存指针。脏节点下刷时将增量Append Only写入盘上，Delta之间用物理地址串起来，需和SSD配合应对随机读。
    - **内存结构**：包括Mapping Table和Base、Delta的内存缓存。增删改时通过CAS操作Mapping Table连接新Delta，读不存在节点时根据Mapping Table加载并CAS地址。
    - **索引**：Mapping Table是索引，保存所有节点物理地址；逻辑B+树非叶节点也可看作索引。磁盘上日志可理解为Mapping Table日志，为提高恢复效率需做checkpoint。
    - **垃圾回收**：未详细提及。

**其他索引**包括：

- **二级索引**：方便其他场景快速找数据，可能指向主键或数据，实现似键索引，B+树和LSM树均可用于实现，组合索引类似，判断顺序元素为组。
- **多维索引**：如处理地理位置数据，采用Z-Curve编码结合B+树（如树）或R树（节点保存最大邻接矩形）等结构，减少查询筛选数据量。
- **模糊索引**：用于匹配字符串编辑距离实现字符串搜索，较复杂未详述。

**垃圾回收**：存储引擎需保障空间使用效率，减少空间放大，垃圾来源包括数据区域不再被引用的数据（如LSM旧数据、Bw树合并后旧的Base+Delta）和已下刷脏数据的WAL等，一般在后台处理，如LSM树通过SST之间的Compaction清理垃圾。

衡量指标：

* 面向用户：SLA、读性能、写性能、延迟
* 读放大、写放大、空间放大、故障恢复时间
  
### 一般分类

* 交互协议
  * SQL
  * NoSQL
    * KV
    * 图
    * 对象 BLOB
    * 时序
    * 数仓
* 使用场景
  * OLTP  在线事务
  * OLAP  在线分析
  * HTAP  混合事务和分析

### 对于写流程，大部分存储都由两部分流程组成：

* 前台流程（这是用户触发并在等待执行的逻辑）：
  * 提交WAL，保障用户数据快速落库（落不落盘看配置）
  * 修改内存数据，以便服务读流程
* 后台流程（后台聚合执行，可以晚些执行，用户不感知）
  * 将内存中的脏数据（也就是仅仅写了WAL的那些数据）下刷到数据部分，拿到地址
  * 将新的数据地址也下刷到索引区域
  * 更新内存索引
  * （垃圾回收逻辑更偏后台，用于提高存储空间、读取效率）

关注点：读写线程模型、存储引擎的关键选型、主索引的实现

写线程模型：单线程模型、多IO线程模型、多线程模型随机分发、多线程模型hahs分发、多线程模型 + 随机/hash + GroupCommiter串行提交

### 复制：从单机到集群

1. **复制模式概述**：构建副本可扩展系统可用性、降低负载，复制的关键是在数据不断变更中同步数据并确定一致性语义。主要有三种复制模式：
    - **主从模式**：也叫单主、主动/被动模式（AP），是最常见模式，能提供最终一致性保障。
    - **多主模式**：即主-主（AA），多个主接受写请求并相互同步，解决冲突，还有从节点接受复制数据。
    - **无主模式**：无主从概念，客户端同时向若干实例发写请求，读时也同时读若干节点，与多主模式的区别在于是否多发。
2. **主从模式详细介绍**
    - **同步模式**：取决于业务对数据丢失的容忍度，包括同步（变更同步给所有从才返回）、半同步（变更至少同步给一个从才返回）、异步（不用等待直接返回，切主会丢数据）。共享存储类似同步模式，可解决切主丢数据问题。
    - **存量/增量数据同步**：采用全量（快照）+增量（增量日志，如WAL等）方式，系统需具备快照能力和回放增量日志的能力。不同存储引擎获取快照的方式不同，如Redis通过fork进程获取全内存快照，LSM树和Bw树相对容易获取，B+树较麻烦（如Mysql结合事务和外部工具）。
    - **独享&共享存储**：共享存储提供存算分离和更高效的主从同步能力，获取快照只需索引的checkpoint+回放WAL。增量方面，主需将下刷数据/索引结果同步给从。共享存储的主从模式接近同步模式，切主无丢数据风险。
    - **一致性模型**
        - **读己之写/写后读一致性/读写一致性**：保障同一个客户端能读到自己的写，实现方式包括读主、客户端记录时间戳或LSN来决定读主还是读从等。
        - **单调读**：保证用户只会读到更新的数据，可让客户端会话与固定从连接，或在会话内传递LSN，从根据LSN决定返回或路由读主，但会增加客户端复杂度。
        - **一致前缀读/因果一致**：有因果关系的多次写入在从上不会乱序读，主要发生在主从模式+「分片」场景，非分片模式的从按顺序写入无此问题。
    - **事务**：是一致性问题的更好解决方案，在确定的事务隔离性下能保障相关一致性，但性能开销大，部分NoSql数据库对事务支持不足。
3. **多主模式**：多个主节点提供更强的高可用写入保障，同地域副本需解决冲突问题，策略包括避免冲突（如按主键hash更新、数据分片）、LWW（Last Write Wins，用时间戳等确定终值）、记录冲突（按固定算法记录给用户或调用用户冲突解决算法）、CRDT等数据结构（自动解决冲突但有额外开销）。
4. **无主模式**：多写多读，依赖客户端或后台任务追齐数据，有效的读写依赖Quorum（W+R>N），如三副本场景。
5. **多主/无主一致性模型**：多点写入带来更复杂的一致性问题和模型，建议查阅相关书籍。事务基于具体存储引擎，后续分享具体存储时再讨论事务的定义和隔离级别。 

### 存储系统的演进：从单机到分布式的分片与副本技术

#### 一、单机存储的局限与分布式存储的诞生

单机存储引擎（如LevelDB、RocksDB）虽具备高效读写能力，但存在两大核心痛点：
1. **容量与性能瓶颈**：单节点存储容量有限，且IO与CPU资源易成为瓶颈。
2. **单点故障风险**：单机故障会导致数据丢失或服务不可用。

**解决方案**：通过**副本（Replication）** 解决数据安全性，通过**分片（Sharding）** 解决容量与性能扩展问题。两者结合形成了分布式存储的基础架构。


#### 二、分片技术的演进：从业务层到存储层的架构拆分

##### 1. 传统分库分表（Partition at Application Layer）

- **架构特点**：  
  通过Proxy或Client层将数据路由到不同数据库实例，分片内采用主从复制，分片间数据独立。  
  **典型案例**：Abase、MySQL分库分表。

- **核心实现**：  
  - **分片策略**：  
    - 水平分片（按哈希、范围）：如按用户ID哈希到不同库。  
    - 垂直分片（按业务模块）：如订单库与用户库分离。  
  - **挑战**：  
    - 分布式事务处理复杂（需2PC或最终一致性）。  
    - 跨分片查询性能低下（如JOIN操作需结果合并）。  
    - 全局自增ID生成困难（需额外服务如雪花算法）。

- **优缺点**：  
  - 优点：架构简单，易于实现，适合初期业务快速迭代。  
  - 缺点：业务侵入性强，分布式事务和复杂查询需业务层处理。


##### 2. 引擎层分片（Shared Nothing架构，Partition at Engine Layer）

- **架构特点**：  
  将数据库拆分为计算层（Server）与存储引擎层（Engine），仅在引擎层分片，节点间无共享资源。  
  **典型案例**：Spanner、TiDB、CockroachDB。

- **核心技术**：  
  - **一致性算法**：采用Multi-Paxos/Raft变种保证副本一致性（如TiDB的Raft Group）。  
  - **分布式事务**：  
    - Spanner：Paxos保证单分片一致性，跨分片事务用2PC。  
    - TiDB：Percolator模型结合Raft实现分布式事务。  
  - **存储引擎**：基于LSM树（如TiKV）或B+树（如CockroachDB）。

- **优缺点**：  
  - 优点：数据库内部处理分布式事务，业务无感知；支持弹性扩展。  
  - 缺点：架构复杂，运维成本高；跨分片事务仍有性能损耗。


##### 3. 存储层分片（计算存储分离，Shared Storage架构）

- **架构特点**：  
  将分片边界下移至存储层，计算层保留完整事务系统，存储层共享数据（如REDO日志）。  
  **典型案例**：Amazon Aurora、阿里云PolarDB、Google Spanner（部分实现）。

- **核心创新**：  
  - **日志即数据（Log is Database）**：  
    计算节点仅负责事务逻辑，存储节点共享REDO日志，通过LSN（日志序列号）保证一致性。  
  - **无分布式事务**：单计算节点处理事务，避免跨节点2PC。  
  - **存储层高可用**：  
    存储节点多副本（如Aurora的6副本）通过Quorum机制保证持久化。

- **优缺点**：  
  - 优点：计算与存储独立扩展，存储层成本低（共享存储）；故障恢复快（仅需重放日志）。  
  - 缺点：依赖存储层网络带宽，复杂查询仍受计算节点性能限制。


#### 三、副本技术的演进：从主从复制到一致性算法

##### 1. 主从复制（Master-Slave Replication）

- **应用场景**：传统分库分表、单机数据库高可用。  
- **实现**：主库写入后异步/半同步复制到从库。  
- **局限**：异步复制有数据丢失风险，半同步影响写入性能。

##### 2. 一致性算法（Consensus Algorithm）

- **应用场景**：Shared Nothing架构（如TiDB、CockroachDB）。  
- **核心算法**：  
  - Paxos/Raft：保证单分片内多副本一致性，如TiDB的Raft Group。  
  - Multi-Paxos：Spanner使用Paxos变种处理跨分片事务。  
- **优势**：强一致性，自动选主，故障时快速切换。

##### 3. 共享存储副本（Shared Storage Replication）

- **应用场景**：计算存储分离架构（Aurora、PolarDB）。  
- **实现**：  
  存储层多副本共享日志，计算节点通过LSN协调日志持久化。  
- **优势**：副本一致性由存储层保证，计算节点无状态，可快速扩展。


#### 四、不同架构的核心差异与适用场景

| 维度                | 分库分表               | Shared Nothing（TiDB）   | Shared Storage（Aurora） |
|---------------------|------------------------|--------------------------|--------------------------|
| **分片层级**        | 应用层/业务层          | 引擎层                   | 存储层                   |
| **分布式事务**      | 业务层处理（难）        | 数据库内部处理（较易）    | 避免分布式事务（最优）   |
| **扩展能力**        | 有限（受限于分片策略）  | 弹性扩展（计算+存储）     | 计算与存储独立扩展       |
| **典型场景**        | 中小规模业务，读多写少  | 大规模分布式业务         | 核心业务（金融、电商）   |
| **代表产品**        | 传统分库分表、Abase    | TiDB、CockroachDB        | Aurora、PolarDB          |


#### 五、存储架构演进的核心驱动力

1. **业务需求**：  
   - 互联网业务海量数据存储与高并发访问推动分片技术发展。  
   - 金融等场景对ACID强一致性要求催生一致性算法应用。

2. **技术突破**：  
   - 分布式一致性算法（Paxos/Raft）的工程化实现。  
   - 计算存储分离架构降低存储成本（如Aurora的存储成本仅为传统的1/10）。

3. **成本优化**：  
   - 共享存储架构通过存储资源池化降低硬件成本。  
   - 容器化与云原生技术推动存储架构弹性扩展。


#### 六、未来趋势：融合与智能化

1. **存算分离+存算融合**：  
   - 热数据计算存储融合以提升性能，冷数据存算分离降低成本。

2. **智能化存储**：  
   - AI驱动的自动分片、故障预测（如根据访问模式动态调整分片策略）。

3. **多模存储**：  
   - 支持SQL与NoSQL混合场景，如MongoDB的分布式架构演进。

存储系统的发展始终围绕“性能、成本、可用性”三角平衡，从业务层分片到存储层分片的演进，本质是将分布式复杂性从应用层下沉到系统层，最终实现“业务无感知”的分布式存储能力。