---
author: "李昌"
title: "概览MySQL篇二：日志"
date: "2022-03-23"
tags: ["mysql"]
categories: ["数据库"]
ShowToc: true
TocOpen: true
---

### 什么是redo log， 有什么作用

在MySQL中，如果每一次的更新操作都需要写进磁盘，然后磁盘也要找到对应的那条记录，然后再更新，整个过程 IO 成本、查找成本都很高。因此，我们可以使用类似“缓存”的思路来解决这个问题。

**WAL**:Write-Ahead Logging，先写日志，再写磁盘。注意这里不是*Write ahead logging(在记日志之前写)*，而是*Write-Ahead Logging*：在写之前记日志。

具体说来，当有一条记录需要更新的时候，InnoDB 引擎就会先把记录写到 `redo log` 里面，并更新内存，这个时候更新就算完成了（WAL）。同时，InnoDB 引擎会在适当的时候(系统比较空闲的时候或其他情况)，将这个操作记录更新到磁盘里面.

但`redo log`并不是无限大的，如果当前系统比较繁忙，`redo log`很快就被写满，那么这时候系统只能先暂停处理请求，转而把`redo log`中的内容刷到磁盘上。`redo log`的结构类似循环队列。

![20220323135324](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20220323135324.png)

`redo log`并不是在内存里，而是存储在磁盘上，其提高读写速度的关键在于将原来查询数据的随机读写转化为顺序读写，因此速度快的多。

`redo log`为MySQL提供了**crash-safe**的能力，当系统突然宕机，数据库中的更新不会丢失，可以完整的恢复。

### 什么是binlog，有什么作用

`redo log`是`InnoDB`引擎特有的日志系统，而`binlog`才是MySQL的“亲儿子”。

`binlog`是记录所有数据库表结构变更以及表数据修改的二进制日志，不会记录SELECT和SHOW这类操作。

`binlog`日志是以事件形式记录，还包含语句所执行的消耗时间。

`binlog`对数据进行“存档”，从而可通过`binlog`对数据进行恢复，同时通过`binlog`还可进行主从备份、复制等。

### 为什么MySQL有两个日志系统，有什么差别吗，能不能只用其中一个

MySQL分为server层和引擎层，而引擎层是可被替换的。MySQL自带的引擎叫`MyISAM`，但是 `MyISAM` 没有 `crash-safe` 的能力，`binlog` 日志只能用于归档. InnoDB是另一家公司以插件形式引入MySQL的，而`redo log`就是InnoDB用来实现`crash-safe`的日志系统。

这两种日志有以下三点不同：

1. redo log 是 InnoDB 引擎特有的；binlog 是 MySQL 的 Server 层实现的，所有引擎都可以使用。

2. redo log 是物理日志，记录的是“在某个数据页上做了什么修改”；binlog 是逻辑日志，记录的是这个语句的原始逻辑，比如“给 ID=2 这一行的 c 字段加 1 ”。

3. redo log 是循环写的，空间固定会用完；binlog 是可以追加写入的。“追加写”是指 binlog 文件写到一定大小后会切换到下一个，并不会覆盖以前的日志。

能不能只使用其中某一个日志？答曰：最好不要.

- 只使用binlog，则MySQL不能做到`crash-safe`，也就是说如果数据库宕机，那么可能会造成数据的丢失。
- 只使用redo log，则MySQL不能做到数据存档，无法进行数据恢复，主从备份等操作。

因此，`binlog`和`redo log`缺一不可，`redo log`虽然是“后娘养的”，但是对MySQL有很大的作用。

### 什么是两阶段提交，为什么需要两阶段提交

一个更新语句的执行流程如下：

1. 执行器先找引擎取 ID=2 这一行。ID 是主键，引擎直接用树搜索找到这一行。如果 ID=2 这一行所在的数据页本来就在内存中，就直接返回给执行器；否则，需要先从磁盘读入内存，然后再返回。

2. 执行器拿到引擎给的行数据，把这个值加上 1，比如原来是 N，现在就是 N+1，得到新的一行数据，再调用引擎接口写入这行新数据。

3. 引擎将这行新数据更新到内存中，同时将这个更新操作记录到 redo log 里面，此时 redo log 处于 prepare 状态。然后告知执行器执行完成了，随时可以提交事务。

