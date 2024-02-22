从系统架构来看，目前的商用服务器大体可以分为三类，即对称多处理器结构 (SMP ： Symmetric Multi-Processor) ，非一致存储访问结构 (NUMA ： Non-Uniform Memory Access) ，以及海量并行处理结构 (MPP ： Massive Parallel Processing) 。

## SMP(Symmetric Multi-Processing)

对称多处理是指服务器中多个CPU对称工作，无主次或从属关系的硬件架构。 各CPU共享相同的物理内存，每个CPU访问内存中的任何地址所需时间是相同的，因此SMP也被称为一致存储器访问结构(UMA：Uniform Memory Access)。

对称多处理系统内有许多紧耦合的多处理器，所有的CPU共享全部资源。

操作系统管理着一个队列，每个处理器依次处理队列中的进程。如果两个处理器同时请求访问一个资源（例如同一段内存地址），由硬件、软件的锁机制去解决资源争用问题。Access to RAM is serialized; this and cache coherency issues causes performance to lag slightly behind the number of additional processors in the system.

## NUMA(Non-uniform memory access)

NUMA 服务器的基本特征是具有多个 CPU 模块，每个 CPU 模块由多个 CPU( 如 4 个 ) 组成，并且具有独立的本地内存、 I/O 槽口等。由于其节点之间可以通过互联模块 ( 如称为 Crossbar Switch) 进行连接和信息交互，因此每个 CPU 可以访问整个系统的内存 ( 这是 NUMA 系统与 MPP 系统的重要差别 ) 。显然，访问本地内存的速度将远远高于访问远地内存 ( 系统内其它节点的内存 ) 的速度，这也是非一致存储访问 NUMA 的由来。由于这个特点，为了更好地发挥系统性能，开发应用程序时需要尽量减少不同 CPU 模块之间的信息交互。

## MPP

和 NUMA 不同， MPP 提供了另外一种进行系统扩展的方式，它由多个 SMP 服务器通过一定的节点互联网络进行连接，协同工作，完成相同的任务，从用户的角度来看是一个服务器系统。其基本特征是由多个 SMP 服务器 ( 每个 SMP 服务器称节点 ) 通过节点互联网络连接而成，每个节点只访问自己的本地资源 ( 内存、存储等 ) ，是一种完全无共享 (Share Nothing) 结构，因而扩展能力最好，理论上其扩展无限制，目前的技术可实现 512 个节点互联，数千个 CPU 。目前业界对节点互联网络暂无标准，如 NCR 的 Bynet ， IBM 的 SPSwitch ，它们都采用了不同的内部实现机制。但节点互联网仅供 MPP 服务器内部使用，对用户而言是透明的。

- [五分钟理解服务器 SMP、NUMA、MPP 三大体系结构](https://www.51cto.com/article/713075.html)
- 