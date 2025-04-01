---
title: "矢量存算新方案设计"  
date: 2025-04-01          
---


# 1 当前模式

## 1.1 总体架构

### 数据表关系

![](data:image/png;base64...)

### 1.1.2 函数调用关系

OGE包括两种矢量加载模式：

（1）用户上传矢量（当前仅支持geojson）

![](data:image/png;base64...)

（2）用户指定矢量产品（productID），从数据库加载

![](data:image/png;base64...)

## 存储设计

矢量元数据存储在Postgres，矢量数据存储在HBase。

矢量数据按照产品（图层）类别存储，在HBase中列族名为VectorData，每个矢量地理要素对应一个行键RowKey，RowKey由order+product\_key+feature\_id组成。order表示该网格点的索引编码，对于矢量点数据采用Z2编码方式，线、面数据采用XZ2多层级编码格式，确保每一矢量元素都存在对应唯一的编码order。product\_key对应产品key，feature\_id对应产品中各个矢量数据的编号。

![](data:image/png;base64...)

![](data:image/png;base64...)

| RowKey | TimeStamp | VectorData | |
| --- | --- | --- | --- |
| order+product\_key  +feature\_id |  | geom | metadata |

## 1.3 计算设计

OGE中的算子主要输入输出类型包括如下两类：

（1）Geometry

Geometry来自org.locationtech.jts.geom.Geometry（JTS库）

（2）Feature

在OGE中，当前矢量数据结构是RDD[(String, (Geometry, Map[String, Any]))]。其中Geometry来自org.locationtech.jts.geom.Geometry（JTS库）。

**表**：Feature已实现的原生算子

| 类别 | 描述 | 实现方式 |
| --- | --- | --- |
| 构造函数 | 从coors和properties构造 | 封装Geometry方法 |
| 从geojson构造 |
| 属性计算 | area/bounds/centroid/buffer | 遍历，调用Geometry方法 |
| 叠置分析 | contains/difference/intersection | （有误） |
| 其它 | 简单克里金插值/反距离加权插值/栅格化/visualize | 原生实现 |

## 不足

**1、Feature部分算子编写不合理**

![](data:image/png;base64...)

**2、算子种类不齐全**

当前仅包含少量基础算子，还需要更多支持复杂空间分析的矢量算子。

| 算子类型 | 解释 | 举例 |
| --- | --- | --- |
| 构造 | 构造矢量 | - |
| 执行 | 基于给定列执行一个函数 | Envelope/union |
| 聚合 | 基于给定列返回一个聚合值 | aggregate\_mean |
| 谓词 | 基于给定列进行逻辑判断并返回布尔值 | disjoint |
| 其它 | 更复杂的空间分析 | 空间连接 |

**3、矢量当前未实现导出（批处理）功能**

import oge # 初始化

oge.initialize()

service = oge.Service.initialize()

feature = service.getProcess("Feature.point").execute("[114.2, 30.3]", "{a:10}", "EPSG:4326")

feature.styles(["#000000"])**.export**("point") #未实现

oge.mapclient.centerMap(114.2,30.3,15)

**4、未区分Feature和FeatureCollection两个概念**

* **Feature** -> (Geometry, Map[String, Any])

Feature可以理解成Geometry + 属性

* **FeatureCollection** -> RDD[(String, (Geometry, Map[String, Any]))]

FeatureCollection可以理解成由众多Feature的集合

OGE当前业务场景无需区分，以后可以考虑区分。

![](data:image/png;base64...)

![](data:image/png;base64...)

**5、未进行空间分区**

![](data:image/png;base64...)

sc.parralize是不考虑空间属性的普通分区，在较为复杂的分析场景下（空间连接）性能较差。

空间分区：对空间进行格网剖分，从而进行分区（partition）操作，格网的生成算法是决定并行计算的关键，常见的格网生成算法包括四叉树（QuadTree）、KDB树（KDBTree）等。

6、**模式单一，无法支持多样化的矢量存储和分析需求**

因此需要对矢量存储计算模式进行优化，期望模式如下。

# 2 期望模式

## 总体架构

### 示意图

![](data:image/png;base64...)

