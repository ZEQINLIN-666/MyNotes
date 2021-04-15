**MySQL 的 Binlog 日志是一种二进制格式的日志，Binlog 记录所有的 DDL 和 DML 语句(除了数据查询语句SELECT、SHOW等)，以 Event 的形式记录，同时记录语句执行时间。**

Binlog 的主要作用有两个：

1. 数据恢复

   因为 Binlog 详细记录了所有修改数据的 SQL，当某一时刻的数据误操作而导致出问题，或者数据库宕机数据丢失，那么可以根据 Binlog 来回放历史数据。

2. 主从复制

   想要做多机备份的业务，可以去监听当前写库的 Binlog 日志，同步写库的所有更改。

Binlog 包括两类文件：

- 二进制日志索引文件(.index)：记录所有的二进制文件。
- 二进制日志文件(.00000*)：记录所有 DDL 和 DML 语句事件。

Binlog 日志功能默认是开启的，线上情况下 Binlog 日志的增长速度是很快的，在 MySQL 的配置文件 `my.cnf` 中提供一些参数来对 Binlog 进行设置。

```mysql
Copy设置此参数表示启用binlog功能，并制定二进制日志的存储目录
log-bin=/home/mysql/binlog/

#mysql-bin.*日志文件最大字节（单位：字节）
#设置最大100MB
max_binlog_size=104857600

#设置了只保留7天BINLOG（单位：天）
expire_logs_days = 7

#binlog日志只记录指定库的更新
#binlog-do-db=db_name

#binlog日志不记录指定库的更新
#binlog-ignore-db=db_name

#写缓冲多少次，刷一次磁盘，默认0
sync_binlog=0
```

需要注意的是：

**max_binlog_size** ：Binlog 最大和默认值是 1G，该设置并不能严格控制 Binlog 的大小，尤其是 Binlog 比较靠近最大值而又遇到一个比较大事务时，为了保证事务的完整性不可能做切换日志的动作，只能将该事务的所有 SQL 都记录进当前日志直到事务结束。所以真实文件有时候会大于 max_binlog_size 设定值。
**expire_logs_days** ：Binlog 过期删除不是服务定时执行，是需要借助事件触发才执行，事件包括：

- 服务器重启
- 服务器被更新
- 日志达到了最大日志长度 `max_binlog_size`
- 日志被刷新

二进制日志由配置文件的 `log-bin` 选项负责启用，MySQL 服务器将在数据根目录创建两个新文件`mysql-bin.000001` 和 `mysql-bin.index`，若配置选项没有给出文件名，MySQL 将使用主机名称命名这两个文件，其中 `.index` 文件包含一份全体日志文件的清单。

**sync_binlog**：这个参数决定了 Binlog 日志的更新频率。默认 0 ，表示该操作由操作系统根据自身负载自行决定多久写一次磁盘。

sync_binlog = 1 表示每一条事务提交都会立刻写盘。sync_binlog=n 表示 n 个事务提交才会写盘。

根据 MySQL 文档，写 Binlog 的时机是：SQL transaction 执行完，但任何相关的 Locks 还未释放或事务还未最终 commit 前。这样保证了 Binlog 记录的操作时序与数据库实际的数据变更顺序一致。

检查 Binlog 文件是否已开启：

```mysql
Copymysql> show variables like '%log_bin%';
+---------------------------------+------------------------------------+
| Variable_name                   | Value                              |
+---------------------------------+------------------------------------+
| log_bin                         | ON                                 |
| log_bin_basename                | /usr/local/mysql/data/binlog       |
| log_bin_index                   | /usr/local/mysql/data/binlog.index |
| log_bin_trust_function_creators | OFF                                |
| log_bin_use_v1_row_events       | OFF                                |
| sql_log_bin                     | ON                                 |
+---------------------------------+------------------------------------+
6 rows in set (0.00 sec)
```

MySQL 会把用户对所有数据库的内容和结构的修改情况记入 `mysql-bin.n` 文件，而不会记录 SELECT 和没有实际更新的 UPDATE 语句。

如果你不知道现在有哪些 Binlog 文件，可以使用如下命令：

```mysql
Copyshow binary logs; #查看binlog列表
show master status; #查看最新的binlog

mysql> show binary logs;
+------------------+-----------+-----------+
| Log_name         | File_size | Encrypted |
+------------------+-----------+-----------+
| mysql-bin.000001 |       179 | No        |
| mysql-bin.000002 |       156 | No        |
+------------------+-----------+-----------+
2 rows in set (0.00 sec)
```

