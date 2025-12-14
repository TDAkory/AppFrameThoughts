# [Improving Time Series Data Compression in Apache IoTDB](https://www.vldb.org/pvldb/vol18/p3406-tang.pdf)

![oT data processing in Apache IoTDB](https://raw.githubusercontent.com/TDAkory/ImageResources/master/img/AppFrameThoughts/Homomorphic_compression_IoTDB.png)

通过2类核心优化策略，实现前文定义的“有效同态查询”（降低查询成本）：

- **Late Decompression（晚解压）**：利用前文“数据大小单调性”引理，尽量延迟解压操作——仅在必须解压的算子（极少数不支持同态的算子）执行时，才解压**最小范围的局部数据**，避免全量解压的开销。
- **Dynamic Auxiliary Management（动态辅助管理）**：对应前文“有效恢复”定义，动态管理辅助结构（如空值位图、删除列表），在压缩数据上直接恢复辅助信息，且恢复成本低于未压缩数据的传统恢复方式。

提供适配压缩查询的核心数据结构，是算子模块能直接操作压缩数据的基础：

- **CompColumn（压缩列结构）**：CompressIoTDB的核心存储结构，将时序数据按列压缩存储，并关联对应的压缩算法元信息，支撑算子直接访问压缩内容。
- **Compression Offset Index（压缩偏移索引）**：快速定位压缩数据块的位置，避免遍历全量压缩数据的开销。
- **HintIndex（提示索引）**：辅助快速匹配压缩数据的特征（如时间范围、值范围），提升过滤、聚合的效率。

CompressIoTDB的查询流程（对应图中箭头交互）：

1. 存储层的**压缩数据块（①）** 被读取到`Data Structure Module`，解析为CompColumn等结构；
2. `Operator Module`调用对应算子执行查询逻辑，同时通过**③的交互**从`Data Structure Module`获取压缩数据；
3. `Optimization Module`通过**②的交互**，向`Operator Module`注入晚解压、动态辅助管理等优化策略；
4. 算子在压缩数据上完成计算后，将结果返回给IoTQuery Layer，最终封装为`Query Result`返回给IoTClient。

