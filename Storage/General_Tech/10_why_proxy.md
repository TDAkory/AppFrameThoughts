# Proxy

为什么要使用数据库Proxy

一句话概括：有了分布式，引入了数据分片，结合主从架构，就有了Proxy存在的意义：让用户尽可能不感知到Replication和Sharding，屏蔽各种中间态的处理逻辑以及数据迁移带来的暂态问题。

需要权衡的方面：

* Client带宽 VS Proxy带宽
* 运维成本，统一ACL逻辑处理，负载均衡功能
* 多一次RPC转发的CPU开销（如果有协议转换，开销可能更大）以及延迟增加
* 独立部署 VS 逻辑组件
* 部署在靠近Client侧 VS 部署在靠近Server侧 VS 中间
* etc

需要考虑的功能点：

* 性能上：线程模型、Cache、连接管理、Acceptor模式、连接数限流、断链检测、链接是否共享、数据零拷贝
* 控制层面：元数据、Server心跳、服务注册、路由策略
* 服务治理：客户端重试逻辑、限流、熔断
* 可观测性

## 通用数据库Proxy相关技术

### 零拷贝

* read + write（带DMA）
* mmap + write
* sendfile
* splice
* Userspace I/O

### 负载均衡算法

* 轮询（RR）Round-Robin
* 加权轮询（WRR）Weighted-Round-Robin
* 最少连接数（LC）Least-Connections
* 加权最少连接数（WLC）Weighted-Least-Connections
* 哈希算法 Hash
* 一致性哈希 Consistent Hashing
* 基于地理位置的GeoHash算法
* 基于各种Server性能的动态指标反馈算法

现实世界里，凡存在有状态分片的分布式服务，或者存在服务流量不同的服务（例如主从部署只能写主，希望读从），不仅限于数据库或存储产品组件，甚至是各类基于其实现的数据中间件服务，以及LocalCache型的RPC服务，都很有可能会提供功能相似的Proxy组件，而Proxy组件至少需要具备连接的管理、分片路由的管理与更新、元信息管理与心跳机制，以及一系列服务QoS的功能。