4. 执行器生成这个操作的 binlog，并把 binlog 写入磁盘。

5. 执行器调用引擎的提交事务接口，引擎把刚刚写入的 redo log 改成提交（commit）状态，更新完成。

![20220323141403](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20220323141403.png)

在上面步骤的后三部中，我们先是将更新操作记录到`redo log`中，但并不提交事务，再写入`binlog`，最后再把刚才写入的`redo log`提交。这个步骤称为**两阶段提交**

为什么要使用两阶段提交？答曰：为了使两份日志之间的逻辑一致。

为了说明这个问题。对于如下语句：`update a = 1 where a = 0`, 考虑下面两种情况:

1. 先提交`redo log`，再提交`binlog`。

如果在`redo log`提交完成后数据库宕机，系统恢复后`redo log`将数据恢复, 这一行已经成功的被修改为1，但`binlog`中并没有这条数据，如果这时候用`binlog`来恢复临时库的话，由于这个语句的`binlog`丢失，因此刚才的改动并不会应用到临时库上，临时库与原库的值不同，造成了数据不一致。

2. 先写`binlog`再写`redo log`

如果写完`binlog`之后系统崩溃，这时候`binlog`已经记录了修改数据的逻辑。但崩溃恢复后由于`redo log`中没有这条更新，因此改动并不会应用到数据表上,这一行依然是0.如果这时用`binlog`来恢复数据或主从复制，那么`binlog`的记录中多了一个事务出来，恢复出来的行为1,造成了数据不一致。

对于两阶段提交，它是如何保证数据一致性的呢？

对于两阶段提交：`①redo log prepare ----> ②写binlog ----> ③redo log commit`

如果在②之前崩溃，重启恢复后发现没有commit，回滚，事务不成功。备份恢复时，没有`binlog`，数据一致。

如果在③之前崩溃，重启恢复后虽然没有commit，但`prepare`和`binlog`都完成，将自动commit。备份恢复时，binlog完整，数据一致。

### binlog的写入机制

`binlog`的写入逻辑比较简单：事务执行过程中，先把日志写道`binlog cache`，事务提交后，再把`binlog cache`写到`binlog`文件中。

但，一个事务的 `binlog` 是不能被拆开的，因此不论这个事务多大，也要确保一次性写入。这就涉及到了 `binlog cache` 的保存问题。

系统给 binlog cache 分配了一片内存，每个线程一个，参数 `binlog_cache_size` 用于控制单个线程内 `binlog cache` 所占内存的大小。如果超过了这个参数规定的大小，就要暂存到磁盘。

事务提交的时候，执行器把 `binlog cache` 里的完整事务写入到 `binlog` 中，并清空 `binlog cache`。状态如图所示。

![20220324131808](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20220324131808.png)

可以看到，每个线程有自己 binlog cache，但是共用同一份 binlog 文件。

- 图中的 write，指的就是指把日志写入到文件系统的 page cache，并没有把数据持久化到磁盘，所以速度比较快。
- 图中的 fsync，才是将数据持久化到磁盘的操作。一般情况下，我们认为 fsync 才占磁盘的 IOPS。

write 和 fsync 的时机，是由参数 sync_binlog 控制的：

1. sync_binlog=0 的时候，表示每次提交事务都只 write，不 fsync；
2. sync_binlog=1 的时候，表示每次提交事务都会执行 fsync；
3. sync_binlog=N(N>1) 的时候，表示每次提交事务都 write，但累积 N 个事务后才 fsync。

因此，在出现 IO 瓶颈的场景里，将 sync_binlog 设置成一个比较大的值，可以提升性能。在实际的业务场景中，考虑到丢失日志量的可控性，一般不建议将这个参数设成 0，比较常见的是将其设置为 100~1000 中的某个数值。

但是，将 sync_binlog 设置为 N，对应的风险是：如果主机发生异常重启，会丢失最近 N 个事务的 binlog 日志。

### redo log的写入机制

#### redo log buffer

在一个事务的更新过程中，日志是要写多次的。比如下面这个事务：
```sql
begin;
insert into t1 ...
insert into t2 ...
commit;
```

这个事务要往两个表中插入记录，插入数据的过程中，生成的日志都得先保存起来，但又不能在还没 commit 的时候就直接写到 redo log 文件里。

