# 理解NUMA中的概念

> [Non-uniform memory access](https://en.wikipedia.org/wiki/Non-uniform_memory_access)

## 基本概念

NUMA 的几个概念：Node->Socket->Core->Thread，整体是一个从Node开始的对称树状结构。

NUMA 技术的主要思想是将 CPU 进行分组，Node 即是分组的抽象，一个 Node 表示一个分组，一个分组可以由多个 CPU 组成。每个 Node 都有自己的本地资源，包括内存、IO 等。每个 Node 之间通过互联模块（QPI）进行通信，因此每个 Node 除了可以访问自己的本地内存之外，还可以访问远端 Node 的内存，只不过性能会差一些，一般用 distance 这个抽象的概念来表示各个 Node 之间互访资源的开销。

随着多核技术的发展，我们将多个CPU封装在一起，这个封装一般被称为Socket（插槽的意思）。

而Socket中的每个核心被称为Core，也就是常说到的物理核。

为了进一步提升CPU的处理能力，Intel又引入了HT（Hyper-Threading，超线程）的技术，一个Core打开HT之后，在OS看来就是两个核，也就是逻辑核。由于逻辑核运行在同一个物理核上，因此其频繁的调度可能导致性能下降。

通过`lscpu`可以看到NUMA的基本信息：

```shell
$ lscpu
Architecture:        x86_64
CPU op-mode(s):      32-bit, 64-bit
Byte Order:          Little Endian
Address sizes:       40 bits physical, 48 bits virtual
CPU(s):              8
On-line CPU(s) list: 0-7
Thread(s) per core:  1  # 每个Core有1个逻辑核
Core(s) per socket:  4  # 每个Socket有4个Core
Socket(s):           2  # 共2个Socket
NUMA node(s):        2  # 共2两个NUMA Node
Vendor ID:           GenuineIntel
CPU family:          6
Model:               85
Model name:          Intel(R) Xeon(R) Platinum 8260 CPU @ 2.40GHz
Stepping:            7
CPU MHz:             2399.998
BogoMIPS:            4799.99
Hypervisor vendor:   KVM
Virtualization type: full
L1d cache:           32K    # L1 data cache
L1i cache:           32K    # L1 instruction cache
L2 cache:            1024K
L3 cache:            36608K
NUMA node0 CPU(s):   0-3    # 0-3号CPU在NUMA node0
NUMA node1 CPU(s):   4-7    # 4-7号CPU在NUMA node1
Flags:               fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ss ht syscall nx pdpe1gb rdtscp lm constant_tsc rep_good nopl xtopology cpuid tsc_known_freq pni pclmulqdq ssse3 fma cx16 pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand hypervisor lahf_lm abm 3dnowprefetch cpuid_fault invpcid_single ssbd ibrs ibpb stibp ibrs_enhanced fsgsbase tsc_adjust bmi1 avx2 smep bmi2 erms invpcid mpx avx512f avx512dq rdseed adx smap clflushopt clwb avx512cd avx512bw avx512vl xsaveopt xsavec xgetbv1 xsaves arat umip pku ospke avx512_vnni md_clear arch_capabilities
```

## 其他命令

1. 机器上有两个NUMA Node：node0、node1

```shell
$ ls /sys/devices/system/node/
has_cpu  has_memory  has_normal_memory	node0  node1  online  possible	power  uevent
```

2. 深入即可看到每个node下的cpu分布

```shell
$ ls /sys/devices/system/node/node0
compact  cpu3	   hugepages  memory10	memory14  memory18  memory21  memory32	memory36  memory4   memory43  memory47	memory50  memory54  memory58  memory61	memory65  memory69  memory8   subsystem
cpu0	 cpulist   meminfo    memory11	memory15  memory19  memory22  memory33	memory37  memory40  memory44  memory48	memory51  memory55  memory59  memory62	memory66  memory7   memory9   uevent
cpu1	 cpumap    memory0    memory12	memory16  memory2   memory23  memory34	memory38  memory41  memory45  memory49	memory52  memory56  memory6   memory63	memory67  memory70  numastat  vmstat
cpu2	 distance  memory1    memory13	memory17  memory20  memory3   memory35	memory39  memory42  memory46  memory5	memory53  memory57  memory60  memory64	memory68  memory71  power
```

3. 在`/proc/cpuinfo`的`physical id`字段，可以看到每个物理核所属的`socket`插槽

```shell
$ cat /proc/cpuinfo | grep "physical id"
physical id	: 0
physical id	: 0
physical id	: 0
physical id	: 0
physical id	: 1
physical id	: 1
physical id	: 1
physical id	: 1
```

4. 在`/proc/cpuinfo`的`core id`字段，可以看到每个物理核在所属的`socket`中的编号

```shell
$ cat /proc/cpuinfo | grep "core id"
core id		: 0
core id		: 1
core id		: 2
core id		: 3
core id		: 0
core id		: 1
core id		: 2
core id		: 3
```

5. 具体每个cpu的详细信息，可以在如下路径查看

```shell
$ ls /sys/devices/system/cpu/
cpu0  cpu1  cpu2  cpu3	cpu4  cpu5  cpu6  cpu7	cpufreq  cpuidle  hotplug  isolated  kernel_max  modalias  nohz_full  offline  online  possible  power	present  smt  uevent  vulnerabilities

$ ls /sys/devices/system/cpu/cpu0
cache  crash_notes  crash_notes_size  driver  firmware_node  hotplug  node0  power  subsystem  topology  uevent

$ ls /sys/devices/system/cpu/cpu0/cache
index0	index1	index2	index3	uevent

$ cat /sys/devices/system/cpu/cpu0/cache/index0/type
Data
$ cat /sys/devices/system/cpu/cpu0/cache/index1/type
Instruction
$ cat /sys/devices/system/cpu/cpu0/cache/index0/shared_cpu_list
0-1     # 共享这个cache的cpu编号
```

6. 查看NUMA的整体信息

```shell
$ numactl --hardware
available: 2 nodes (0-1)
node 0 cpus: 0 1 2 3
node 0 size: 7838 MB
node 0 free: 3808 MB
node 1 cpus: 4 5 6 7
node 1 size: 7934 MB
node 1 free: 3850 MB
node distances:
node   0   1
  0:  10  20
  1:  20  10
```
