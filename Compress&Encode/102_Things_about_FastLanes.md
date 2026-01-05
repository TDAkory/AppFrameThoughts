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

**现有格式痛点**：

- Parquet/ORC：依赖通用压缩（HWC），SIMD/GPU不友好，解压慢；无多列压缩，浪费列关联信息。
- BtrBlocks：虽支持级联LWC，但非全数据并行，无压缩执行，依赖外部依赖。

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
