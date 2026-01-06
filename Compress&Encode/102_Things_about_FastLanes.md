# FastLanes

1. [The FastLanes Compression Layout: Decoding > 100 Billion Integers per Second with Scalar Code.](https://www.vldb.org/pvldb/vol16/p2132-afroozeh.pdf): significantly improving data decoding performance over the state-of-the-art by introducing a 1024-bit interleaved and Unified Transposed Layout, enabling data-parallel decoding even with scalar code. 
2. [FastLanes on GPU: Analysing DataParallelized Compression Schemes](https://dbdbd2023.ugent.be/abstracts/felius_fastlanes.pdf): data-parallelized layouts are essential to fully exploit GPU parallelism
3. [ALP: Adaptive Lossless Floating-Point Compression.](https://dl.acm.org/doi/pdf/10.1145/3626717): designed and implemented ALP, a new vectorized and data-parallel encoding for floating-point data
4. [The FastLanes File Format](https://www.vldb.org/pvldb/vol18/p4629-afroozeh.pdf)

## FastLanes Compression Layout

`transpose_i`

本文围绕“提升轻量级压缩（LWC）解码效率”展开，提出一套适配SIMD架构、跨硬件兼容的压缩布局方案，核心目标是通过数据布局重构与硬件友好设计，突破传统LWC解码的性能瓶颈，实现超高速并行解码，为下一代大数据格式奠定基础。

**行业痛点**：

- 列式存储广泛应用于分析型系统，但传统LWC（DICT/FOR/DELTA/RLE）解码存在SIMD宽度异构（如AVX512/NEON寄存器宽度差异）、数据依赖（DELTA/RLE需顺序计算）、布局与车道宽度绑定等问题；
- 现有方案要么依赖平台特定SIMD intrinsics导致技术债务，要么解码效率低，难以充分利用现代CPU的向量计算能力。

| 挑战 | 核心矛盾 |
|------|----------|
| 多SIMD宽度 | 不同硬件寄存器宽度（64~512位）需适配统一布局 |
| 异构ISA | x86_64/ARM64等架构指令集差异，需保证代码可移植 |
| 解码数据依赖 | DELTA依赖前值、RLE依赖循环计数，阻碍并行 |
| 布局-位宽绑定 | 不同列数据类型（8/16/32/64位）需统一布局 |
| 代码可移植性 | 避免平台特定 intrinsics，降低维护成本 |
| 内存读写瓶颈 | 解码速度过快导致LOAD/STORE-bound，浪费CPU资源 |

**核心技术方案**：

- 虚拟1024位SIMD寄存器（FLMM1024）
  - 定义虚拟1024位寄存器作为统一适配目标，无需绑定实际硬件SIMD宽度；
  - 配套轻量级指令集（LOAD/STORE/AND/ADD等基础操作），兼容所有ISA，可通过 scalar 代码模拟，支持自动向量化。

- 1024位交错位打包布局
  - 将1024值向量按“索引取模+整除”规则，分布到`S=1024/T`个T位车道（如T=8→128个车道）；
  - 逻辑连续值被循环分配到不同车道，避免跨车道重排（PERMUTE/BITSHUFFLE），适配任意SIMD宽度（窄寄存器分多轮处理，无性能损失）。

- 统一转置布局（Unified Transposed Layout）
  - 基础单元：8×16值的瓦片（Tile），1024值向量拆分为8个瓦片；
  - 核心规则：瓦片按04261537顺序重组，通过瓦片拆分适配8/16/32/64位所有车道宽度；
  - 核心作用：打破DELTA/RLE的数据依赖，使每个瓦片内数据独立，支持SIMD并行计算。

性能评估结果：覆盖Intel Ice Lake（AVX512）、AMD Zen3/4、Apple M1、AWS Graviton2/3等多架构CPU。

- 解码速度：8位类型达70值/CPU周期（1400亿值/秒），64位类型达3~4倍标量速度；
- DELTA解码：8位类型>40值/CPU周期，较传统方案快3~40倍；
- 端到端查询：SELECT SUM(COL)场景，8线程下较未压缩数据提升7倍，较 scalar 解码提升4倍；
- 自动向量化：scalar 代码性能与显式SIMD intrinsics持平，无需手动优化。
- vs 4-way交错布局：AVX512下快4倍，充分利用宽寄存器并行性；
- vs 传统RLE：短运行长度（>80个/向量）解码快数倍，无分支预测损失；
- vs D4/DM布局：DELTA编码比特数更少，压缩比更优。

**统一转置布局**：有些复杂，看了半天论文和代码，核心思路就是：根据数据总量C、数据类型T、SIMD指令位宽W，先对数据重新分组，然后计算。比如，要对 1024个double进行delta压缩，采用SIMD256指令处理

```shell
# 常规算法
_m256 pre = load[0..3]
_m256 now = load[1..4]
_m256 delta[1..4] = now - pre 
store(delta[1..4] --> ret[1..3]) # 涉及跨Lane重排, ret[0]=0

pre = load[3..6]
now = load[4..7]
delta[4..7] = now - pre
store(delta[4..7] --> ret[4..7]) # 无跨Lane，但头尾需要特殊处理，循环自增不一致
store(delta[4..7] --> ret[4..6]) # 全局一个循环，但是有跨Lane

# SIMD friendly 算法
# 重分组 一条指令加载4个double，因此分4组[0~255],[256~511] ... 
# 转置 使内存连续，方便SIMD加载
_m256 pre = load[0, 256, 512, 768]
_m256 now = load[1, 257, 513, 769]
delta[1, 257, 513, 769] = now - pre
pre = now
now = load next line
```

从**架构层面消除跨Lane操作的必要性、提升循环规整性、释放Tile级并行度**；即便常规算法通过头尾特殊处理减少跨Lane指令的调用次数，也无法解决其“数据依赖串行、循环不规整、硬件利用率低”的本质问题

| 术语 | 常规算法中的表现 | 转置布局中的表现 |
|------|------------------|------------------|
| 跨Lane操作 | 要么“每个循环调用”（如`store(delta[1..4]→ret[1..3])`），要么“全局一次但开销仍在”（如ret[0]=0的重排） | 完全消除，无需任何跨Lane指令（PERMUTE/BITSHUFFLE/切片） |
| 循环规整性 | 步长不一致（加载[0..3]→[3..6]，步长3）、store需切片，编译器无法自动向量化 | 固定步长（128）、连续加载/存储，编译器可自动生成SIMD指令 |
| 数据依赖 | 下一个SIMD块必须等待上一个块完成（如[3..6]依赖[0..3]的最后一个值），仅能串行 | 瓦片级并行，8个瓦片可同时计算，并行度提升8倍 |
| 硬件利用率 | SIMD车道闲置（如_m256计算4个值仅存储3个，利用率75%） | 车道100%利用，无闲置/丢弃的计算结果 |

![Unified Transposed Layout:](https://raw.githubusercontent.com/TDAkory/ImageResources/master/img/AppFrameThoughts/fastLane_unified_transposed_layout.png)

## File Format

**FastLanes** 是一款针对现代数据并行执行（SIMD/GPU）设计的开源大数据文件格式，核心创新在于 **表达式编码机制** 与 **分段块布局**，通过摒弃通用压缩（如Snappy/Zstd）、采用 **轻量级编码级联（LWC）** 和 **多列压缩（MCC）**，在保证 **压缩比优于Parquet（Public_BI数据集上比Parquet+Snappy高41%，接近Parquet+Zstd）** 的同时，实现 **解压速度较Parquet快43倍、较BtrBlocks快7倍** 的突破；支持向量级细粒度访问（1024值向量）和部分解压，适配AI管道与数据库查询，解决了现有格式SIMD不友好、缺乏多列关联压缩的痛点。

---

传统格式存在以下局限性：

- **重度压缩算法（HWC）依赖：** 如Snappy和Zstd等通用压缩算法难以数据并行，是数据并行计算的“对立面”，导致解压速度成为性能瓶颈。
- **粗粒度访问：** 传统格式通常以行组（Rowgroup）为单位解压，内存占用大且无法有效利用CPU/GPU缓存。
- **列间相关性利用不足：** 传统列式存储独立处理各列，忽略了列与列之间的相关性压缩机会。

**FastLanes目标**：

- 适配现代硬件（多架构CPU、GPU），支持数据并行执行。
- 结合级联LWC与MCC，达到HWC压缩比，保持LWC速度。
- 支持细粒度访问与部分解压，适配压缩执行引擎。

**核心设计理念**：

| 设计理念                | 具体说明         |
|------------------------|------------------|
| 数据并行优先            | 所有编码/解码操作无数据/控制依赖，适配1024值向量，支持编译器自动向量化    |
| 轻量级编码级联          | 组合多个LWC（如DICT+FFOR+RLE），捕捉多类数据模式，替代HWC              |
| 多列压缩（MCC）         | 利用列间关联（相等、一对一映射）或列拆分，提升压缩比                     |
| 细粒度访问              | 向量级（1024值）访问，减少解压内存占用，适配CPU/GPU缓存                 |

### **核心技术创新**

#### **1. 表达式编码（Expression Encoding）**

FastLanes引入了一种全新的压缩模型，将复杂的轻量级编码（LWC）分解为可重用的**算子（Operators）**，并通过**表达式链**进行组合。

- **级联压缩：** 通过组合多个算子（如先进行字典编码再进行RLE编码），FastLanes能在保持解压速度的同时达到或超过重度压缩的压缩率。
- **算子库：** 包含FFOR（融合位打包的框架编码）、DELTA、ALP（针对浮点数）、DICT（字典）、FSST（字符串压缩）等多种数据并行算子。
- **序列化方式：** 使用修改后的**逆波兰表示法（RPN）**存储表达式，最小化运行时解析开销。

#### **2. 多列压缩（Multi-Column Compression, MCC）**

FastLanes将多列相关性视为压缩突破口，支持以下模式：

- **相等性（EQUALITY）：** 识别并压缩完全相同的列。
- **一对一映射：** 利用两列之间的映射关系，共享字典或引用。
- **类型转换（Cast）：** 将字符串或双精度浮点数转换为更易压缩的整数类型。
实验显示，MCC方案能为压缩率和解码速度带来约**8%-20%的提升**。

#### **3. 分段页面布局（Segmented Page Layout）**

为了支持细粒度的向量化解码，FastLanes设计了分段布局：

- **向量化API：** 以**1024个值（一个向量）**为基本单位进行读写，完美匹配SIMD寄存器和GPU Warp的并行度。
- **入口点数组（Entry Points）：** 每个段通过入口点跟踪每个向量的数据偏移，实现真正的**随机访问**，避免了块级压缩必须解压整个块的弊端。
- **元数据分离：** 脚注（Footer）使用**FlatBuffers**序列化，支持不读取数据文件即可进行投影下推和统计过滤。

### **编码检测算法**

FastLanes采用**两阶段算法**快速寻找最优编码表达式：

1. **规则阶段：** 基于启发式规则识别常量列、相等列、相关列及数据类型优化（如缩小整数位宽）。
2. **采样阶段（三向采样）：** 抽取行组的首、中、尾三个向量（共3072个值），在预设的表达式池中尝试所有组合。实验证明，仅需采样3个向量，准确率即可超过**99%**。

**总结与洞察：**
FastLanes的核心哲学是**“用计算换取带宽，但不牺牲并行性”**。它通过将复杂的解压逻辑拆解为极简、无依赖的SIMD算子，解决了数据存储与现代加速硬件之间的鸿沟。

**类比理解：**
如果说**Parquet**像是一个装满大箱子的**集装箱货轮**（必须吊起整个箱子才能拿到里面的一件衣服），那么**FastLanes**就像是一条**全自动智能传送带**。传送带上的商品（数据）被精准地贴上了微小的电子标签（表达式元数据），机械臂（SIMD/GPU）可以在高速移动中，仅凭标签就瞬间抓取并处理任何一个特定的小包裹，而无需停下整条生产线。