Binlog 文件是二进制文件，强行打开看到的必然是乱码，MySQL 提供了命令行的方式来展示 Binlog 日志：

```shell
Copymysqlbinlog mysql-bin.000002 | more
```

`mysqlbinlog` 命令即可查看。

![1](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjusawxe99j31ne0u07kb.jpg)

看起来凌乱其实也有迹可循。Binlog 通过事件的方式来管理日志信息，可以通过 `show binlog events in` 的语法来查看当前 Binlog 文件对应的详细事件信息。

```mysql
Copymysql> show binlog events in 'mysql-bin.000001';
+------------------+-----+----------------+-----------+-------------+-----------------------------------+
| Log_name         | Pos | Event_type     | Server_id | End_log_pos | Info                              |
+------------------+-----+----------------+-----------+-------------+-----------------------------------+
| mysql-bin.000001 |   4 | Format_desc    |         1 |         125 | Server ver: 8.0.21, Binlog ver: 4 |
| mysql-bin.000001 | 125 | Previous_gtids |         1 |         156 |                                   |
| mysql-bin.000001 | 156 | Stop           |         1 |         179 |                                   |
+------------------+-----+----------------+-----------+-------------+-----------------------------------+
3 rows in set (0.01 sec)
```

这是一份没有任何写入数据的 Binlog 日志文件。

Binlog 的版本是V4，可以看到日志的结束时间为 Stop。出现 Stop event 有两种情况：

1. 是 master shut down 的时候会在 Binlog 文件结尾出现
2. 是备机在关闭的时候会写入 relay log 结尾，或者执行 RESET SLAVE 命令执行

本文出现的原因是我有手动停止过 MySQL 服务。

一般来说一份正常的 Binlog 日志文件会以 **Rotate event** 结束。当 Binlog 文件超过指定大小，Rotate event 会写在文件最后，指向下一个 Binlog 文件。

我们来看看有过数据操作的 Binlog 日志文件是什么样子的。

```mysql
Copymysql> show binlog events in 'mysql-bin.000002';
+------------------+-----+----------------+-----------+-------------+-----------------------------------+
| Log_name         | Pos | Event_type     | Server_id | End_log_pos | Info                              |
+------------------+-----+----------------+-----------+-------------+-----------------------------------+
| mysql-bin.000002 |   4 | Format_desc    |         1 |         125 | Server ver: 8.0.21, Binlog ver: 4 |
| mysql-bin.000002 | 125 | Previous_gtids |         1 |         156 |                                   |
+------------------+-----+----------------+-----------+-------------+-----------------------------------+
2 rows in set (0.00 sec)
```

上面是没有任何数据操作且没有被截断的 Binlog。接下来我们插入一条数据，再看看 Binlog 事件。

```mysql
Copymysql> show binlog events in 'mysql-bin.000002';
+------------------+-----+----------------+-----------+-------------+-------------------------------------------------------------------------+
| Log_name         | Pos | Event_type     | Server_id | End_log_pos | Info                                                                    |
+------------------+-----+----------------+-----------+-------------+-------------------------------------------------------------------------+
| mysql-bin.000002 |   4 | Format_desc    |         1 |         125 | Server ver: 8.0.21, Binlog ver: 4                                       |
| mysql-bin.000002 | 125 | Previous_gtids |         1 |         156 |                                                                         |
| mysql-bin.000002 | 156 | Anonymous_Gtid |         1 |         235 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'                                    |
| mysql-bin.000002 | 235 | Query          |         1 |         323 | BEGIN                                                                   |
| mysql-bin.000002 | 323 | Intvar         |         1 |         355 | INSERT_ID=13                                                            |
| mysql-bin.000002 | 355 | Query          |         1 |         494 | use `test_db`; INSERT INTO `test_db`.`test_db`(`name`) VALUES ('xdfdf') |
| mysql-bin.000002 | 494 | Xid            |         1 |         525 | COMMIT /* xid=192 */                                                    |
+------------------+-----+----------------+-----------+-------------+-------------------------------------------------------------------------+
7 rows in set (0.00 sec)
```

这是加入一条数据之后的 Binlog 事件。

我们对 event 查询的数据行关键字段来解释一下：

- Pos：当前事件的开始位置，每个事件都占用固定的字节大小，结束位置(End_log_position)减去Pos，就是这个事件占用的字节数。

  上面的日志中我们能看到，第一个事件位置并不是从 0 开始，而是从 4。MySQL 通过文件中的前 4 个字节，来判断这是不是一个 Binlog 文件。这种方式很常见，很多格式的文件，如 pdf、doc、jpg等，都会通常前几个特定字符判断是否是合法文件。

