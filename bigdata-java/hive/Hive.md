# Hive

## Hive优缺点

#### 优点

- **操作接口采用类SQL语法**，提供快速开发的能力（简单、容易上手）。
- **避免了去写MapReduce**，减少开发人员的学习成本。
- **Hive支持用户自定义函数**，用户可以根据自己的需求来实现自己的函数。

#### 缺点

- **Hive 的查询延迟很严重**

- **Hive 不支持事务**

## HIVE体系架构

![Hive体系架构17](.\pic\Hive体系架构17.png)

### command-line shell & thrift/jdbc

可以用 command-line shell 和 thrift／jdbc 两种方式来操作数据：

- **command-line shell**：通过 hive 命令行的的方式来操作数据；

- **thrift／jdbc**：通过 thrift 协议按照标准的 JDBC 的方式操作数据。

### Metastore

在 Hive 中，表名、表结构、字段名、字段类型、表的分隔符等统一被称为元数据。所有的元数据默认存储在 Hive 内置的 derby 数据库中，但由于 derby 只能有一个实例，也就是说不能有多个命令行客户端同时访问，所以在实际生产环境中，通常使用 MySQL 代替 derby。

Hive 进行的是统一的元数据管理，就是说你在 Hive 上创建了一张表，然后在 presto／impala／sparksql 中都是可以直接使用的，它们会从 Metastore 中获取统一的元数据信息，同样的你在 presto／impala／sparksql 中创建一张表，在 Hive 中也可以直接使用。

### HQL的执行流程

Hive 在执行一条 HQL 的时候，会经过以下步骤：

1. 语法解析：Antlr 定义 SQL 的语法规则，完成 SQL 词法，语法解析，将 SQL 转化为抽象 语法树 AST Tree；

2. 语义解析：遍历 AST Tree，抽象出查询的基本组成单元 QueryBlock；

3. 生成逻辑执行计划：遍历 QueryBlock，翻译为执行操作树 OperatorTree；

4. 优化逻辑执行计划：逻辑层优化器进行 OperatorTree 变换，合并不必要的 ReduceSinkOperator，减少 shuffle 数据量；

5. 生成物理执行计划：遍历 OperatorTree，翻译为 MapReduce 任务；

6. 优化物理执行计划：物理层优化器进行 MapReduce 任务的变换，生成最终的执行计划。
    关于 Hive SQL 的详细执行流程可以参考美团技术团队的文章：[Hive SQL 的编译过程](https://tech.meituan.com/2014/02/12/hive-sql-to-mapreduce.html)

## 内部表与外部表

内部表又叫做管理表 (Managed/Internal Table)，创建表时不做任何指定，默认创建的就是内部表。想要创建外部表 (External Table)，则需要使用 External 进行修饰。 内部表和外部表主要区别如下：

|              | 内部表                                                       | 外部表                                                       |
| ------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 数据存储位置 | 内部表数据存储的位置由 hive.metastore.warehouse.dir 参数指定，默认情况下表的数据存储在 HDFS 的 `/user/hive/warehouse/数据库名.db/表名/` 目录下 | 外部表数据的存储位置创建表时由 `Location` 参数指定；         |
| 导入数据     | 在导入数据到内部表，内部表将数据移动到自己的数据仓库目录下，数据的生命周期由 Hive 来进行管理 | 外部表不会将数据移动到自己的数据仓库目录下，只是在元数据中存储了数据的位置 |
| 删除数据     | 删除元数据（metadata）和文件                                 | 只删除元数据（metadata）                                     |
|              | 一般作为数仓的ODS层                                          | 一般作为数仓的DW层                                           |

## Hive分区表与分桶表

#### [分区表](https://www.cnblogs.com/jimmy888/tag/Hive/default.html?page=1)



- 如果`hive`当中所有的数据都存入到一个文件夹下面，那么在使用`MR`计算程序的时候，读取一整个目录下面的所有文件来进行计算（全表扫描），就会变得特别慢，因为数据量太大了

- 实际工作当中一般都是计算前一天的数据，所以我们只需要将前一天的数据挑出来放到一个文件夹下面即可，专门去计算前一天的数据。

- 这样就可以使用`hive`当中的分区表，通过分文件夹的形式，将每一天的数据都分成为一个文件夹，然后我们计算数据的时候，通过指定前一天的文件夹即可只计算前一天的数据。

- 在大数据中，最常用的一种思想就是分治，我们可以把大的文件切割划分成一个个的小的文件，这样每次操作一个小的文件就会很容易了，同样的道理，在`hive`当中也是支持这种思想的，就是我们可以把大的数据，按照每天，或者每小时进行切分成一个个的小的文件，这样去操作小的文件就会容易得多了。

- 在文件系统上建立文件夹，把表的数据放在不同文件夹下面，加快查询速度。**示意图如下：**

![hive分区表4](.\pic\hive分区表4.png)

创建分区表并加载数据

```sql
CREATE EXTERNAL TABLE emp_partition(
    empno INT,
    ename STRING,
    job STRING,
    mgr INT,
    hiredate TIMESTAMP,
    sal DECIMAL(7,2),
    comm DECIMAL(7,2)
    )
    PARTITIONED BY (deptno INT)   -- 按照部门编号进行分区
    ROW FORMAT DELIMITED FIELDS TERMINATED BY "\t"
    LOCATION '/hive/emp_partition';
    
# 加载部门编号为20的数据到表中
LOAD DATA LOCAL INPATH "/usr/file/emp20.txt" OVERWRITE INTO TABLE emp_partition PARTITION (deptno=20)
# 加载部门编号为30的数据到表中
LOAD DATA LOCAL INPATH "/usr/file/emp30.txt" OVERWRITE INTO TABLE emp_partition PARTITION (deptno=30)
```

#### 分桶表

分区表是为了减少扫描量，提高效率，那分桶表是干嘛的？分桶表的作用也是一样，只不过会进一步会细化而已。

分桶将整个数据内容按照某列属性值去`hash`值进行区分，具有相同的`hash`值的结果的数据进入到同一个文件中。

![分桶表16](.\pic\分桶表16.png)

- 分桶是相对分区进行**更细粒**度的划分

  - `Hive`表或分区表可进一步的分桶
  - 分桶将整个数据内容按照某列取`hash`值，对桶的个数取模的方式决定该条记录（行）存放在哪个桶当中；具有相同`hash`值的数据进入到同一个文件中。
  - 比如按照`name`属性分为`3`个桶，就是对`name`属性值的`hash`值对`3`取摸，按照取模结果对数据分桶。
    - 取模结果为 **0** 的数据记录存放到一个文件
    - 取模结果为 **1** 的数据记录存放到一个文件
    - 取模结果为 **2** 的数据记录存放到一个文件

- 如果一个表既分区又分桶，则必须先分区再分桶。如下：

  ```sql
  CREATE TABLE user_info_bucketed(user_id BIGINT, firstname STRING, lastname STRING)
  COMMENT 'A bucketed copy of user_info'  
  PARTITIONED BY(ds STRING)
  CLUSTERED BY(user_id) INTO 256 BUCKETS;
  ```

  

