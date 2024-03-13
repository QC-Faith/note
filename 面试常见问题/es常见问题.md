es常见问题



什么是倒排序索引？



es通过分词操作将不同的词存入不通的文档将其归类到字典，当用户查询时使其时间复杂度大体控制在o(1)的程度





es的数据结构

一个分片中包含多个Mapping和Document 




Mapping（映射）是 Elasticsearch 中的一个重要概念，它定义了索引中存储的每个文档的字段和属性。Mapping 可以理解为索引的模式或者架构，它描述了索引中文档的结构，包括每个字段的数据类型、分析器、索引方式等信息。

在 Elasticsearch 中，Mapping 与 Document 是紧密相关的。Document 包含了实际的数据，而 Mapping 则定义了文档的结构和字段信息。在索引中，每个文档都必须遵循相应的 Mapping 规定的字段结构，否则 Elasticsearch 将无法正确解析和处理文档。



可以说 Mapping 是 Document 的一个补充，它规定了文档中每个字段的类型、分析器等属性，为文档的存储和检索提供了基础。