- Event_type：表示事件的类型

- Server_id：表示产生这个事件的 MySQL server_id，通过设置 my.cnf 中的 **server-id** 选项进行配置

- End_log_position：下一个事件的开始位置

- Info：包含事件的具体信息

#### Binlog 日志格式[#](https://www.cnblogs.com/rickiyang/p/13841811.html#2968765751)

针对不同的使用场景，Binlog 也提供了可定制化的服务，提供了三种模式来提供不同详细程度的日志内容。

- Statement 模式：基于 SQL 语句的复制(statement-based replication-SBR)
- Row 模式：基于行的复制(row-based replication-RBR)
- Mixed 模式：混合模式复制(mixed-based replication-MBR)

##### Statement 模式

保存每一条修改数据的SQL。

该模式只保存一条普通的SQL语句，不涉及到执行的上下文信息。

因为每台 MySQL 数据库的本地环境可能不一样，那么对于依赖到本地环境的函数或者上下文处理的逻辑 SQL 去处理的时候可能同样的语句在不同的机器上执行出来的效果不一致。

比如像 `sleep()`函数，`last_insert_id()`函数，等等，这些都跟特定时间的本地环境有关。

##### Row 模式

MySQL V5.1.5 版本开始支持Row模式的 Binlog，它与 Statement 模式的区别在于它不保存具体的 SQL 语句，而是记录具体被修改的信息。

比如一条 update 语句更新10条数据，如果是 Statement 模式那就保存一条 SQL 就够，但是 Row 模式会保存每一行分别更新了什么，有10条数据。

Row 模式的优缺点就很明显了。保存每一个更改的详细信息必然会带来存储空间的快速膨胀，换来的是事件操作的详细记录。所以要求越高代价越高。

##### Mixed 模式

Mixed 模式即以上两种模式的综合体。既然上面两种模式分别走了极简和一丝不苟的极端，那是否可以区分使用场景的情况下将这两种模式综合起来呢？

在 Mixed 模式中，一般的更新语句使用 Statement 模式来保存 Binlog，但是遇到一些函数操作，可能会影响数据准确性的操作则使用 Row 模式来保存。这种方式需要根据每一条具体的 SQL 语句来区分选择哪种模式。

MySQL 从 V5.1.8 开始提供 Mixed 模式，V5.7.7 之前的版本默认是Statement 模式，之后默认使用Row模式， 但是在 8.0 以上版本已经默认使用 Mixed 模式了。

查询当前 Binlog 日志使用格式：

```mysql
Copymysql> show global variables like '%binlog_format%';
+---------------------------------+---------+
| Variable_name                   | Value   |
+---------------------------------+---------+
| binlog_format                   | MIXED   |
| default_week_format             | 0       |
| information_schema_stats_expiry | 86400   |
| innodb_default_row_format       | dynamic |
| require_row_format              | OFF     |
+---------------------------------+---------+
5 rows in set (0.01 sec)
```

#### 如何通过 `mysqlbinlog` 命令手动恢复数据[#](https://www.cnblogs.com/rickiyang/p/13841811.html#2939530574)

上面说过每一条 event 都有位点信息，如果我们当前的 MySQL 库被无操作或者误删除了，那么该如何通过 Binlog 来恢复到删除之前的数据状态呢？

首先发现误操作之后，先停止 MySQL 服务，防止继续更新。

接着通过 `mysqlbinlog`命令对二进制文件进行分析，查看误操作之前的位点信息在哪里。

接下来肯定就是恢复数据，当前数据库的数据已经是错的，那么就从开始位置到误操作之前位点的数据肯定的都是正确的；如果误操作之后也有正常的数据进来，这一段时间的位点数据也要备份。

比如说：

误操作的位点开始值为 501，误操作结束的位置为705，之后到800的位点都是正确数据。

那么从 0 - 500 ，706 - 800 都是有效数据，接着我们就可以进行数据恢复了。

先将数据库备份并清空。

接着使用 `mysqlbinlog` 来恢复数据：

0 - 500 的数据：

```mysql
Copymysqlbinlog --start-position=0  --stop-position=500  bin-log.000003 > /root/back.sql;
```

上面命令的作用就是将 0 -500 位点的数据恢复到自定义的 SQL 文件中。同理 706 - 800 的数据也是一样操作。之后我们执行这两个 SQL 文件就行了。

#### Binlog 事件类型[#](https://www.cnblogs.com/rickiyang/p/13841811.html#2294710563)