### 2.1.2 数据表关系

![](data:image/png;base64...)

iceberg

## 2.2 存储设计

### 2.2.1 HBase

**场景：**存储复杂、更新频繁的矢量

**Hbase优势：**

1. 横向扩展能力

支持PB级数据存储，可线性扩展RegionServer节点；通过Region自动分裂实现数据分布式存储；支持动态Region迁移平衡集群负载。

（2）高性能访问

基于LSM树结构优化写操作，毫秒级响应；支持Scan操作高效遍历大量矢量要素；BlockCache缓存热点数据，读性能提升10倍+。

（3）灵活数据模型

单行可存储百万级列（适合存储矢量属性字段）；动态列 无需预定义列，适合异构矢量数据结构；多版本控制 支持时间戳版本管理（历史数据追溯）。

（4）空间数据优化

原生支持空间索引（Geohash/Z曲线等）；可通过协处理器实现ST\_Within等空间谓词下推； RowKey可集成空间编码（如Hilbert曲线）。

（5）高可用性

RegionServer故障时WAL日志快速恢复；HDFS多副本机制保障数据安全；行级ACID保证拓扑关系完整性。

### 2.2.2 PostgreSQL

**场景：**存储常规矢量

**PostgreSQL优势：**

（1）全功能空间数据处理

支持POINT、LINESTRING、POLYGON等OGC标准类型；包含ST\_Within、ST\_Intersects等空间谓词和ST\_Buffer等空间运算；可处理3D/4D数据，支持拓扑网络模型（TopoGeometry）。

（2）高性能空间索引

基于R树的通用搜索树索引，加速空间查询；支持<->距离运算符实现"最近邻"查询；多核并行化空间索引查询。

（3）标准兼容与互操作性

完全兼容ISO 13249-3和SQL/MM Part 3标准；支持WKT/WKB/GeoJSON等格式与内部类型互转；被QGIS/ArcGIS等主流GIS工具原生支持。

（4）数据库高级特性

复杂空间操作（如拓扑编辑）的原子性保证；可创建空间数据视图简化查询；空间数据变更时自动触发业务逻辑。

（5）扩展性与生态系统

PostGIS插件生态 支持pgRouting（路径分析）、PointCloud（点云）等扩展；多版本并发控制，读写操作互不阻塞，适合高频编辑场景；可同库存储分析栅格数据（PostGIS Raster）。

### 2.2.3 Parquet（一种文件格式）

**场景举例：**图斑的叠置分析 全量计算

**Parquet 文件格式的优势**包括：

（1）列式存储（Columnar Storage）

不同于行式存储（如 CSV、JSON），Parquet 按列存储数据，适合 只读取部分列 的场景（如机器学习特征提取）。能够显著减少 I/O 和存储占用，提升查询速度。此外，Parquet也可以以柱状方式存储具有嵌套结构的数据，即在 Parquet 文件格式中，即使是嵌套字段也可以单独读取，而无需读取嵌套结构中的所有字段。

（2）高性能压缩

Parquet 支持 Snappy、Gzip 等压缩算法，大幅降低存储成本（尤其适合海量数据）。例如：原始日志文件转为 Parquet 后，存储空间可能减少 50%~90%。

（3）Schema 演化支持

允许数据结构随时间变化（如新增/删除列），而无需重写全部数据。适合长期积累的训练数据集。（列级 TTL）

（4）与 Spark SQL 深度集成

Spark SQL 可直接高效读取 Parquet 文件，并利用其 谓词下推（Predicate Pushdown）、列裁剪（Column Pruning） 等优化技术，加速查询。例如：训练模型时只需读取部分特征列，Parquet 可跳过无关列的数据加载。

**如果训练数据需要 全量扫描（如特征工程），Parquet 比 HBase 更高效。Parquet可用于记录某一时刻全量数据，更适合全量计算。**

**如果数据需要 频繁更新，HBase 更合适，而 Parquet 适合静态数据（可通过覆写更新）。**

![](data:image/png;base64...)

Batch layer两个作用：（1）管理全量数据；（2）处理全量数据

