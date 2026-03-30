# ClickHouse学习

### 一、基础概念

##### 1、简单介绍：

##### 2、适用场景：

​	**数据分析**：用户行为分析、日志统计、广告分析等。

​	**大数据实时统计**：秒级/分钟级的聚合分析。

​	**高并发读、低延迟查询**：千万级/亿级条数据的聚合查询也很快。

​	**不适合**：高频单行更新/删除

##### 3、数据存储结构概览

	- 列存：每列数据独立存储，连续在磁盘上，所以聚合、压缩、IO都较为高效
	- Part(数据分片)： MergeTree引擎将数据分为多个Part(文件夹级别的分块)
	- Merge机制: 后台异步合并Part，减少碎片，以提高查询性能
	- 分区(Partition)：用于划分大表，如按日期分区，查询时可快速跳过不相关的分区，以提高效率。
	- 主键与索引：主键不是唯一约束，而是排序索引。用于范围查询、过滤加速。

##### 4、行存储 VS 列存储：

| 特性     | 行存储（MySQL/传统数据库） | 列存储（ClickHouse）                 |
| -------- | -------------------------- | ------------------------------------ |
| 存储方式 | 一行数据连续存储           | 每列数据连续存储                     |
| 适合场景 | OLTP：大量写、单行操作     | OLAP：分析聚合、扫描特定列           |
| 查询效率 | 单行查询快                 | 聚合扫描快，列裁剪（只读取需要的列） |
| 压缩效率 | 较低                       | 高（同列相似性高，压缩率高）         |

> 💡 简单理解：行存储像“一本笔记本，每页写一条记录”，列存储像“每页只写某一列的所有值”，所以分析某一列时无需翻整本笔记本，直接读对应页面就好。	

### 二、核心引擎与架构

##### 1、MergeTree引擎概览

提到引擎，首先我们需要比较一下和mysql的引擎：

| 特性     | MySQL 引擎            | ClickHouse 引擎              |
| -------- | --------------------- | ---------------------------- |
| 数据结构 | 行存                  | 列存                         |
| 事务支持 | 完全支持（ACID）      | 单行原子写入，批量最终一致性 |
| 查询优化 | B+Tree、二级索引      | 主键稀疏索引、分区裁剪       |
| 适用场景 | OLTP：高并发插入/更新 | OLAP：分析、聚合、日志统计   |
| 合并机制 | 不需要                | MergeTree 异步合并 Part      |

**MergeTree 是 ClickHouse 最常用的表引擎**，也是其他系列引擎（ReplacingMergeTree、SummingMergeTree 等）的基础。

**核心作用**：

1. 支持大规模数据存储。
2. 提供高效的**范围查询**和**排序索引**。
3. 自动管理数据的**合并（Merge）**和 **压缩**

##### 2、数据组织结构

- Part(数据块)：
  - 数据写入MergeTree后，会先生成Part
  - 每个Part内部按照主键排序，便于快速查询
  - Part的大小通常是几十MB到几百MB，后台会合并成更大的Part
- Partition(分区)
  - 可以按照日期、ID范围等划分
  - 查询时，如果过滤条件涉及分区列，Clickhouse可以直接跳过不相关分区，极大提升查询效率。
- Primary Key(主键索引)
  - Clickhouse的主键是稀疏索引。
  - 每隔N行记录存一次主键索引信息，用于快速定位数据范围。
  - 并不是唯一约束，而是为了快速范围过滤。

##### 3、Merge机制

   - 数据写入后会形成许多Part，如果不处理，会导致查询慢、碎片多。
   - MergeTree的后台线程会自动合并小Part成大Part
   - 合并策略和规则：
     - 小Part会先合并成为中等Part，再逐步合并成更大Part
     - 合并的过程是增量异步的，对查询不阻塞。

Merge机制是Clickhouse高性能的关键之一。

##### 4、写入数据

- 写入
  - 批量写入性能最好
  - 每次写入生成新的Part，后台合并
  - 单条写入也支持，但是会产生许多小Part，影响性能。
- 为什么Clickhouse推荐批量写入？
  - 批量写入----> 每次生成一个较大的Part ---->后台Merge---->高性能
  - 单条写入----->每次生成一个Part---->过多小Part---->查询慢、Merge压力大

##### 5、查询执行流程简化

-  查询到达Clickhouse
-  分区裁剪+主键索引定位--->确定需要读取的Part和数据范围
-  读取相关列的数据---->CPU并行聚合计算
-  返回结构
   -  高并行：每个Part都可以独立计算，充分利用多核CPU
   -  列裁剪：只读取必要列，减少IO

##### 6、分区裁剪

	- 表可以按列分区(如date列按天分区)
	- 查询时只扫描相关分区，跳过无关分区，比如查询2025-11-01的数据，只读取2025-11-01的分区，其他天的数据完全不用管。

##### 7、列式存储的优势

​	通常在分析型查询里，我们并不需要全量的字段，而是某些，例如

```
SELECT user_id, count(*) FROM logs WHERE event = 'click';
```

对于mysql来说 即使只查某些字段，但是innodb在初步获取数据的时候 还是会在对应的页中 把整条数据都薅出来，而对于Clickhouse来说，因为它是列式存储的，就只需要读取查询所需要的列，从第一步就极大的减少了磁盘和内存的读取量，

### 三、集群与分布式

### 四、事务与一致性

### 五、高级特性

### 六、性能优化

### 七、运维与监控

### 八、一些好用的SQL

```
清空表
TRUNCATE TABLE table_name
```



```
复制表
use team_study_data;
show create table team_pending_data_0;

CREATE TABLE team_study_data.team_pending_data_65
(
    `judgmentNumCount` Int64,
    `minuteTime` DateTime,
    `hourTime` DateTime,
    `dayDate` Date,
    `completeTime` DateTime,
    `completeDate` Date,
    `collectTime` DateTime,
    `createTime` DateTime,
    `cumulativeNumCount` Int64 DEFAULT 0
)
ENGINE = MergeTree
ORDER BY createTime
SETTINGS index_granularity = 8192
```







```
创建一个Typora序列号验证工具，输入序列号自动检测其有效性。工具需包含以下功能：1) 序列号有效性检测 2) 使用期限查询 3) 自动从云端更新可用序列号库 4) 一键复制有效序列号。使用Electron开发跨平台应用。CREATE TABLE team_study_data.team_pending_data_11
(
    `judgmentNumCount` Int64,
    `minuteTime` DateTime,
    `hourTime` DateTime,
    `dayDate` Date,
    `completeTime` DateTime,
    `completeDate` Date,
    `collectTime` DateTime,
    `createTime` DateTime,
    `cumulativeNumCount` Int64 DEFAULT 0
)
ENGINE = MergeTree
ORDER BY createTime
SETTINGS index_granularity = 8192
```

```
CREATE TABLE team_study_data.team_processed_data_2
(
    `dataId` Int64,
    `userId` Int32,
    `timeType` UInt8 DEFAULT 0,
    `dataWarnFlag` UInt8 DEFAULT 0,
    `minuteTime` DateTime,
    `hourTime` DateTime,
    `dayDate` Date,
    `completeTime` DateTime,
    `completeDate` Date,
    `judgmentTime` DateTime,
    `createTime` DateTime
)
ENGINE = MergeTree
ORDER BY createTime
SETTINGS index_granularity = 8192
```

