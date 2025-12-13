# 数据压缩算法

## Why Compression?

1. 存储成本高（Storage is costly）
2. 信息处理的纯粹形式（Purest form of information processing）  
   • 提炼信息的本质（distilling your information to its essence）  
   • 实现无噪通信（communication without the noise）
3. 压缩表示的优势（Compressed representations）  
   • 易于传输和搜索（easy to transmit and search）  
   • 简化系统实现（can simplify implementations）
4. 与其他领域的关联  
   • 与数据建模、预测、机器学习等密切相关（Connection with data modeling, prediction, ML etc.）
5. 系统构建的关键  
   • 是可扩展高效系统的核心构建模块（Critical building block in scalable and efficient systems）

   这张幻灯片的核心内容是**期望码长**的定义和计算方法，这是数据压缩理论中的一个基础且重要的概念。

### **期望码长（Expected Code Length）**

它衡量的是对一个数据源（如一个文件、一段信息）中所有符号进行编码后，**平均每个符号所占用的比特数**。单位通常是 **比特/符号**。

期望码长直接反映了压缩算法的效率。期望码长越低，表示压缩效率越高。

**E[l(X)] = Σ P(x) * l(x)** （对所有符号 x ∈ X 求和）

- **E[l(X)]**： 表示随机变量 `l(X)` 的**期望值**，即期望码长。
- **X**： 代表数据源中所有可能符号的集合。
- **x**： 集合 `X` 中的一个具体符号。
- **P(x)**： 符号 `x` 在数据源中出现的**概率**。
- **l(x)**： 为符号 `x` 分配的**码字长度**（单位：比特）。

假设一个数据源只有两个符号 `A` 和 `B`：
- `P(A) = 0.8` （A出现的概率是80%）
- `P(B) = 0.2` （B出现的概率是20%）
- 我们设计的编码方案是：`A` 用码字 `0` 表示（长度 `l(A) = 1` 比特），`B` 用码字 `11` 表示（长度 `l(B) = 2` 比特）。

那么，这个编码方案的期望码长为：
**E[l(X)] = P(A) * l(A) + P(B) * l(B) = 0.8 * 1 + 0.2 * 2 = 0.8 + 0.4 = 1.2 比特/符号**

### **唯一可译码**

一个码是唯一可译码的，当且仅当**不存在**两个不同的输入序列（例如 \(x^n\) 和 \(y^m\)，其中 \(m, n \geq 1\)）被编码成相同的输出序列。

## Refs

- [EE274数据压缩理论与应用 From Stanford](https://www.bilibili.com/video/BV1LyreYKEUD/?spm_id_from=333.337.search-card.all.click&vd_source=3b30eb8157301900ba82e04d7f8fa9d3)
  - [EE274 Course site](https://stanforddatacompressionclass.github.io/Fall25/)
  - [Github Page](https://github.com/StanfordDataCompressionClass)
  - [Notes](https://stanforddatacompressionclass.github.io/notes/)

- [数据流压缩原理（Deflate压缩算法、gzip、zlib）](https://blog.51cto.com/u_15346415/5026718)
- [Zstandard Compression and the 'application/zstd' Media Type](https://datatracker.ietf.org/doc/html/rfc8878#name-entropy-encoding)
- [bit packing](https://kinematicsoup.com/news/2016/9/6/data-compression-bit-packing-101)

- [Intel(R) Intelligent Storage Acceleration Library](https://github.com/intel/isa-l)