# [Raft](https://zh.wikipedia.org/wiki/Raft)

* [In Search of an Understandable Consensus Algorithm(Extended Version)](https://raft.github.io/raft.pdf)
* [万字详解Raft一致性算法](https://zhuanlan.zhihu.com/p/522435604)
* [Raft协议详解--背景+概念介绍+算法剖析](https://blog.csdn.net/tiancaidddddd/article/details/135995057)

## RaftLog

RaftLog（日志）是 Raft 一致性算法的核心组件之一，用于记录分布式系统中各节点需要达成共识的**状态变更操作**，确保集群中所有节点的状态最终一致。以下从定义、核心作用、关键机制三方面展开说明：

### **一、RaftLog 的定义**

RaftLog 是一个**有序的日志条目序列**，每个日志条目包含以下关键信息：

- **索引（Index）**：该条目在日志中的位置（全局唯一递增，从 1 开始）。
- **任期（Term）**：该条目被创建时的 Raft 任期号（用于版本控制和冲突检测）。
- **命令（Command）**：客户端请求的具体操作（如“修改某个键的值”）。
- **状态标记**：是否已提交（Committed）、是否已应用（Applied）等。

### **二、RaftLog 在 Raft 算法中的核心作用**

Raft 的核心目标是让分布式系统中的所有节点对“一系列操作的执行顺序”达成共识。RaftLog 是这一共识的**载体**，具体作用如下：

#### 1. **状态复制的基础**

- 客户端的请求（如写操作）会被领导者（Leader）封装为日志条目，追加到自己的 RaftLog 中。
- 领导者通过 `AppendEntries RPC` 将日志条目复制到其他跟随者（Follower）节点的 RaftLog 中。
- 当多数节点（超过半数）成功复制该日志条目后，领导者标记该条目为“已提交（Committed）”，并将其应用到本地状态机（如数据库）。
- 最终，所有节点的 RaftLog 会同步，状态机执行相同顺序的操作，保证状态一致。

#### 2. **冲突检测与修复**

- 当节点故障或网络分区时，不同节点的 RaftLog 可能出现不一致（如某个 Follower 缺少部分条目，或存在过期条目）。
- Raft 通过以下机制修复冲突：
  - **任期校验**：日志条目中的 `term` 用于判断新旧。如果 Follower 的日志中存在与 Leader 日志在相同索引但不同 `term` 的条目，Follower 会删除该条目及之后的所有条目，复制 Leader 的日志。
  - **索引对齐**：Leader 通过 `AppendEntries RPC` 携带前一条日志的索引和 `term`，确保 Follower 的日志前缀与 Leader 一致后，再追加新条目。

#### 3. **任期与领导者变更的容错**

- 每个任期（Term）最多有一个领导者。当领导者故障时，新的领导者通过选举产生（基于 `term` 递增）。
- 新领导者的 RaftLog 必须包含所有已提交的日志条目（Raft 的“领导者完整性”特性），确保其能继续推进日志复制，避免数据丢失。


### **三、关键机制示例**

以“客户端写请求”流程为例，RaftLog 的作用体现如下：

1. 客户端向领导者发送写请求（如 `Set(key=1, value=100)`）。
2. 领导者将该请求封装为日志条目（`index=5, term=3, command=Set(1,100)`），追加到自己的 RaftLog 中。
3. 领导者通过 `AppendEntries RPC` 向所有 Follower 发送该条目。
4. 当多数 Follower 确认接收（日志中存在 `index=5, term=3` 的条目），领导者标记该条目为“已提交”。
5. 领导者将该条目应用到本地状态机（如更新 `key=1` 的值为 100），并通知 Follower 提交该条目。
6. 所有 Follower 应用已提交的日志条目，最终所有节点的状态机均显示 `key=1` 的值为 100。
