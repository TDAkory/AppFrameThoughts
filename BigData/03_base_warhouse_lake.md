# 数据库、数据仓、数据湖、湖仓一体化

数据库主要用于「事务处理」，存取款这种算是最典型的，特别强调每秒能干多少事儿：QPS（每秒查询数）、TPS（每秒事务数）、IOPS（每秒读写数）等等。主要负责事务处理相关的事

数据仓库相当于一个集成化数据管理的平台，从多个数据源抽取有价值的数据，在仓库内转换和流动，并提供给BI等分析工具来输出干货。主要负责业务分析相关的事

数据湖的本质，是由“数据存储架构+数据处理工具”组成的解决方案，而不是某个单一独立产品。

Lake House架构最重要的一点，是实现“湖里”和“仓里”的数据/元数据能够无缝打通，并且“自由”流动。湖里的“新鲜”数据可以流到仓里，甚至可以直接被数仓使用，而仓里的“不新鲜”数据，也可以流到湖里，低成本长久保存，供未来的数据挖掘使用。

- [数据库、数据仓库、数据湖、湖仓一体分别是什么？](https://support.huaweicloud.com/dws_faq/dws_03_2121.html)
- [数据仓库、数据湖和数据集市之间有什么区别](https://aws.amazon.com/cn/compare/the-difference-between-a-data-warehouse-data-lake-and-data-mart/)
- [数据湖 VS 数据仓库](https://www.alibabacloud.com/zh/knowledge/data-lake-vs-data-warehouse?_p_lc=1)