上面我们说到了 Binlog 日志中的事件，不同的操作会对应着不同的事件类型，且不同的 Binlog 日志模式同一个操作的事件类型也不同，下面我们一起看看常见的事件类型。

首先我们看看源码中的事件类型定义：

源码位置：/libbinlogevents/include/binlog_event.h

```c++
Copyenum Log_event_type
{
  /**
    Every time you update this enum (when you add a type), you have to
    fix Format_description_event::Format_description_event().
  */
  UNKNOWN_EVENT= 0,
  START_EVENT_V3= 1,
  QUERY_EVENT= 2,
  STOP_EVENT= 3,
  ROTATE_EVENT= 4,
  INTVAR_EVENT= 5,
  LOAD_EVENT= 6,
  SLAVE_EVENT= 7,
  CREATE_FILE_EVENT= 8,
  APPEND_BLOCK_EVENT= 9,
  EXEC_LOAD_EVENT= 10,
  DELETE_FILE_EVENT= 11,
  /**
    NEW_LOAD_EVENT is like LOAD_EVENT except that it has a longer
    sql_ex, allowing multibyte TERMINATED BY etc; both types share the
    same class (Load_event)
  */
  NEW_LOAD_EVENT= 12,
  RAND_EVENT= 13,
  USER_VAR_EVENT= 14,
  FORMAT_DESCRIPTION_EVENT= 15,
  XID_EVENT= 16,
  BEGIN_LOAD_QUERY_EVENT= 17,
  EXECUTE_LOAD_QUERY_EVENT= 18,

  TABLE_MAP_EVENT = 19,

  /**
    The PRE_GA event numbers were used for 5.1.0 to 5.1.15 and are
    therefore obsolete.
   */
  PRE_GA_WRITE_ROWS_EVENT = 20,
  PRE_GA_UPDATE_ROWS_EVENT = 21,
  PRE_GA_DELETE_ROWS_EVENT = 22,

  /**
    The V1 event numbers are used from 5.1.16 until mysql-trunk-xx
  */
  WRITE_ROWS_EVENT_V1 = 23,
  UPDATE_ROWS_EVENT_V1 = 24,
  DELETE_ROWS_EVENT_V1 = 25,

  /**
    Something out of the ordinary happened on the master
   */
  INCIDENT_EVENT= 26,

  /**
    Heartbeat event to be send by master at its idle time
    to ensure master's online status to slave
  */
  HEARTBEAT_LOG_EVENT= 27,

  /**
    In some situations, it is necessary to send over ignorable
    data to the slave: data that a slave can handle in case there
    is code for handling it, but which can be ignored if it is not
    recognized.
  */
  IGNORABLE_LOG_EVENT= 28,
  ROWS_QUERY_LOG_EVENT= 29,

  /** Version 2 of the Row events */
  WRITE_ROWS_EVENT = 30,
  UPDATE_ROWS_EVENT = 31,
  DELETE_ROWS_EVENT = 32,

  GTID_LOG_EVENT= 33,
  ANONYMOUS_GTID_LOG_EVENT= 34,

  PREVIOUS_GTIDS_LOG_EVENT= 35,

  TRANSACTION_CONTEXT_EVENT= 36,

  VIEW_CHANGE_EVENT= 37,

  /* Prepared XA transaction terminal event similar to Xid */
  XA_PREPARE_LOG_EVENT= 38,
  /**
    Add new events here - right above this comment!
    Existing events (except ENUM_END_EVENT) should never change their numbers
  */
  ENUM_END_EVENT /* end marker */
};
```

这么多的事件类型我们就不一一介绍，挑出来一些常用的来看看。

**FORMAT_DESCRIPTION_EVENT**

FORMAT_DESCRIPTION_EVENT 是 Binlog V4 中为了取代之前版本中的 START_EVENT_V3 事件而引入的。它是 Binlog 文件中的第一个事件，而且，该事件只会在 Binlog 中出现一次。MySQL 根据 FORMAT_DESCRIPTION_EVENT 的定义来解析其它事件。

它通常指定了 MySQL 的版本，Binlog 的版本，该 Binlog 文件的创建时间。

**QUERY_EVENT**

QUERY_EVENT 类型的事件通常在以下几种情况下使用：

- 事务开始时，执行的 BEGIN 操作
- STATEMENT 格式中的 DML 操作
- ROW 格式中的 DDL 操作

比如上文我们插入一条数据之后的 Binlog 日志：

