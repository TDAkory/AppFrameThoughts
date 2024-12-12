# [Erasure Coding](https://blog.min.io/erasure-coding/)

> Erasure coding is a key data protection method for distributed storage systems. 

- [分布式系统之数据可靠性（冗余存储中的EC算法）](https://www.cnblogs.com/z-sm/p/16254175.html)

## 冗余存储方案

若在最多允许m个数据丢失或损坏的情况下做到数据整体上仍可靠，该怎么做？（在实际分布式系统中，m通常取3）

* 多副本：简单的做法是每个数据都多存m个副本即可，例如Kafka内部数据的replication就是这种。其优点是简单、缺点是冗余率太高了，为 m/(1+m)。此方案可认为是下述方案的特例。
* 纠删码：EC算法的做法则是针对k个原始数据算出m个冗余数据，将k+m个数据一起存储，这样冗余率大大降低，为 m/(k+m)。其k+m个数据中任意最多m个丢失也能恢复。可见，上述方案就是这里当k=1时的情形！！