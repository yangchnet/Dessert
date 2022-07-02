---
author: "李昌"
title: "概览MySQL篇三：主备与高可用"
date: "2022-03-23"
tags: ["MySQL"]
categories: ["数据库"]
ShowToc: true
TocOpen: true
---

> 极客时间[《MySQL实战45讲》](https://time.geekbang.org/column/intro/100020801?tab=catalog)笔记

### 主备的基本原理

![20220614154547](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20220614154547.png)

在状态 1 中，客户端的读写都直接访问节点 A，而节点 B 是 A 的备库，只是将 A 的更新都同步过来，到本地执行。这样可以保持节点 B 和 A 的数据是相同的。

当需要切换的时候，就切成状态 2。这时候客户端读写访问的都是节点 B，而节点 A 是 B 的备库。

那么从状态1切换到状态2的内部流程是什么样的？

![20220614154646](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20220614154646.png)

备库 B 跟主库 A 之间维持了一个长连接。主库 A 内部有一个线程，专门用于服务备库 B 的这个长连接。一个事务日志同步的完整过程是这样的：

1. 在备库 B 上通过 change master 命令，设置主库 A 的 IP、端口、用户名、密码，以及要从哪个位置开始请求 binlog，这个位置包含文件名和日志偏移量。
2. 在备库 B 上执行 start slave 命令，这时候备库会启动两个线程，就是图中的 io_thread 和 sql_thread。其中 io_thread 负责与主库建立连接。
3. 主库 A 校验完用户名、密码后，开始按照备库 B 传过来的位置，从本地读取 binlog，发给 B。
4. 备库 B 拿到 binlog 后，写到本地文件，称为中转日志（relay log）。
5. sql_thread 读取中转日志，解析出日志里的命令，并执行。

#### binlog 的三种格式对比

主备复制依赖于bin log，那么bin log中是什么内容。

bin log的三种格式：
- statement
- row
- mixed

对于下表：
```sql
mysql> CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `a` int(11) DEFAULT NULL,
  `t_modified` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  KEY `a` (`a`),
  KEY `t_modified`(`t_modified`)
) ENGINE=InnoDB;

insert into t values(1,1,'2018-11-13');
insert into t values(2,2,'2018-11-12');
insert into t values(3,3,'2018-11-11');
insert into t values(4,4,'2018-11-10');
insert into t values(5,5,'2018-11-09');
```

如果要在表中删除一行数据的话，我们来看看这个 delete 语句的 binlog 是怎么记录的：
```sql

mysql> delete from t /*comment*/  where a>=4 and t_modified<='2018-11-10' limit 1;
```

当`binlog_format=statement`时，binlog 里面记录的就是 SQL 语句的原文

```sql
mysql> show binlog events in 'binlog.000002';
+---------------+-----+----------------+-----------+-------------+------------------------------------------------------------------------------------------+
| Log_name      | Pos | Event_type     | Server_id | End_log_pos | Info                                                                                     |
+---------------+-----+----------------+-----------+-------------+------------------------------------------------------------------------------------------+
| binlog.000002 |   4 | Format_desc    |         1 |         126 | Server ver: 8.0.28, Binlog ver: 4                                                        |
| binlog.000002 | 126 | Previous_gtids |         1 |         157 |                                                                                          |
| binlog.000002 | 157 | Anonymous_Gtid |         1 |         236 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'                                                     |
| binlog.000002 | 236 | Query          |         1 |         332 | BEGIN                                                                                    |
| binlog.000002 | 332 | Query          |         1 |         496 | use `test_db`; delete from t /*comment*/ where a>=4 and t_modified<='2018-11-10' limit 1 |
| binlog.000002 | 496 | Xid            |         1 |         527 | COMMIT /* xid=12 */                                                                      |
+---------------+-----+----------------+-----------+-------------+------------------------------------------------------------------------------------------+

```

当使用steaement格式日志时，可能会存在主备数据不一致的情况，例如：
1. 如果 delete 语句使用的是索引 a，那么会根据索引 a 找到第一个满足条件的行，也就是说删除的是 a=4 这一行；
2. 但如果使用的是索引 t_modified，那么删除的就是 t_modified='2018-11-09’也就是 a=5 这一行。

由于 statement 格式下，记录到 binlog 里的是语句原文，因此可能会出现这样一种情况：在主库执行这条 SQL 语句的时候，用的是索引 a；而在备库执行这条 SQL 语句的时候，却使用了索引 t_modified。因此，MySQL 认为这样写是有风险的。


当binlog_format=‘row’时：
```sql
mysql> show binlog events in 'binlog.000002';
+---------------+-----+----------------+-----------+-------------+--------------------------------------+
| Log_name      | Pos | Event_type     | Server_id | End_log_pos | Info                                 |
+---------------+-----+----------------+-----------+-------------+--------------------------------------+
| binlog.000002 |   4 | Format_desc    |         1 |         126 | Server ver: 8.0.28, Binlog ver: 4    |
| binlog.000002 | 126 | Previous_gtids |         1 |         157 |                                      |
| binlog.000002 | 157 | Anonymous_Gtid |         1 |         236 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS' |
| binlog.000002 | 236 | Query          |         1 |         322 | BEGIN                                |
| binlog.000002 | 322 | Table_map      |         1 |         375 | table_id: 88 (test_db.t)             |
| binlog.000002 | 375 | Delete_rows    |         1 |         423 | table_id: 88 flags: STMT_END_F       |
| binlog.000002 | 423 | Xid            |         1 |         454 | COMMIT /* xid=8 */                   |
+---------------+-----+----------------+-----------+-------------+--------------------------------------+
```

与 statement 格式的 binlog 相比，前后的 BEGIN 和 COMMIT 是一样的。但是，row 格式的 binlog 里没有了 SQL 语句的原文，而是替换成了两个 event：Table_map 和 Delete_rows。
- Table_map event，用于说明接下来要操作的表是 test 库的表 t;
- Delete_rows event，用于定义删除的行为。

但这里我们看不到详细信息，需要借助`mysqlbinlog`工具。用下面这个命令解析和查看 binlog 中的内容。

```
mysqlbinlog  -vv data/master.000001 --start-position=8900;
```

```
# The proper term is pseudo_replica_mode, but we use this compatibility alias
# to make the statement usable on server versions 8.0.24 and older.
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=1*/;
/*!50003 SET @OLD_COMPLETION_TYPE=@@COMPLETION_TYPE,COMPLETION_TYPE=0*/;
DELIMITER /*!*/;
# at 126
#220615  6:51:35 server id 1  end_log_pos 126 CRC32 0xc895f13c  Start: binlog v 4, server v 8.0.28 created 220615  6:51:35 at startup
# Warning: this binlog is either in use or was not closed properly.
ROLLBACK/*!*/;
BINLOG '
d4GpYg8BAAAAegAAAH4AAAABAAQAOC4wLjI4AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAB3galiEwANAAgAAAAABAAEAAAAYgAEGggAAAAICAgCAAAACgoKKioAEjQA
CigAATzxlcg=
'/*!*/;
# at 157
#220615  6:52:14 server id 1  end_log_pos 236 CRC32 0x245f18b1  Anonymous_GTID  last_committed=0        sequence_number=1       rbr_only=yes    original_committed_timestamp=1655275934841953immediate_commit_timestamp=1655275934841953     transaction_length=297
/*!50718 SET TRANSACTION ISOLATION LEVEL READ COMMITTED*//*!*/;
# original_commit_timestamp=1655275934841953 (2022-06-15 06:52:14.841953 UTC)
# immediate_commit_timestamp=1655275934841953 (2022-06-15 06:52:14.841953 UTC)
/*!80001 SET @@session.original_commit_timestamp=1655275934841953*//*!*/;
/*!80014 SET @@session.original_server_version=80028*//*!*/;
/*!80014 SET @@session.immediate_server_version=80028*//*!*/;
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 236
#220615  6:52:14 server id 1  end_log_pos 322 CRC32 0xfa351183  Query   thread_id=8     exec_time=0     error_code=0
SET TIMESTAMP=1655275934/*!*/;
SET @@session.pseudo_thread_id=8/*!*/;
SET @@session.foreign_key_checks=1, @@session.sql_auto_is_null=0, @@session.unique_checks=1, @@session.autocommit=1/*!*/;
SET @@session.sql_mode=1168113696/*!*/;
SET @@session.auto_increment_increment=1, @@session.auto_increment_offset=1/*!*/;
/*!\C latin1 *//*!*/;
SET @@session.character_set_client=8,@@session.collation_connection=8,@@session.collation_server=255/*!*/;
SET @@session.time_zone='SYSTEM'/*!*/;
SET @@session.lc_time_names=0/*!*/;
SET @@session.collation_database=DEFAULT/*!*/;
/*!80011 SET @@session.default_collation_for_utf8mb4=255*//*!*/;
BEGIN
/*!*/;
# at 322
#220615  6:52:14 server id 1  end_log_pos 375 CRC32 0xe571a47d  Table_map: `test_db`.`t` mapped to number 88
# at 375
#220615  6:52:14 server id 1  end_log_pos 423 CRC32 0xafba99da  Delete_rows: table id 88 flags: STMT_END_F

BINLOG '
noGpYhMBAAAANQAAAHcBAAAAAFgAAAAAAAEAB3Rlc3RfZGIAAXQAAwMDEQEAAgEBAH2kceU=
noGpYiABAAAAMAAAAKcBAAAAAFgAAAAAAAEAAgAD/wAEAAAABAAAAFvmH4Dambqv
'/*!*/;
### DELETE FROM `test_db`.`t`
### WHERE
###   @1=4 /* INT meta=0 nullable=0 is_null=0 */
###   @2=4 /* INT meta=0 nullable=1 is_null=0 */
###   @3=1541808000 /* TIMESTAMP(0) meta=0 nullable=0 is_null=0 */
# at 423
#220615  6:52:14 server id 1  end_log_pos 454 CRC32 0x7e73732f  Xid = 8
COMMIT/*!*/;
SET @@SESSION.GTID_NEXT= 'AUTOMATIC' /* added by mysqlbinlog */ /*!*/;
DELIMITER ;
# End of log file
/*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;
```

从上文日志中可以看出：


- server id 1，表示这个事务是在 server_id=1 的这个库上执行的。
- 每个 event 都有 CRC32 的值，这是因为我把参数 binlog_checksum 设置成了 CRC32。
- Table_map event 跟在statement类型日志中看到的相同，显示了接下来要打开的表，map 到数字 226。现在我们这条 SQL 语句只操作了一张表，如果要操作多张表呢？每个表都有一个对应的 Table_map event、都会 map 到一个单独的数字，用于区分对不同表的操作。
- 在 mysqlbinlog 的命令中，使用了 -vv 参数是为了把内容都解析出来，所以从结果里面可以看到各个字段的值（比如，@1=4、 @2=4 这些值）。
- binlog_row_image 的默认配置是 FULL，因此 Delete_event 里面，包含了删掉的行的所有字段的值。如果把 binlog_row_image 设置为 MINIMAL，则只会记录必要的信息，在这个例子里，就是只会记录 id=4 这个信息。
- 最后的 Xid event，用于表示事务被正确地提交了。

当 binlog_format 使用 row 格式的时候，binlog 里面记录了真实删除行的主键 id，这样 binlog 传到备库去的时候，就肯定会删除 id=4 的行，不会有主备删除不同行的问题。

#### 为什么会有 mixed 格式的 binlog？

- 因为有些 statement 格式的 binlog 可能会导致主备不一致，所以要使用 row 格式。
- 但 row 格式的缺点是，很占空间。比如你用一个 delete 语句删掉 10 万行数据，用 statement 的话就是一个 SQL 语句被记录到 binlog 中，占用几十个字节的空间。但如果用 row 格式的 binlog，就要把这 10 万条记录都写到 binlog 中。这样做，不仅会占用更大的空间，同时写 binlog 也要耗费 IO 资源，影响执行速度。
- 所以，MySQL 就取了个折中方案，也就是有了 mixed 格式的 binlog。mixed 格式的意思是，MySQL 自己会判断这条 SQL 语句是否可能引起主备不一致，如果有可能，就用 row 格式，否则就用 statement 格式。

也就是说，mixed 格式可以利用 statment 格式的优点，同时又避免了数据不一致的风险。

因此，如果你的线上 MySQL 设置的 binlog 格式是 statement 的话，那基本上就可以认为这是一个不合理的设置。你至少应该把 binlog 的格式设置为 mixed。

但现在越来越多的场景要求把 MySQL 的 binlog 格式设置成 row。这么做的理由有很多，一个最直接的好处：恢复数据。

- 从row格式的日志可以看出，binlog会把删掉的整行信息都保存下来，所以，如果你在执行完一条 delete 语句以后，发现删错数据了，可以直接把 binlog 中记录的 delete 语句转成 insert，把被错删的数据插入回去就可以恢复了。

- 如果你是执行错了 insert 语句呢？那就更直接了。row 格式下，insert 语句的 binlog 里会记录所有的字段信息，这些信息可以用来精确定位刚刚被插入的那一行。这时，你直接把 insert 语句转成 delete 语句，删除掉这被误插入的一行数据就可以了。

- 如果执行的是 update 语句的话，binlog 里面会记录修改前整行的数据和修改后的整行数据。所以，如果你误执行了 update 语句的话，只需要把这个 event 前后的两行信息对调一下，再去数据库里面执行，就能恢复这个更新操作了。

#### 循环复制问题

现在可以知道，binlog 的特性确保了在备库执行相同的 binlog，可以得到与主库相同的状态。因此我们可以认为图1中所示的M-S结构中A、B两个节点的内容是一致的。

但实际生产上使用比较多的是双 M 结构，也就是下图所示的主备切换流程。

![20220615151126](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20220615151126.png)

对比上图和图 1，你可以发现，双 M 结构和 M-S 结构，其实区别只是多了一条线，即：节点 A 和 B 之间总是互为主备关系。这样在切换的时候就不用再修改主备关系。

但是，双 M 结构还有一个问题需要解决。

业务逻辑在节点 A 上更新了一条语句，然后再把生成的 binlog 发给节点 B，节点 B 执行完这条更新语句后也会生成 binlog。（我建议你把参数 log_slave_updates 设置为 on，表示备库执行 relay log 后生成 binlog）。

那么，如果节点 A 同时是节点 B 的备库，相当于又把节点 B 新生成的 binlog 拿过来执行了一次，然后节点 A 和 B 间，会不断地循环执行这个更新语句，也就是循环复制了。这个要怎么解决呢？

从row格式的日志中可以看到，MySQL 在 binlog 中记录了这个命令第一次执行时所在实例的 server id。因此，可以用下面的逻辑，来解决两个节点间的循环复制的问题：

- 规定两个库的 server id 必须不同，如果相同，则它们之间不能设定为主备关系；
- 一个备库接到 binlog 并在重放的过程中，生成与原 binlog 的 server id 相同的新的 binlog；
- 每个库在收到从自己的主库发过来的日志后，先判断 server id，如果跟自己的相同，表示这个日志是自己生成的，就直接丢弃这个日志。

按照这个逻辑，如果我们设置了双 M 结构，日志的执行流就会变成这样：
- 从节点 A 更新的事务，binlog 里面记的都是 A 的 server id；
- 传到节点 B 执行一次以后，节点 B 生成的 binlog 的 server id 也是 A 的 server id；
- 再传回给节点 A，A 判断到这个 server id 与自己的相同，就不会再处理这个日志。所以，死循环在这里就断掉了。