```mysql
Copymysql> show binlog events in 'mysql-bin.000002';
+------------------+-----+----------------+-----------+-------------+-------------------------------------------------------------------------+
| Log_name         | Pos | Event_type     | Server_id | End_log_pos | Info                                                                    |
+------------------+-----+----------------+-----------+-------------+-------------------------------------------------------------------------+
| mysql-bin.000002 |   4 | Format_desc    |         1 |         125 | Server ver: 8.0.21, Binlog ver: 4                                       |
| mysql-bin.000002 | 125 | Previous_gtids |         1 |         156 |                                                                         |
| mysql-bin.000002 | 156 | Anonymous_Gtid |         1 |         235 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'                                    |
| mysql-bin.000002 | 235 | Query          |         1 |         323 | BEGIN                                                                   |
| mysql-bin.000002 | 323 | Intvar         |         1 |         355 | INSERT_ID=13                                                            |
| mysql-bin.000002 | 355 | Query          |         1 |         494 | use `test_db`; INSERT INTO `test_db`.`test_db`(`name`) VALUES ('xdfdf') |
| mysql-bin.000002 | 494 | Xid            |         1 |         525 | COMMIT /* xid=192 */                                                    |
+------------------+-----+----------------+-----------+-------------+-------------------------------------------------------------------------+
7 rows in set (0.00 sec)
```

**XID_EVENT**

在事务提交时，不管是 STATEMENT 还 是ROW 格式的 Binlog，都会在末尾添加一个 XID_EVENT 事件代表事务的结束。该事件记录了该事务的 ID，在 MySQL 进行崩溃恢复时，根据事务在 Binlog 中的提交情况来决定是否提交存储引擎中状态为 prepared 的事务。

**ROWS_EVENT**

对于 ROW 格式的 Binlog，所有的 DML 语句都是记录在 ROWS_EVENT 中。

ROWS_EVENT分为三种：

- WRITE_ROWS_EVENT
- UPDATE_ROWS_EVENT
- DELETE_ROWS_EVENT

分别对应 insert，update 和 delete 操作。

对于 insert 操作，WRITE_ROWS_EVENT 包含了要插入的数据。

对于 update 操作，UPDATE_ROWS_EVENT 不仅包含了修改后的数据，还包含了修改前的值。

对于 delete 操作，仅仅需要指定删除的主键（在没有主键的情况下，会给定所有列）。

**对比 QUERY_EVENT 事件，是以文本形式记录 DML 操作的。而对于 ROWS_EVENT 事件，并不是文本形式，所以在通过 mysqlbinlog 查看基于 ROW 格式的 Binlog 时，需要指定 `-vv --base64-output=decode-rows`。**

我们来测试一下，首先将日志格式改为 Rows：

```mysql
Copymysql> set binlog_format=row;
Query OK, 0 rows affected (0.00 sec)

mysql>
mysql> flush logs;
Query OK, 0 rows affected (0.01 sec)
```

然后刷新一下日志文件，重新开始一个 Binlog 日志。我们插入一条数据之后看一下日志：

```mysql
Copymysql> show binlog events in 'binlog.000008';
+---------------+-----+----------------+-----------+-------------+--------------------------------------+
| Log_name      | Pos | Event_type     | Server_id | End_log_pos | Info                                 |
+---------------+-----+----------------+-----------+-------------+--------------------------------------+
| binlog.000008 |   4 | Format_desc    |         1 |         125 | Server ver: 8.0.21, Binlog ver: 4    |
| binlog.000008 | 125 | Previous_gtids |         1 |         156 |                                      |
| binlog.000008 | 156 | Anonymous_Gtid |         1 |         235 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS' |
| binlog.000008 | 235 | Query          |         1 |         313 | BEGIN                                |
| binlog.000008 | 313 | Table_map      |         1 |         377 | table_id: 85 (test_db.test_db)       |
| binlog.000008 | 377 | Write_rows     |         1 |         423 | table_id: 85 flags: STMT_END_F       |
| binlog.000008 | 423 | Xid            |         1 |         454 | COMMIT /* xid=44 */                  |
+---------------+-----+----------------+-----------+-------------+--------------------------------------+
7 rows in set (0.01 sec)
```

#### 总结[#](https://www.cnblogs.com/rickiyang/p/13841811.html#370791871)

这一篇我们详解了解 Binlog 日志是什么，里面都有什么内容，Binlog 事件，如何通过 Binlog 来恢复数据。Binlog 目前最重要的应用就是用于主从同步，那么下一篇我们讲来讲讲如何通过 Binlog 实现主从同步。