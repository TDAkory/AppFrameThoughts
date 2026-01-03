# FastLanes

1. [The FastLanes Compression Layout: Decoding > 100 Billion Integers per Second with Scalar Code.](https://www.vldb.org/pvldb/vol16/p2132-afroozeh.pdf): significantly improving data decoding performance over the state-of-the-art by introducing a 1024-bit interleaved and Unified Transposed Layout, enabling data-parallel decoding even with scalar code. 
2. [FastLanes on GPU: Analysing DataParallelized Compression Schemes](https://dbdbd2023.ugent.be/abstracts/felius_fastlanes.pdf): data-parallelized layouts are essential to fully exploit GPU parallelism
3. [ALP: Adaptive Lossless Floating-Point Compression.](https://dl.acm.org/doi/pdf/10.1145/3626717): designed and implemented ALP, a new vectorized and data-parallel encoding for floating-point data
4. [The FastLanes File Format](https://www.vldb.org/pvldb/vol18/p4629-afroozeh.pdf)

**FastLanes** 是一款针对现代数据并行执行（SIMD/GPU）设计的开源大数据文件格式，核心创新在于 **表达式编码机制** 与 **分段块布局**，通过摒弃通用压缩（如Snappy/Zstd）、采用 **轻量级编码级联（LWC）** 和 **多列压缩（MCC）**，在保证 **压缩比优于Parquet（Public_BI数据集上比Parquet+Snappy高41%，接近Parquet+Zstd）** 的同时，实现 **解压速度较Parquet快43倍、较BtrBlocks快7倍** 的突破；支持向量级细粒度访问（1024值向量）和部分解压，适配AI管道与数据库查询，解决了现有格式SIMD不友好、缺乏多列关联压缩的痛点。

---

**现有格式痛点**
- Parquet/ORC：依赖通用压缩（HWC），SIMD/GPU不友好，解压慢；无多列压缩，浪费列关联信息。
- BtrBlocks：虽支持级联LWC，但非全数据并行，无压缩执行，依赖外部依赖。
**FastLanes目标**
- 适配现代硬件（多架构CPU、GPU），支持数据并行执行。
- 结合级联LWC与MCC，达到HWC压缩比，保持LWC速度。
- 支持细粒度访问与部分解压，适配压缩执行引擎。

**核心设计理念**

| 设计理念                | 具体说明         |
|------------------------|------------------|
| 数据并行优先            | 所有编码/解码操作无数据/控制依赖，适配1024值向量，支持编译器自动向量化    |
| 轻量级编码级联          | 组合多个LWC（如DICT+FFOR+RLE），捕捉多类数据模式，替代HWC              |
| 多列压缩（MCC）         | 利用列间关联（相等、一对一映射）或列拆分，提升压缩比                     |
| 细粒度访问              | 向量级（1024值）访问，减少解压内存占用，适配CPU/GPU缓存                 |


### 4. 关键问题
#### 问题1：FastLanes的核心创新是什么？如何解决现有格式的痛点？
答：核心创新是 **表达式编码机制** 和 **分段块布局**，搭配 **轻量级编码级联** 与 **多列压缩（MCC）**。  
- 解决通用压缩痛点：摒弃Snappy/Zstd等HWC，采用LWC级联（如DICT+FFOR+FSST），兼顾压缩比与数据并行性。  
- 解决单列压缩局限：通过MCC利用列间关联（相等、一对一映射），突破列式存储“孤立压缩”的缺陷。  
- 解决细粒度访问痛点：分段布局+向量级（1024值）访问，适配CPU/GPU缓存，支持部分解压与压缩执行。

#### 问题2：FastLanes在性能上与Parquet、BtrBlocks的核心差异是什么？关键原因是什么？
答：核心差异体现在 **解压速度** 和 **细粒度访问性能**，压缩比接近顶级方案：  
- 解压速度：FastLanes较Parquet快43倍、较BtrBlocks快7倍，关键原因是 **全数据并行设计**（无控制/数据依赖，适配SIMD）和 **LWC级联**（避免HWC的CPU密集型操作）。  
- 压缩比：Public_BI数据集上比Parquet+Snappy高41%，接近Parquet+Zstd，关键原因是 **MCC** 和 **LWC级联** 捕捉多类数据模式。  
- 随机访问：0.14ms较Parquet快315倍，关键原因是 **向量级细粒度访问**（无需解压整个块）和 **分段布局的入口点数组**。

#### 问题3：表达式编码的具体实现的是什么？如何保证编码选择的高效性？
答：表达式编码是FastLanes的统一压缩框架，实现如下：  
- 结构：以 **逆波兰表示（RPN）** 存储操作符（整数ID）和操作数（列/段索引），无字符串解析开销，支持任意LWC组合。  
- 操作符池：含15+种操作符，覆盖数值、字符串、浮点、MCC等场景，如ALP处理浮点、FSST处理字符串、EQUALITY处理列相等。  
- 高效编码选择：通过 **两级检测策略** 保证效率：  
  1. 规则匹配：优先处理常量列、相等列等简单场景，直接生成表达式（无采样开销）。  
  2. 三向采样：仅采样3个向量（0、中间、末尾），从预定义表达式池选择最优，压缩比准确率达99%，采样开销仅占压缩时间的6%。