# 如何排查 Amazon RDS for MySQL 复制延迟高的问题？

*上次更新日期：2022 年 6 月 17 日*

**我正在尝试查明在使用 Amazon Relational Database Service (Amazon RDS) for MySQL 时副本滞后的原因。该如何操作？**

## 简短描述

Amazon RDS for MySQL 使用异步复制。这意味着副本有时无法与主数据库实例保持同步。因此，可能会出现复制滞后。

在您将 Amazon RDS for MySQL 只读副本用于基于二进制日志文件位置的复制时，您可以会监控到复制延迟。在 Amazon CloudWatch 中，检查 Amazon RDS 的 **ReplicaLag** 指标。**ReplicaLag** 指标可报告 **SHOW SLAVE STATUS** 命令的 **Seconds_Behind_Master** 字段的值。

**Seconds_Behind_Master** 字段显示副本数据库实例上的当前时间戳之间的差别。此外，还会显示主数据库实例上记录的副本数据库实例上正在处理的事件的原始时间戳。

MySQL 复制适用于三个线程：**Binlog Dump** 线程、IO_THREAD 和 SQL_THREAD。有关这些线程具体工作方式的更多信息，请参阅[复制实施细节](https://dev.mysql.com/doc/refman/5.6/en/replication-implementation-details.html)的 MySQL 部分。如果复制中存在延迟，请确定滞后是由副本 IO_THREAD 还是副本 SQL_THREAD 引起的。然后，您可以确定滞后的根本原因。

## 解决方法

要确定哪个复制线程发生了滞后，请参阅以下示例：

\1.  在主数据库实例上运行 **SHOW MASTER STATUS** 命令，然后查看输出：

```plainText
mysql> SHOW MASTER STATUS;
+----------------------------+----------+--------------+------------------+-------------------+
| File                       | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+----------------------------+----------+--------------+------------------+-------------------+
| mysql-bin-changelog.066552|      521 |              |                  |                   |
+----------------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```

**注意：**在示例输出中，源或主数据库实例将二进制日志写入到文件 **mysql-bin.066552**。

\2.  在副本数据库实例上运行 **SHOW SLAVE STATUS** 命令并查看输出：

**示例 1：**

```plainText
mysql> SHOW SLAVE STATUS\G;
*************************** 1. row ***************************
Master_Log_File: mysql-bin.066548
Read_Master_Log_Pos: 10050480
Relay_Master_Log_File: mysql-bin.066548
Exec_Master_Log_Pos: 10050300
Slave_IO_Running: Yes
Slave_SQL_Running: Yes
```

在示例 1 中，**Master_Log_File: mysql-bin.066548** 指示副本 **IO_THREAD** 正在从二进制日志文件 **mysql-bin.066548** 读取数据。主数据库实例正在将二进制日志写入 **mysql-bin.066552** 文件。此输出显示副本 **IO_THREAD** 滞后 4 个二进制日志。但是，**Relay_Master_Log_File** 是 **mysql-bin.066548**，这表示副本 **SQL_THREAD** 正在从与 **IO_THREAD** 相同的文件中读取数据。这意味着副本 **SQL_THREAD** 保持同步，但副本 **IO_THREAD** 出现滞后。

示例 2：

```plainText
mysql> SHOW SLAVE STATUS\G
*************************** 1. row ***************************
Master_Log_File: mysql-bin.066552
Read_Master_Log_Pos: 430
Relay_Master_Log_File: mysql-bin.066530
Exec_Master_Log_Pos: 50360
Slave_IO_Running: Yes
Slave_SQL_Running: Yes
```

示例 2 显示主实例的日志文件为 **mysql-bin-changelog.066552**。输出显示 IO_THREAD 目前与主数据库实例同步。在副本输出中，SQL 线程正在执行 **Relay_Master_Log_File: mysql-bin-changelog.066530**。因此，SQL_THREAD 滞后 22 个二进制日志。

通常，IO_THREAD 不会导致较高的复制延迟，因为 IO_THREAD 仅从主或源实例读取二进制日志。但是，网络连接和网络延迟会影响服务器之间的读取速度。IO_THREAD 副本可能会因高带宽使用率而执行较慢。

如果副本 SQL_THREAD 是复制延迟的原因，那么这些延迟可能是由于以下原因导致的：

- 主数据库实例上长时间运行的查询
- 数据库实例类大小或存储空间不足
- 在主数据库实例上执行的并行查询
- 同步到副本数据库实例上的磁盘的二进制日志
- 副本上的 Binlog_format 设置为 ROW
- 副本创建滞后

### 主实例上长时间运行的查询

在主数据库实例上长时间运行的查询需要花费相同的时间在副本数据库实例上运行，这会增加 **seconds_behind_master**。例如，如果您在主实例上发起更改且这需要运行一个小时，则滞后为一个小时。由于更改可能还需要一个小时才能在副本上完成，因此在更改完成时，总滞后大约为两小时。这种延迟在预料之中，但您可以通过监控主实例上的缓慢查询日志来尽可能减少这种滞后。您还可以通过识别长时间运行的语句来减少滞后。然后，将长时间运行的语句分解为多个较小的语句或事务。有关更多信息，请参阅 [RDS for MySQL 慢速查询日志和常规日志](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_LogAccess.MySQL.LogFileSize.html#USER_LogAccess.MySQL.Generallog)。

### 数据库实例类大小或存储空间不足

如果副本数据库实例类或存储配置小于主实例，则副本可能因资源不足而受到限制。副本将无法与主实例上所做的更改同步。确保副本的数据库实例类型与主数据库实例相同或级别更高。为保证复制有效运行，每个只读副本需要与源数据库实例相同数量的计算和存储资源。有关更多信息，请参阅[数据库实例类](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.DBInstanceClass.html)。

### 在主数据库实例上执行的并行查询

如果您在主实例上执行并行查询，则它们将以串行顺序在副本上提交。这是因为默认情况下，MySQL 复制是单线程 (SQL_THREAD)。如果并行执行对源数据库实例的大量写入，则应使用单个 SQL_THREAD 序列化对只读副本的写入。这可能会导致源数据库实例和只读副本之间的滞后。

多线程（并行）复制适用于 MySQL 5.6、MySQL 5.7 及更高版本。有关多线程复制的更多信息，请参阅 [二进制日志记录选项和变量](https://dev.mysql.com/doc/refman/5.7/en/replication-options-binary-log.html)的 MySQL 文档部分。

多线程复制会导致复制中出现差距。例如，在跳过复制错误时，因为很难识别跳过了哪些事务，所以多线程复制并非最佳实践。这可能会导致主数据库实例和副本数据库实例之间的数据一致性存在差距。

### 同步到副本数据库实例上的磁盘的二进制日志

在副本上启用自动备份可能会导致需要额外操作才能将二进制日志同步到副本上的磁盘。**sync_binlog** 参数的默认值设置为 **1**。如果将此值更改为 **0**，则您将禁用 MySQL 服务器将二进制日志同步到磁盘的功能。操作系统 (OS) 偶尔会将二进制日志刷入磁盘，而非直接将日志记录到磁盘。

禁用二进制日志同步可以减少在每次提交时将二进制日志同步到磁盘所需的性能开销。然而，如果出现电源故障或操作系统崩溃，则某些提交可能无法同步到二进制日志。此异步可能会影响时间点恢复 (PITR) 功能。有关更多信息，请参阅 [sync_binlog](https://dev.mysql.com/doc/refman/5.7/en/replication-options-binary-log.html#sysvar_sync_binlog) 的 MySQL 文档部分。

### 将 Binlog_format 设为 ROW

如果将主数据库实例上的 **binlog_format** 设为 ROW，且源表缺失主键，则 SQL 线程将在副本上执行全表扫描。这是因为**参数 slave_rows_search_algorithms** 的默认值为 TABLE_SCAN,INDEX_SCAN。要短期解决此问题，请将搜索算法更改为 INDEX_SCAN,HASH_SCAN，以减少全表扫描的开销。如果要长期解决此问题，最佳做法是向每个表添加显式主键。

有关 slave-rows-search-algorithms 参数的更多信息，请参阅 [slave-rows-search-algorithms](https://dev.mysql.com/doc/refman/5.7/en/replication-options-replica.html#sysvar_slave_rows_search_algorithms) 的 MySQL 文档部分。

### 副本创建滞后

Amazon RDS 通过拍摄数据库快照来创建 MySQL 主实例的只读副本。然后，Amazon RDS 会恢复快照以创建新的数据库实例（副本）并在两者之间建立复制关系。

Amazon RDS 需要花时间来创建新的只读副本。建立复制关系后，会出现相当于创建主实例备份所需时长的滞后。要尽可能减少此滞后，请在调用副本创建之前创建手动备份。然后，副本创建过程生成的快照属于增量备份，速度更快。

从快照恢复只读副本时，副本不会等待所有要从源传输的所有数据。副本数据库实例可用于执行数据库操作。系统将会在后台从现有 Amazon Elastic Block Store (Amazon EBS) 快照负载创建新卷。

**注意：**对于 Amazon RDS for MySQL 副本（基于 EBS 的卷），最初副本滞后可能会增加，因为延迟加载效应可能会影响复制性能。

考虑[启用 InnoDB 缓存预热功能](https://aws.amazon.com/blogs/aws/rds-mysql-cache-warming/)，该功能可通过保存主数据库实例缓存池的当前状态来提高性能。然后，尝试在恢复的只读副本上重新加载缓存池。有关 InnoDB 缓存预热的更多信息，请参阅 [Amazon RDS 上的 MySQL](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_MySQL.html)。