所以，redo log buffer 就是一块内存，用来先存 redo 日志的。也就是说，在执行第一个 insert 的时候，数据的内存被修改了，redo log buffer 也写入了日志。

但是，真正把日志写到 redo log 文件（文件名是 ib_logfile+ 数字），是在执行 commit 语句的时候做的。

但redo log buffer 里面的内容，不是每次生成后都要直接持久化到磁盘。

如果事务执行期间 MySQL 发生异常重启，那这部分日志就丢了。由于事务并没有提交，所以这时日志丢了也不会有损失。

#### redo log的三种状态

![20220612184126](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20220612184126.png)

1. 存在 redo log buffer 中，物理上是在 MySQL 进程内存中，就是图中的红色部分；
2. 写到磁盘 (write)，但是没有持久化（fsync），物理上是在文件系统的 page cache 里面，也就是图中的黄色部分；
3. 持久化到磁盘，对应的是 hard disk，也就是图中的绿色部分。

日志写到 redo log buffer 是很快的，wirte 到 page cache 也差不多，但是持久化到磁盘的速度就慢多了。

#### redo log 的写入策略

为了控制 redo log 的写入策略，InnoDB 提供了 innodb_flush_log_at_trx_commit 参数，它有三种可能取值：

- 设置为 0 的时候，表示每次事务提交时都只是把 redo log 留在 redo log buffer 中 ;
- 设置为 1 的时候，表示每次事务提交时都将 redo log 直接持久化到磁盘；
- 设置为 2 的时候，表示每次事务提交时都只是把 redo log 写到 page cache。

#### redo log的写入时机

InnoDB 有一个后台线程，**每隔 1 秒，就会把 redo log buffer 中的日志，调用 write 写到文件系统的 page cache，然后调用 fsync 持久化到磁盘**。

注意，事务执行中间过程的 redo log 也是直接写在 redo log buffer 中的，这些 redo log 也会被后台线程一起持久化到磁盘。

也就是说，一个没有提交的事务的 redo log，也是可能已经持久化到磁盘的。

实际上，除了后台线程每秒一次的轮询操作外，还有两种场景会让一个没有提交的事务的 redo log 写入到磁盘中。

- 一种是，redo log buffer 占用的空间即将达到 innodb_log_buffer_size 一半的时候，后台线程会主动写盘。注意，由于这个事务并没有提交，所以这个写盘动作只是 write，而没有调用 fsync，也就是只留在了文件系统的 page cache。

- 另一种是，并行的事务提交的时候，顺带将这个事务的 redo log buffer 持久化到磁盘。假设一个事务 A 执行到一半，已经写了一些 redo log 到 buffer 中，这时候有另外一个线程的事务 B 提交，如果 innodb_flush_log_at_trx_commit 设置的是 1，那么按照这个参数的逻辑，事务 B 要把 redo log buffer 里的日志全部持久化到磁盘。这时候，就会带上事务 A 在 redo log buffer 里的日志一起持久化到磁盘。

#### 双1配置

指的就是 sync_binlog 和 innodb_flush_log_at_trx_commit 都设置成 1。也就是说，一个事务完整提交前，需要等待两次刷盘，一次是 redo log（prepare 阶段），一次是 binlog。

但事务提交时使用的是两阶段提交，redo log除了prepare阶段外还需要commit阶段，为什么这里只需要在prepare阶段进行一次fsync呢。

如果把 innodb_flush_log_at_trx_commit 设置成 1，那么 redo log 在 prepare 阶段就要持久化一次，因为有一个崩溃恢复逻辑是要依赖于 prepare 的 redo log，再加上 binlog 来恢复的。

每秒一次后台轮询刷盘，再加上崩溃恢复这个逻辑，InnoDB 就认为 redo log 在 commit 的时候就不需要 fsync 了，只会 write 到文件系统的 page cache 中就够了。


#### 组提交 group commit

LSN: 日志逻辑序列号（log sequence number，LSN）。LSN 是单调递增的，用来对应 redo log 的一个个写入点。每次写入长度为 length 的 redo log， LSN 的值就会加上 length。LSN 也会写到 InnoDB 的数据页中，来确保数据页不会被多次执行重复的 redo log.

![20220612190342](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20220612190342.png)

