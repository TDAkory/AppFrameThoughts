# FastLanes

1. [The FastLanes Compression Layout: Decoding > 100 Billion Integers per Second with Scalar Code.](https://www.vldb.org/pvldb/vol16/p2132-afroozeh.pdf): significantly improving data decoding performance over the state-of-the-art by introducing a 1024-bit interleaved and Unified Transposed Layout, enabling data-parallel decoding even with scalar code. 
2. [FastLanes on GPU: Analysing DataParallelized Compression Schemes](https://dbdbd2023.ugent.be/abstracts/felius_fastlanes.pdf): data-parallelized layouts are essential to fully exploit GPU parallelism
3. [ALP: Adaptive Lossless Floating-Point Compression.](https://dl.acm.org/doi/pdf/10.1145/3626717): designed and implemented ALP, a new vectorized and data-parallel encoding for floating-point data
4. [The FastLanes File Format](https://www.vldb.org/pvldb/vol18/p4629-afroozeh.pdf)

### 浮点数精度丢失问题

在将**IEEE 754双精度浮点数（doubles）转换为整数以实现压缩时**，核心存在两类精度丢失问题，具体表现及根源如下：

#### 1. 浮点数转整数解码阶段的精度偏差

- **问题表现**：通过传统十进制编码（如PDE的`P_enc`/`P_dec`流程）转换时，解码结果无法还原原始浮点数的精确位模式。  
  示例：对于浮点数`n=8.0605`，按`P_enc=round(n×10⁴)`编码得到整数`d=80605`，再通过`P_dec=d×10⁻⁴`解码时，结果为`8.0605000000000011084`，而非原始IEEE 754双精度表示的`8.06049999999999933209`，导致无损压缩失败。
- **根本原因**：  
  双精度浮点数无法精确表示`10⁻ᵉ`（e为小数点右移位数）。例如`10⁻⁴`（即0.0001）的实际双精度存储值为`0.000100000000000000002082`，该误差在`P_dec`的乘法步骤中被放大，最终导致解码结果偏离原始值。  
  注：`P_enc`阶段（`n×10ᵉ`）无精度损失，因为`10ᵉ`在`e≤21`时可被双精度精确表示。

#### 2. 整数表示的52位限制导致的舍入误差
- **问题表现**：当浮点数的“数量级+可见 decimal 精度”≥16时，`P_enc`计算出的整数会超出双精度的精确整数表示范围，触发自动舍入，导致编码不可逆。  
  示例：POI-lat、POI-lon数据集（平均decimal精度15.7-15.9），使用`e=14`编码时，`n×10¹⁴`的结果常超过`2⁵³`（双精度可精确表示的最大整数范围为`-2⁵³~2⁵³`），自动舍入后无法通过`P_dec`还原原始值，成功编码率仅70.5%-76.4%。
- **根本原因**：  
  双精度浮点数的尾数（fraction）仅52位，仅能精确表示`-2⁵³~2⁵³`之间的整数；超出该范围后，整数需按“2的幂次步进”存储（如`2⁵³~2⁵⁴`仅存偶数、`2⁵⁴~2⁵⁵`仅存4的倍数），`P_enc`的乘法结果若超出此范围，会被强制舍入，丢失原始精度。


### 二、对应的解决方案
针对上述两类精度丢失问题，文献[1] 2.5节及后续设计提出4类核心解法，兼顾精度保留与压缩效率：

#### 1. 采用高指数e提升解码精度
- **核心思路**：使用更高的指数`e`（如14、16），使`10⁻ᵉ`的双精度表示更接近真实值，减少`P_dec`的误差。  
- **效果**：  
  - 高指数`e`（如14）可使`10⁻¹⁴`的双精度表示为`1.00000000000000007771E-15`，误差极小，解码结果更接近原始浮点数；  
  - 全数据集平均成功编码率从“按可见精度选e”的82.5%提升至95%，部分数据集（如SD-bench、Stocks-UK）达99.9%；  
  - 进一步优化为“按向量选e”（每个1024值向量用1个e），成功编码率再提升至97.2%。

#### 2. 引入因子f裁剪尾部零，规避大整数存储
- **核心思路**：在`P_enc`基础上增加因子`f`（`f≤e`），通过`ALP_enc=round(n×10ᵉ×10⁻ᶠ)`裁剪整数`d`的尾部零，减少整数长度，同时避免新误差。  
- **原理与示例**：  
  对`n=8.0605`（原始双精度`8.06049999999999933209`），取`e=14`、`f=10`：  
  `ALP_enc=round(8.06049999999999933209×10¹⁴×10⁻¹⁰)=round(80604.9999999985448)=80605`，解码时`ALP_dec=80605×10¹⁰×10⁻¹⁴`，可精确还原原始值；  
  尾部零裁剪不引入新误差，因`10⁻ᶠ`的误差对整数`d`的round结果影响可忽略。

#### 3. 向量级自适应优化，减少参数开销与搜索成本
- **核心思路**：放弃“逐值选e”（如PDE），改为“按向量选e和f”，利用向量内decimal精度、数量级方差小的特性，缩减搜索空间。  
- **具体措施**：  
  - 搜索空间从253种（`0≤e≤21`、`f≤e`）缩减至5种以内，多数数据集（如Basel-wind、City-Temp）仅需1种最优`(e,f)`组合；  
  - 向量级参数存储（e、f、位宽仅存1次/1024值），避免逐值存储e的空间开销，同时减少CPU分支预测错误（无逐值if-else）。

#### 4. 针对高精度数据的`ALP_rd` fallback方案
- **核心思路**：对“数量级+decimal精度≥16”的高压缩难度数据（如POI-lat），放弃“整数转换”，转而压缩浮点数的前导位（sign+exponent+尾数高位），保留尾随位以保精度。  
- **实现**：  
  - 分割浮点数为前导位（低方差，用倾斜字典+位打包压缩）和尾随位（高随机，直接位打包）；  
  - 字典大小限制为`2³`（8个值），异常值（非字典值）用16位存储位置和值，确保精度无损，同时压缩比优于Gorilla、Elf等方案。