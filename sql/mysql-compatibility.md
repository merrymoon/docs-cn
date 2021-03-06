---
title: 与 MySQL 兼容性对比
category: compatibility
---

# 与 MySQL 兼容性对比

TiDB 支持包括跨行事务、JOIN 及子查询在内的绝大多数 MySQL 5.7 的语法，用户可以直接使用现有的 MySQL 客户端连接。如果现有的业务已经基于 MySQL 开发，大多数情况不需要修改代码即可直接替换单机的 MySQL。

包括现有的大多数 MySQL 运维工具（如 PHPMyAdmin, Navicat, MySQL Workbench 等），以及备份恢复工具（如 mysqldump, mydumper/myloader）等都可以直接使用。

不过一些特性由于在分布式环境下没法很好的实现，目前暂时不支持或者是表现与 MySQL 有差异。

一些 MySQL 语法在 TiDB 中可以解析通过，但是不会做任何后续的处理，例如 Create Table 语句中 `Engine` 以及 `Partition` 选项，都是解析并忽略。更多兼容性差异请参考具体的文档。


## 不支持的特性

* 存储过程
* 视图
* 触发器
* 自定义函数
* 外键约束
* 全文索引
* 空间索引
* 非 UTF8 字符集

## 与 MySQL 有差异的特性

### 自增 ID

TiDB 的自增 ID (Auto Increment ID) 只保证自增且唯一，并不保证连续分配。TiDB 目前采用批量分配的方式，所以如果在多台 TiDB 上同时插入数据，分配的自增 ID 会不连续。

> **注意：**
>
> 在集群中有多个 tidb-server 实例时，如果表结构中有自增 ID，建议不要混用缺省值和自定义值。否则在如下情况下会遇到问题：
> 
> 假设有这样一个带有自增 ID 的表：
>
> ```
> create table t(id int unique key auto_increment, c int);
> ```
> 
> TiDB 实现自增 ID 的原理是每个 tidb-server 实例缓存一段 ID 值用于分配（目前会缓存 30000 个 ID），用完这段值再去取下一段。

> 假设集群中有两个 tidb-server 实例 A 和 B（A 缓存 [1,30000] 的自增 ID，B 缓存 [30001,60000] 的自增 ID），
>
> 依次执行操作：
>
> 1. 客户端向 B 插入一条将 `id` 设置为 1 的语句 `insert into t values (1, 1)`，并执行成功。
> 2. 客户端向 A 发送 Insert 语句 `insert into t (c) (1)`，这条语句中没有指定 `id` 的值，所以会由 A 分配，当前 A 缓存了 [1, 30000] 这段 ID，所以会分配 1 为自增 ID 的值，并把本地计数器加 1。而此时数据库中已经存在 `id` 为 1 的数据，最终返回 `Duplicated Error` 错误。


### 内建函数

TiDB 支持常用的 MySQL 内建函数，但是不是所有的函数都已经支持，具体请参考[语法文档](https://pingcap.github.io/sqlgram/#FunctionCallKeyword)。

### DDL

TiDB 实现了 F1 的异步 Schema 变更算法，DDL 执行过程中不会阻塞线上的 DML 操作。目前已经支持的 DDL 包括：

+ Create Database
+ Drop Database
+ Create Table
+ Drop Table
+ Add Index
+ Drop Index
+ Add Column
+ Drop Column
+ Alter Column
+ Change Column
+ Modify Column
+ Truncate Table
+ Rename Table
+ Create Table Like

以上语句还有一些支持不完善的地方，具体包括如下：

+ Add/Drop primary key 操作目前不支持。
+ Add Index/Column 操作不支持同时创建多个索引或列。
+ Drop Column 操作不支持删除的列为主键列或索引列。
+ Add Column 操作不支持同时将新添加的列设为主键或唯一索引，也不支持将此列设成 auto_increment 属性。
+ Change/Modify Column 操作目前支持部分语法，细节如下：
    - 在修改类型方面，只支持整数类型之间修改，字符串类型之间修改和 Blob 类型之间的修改，且只能使原类型长度变长。此外，不能改变列的 unsigned/charset/collate 属性。这里的类型分类如下：
        * 具体支持的整型类型有：TinyInt，SmallInt，MediumInt，Int，BigInt。
        * 具体支持的字符串类型有：Char，Varchar，Text，TinyText，MediumText，LongText。
        * 具体支持的 Blob 类型有：Blob，TinyBlob，MediumBlob，LongBlob。
    - 在修改类型定义方面，支持的包括 default value，comment，null，not null 和 OnUpdate，但是不支持从 null 到 not null 的修改。
    - 支持 LOCK [=] {DEFAULT|NONE|SHARED|EXCLUSIVE} 语法，但是不做任何事情（pass through）。
    - 不支持对enum类型的列进行修改

### 事务
TiDB 使用乐观事务模型，在执行 Update、Insert、Delete 等语句时，只有在提交过程中才会检查写写冲突，而不是像 MySQL 一样使用行锁来避免写写冲突。所以业务端在执行 SQL 语句后，需要注意检查 commit 的返回值，即使执行时没有出错，commit的时候也可能会出错。

### Load data

+  语法：

    ```sql
    LOAD DATA LOCAL INFILE 'file_name' INTO TABLE table_name
        {FIELDS | COLUMNS} TERMINATED BY 'string' ENCLOSED BY 'char' ESCAPED BY 'char'
        LINES STARTING BY 'string' TERMINATED BY 'string'
        IGNORE n LINES
        (col_name ...);
    ```

    其中 ESCAPED BY 目前只支持 '/\/\'。

+   事务的处理：

    TiDB 在执行 load data 时，默认每 2 万行记录作为一个事务进行持久化存储。如果一次 load data 操作插入的数据超过 2 万行，那么会分为多个事务进行提交。如果某个事务出错，这个事务会提交失败，但它前面的事务仍然会提交成功，在这种情况下一次 load data 操作会有部分数据插入成功，部分数据插入失败。而 MySQL 中会将一次 load data 操作视为一个事务，如果其中发生失败情况，将会导致整个 load data 操作失败。

### 默认设置的区别

+ 默认字符集不同：
    + MySQL 5.7 中使用 `latin1`（MySQL 8.0 中使用 `UTF-8`）
    + TiDB 使用 `utf8mb4`
+ 默认排序规则不同：
    + MySQL 5.7 中使用 `latin1_swedish_ci`
    + TiDB 使用 `binary`
+ `lower_case_table_names` 的默认值不同：
    + TiDB 中该值默认为 2，并且目前 TiDB 只支持设置该值为 2
    + MySQL 中默认设置：
        + Linux 系统中该值为 0
        + Windows 系统中该值为 1
        + macOS 系统中该值为 2