Batch layer主要用于全量计算，处理所有历史数据，它可以使用HBase+Parquet+Spark SQL的机制实现，首先将所有原数据保存到HBase的一张表中，然后根据row key（可加入时间戳）读取HBase数据，根据读取到的数据从remote server文件数据源服务器获取文件到平台的HDFS，用**Parquet记录**信息，使用时再通过Spark SQL去读parquet file。

## 2.3 计算设计

### 2.3.1 Sedona

Apache Sedona是一个用户处理大规模空间数据的集群计算系统。Sedona通过一组开箱即用的分布式空间数据集和空间SQL扩展了现有的集群计算系统，例如Apache Spark，这些系统可以跨计算机高效加载、处理和分析大规模空间数据。

![](data:image/png;base64...)

![](data:image/png;base64...)

核心示例：通过SQL语句实现功能

//sedona: SparkSession

var polygonCsvDf = **sedona.read**.format("csv").option("delimiter",",").option("header","false").load(*csvPolygonInputLocation*)
polygonCsvDf.createOrReplaceTempView("polygontable")
polygonCsvDf.show()
var polygonDf = **sedona.sql**("select ST\_PolygonFromEnvelope(cast(polygontable.\_c0 as Decimal(24,20)),cast(polygontable.\_c1 as Decimal(24,20)), cast(polygontable.\_c2 as Decimal(24,20)), cast(polygontable.\_c3 as Decimal(24,20))) as polygonshape from polygontable")
**polygonDf. write**.format("geoparquet").mode("overwrite").save("polygon.parquet")

核心示例：DataFrame与SpatialRDD的转换

var tripDf = sedona.read.format("csv").option("delimiter",",").option("header","false").load(*nyctripCSVLocation*)
tripDf.createOrReplaceTempView("tripdf")
var tripRDD = **Adapter.*toSpatialRdd***(sedona.sql("select ST\_Point(cast(tripdf.\_c0 as Decimal(24, 14)), cast(tripdf.\_c1 as Decimal(24, 14))) as point, 'def' as trip\_attr from tripdf") , "point")
arealmRDD.analyze()
tripRDD.analyze()
tripRDD.spatialPartitioning(GridType.*KDBTREE*)
tripRDD.buildIndex(IndexType.*QUADTREE*, true)
tripRDD.*indexedRDD* = tripRDD.*indexedRDD*.cache()

### 2.3.2 PostGIS

PostGIS通过向PostgreSQL添加对**空间数据类型、空间索引和空间函数**的支持，将PostgreSQL数据库管理系统转换为空间数据库。

空间数据库支持在查询的 WHERE 或 ON 子句中使用空间运算符或索引感知函数，PostGIS 空间数据库索引通过将数据组织成搜索树来加快搜索速度，该搜索树可以快速遍历以查找特定记录。

空间数据库提供的空间函数可归纳为以下五个类：

• 转换 —— 在geometry（PostGIS中存储空间信息的格式）和外部数据格式之间进行转换的函数

• 管理 —— 管理关于空间表和PostGIS组织的信息的函数

• 检索 —— 检索几何图形的属性和空间信息测量的函数

• 比较 —— 比较两种几何图形的空间关系的函数

• 生成 —— 基于其他几何图形生成新图形的函数

此外，PostGIS可扩展性强。PostgreSQL从一开始就考虑到类型扩展 —— 能够在运行时添加新的数据类型、函数和访问方法的机制。

### 2.3.3 PostGIS与Sedona自适应计算框架设计

![](data:image/png;base64...)

为了实现根据数据量自动选择PostGIS或Sedona进行计算的功能，可以设计一个智能路由系统，通过评估查询特征（基于数据量和查询复杂度）选择最佳执行引擎。

**核心组件：**

路由决策器 (RoutingDecider)

PostGIS执行器 (PostGISExecutor)

Sedona执行器 (SedonaExecutor)

**决策流程：**

用户请求 → 路由决策器 → (PostGIS/Sedona)执行器 → 返回结果

↑

性能监控器反馈

## 2.4 算子接口设计

考虑参考Sedona的SpatialRDD类，实现一个灵活、多态的FeatureCollection类。

![](data:image/png;base64...)