如图所示，是三个并发事务 (trx1, trx2, trx3) 在 prepare 阶段，都写完 redo log buffer，持久化到磁盘的过程，对应的 LSN 分别是 50、120 和 160。

从图中可以看到，
trx1 是第一个到达的，会被选为这组的 leader；等 trx1 要开始写盘的时候，这个组里面已经有了三个事务，这时候 LSN 也变成了 160；trx1 去写盘的时候，带的就是 LSN=160，因此等 trx1 返回时，所有 LSN 小于等于 160 的 redo log，都已经被持久化到磁盘；这时候 trx2 和 trx3 就可以直接返回了。所以，一次组提交里面，组员越多，节约磁盘 IOPS 的效果越好。但如果只有单线程压测，那就只能老老实实地一个事务对应一次持久化操作了。

LSN: 日志逻辑序列号（log sequence number，LSN）。LSN 是单调递增的，用来对应 redo log 的一个个写入点。每次写入长度为 length 的 redo log， LSN 的值就会加上 length。LSN 也会写到 InnoDB 的数据页中，来确保数据页不会被多次执行重复的 redo log.

![20220612190342](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20220612190342.png)

如图所示，是三个并发事务 (trx1, trx2, trx3) 在 prepare 阶段，都写完 redo log buffer，持久化到磁盘的过程，对应的 LSN 分别是 50、120 和 160。

从图中可以看到，
1. trx1 是第一个到达的，会被选为这组的 leader；
2. 等 trx1 要开始写盘的时候，这个组里面已经有了三个事务，这时候 LSN 也变成了 160；
3. trx1 去写盘的时候，带的就是 LSN=160，因此等 trx1 返回时，所有 LSN 小于等于 160 的 redo log，都已经被持久化到磁盘；这时候 trx2 和 trx3 就可以直接返回了。
4. 所以，一次组提交里面，组员越多，节约磁盘 IOPS 的效果越好。

但如果只有单线程压测，那就只能老老实实地一个事务对应一次持久化操作了。

因此可能会存在这种现象：从 MySQL 看到的 TPS 是每秒两万的话，也即每秒会写四万次磁盘。但用工具测试出来，磁盘能力两万左右，怎么能实现两万的 TPS？了解了组提交后我们就会知道，这是用组提交完成的。

在并发更新场景下，第一个事务写完 redo log buffer 以后，接下来这个 fsync 越晚调用，组员可能越多，节约 IOPS 的效果就越好。

#### MySQL 对组提交的优化

前面我们知道，日志提交为两阶段提交，`①redo log prepare ----> ②写binlog ----> ③redo log commit`。但实际上写bin log是分为两步的：

1. 先把 binlog 从 binlog cache 中写到磁盘上的 binlog 文件；
2. 调用 fsync 持久化。

MySQL 为了让组提交的效果更好，把 redo log 做 fsync 的时间拖到了步骤 1 之后。也就是说，两阶段提交变成了这样：
![20220612191003](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20220612191003.png)

这么一来，binlog 也可以组提交了。在执行图中第 4 步把 binlog fsync 到磁盘时，如果有多个事务的 binlog 已经写完了，也是一起持久化的，这样也可以减少 IOPS 的消耗。

不过通常情况下第 3 步执行得会很快，所以 binlog 的 write 和 fsync 间的间隔时间短，导致能集合到一起持久化的 binlog 比较少，因此 binlog 的组提交的效果通常不如 redo log 的效果那么好。

如果你想提升 binlog 组提交的效果，可以通过设置 `binlog_group_commit_sync_delay` 和 `binlog_group_commit_sync_no_delay_count` 来实现。`binlog_group_commit_sync_delay` 参数，表示延迟多少微秒后才调用 fsync;`binlog_group_commit_sync_no_delay_count` 参数，表示累积多少次以后才调用 fsync。

这两个条件是或的关系，也就是说只要有一个满足条件就会调用 fsync。所以，当 `binlog_group_commit_sync_delay` 设置为 0 的时候，`binlog_group_commit_sync_no_delay_count` 也无效了。

分析到这里可以得知，WAL机制的快速主要得益于两方面：
1. redo log 和 binlog 都是顺序写，磁盘的顺序写比随机写速度要快；
2. 组提交机制，可以大幅度降低磁盘的 IOPS 消耗。