# Shared Nothing vs Shared Disk

## 1. Shared Nothing Architecture

Shared nothing architecture is an architecture which is used in distributed computing in which each node is independent and different nodes are interconnected by a network. Every node is made of a processor, main memory and disk. The main motive of this architecture is to remove the contention among nodes. Here the Nodes do not share memory or storage. The disks have individual nodes which cannot be shared. It works effectively on high volume and read write environment.

## 2. Shared Disk Architecture

Shared Disk Architecture is an architecture which is used in distributed computing in which the nodes share same disk devices but each node has its own private memory. The disks have active nodes which share memory in case of any failures. In this architecture the disks are accessible from all the cluster nodes. This architecture has quick adaptability to the changing workloads. It uses robust optimization techniques.

## Difference

|Shared Nothing Architecture | Shared Disk Architecture|
|----|----|
|In shared nothing architecture the nodes do not share memory or storage.|In shared disk architecture the nodes share memory as well as the storage.
|Here the disks have individual nodes which cannot be shared.|Here the disks have active nodes which are shared in case of failures.|
|It has cheaper hardware as compared to shared disk architecture.|The hardware in shared disk is comparatively expensive.|
|The data is strictly partitioned.|The data is not partitioned.|
|It has fixed load balancing.|It has dynamic load balancing.|
|Its advantage is that it has high availability.|Its advantage is that it has unlimited scalability.|
