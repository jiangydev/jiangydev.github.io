---
layout:     post
title:      "MySQL 优化笔记"
subtitle:   "MySQL 基础, 索引, 主备, 日志, 锁等"
date:       2019-09-29
author:     "jiangydev"
header-img: "img/post-bg-cloud.jpg"
header-mask: 0.5
catalog:    true
tags:
    - MySQL
---

[TOC]

## 1 基础架构

### 1.1 逻辑架构

![MySQL逻辑架构图](/img/in-post/mysql/mysql-note/MySQL逻辑架构图.png)

### 1.2 注意点

#### 1.2.1 连接器

- 长连接使得内存占用大
  - 方案1：定期断开长连接。
  - 方案2： MySQL 5.7 或更新版本，在每次执行较大的操作后，可以执行 mysql_reset_connection 来重新初始化连接资源。

#### 1.2.2 查询缓存

- 建议不使用查询缓存
  - 缓存失效频繁，表更新会导致查询缓存清空。
  - MySQL 8.0 版本开始没有查询缓存功能。

#### 1.2.3 其他

- 如果用户对表没有权限，是在执行器这一步出现 ERROR。
- 如果SQL中字段不存在，是在分析器这一步出现 ERROR。



## 2 主备和日志系统

### 2.1 redo log 重做日志

WAL 技术，WAL 的全称是 Write-Ahead Logging，它的关键点就是先写日志，再写磁盘。

![redo_log](/img/in-post/mysql/mysql-note/redo_log.png)

Redo log，InnoDB 引擎层维护，记录物理日志（有I/O，记录的数据页修改过程），循环写，可以保证即使数据库发生异常重启，之前提交的记录都不会丢失，这个能力称为 crash-safe。 

开启方法：

```ini
innodb_flush_log_at_trx_commit=1
```



### 2.2 binlog 归档日志

binlog 是 MySQL Server 层维护的，记录逻辑日志（有I/O，记录的语句逻辑）。

![update数据库操作流程](/img/in-post/mysql/mysql-note/update数据库操作流程.png)

两阶段提交：将 redo log 的写入拆成了两个步骤 prepare 和 commit。

开启方法：

```ini
# 每次事务的 binlog 都持久化到磁盘
sync_binlog=1
```

#### 2.3.1 binlog 三种格式

statement，row ，mixed

用 mysqlbinlog 工具恢复数据。

### 2.3 binlog 写入机制

![binlog写入机制](/img/in-post/mysql/mysql-note/binlog写入机制.png)

- 图解

  - write：日志写入文件系统的 page cache，并没有持久化到磁盘，速度较快；
  - fsync：数据持久化到磁盘，占磁盘的 IOPS。

- 涉及的参数

  - binlog_cache_size：控制单个线程内 binlog cache 所占内存的大小。如果超过了这个参数规定的大小，就要暂存到磁盘。 

  - sync_binlog：控制 write 和 fsync 的时机

    - sync_binlog=0 的时候，表示每次提交事务都只 write，不 fsync；
    - sync_binlog=1 的时候，表示每次提交事务都会执行 fsync；
    - sync_binlog=N(N>1) 的时候，表示每次提交事务都 write，但累积 N 个事务后才 fsync。

    这个值通常设置为 100 - 1000，提升性能，面临的风险是，如果异常重启，会丢失 N 个事务的 binlog 日志。

### 2.4 redo log 写入机制

![redolog存储状态](/img/in-post/mysql/mysql-note/redolog存储状态.png)

- 涉及的参数
  - innodb_flush_log_at_trx_commit
    - 设置为 0 的时候，表示每次事务提交时都只是把 redo log 留在 redo log buffer 中 ; 
    - 设置为 1 的时候，表示每次事务提交时都将 redo log 直接持久化到磁盘；
    - 设置为 2 的时候，表示每次事务提交时都只是把 redo log 写到 page cache。

#### 2.4.1 没有提交的事务的 redo log 写入到磁盘的三种情况

- InnoDB 有一个后台线程，每隔 1 秒，就会把 redo log buffer 中的日志，调用 write 写到文件系统的 page cache，然后调用 fsync 持久化到磁盘。
- redo log buffer 占用的空间即将达到 innodb_log_buffer_size 一半的时候，后台线程会主动写盘 ；只是 write，不是 fsync；
- 并行的事务提交的时候，顺带将这个事务的 redo log buffer 持久化到磁盘。

### 2.5 双“1”配置

指的就是 sync_binlog 和 innodb_flush_log_at_trx_commit 都设置成 1，即一个事务完整提交前，需要等待两次刷盘，一次 redo log，一次 binlog。

虽然是两次刷盘，但有“组提交机制”。当事务一提交时，若其他事务也准备写入，此时可以一次写入，判断的依据时 LSN（Length Sequence Number）。

#### 2.5.1 设置非双“1”的情况

1. 业务高峰期。一般如果有预知的高峰期，DBA 会有预案，把主库设置成“非双 1”。
2. 备库延迟，为了让备库尽快赶上主库。
3. 用备份恢复主库的副本，应用 binlog 的过程。
4. 批量导入数据的时候。 

一般情况下，把生产库改成“非双 1”配置，是设置 innodb_flush_logs_at_trx_commit=2、sync_binlog=1000。 

### 2.6 主备

#### 2.6.1 原理

![主备](/img/in-post/mysql/mysql-note/主备.png)

日志同步的完整过程：
1. 在备库 B 上通过 change master 命令，设置主库 A 的 IP、端口、用户名、密码，以及要从哪个位置开始请求 binlog，这个位置包含文件名和日志偏移量。
2. 在备库 B 上执行 start slave 命令，这时候备库会启动两个线程，就是图中的 io_thread 和 sql_thread。其中 io_thread 负责与主库建立连接。
3. 主库 A 校验完用户名、密码后，开始按照备库 B 传过来的位置，从本地读取 binlog，发给B。
4. 备库 B 拿到 binlog 后，写到本地文件，称为中转日志（relay log）。
5. sql_thread 读取中转日志，解析出日志里的命令，并执行。 

#### 2.6.2 延迟

主备延迟，就是同一个事务，在备库执行完成的时间和主库执行完成的时间之间的差值。

查看备库延迟时间命令：show slave status，返回结果里面会显示 seconds_behind_master

计算方法：

1. 每个事务的 binlog 里面都有一个时间字段，用于记录主库上写入的时间；
2. 备库取出当前正在执行的事务的时间字段的值，计算它与当前系统时间的差值，得到seconds_behind_master。 

精度为秒，若系统时间不一致，不会影响延迟时间，因为从库会执行 SELECT UNIX_TIMESTAMP() 获得主库的系统时间。

若网络正常，主备延迟的表现为备库消费中转日志（relay log）的速度。

主备延迟的来源：

- 备库所在机器的性能要比主库所在的机器性能差。

- 对称部署的情况下，备库的压力大。（备库用于查询，消费大量CPU资源）

  方案：一主多从；将 binlog 输出到外部系统，如 Hadoop，外部系统提供统计类查询的能力。

- 大事务；不要一次性地用 delete 语句删除太多数据 。如果主库执行语句耗费10分钟，从库会延迟10分钟。

- 大表 DDL；使用 gh-ost 方案。

- 备库的并行复制能力；

#### 2.6.3 并行复制

![主备多线程模型](/img/in-post/mysql/mysql-note/主备多线程模型.png)

coordinator 就是原来的 sql_thread，只负责读取中转日志和分发事务。

worker 线程更新日志。work 线程的个数，由参数 slave_parallel_workers 决定。

coordinator 分发两个原则：

1. 不能造成更新覆盖。这就要求更新同一行的两个事务，必须被分发到同一个 worker 中。
2. 同一个事务不能被拆开，必须放到同一个 worker 中。

并行复制策略：

- MySQL 5.6 版本：按库分发

- MariaDB：按组分发。利用 redo log 组提交，一组中的事务（commit_id 相同）并行。

  缺点：大事务造成资源浪费。

- MySQL 5.7 版本：参数 slave-parallel-type 来控制 

  - DATABASE：按库并行

  - LOGICAL_CLOCK：

    1. 同时处于 prepare 状态的事务，在备库执行时是可以并行的；
    2. 处于 prepare 状态的事务，与处于 commit 状态的事务之间，在备库执行时也是可以并行的。 

    设置参数 binlog_group_commit_sync_delay 和 binlog_group_commit_sync_no_delay_count，提高 IO 性能的同时，也可以提高备库复制的并行度。

- MySQL 5.7.22 版本：基于 WRITESET 的并行复制。遇到无主键和有外键的场景，会退化为单线程模型。

  参数 binlog-transaction-dependency-tracking，可选值如下。

  1. COMMIT_ORDER，表示根据同时进入 prepare 和 commit 来判断是否可以并行的策略。
  2. WRITESET，表示的是对于事务涉及更新的每一行，计算出这一行的 hash 值，组成集合 writeset。如果两个事务没有操作相同的行，也就是说它们的 writeset 没有交集，就可以并行。
  3. WRITESET_SESSION，是在 WRITESET 的基础上多了一个约束，即在主库上同一个线程先后执行的两个事务，在备库执行的时候，要保证相同的先后顺序。 

#### 2.6.4 主备切换

##### 1 基于位点的主备切换

```sql
CHANGE MASTER TO
MASTER_HOST=$host_name
MASTER_PORT=$port
MASTER_USER=$user_name
MASTER_PASSWORD=$password
MASTER_LOG_FILE=$master_log_name  -- 主库对应的日志文件名
MASTER_LOG_POS=$master_log_pos    -- 主库日志偏移量
```

主动跳过错误的方式：

- 跳过事务

  ```sql
  set global sql_slave_skip_counter=1;
  start slave;
  ```

  每次遇到这些错误就停下来执行命令，直到不再出现。

- 跳过指定错误

  设置 slave_skip_errors 参数 。1062 错误是插入数据时唯一键冲突；1032 错误是删除数据时找不到行。 

遇到错误尽量修补数据。

##### 2 GTID(Global Transaction Identifier)

全局事务ID，MySQL 5.6 版本引入，格式为 server_uuid:gno。

启动方式：启动实例时，加上参数 gtid_mode=on 和 enforce_gtid_consistency=on。

```sql
CHANGE MASTER TO
MASTER_HOST=$host_name
MASTER_PORT=$port
MASTER_USER=$user_name
MASTER_PASSWORD=$password
master_auto_position=1
```

#### 2.6.5 读写分离“过期读”

- 强制走主库；

- sleep 方案；（等待一段时间再读，如用户操作后先直接显示结果，不读库）

- 判断主备无延迟（show slave status）；并不是完全精确

  - 判断 seconds_behind_master 是否已经等于 0
  - 对比位点
    - Master_Log_File 和 Read_Master_Log_Pos，表示的是读到的主库的最新位点；
    - Relay_Master_Log_File 和 Exec_Master_Log_Pos，表示的是备库执行的最新位点
  - 对比 GTID 集合
    - Auto_Position=1 ，表示这对主备关系使用了 GTID 协议。
    - Retrieved_Gtid_Set，是备库收到的所有日志的 GTID 集合；
    - Executed_Gtid_Set，是备库所有已经执行完成的 GTID 集合。

- 配合方案：半同步复制 semi-sync replication

  - 从库收到 binlog 后 ack 确认，主库收到 ack 后返回客户端确认。
  - 存在的问题：1. 一主多从时，某些从库查询会过期读；2. 持续延迟时，可能会过度等待。

- 等主库位点方案

  主库用 show master status 命令获得 file 和 pos，然后执行下面的SQL判断是否可以在从库查询。

  ```sql
  -- 从库执行，如果返回 >= 0；则可以执行查询，-1 等待超时；NULL 备库同步线程异常。
  select master_pos_wait(file, pos[, timeout]);
  ```

- 等 GTID 方案

  根据事务返回的 gtid_set 查询（需要调用 mysql_session_track_get_first 这个函数），返回 1 在从库查询，反则主库查询。

  ```sql
  select wait_for_executed_gtid_set(gtid_set, 1);
  ```

  

### 2.7 双主

#### 2.7.1 双主切换

##### 1 可靠性优先策略（建议）

1. 判断备库 B 现在的 seconds_behind_master，如果小于某个值（比如 5 秒）继续下一步，否则持续重试这一步；
2. 把主库 A 改成只读状态，即把 readonly 设置为 true；
3. 判断备库 B 的 seconds_behind_master 的值，直到这个值变成 0 为止；
4. 把备库 B 改成可读写状态，也就是把 readonly 设置为 false；
5. 把业务请求切到备库 B。

步骤3会使得系统处于不可写状态，若要缩短时间，可以控制步骤1的主备延迟时间足够小。

##### 2 可用性优先策略

将步骤 4，5调整为第一步，但会造成数据不一致。

在 mixed 的格式下，数据可能会出错，但正常插入；row 格式下，数据出错，且插入会报错。



#### 2.7.2 循环复制

问题产生：双 M 的情况下，log_slave_updates 设置为 on，表示备库执行 relay log 后生成 binlog。

数据库解决方案 ：

  1. 规定两个库的 server id 必须不同，如果相同，则它们之间不能设定为主备关系；
 2. 一个备库接到 binlog 并在重放的过程中，生成与原 binlog 的 server id 相同的新的 binlog；
 3. 每个库在收到从自己的主库发过来的日志后，先判断 server id，如果跟自己的相同，表示这个日志是自己生成的，就直接丢弃这个日志。 

出现循环复制的场景：

1. 主库更新事务后，用命令 set global server_id=x 修改了 server_id。
2. 三节点

解决方案：

- 临时修改 server_id 为 master 的，种植循环复制后，修改为原来的；
- 若还有更新，此时不能修改 server_id，可以先 stop slave，手动 change master 后 start slave。

## 3 事务隔离

### 3.1 ACID

Atomicity、Consistency、Isolation、Durability，即原子性、一致性、隔离性、持久性

### 3.2 隔离级别

- 读未提交：一个事务还没提交时，它做的变更就能被别的事务看到。
- 读提交：一个事务提交之后，它做的变更才会被其他事务看到。
- 可重复读：一个事务执行过程中看到的数据，总是跟这个事务在启动时看到的数据是一致的。当然在可重复读隔离级别下，未提交变更对其他事务也是不可见的。
- 串行化：顾名思义是对于同一行记录，“写”会加“写锁”，“读”会加“读锁”。当出现读写锁冲突的时候，后访问的事务必须等前一个事务执行完成，才能继续执行。

### 3.3 避免长事务

- 应用端
  - set autocommit=1 显式启动事务
  - 只读业务不加事务
  - 根据业务本身的预估，通过 SET MAX_EXECUTION_TIME 命令，来控制每个语句执行的最长时间，避免单个语句意外执行太长时间。 
- 数据库端
  - 监控 information_schema.Innodb_trx 表，设置长事务阈值，超过就报警 / 或者 kill；
  - Percona 的 pt-kill 这个工具
  - 在业务功能测试阶段要求输出所有的 general_log，分析日志行为提前发现问题； 
  - 如果使用的是 MySQL 5.6 或者更新版本，把 innodb_undo_tablespaces 设置成 2（或更大的值，默认0，使用系统表空间）。如果真的出现大事务导致回滚段过大，这样设置后清理起来更方便（控制undo开启独立的表空间）。

## 4 索引

索引的目的：为了提高数据查询的效率，就像书的目录一样。

### 4.1 索引的常见模型

哈希表：键值+链表，适用于只有等值查询的场景；

有序数组：等值查询和范围查询性能好，只适用于静态存储引擎（更新数据成本高）；

搜索树：

### 4.2 InnoDB 索引模型

B+树索引模型

索引类型分为主键索引和非主键索引。

主键索引的叶子节点存的是整行数据。在 InnoDB 里，主键索引也被称为聚簇索引（clustered index）。
非主键索引的叶子节点内容是主键的值。在 InnoDB 里，非主键索引也被称为二级索引 （secondary index）。

回表：先搜索二级索引，根据二级索引中的主键ID再到主键索引搜索。

### 4.3 索引维护

页分裂：插入新值时，若数据页已满，根据B+树的算法，需要申请新的数据页，挪动部分数据。

“页合并”：相邻两页删除数据，利用率很低后，数据页会做合并。

##### 4.3.1 主键

- 自增主键，保证了有序插入，不会触发叶子节点的分裂，降低写数据成本；

- 主键长度越小，普通索引的叶子节点就越小，普通索引占用的空间也就越小。 

### 4.4 覆盖索引

覆盖索引可以减少树的搜索次数（不需要回表），显著提升查询性能，所以使用覆盖索引是一个常用的性能优化手段。

考虑建立冗余索引以支持覆盖索引。

### 4.5 最左前缀原则

### 4.6 索引下推

在 MySQL 5.6 之前，只能一个个回表。到主键索引上找出数据行，再对比字段值。
而 MySQL 5.6 引入的索引下推优化（index condition pushdown)， 可以在索引遍历过程中，对索引中包含的字段先做判断，直接过滤掉不满足条件的记录，减少回表次数。索引下推在执行计划中显示为：Using index condition。

### 4.7 重建索引

```sql
alter table T engine=InnoDB
```

### 4.8 普通索引和唯一索引

在查询上，两者性能差距微乎其微；普通索引会比唯一索引多查询一次记录，但 InnoDB 是按数据页为单位读写（默认16KB），影响不大。

在更新上，普通索引可以使用 change buffer 机制，而唯一索引无法使用这一机制（要判断唯一性）。

对于写多读少的表，普通索引可以利用 change buffer 机制，收益更大。

##### 4.8.1 change buffer 机制，主机异常重启，数据丢失？

不会丢失，虽然只更新内存，但事务提交的时候，change buffer 的操作也记录到了 redo log 里。

### 4.9 索引选错

选错的因素：扫描行数、是否使用临时表、是否排序等

- 索引统计信息不准确
  - analyze table
- 优化器误判
  - force index 强行指定索引
  - 过修改语句来引导优化器（如 order by b limit 1 和 order by b,a limit 1）
  - 过增加或者删除索引

### 4.10 前缀索引

使用前缀索引，定义好长度，就可以做到既节省空间，又不用额外增加太多的查询成本。

前缀索引会使得覆盖索引用不上，需要回表查询。

如果查询字段使用等值查询，有两种优化方式：

- hash 字段+索引（多使用一个字段；crc32函数，CPU占用比reverse多；查询效率稳定）
- 倒序存储+前缀索引（前缀索引长度对空间的消耗）

## 5 锁

### 5.1 全局锁

全局锁就是对整个数据库实例加锁。MySQL 提供了一个加全局读锁的方法，命令是Flush tables with read lock (FTWRL)。

使用场景：全库逻辑备份

##### 5.1.1 不加锁引发的问题

备份的数据不是一个时间点的，会造成数据不一致。

##### 5.1.2 mysqldump

官方自带的逻辑备份工具，使用参数–single-transaction 的时候，导数据之前就会启动一个事务，来确保拿到一致性视图。（仅支持支持事务的引擎）

##### 5.1.3 set global readonly=true

建议不使用这种方式，原因：

- 修改 global 变量的方式影响面更大，在有些系统中，readonly 的值会被用来做其他逻辑，比如用来判断一个库是主库还是备库。
- 在异常处理机制上有差异。如果执行 FTWRL 命令之后由于客户端发生异常断开，那么 MySQL 会自动释放这个全局锁，整个库回到可以正常更新的状态。而将整个库设置为 readonly 之后，如果客户端发生异常，则数据库就会一直保持 readonly 状态，这样会导致整个库长时间处于不可写状态，风险较高。

### 5.2 表级锁

MySQL 里面表级别的锁有两种：一种是表锁，一种是元数据锁（meta data lock，MDL)。

#### 5.2.1 表锁

语法：lock tables … read/write

手动释放或客户端断开时自动释放

#### 5.2.2 MDL

不需要显式使用，访问时自动加上。

#### 5.2.3 注意事项

由于MDL会自动锁表，读锁虽然共享，但当多个会话掺杂着读写操作，会导致锁表后的读操作都被锁住。应注意该表的查询频率（客户端重试机制），防止 DDL 操作引发整库挂掉。

如何安全地给小表加字段：

- 若请求不频繁地表，考虑 kill 长事务再执行；

- alter table 设置等待时间，拿不到写锁则放弃；

  ```sql
  ALTER TABLE tbl_name WAIT N add column ...
  ```

  

### 5.3 行锁

##### 5.3.1 两阶段锁协议

在 InnoDB 事务中，行锁是在需要的时候才加上的，但并不是不需要了就立刻释放，而是要等到事务结束时才释放。

注：

如果你的事务中需要锁多个行，要把最可能造成锁冲突、最可能影响并发度的锁尽量往后放。 

##### 5.3.2 死锁和死锁检测

![死锁示例](/img/in-post/mysql/mysql-note/死锁示例.png)

两种策略：

- 直接进入等待，直到超时。这个超时时间可以通过参数 innodb_lock_wait_timeout 设置（默认50s）；

- 发起死锁检测，发现死锁后，主动回滚死锁链条中的某一个事务，让其他事务得以继续执行。将参数 innodb_deadlock_detect 设置为 on，表示开启这个逻辑。

  **死锁检测要耗费大量的 CPU 资源。**

### 5.4 间隙锁

解决幻读问题，幻读：事务提交前，行锁无法锁新插入的行，造成幻读。

间隙锁和行锁合称 next-key lock，每个 next-key lock 是前开后闭区间（分析时，须将间隙锁和行锁分开）。 

#### 5.4.1 加锁规则*

- 前提

  - 5.x 系列 <=5.7.24，8.0 系列 <=8.0.13 

- 规则

  - 原则 1：加锁的基本单位是 next-key lock。next-key lock 是前开后闭区间。
  - 原则 2：查找过程中访问到的对象才会加锁。
  - 优化 1：索引上的等值查询，给唯一索引加锁的时候，next-key lock 退化为行锁。
  - 优化 2：索引上的等值查询，向右遍历时且最后一个值不满足等值条件的时候，next-key lock 退化为间隙锁。
  - 一个 bug：唯一索引上的范围查询会访问到不满足条件的第一个值为止。

  如果语句含 limit，当 limit 符合条件后，不会往后加锁。

  建议删除数据时加 limit，减小锁范围。

  RR 级别下，语句执行过程中加上的行锁，在语句执行完成后，就要把“不满足条件的行”上的行锁直接释放了，不需要等到事务提交。 

### 5.5 并发连接和并发查询

通常情况下，把 innodb_thread_concurrency （并发线程）设置为 64~128 之间的值。

并发连接：占用内存

并发查询：占用 CPU，show processlist 结果中当前正在执行的语句。

在线程进入锁等待以后，并发线程的计数会减一，也就是说等行锁（也包括间隙锁）的线程是不算在并发线程中。 避免整个系统死锁。

#### 5.5.1 检测数据库是否可用

select 1 语句无法发现系统因锁等待而不可用的情况。

- 查表判断；

  在 mysql 库中创建一个 health_check 表，定期执行 `select * from mysql.health_check`；

  该方法在 binlog 磁盘空间使用 100%，因读库正常，会无法判断（更新和事务提交的 commit 阻塞）。

- 更新判断；

  创建一个 health_check 表，表结构中有 server_id 和 modifid_time 两个字段，每个库更新自己的字段。

  该方法在 IO 利用率 100% 时，因该 update 语句效率高、不会超时，会无法发现问题。

- 内部统计；

  ```sql
  -- 统计每次 IO 请求的时间
  select * from performance_schema.file_summary_by_event_name where event_name='wait/io/file/innodb/innodb_log_file’
  ```

  如果打开所有的 performance_schema 项，性能大概会下降 10% 左右。所以，建议只打开自己需要的项进行统计。 

### 5.6 死锁现场分析

查看死锁命令：show engine innodb status;

LATESTDETECTED DEADLOCK，就是记录的最后一次死锁信息。 

![死锁现场案例](/img/in-post/mysql/mysql-note/死锁现场案例.png)

1. 这个结果分成三部分：
    (1) TRANSACTION，是第一个事务的信息；
    (2) TRANSACTION，是第二个事务的信息；
    WE ROLL BACK TRANSACTION (1)，是最终的处理结果，表示回滚了第一个事务。

2. 第一个事务的信息中：
WAITING FOR THIS LOCK TO BE GRANTED，表示的是这个事务在等待的锁信息；
index c of table `test`.`t`，说明在等的是表 t 的索引 c 上面的锁；
lock mode S waiting 表示这个语句要自己加一个读锁，当前的状态是等待中；
Record lock 说明这是一个记录锁；
n_fields 2 表示这个记录是两列，也就是字段 c 和主键字段 id；
0: len 4; hex 0000000a; asc ;; 是第一个字段，也就是 c。值是十六进制 a，也就是10；
1: len 4; hex 0000000a; asc ;; 是第二个字段，也就是主键 id，值也是 10；
这两行里面的 asc 表示的是，接下来要打印出值里面的“可打印字符”，但 10 不是可打印字符，因此就显示空格。
第一个事务信息就只显示出了等锁的状态，在等待 (c=10,id=10) 这一行的锁。
当然你是知道的，既然出现死锁了，就表示这个事务也占有别的锁，但是没有显示出来。别着急，我们从第二个事务的信息中推导出来。

3. 第二个事务显示的信息要多一些：

| 字段                                | 含义                                         |
| ----------------------------------- | -------------------------------------------- |
| “ HOLDS THE LOCK(S)”                | 用来显示这个事务持有哪些锁；                 |
| index c of table `test`.`t`         | 表示锁是在表 t 的索引 c 上；                 |
| hex 0000000a 和 hex 00000014        | 表示这个事务持有 c=10 和 c=20 这两个记录锁； |
| WAITING FOR THIS LOCK TO BE GRANTED | 表示在等 (c=5,id=5) 这个记录锁。             |

从上面这些信息中，我们就知道： 

1. “lock in share mode”的这条语句，持有 c=5 的记录锁，在等 c=10 的锁；
2. “for update”这个语句，持有 c=20 和 c=10 的记录锁，在等 c=5 的记录锁。

因此导致了死锁。这里，我们可以得到两个结论：

1. 由于锁是一个个加的，要避免死锁，对同一组资源，要按照尽量相同的顺序访问；
2. 在发生死锁的时刻，for update 这条语句占有的资源更多，回滚成本更大，所以 InnoDB
   选择了回滚成本更小的 lock in share mode 语句，来回滚。

## 6 优化

### 6.1 刷脏页控制策略

当内存数据页跟磁盘数据页内容不一致的时候，我们称这个内存页为“脏页”。内存数据写入到磁盘后，内存和磁盘上的数据页的内容就一致了，称为“干净页”。

#### 6.1.1 innodb_io_capacity

将 innodb_io_capacity 参数设置为磁盘的 IOPS，测试磁盘随机读写的命令：

```shell
$ fio -filename=$filename -direct=1 -iodepth 1 -thread -rw=randrw -rwmixread=70 -ioengine=psync -bs=16k -size=2G -numjobs=30 -runtime=120 -group_reporting -name=$name
```

#### 6.1.2 innodb_max_dirty_pages_pct

参数 innodb_max_dirty_pages_pct 是脏页比例上限，默认值是 75%。（可修改，计算脏页比例和 redo log 写盘速度，取最大值。）

#### 6.1.3 innodb_flush_neighbors

参数 innodb_flush_neighbors=0 ，控制刷脏页的行为不会传播。

在 MySQL 8.0 中，默认值为 0。

### 6.2 收缩表空间

将 innodb_file_per_table 设置为 ON，不使用共享表空间，使得删表时可以回收表空间。

delete 操作不会回收表空间，会产生记录位置复用或数据页复用。不仅是删除数据，插入数据也会造成空洞。

#### 6.2.1 重建表

```sql
alter table A engine=InnoDB
```

MySQL 会自动完成转存数据、交换表名、删除旧表的操作。

注：

- 临时表 tmp_table 在 Server 层创建。
- 重建表，InnoDB 不会把整张表占满，每个页留了1/16给后续的更新用。

![非Online重建表过程](/img/in-post/mysql/mysql-note/非Online重建表过程.png)

#### 6.2.2 Online DDL

MySQL 5.6 版本开始引入 ；**允许在重建表的过程中，允许对表A做增删改操作。**

注：对A增删改操作时，MDL写锁会退化成读锁（保护自己，禁止其他DDL操作），不阻塞增删改操作。

重建表的流程：
1. 建立一个临时文件，扫描表 A 主键的所有数据页；
2. 用数据页中表 A 的记录生成 B+ 树，存储到临时文件中；（耗时）
3. 生成临时文件的过程中，将所有对 A 的操作记录在一个日志文件（row log）中，对应的是
图中 state2 的状态；
4. 临时文件生成后，将日志文件中的操作应用到临时文件，得到一个逻辑数据上与表 A 相同
的数据文件，对应的就是图中 state3 的状态；
5. 用临时文件替换表 A 的数据文件。 

临时文件 tmp_file 由 InnoDB 内部创建，整个 DDL 过程在内部完成。

![Online重建表过程](/img/in-post/mysql/mysql-note/Online重建表过程.png)

推荐工具：开源的 gh-ost

#### 6.2.3 Online 和 inplace

临时文件 tmp_file 由 InnoDB 内部创建，整个 DDL 过程在内部完成。对于 Server 层而言，没有将数据移动到临时表，称为“inplace”。

重建表的语句，其隐含的意思如下：

```sql
alter table t engine=innodb,ALGORITHM=inplace;
```

两者的逻辑关系：

- DDL 过程如果是 Online，就一定是 inplace；
- 反之不成立，如果 DDL 是 inplace，不一定是 Online。截至 MySQL8.0，加全文索引（FULLTEXT index）和空间索引（SPATIAL index）属于这种情况。

#### 6.2.4 三种重建表方式区别

- alter table：即 recreate，如 Online DDL 图示过程；
- analyze table：不是重建表，只是对表的索引信息做重新统计，没有修改数据，该过程加 MDL 读锁。
- optimize table：等于 recreate+analyze。

### 6.3 COUNT()

#### 6.3.1 不同 COUNT 用法

COUNT 语义：一个聚合函数，对返回的结果行一行行判断，如果不是 NULL，累计值加 1。

效率排序：COUNT(*) ≈ COUNT(1) > COUNT(主键ID) > COUNT(字段)

COUNT(*)：优化器会遍历最小的索引树得到最终结果；

COUNT(1)：遍历表，但不取值，返回 Server 层的每一行都是数字"1"；

COUNT(主键ID)：遍历表，取值并返回给 Server 层；

COUNT(字段)：字段为 NOT NULL 时，读取字段返回 Server 层；字段允许 NULL 时，先判断是否为 NULL 再返回。

#### 6.3.2 计数方法

注：不能使用 show table status 中的行数，误差可能为 40％ 至 50％。

- 缓存系统（Redis）计数：无法保证一致性，逻辑上不精确
- 数据库保存计数

### 6.4 ORDER BY

#### 6.4.1 算法一：全字段排序

通过索引查找符合的记录主键 ID，回表取字段存入 sort_buffer，直到不满足查询条件为止，在 sort_buffer 中做快速排序，最终结果返回客户端。

如果排序的内存 sort_buffer_size 设置的小，排序会在文件中进行（归并排序）。

#### 6.4.2 算法二：rowid 排序

如果排序的单行长度太大（max_length_for_sort_data），会采用这个算法。

该算法放入 sort_buffer 的字段为排序字段和主键ID（若没有主键，有系统生成的6字节rowid代替），排序结果出来后回表取字段返回客户端。

#### 6.4.3 优化

- 加索引时将查询条件和排序字段加上索引，也可考虑覆盖索引。

- 在一个两个字段的联合索引中，第一个字段为查询条件且为 in，第二个为排序字段，此时不能利用索引的排序，可以考虑查询条件分开查询，然后代码中做归并排序。（要权衡性能需求和代码复杂度）

#### 6.4.4 ORDER BY RAND()

order by rand() 使用了内存临时表，内存临时表排序的时候使用了 rowid 排序方法（不是用主键）。 

替代方案：

取得表中行数 C，Y = floor(C*rand())，再用 limit Y, 1 取值。若需要多个值，limit min, max-min+1。

#### 6.4.5 临时表

临时表的大小通过 tmp_table_size 设置，默认 16M，小于该值使用内存临时表，大于则使用磁盘临时表。

### 6.5 SQL 优化

慢查的三个原因：索引没有建好、语句没有写好、索引选错

#### 6.5.1 函数操作

- 对索引字段做函数操作，可能会破坏索引值的有序性，因此优化器会放弃走树搜索功能。

#### 6.5.2 隐式转换

- 类型转换：数据库中遇到字符和数字，转换规则是字符转数字。
- 字符编码转换：数据库会默认“按数据长度增加的方向”进行转换
  - CONVERT USING utf8

#### 6.5.3 JOIN 语句

Index Nested Loop Join

在可以使用被驱动表索引的前提下：

1. 使用 join 语句，性能比强行拆成多个单表执行 SQL 语句的性能要好；
2. 如果使用 join 语句的话，需要让小表做驱动表。

Simple Nested Loop Join：被驱动表无索引可用，全表扫描

Block Nested Loop Join：被驱动表无索引可用，但将驱动表分块放入内存（join_buffer）中全表扫描。尽量转成 BKA。



**结论**：更准确地说，在决定哪个表做驱动表的时候，应该是两个表按照各自的条件过滤，过滤完成之后，计算参与 join 的各个字段的总数据量，数据量小的那个表，就是“小表”，应该作为驱动表。 



Multi-Range Read 优化：

因为大多数的数据都是按照主键递增顺序插入得到的，所以我们可以认为，如果按照主键的递增顺序查询的话，对磁盘的读比较接近顺序读，能够提升读性能。 



对 NLJ 的优化：Batched Key Access 算法（依赖 MRR），将驱动表的数据暂存到内存中。



大表 join 操作虽然对 IO 有影响，但是在语句执行结束后，对 IO 的影响也就结束了。但是，对 Buffer Pool 的影响就是持续性的，需要依靠后续的查询请求慢慢恢复内存命中率（大量的冷表数据进入young区域，命中率降低）。 

#### 6.5.4 GROUP BY 语句

不论是使用内存临时表还是磁盘临时表，group by 逻辑都需要构造一个带唯一索引的表，执行代价都是比较高的。

- 使用索引：generated column 机制，用来实现列数据的关联更新。

  ```sql
   alter table t1 add column z int generated always as(id % 100), add index(z);
  ```

- SQL_BIG_RESULT，数据量大，直接用磁盘临时表

指导原则：

1. 如果对 group by 语句的结果没有排序要求，要在语句后面加 order by null；
2. 尽量让 group by 过程用上表的索引，确认方法是 explain 结果里没有 Using temporary 和 Using filesort；
3. 如果 group by 需要统计的数据量不大，尽量只使用内存临时表；也可以通过适当调大tmp_table_size 参数，来避免用到磁盘临时表；
4. 如果数据量实在太大，使用 SQL_BIG_RESULT 这个提示，来告诉优化器直接使用排序算法得到 group by 的结果。 

### 6.5 临时提高性能

#### 6.5.1 短连接风暴

- 先处理掉那些占着连接但是不工作的线程（要注意 kill 的线程对业务的影响）
- 减少连接过程的消耗，如果数据库只是内网能访问的，可以让数据库跳过权限验证阶段（让数据库跳过权限验证阶段，高风险）。

#### 6.5.2 QPS 突增问题

完善规范的运维体系：虚拟化、白名单机制、业务账号分离。及时将新上的业务功能离线，修改后上线。

### 6.6 IO性能提升

- 设置 binlog_group_commit_sync_delay （表示延迟多少微秒后才调用 fsync）和 binlog_group_commit_sync_no_delay_count（表示累积多少次以后才调用 fsync ） 参数，减少 binlog 的写盘次数。这个方法是基于“额外的故意等待”来实现的，因此可能会增加语句的响应时间，但没有丢失数据的风险。可以增加备库复制的并行度。
- 将 sync_binlog 设置为大于 1 的值（比较常见是 100~1000）。这样做的风险是，主机掉电时会丢 binlog 日志。 
- 将 innodb_flush_log_at_trx_commit 设置为 2。这样做的风险是，主机掉电的时候会丢数据。

## 7 慢查分析

### 7.1 查询长时间不返回

查看的命令：show processlist;

- 等 MDL 锁（表级）；
- 等 flush（表级）；
- 等行锁；

### 7.2 查询慢



## 8 其他

### 8.1 误删数据

#### 8.1.1 误删行

如果是使用 delete 语句误删了数据行，可以用 Flashback 工具通过闪回把数据恢复回来。

使用这个方案的前提是，需要确保 binlog_format=row 和 binlog_row_image=FULL。 

使用 truncate /drop table 和 drop database 命令删除的数据，就没办法通过 Flashback 来恢复配置了 binlog_format=row，执行这三个命令时，记录的 binlog 还是 statement 格式。binlog 里面就只有一个 truncate/drop 语句，这些信息是恢复不出数据的。

预防：

1. 把 sql_safe_updates 参数设置为 on。这样一来，如果忘记在 delete 或者 update 语句中写 where 条件，或者 where 条件里面没有包含索引字段的话，这条语句的执行就会报错。
2. 代码上线前，必须经过 SQL 审计 

#### 8.1.2 误删库/表

要想恢复数据，就需要使用全量备份，加增量日志的方式了。这个方案要求线上有定期的全量备份，并且实时备份 binlog。 

注：

需要跳过误操作语句的 binlog。

### 8.2 kill 命令

#### 8.2.1 kill 的方式

- kill query + 线程 id; 终止这个线程中正在执行的语句。
- kill connection + 线程 id; connection 可缺省，表示断开这个线程的连接。

#### 8.2.2 kill 的过程

1. 把 session 的运行状态改成 THD::KILL_QUERY(将变量 killed 赋值为THD::KILL_QUERY)；
2. 给 session 的执行线程发一个信号。

隐含的意思： 

1. 一个语句执行过程中有多处“埋点”，在这些“埋点”的地方判断线程状态，如果发现线程状态是 THD::KILL_QUERY，才开始进入语句终止逻辑；
2. 如果处于等待状态，必须是一个可以被唤醒的等待，否则根本不会执行到“埋点”处；
3. 语句从开始进入终止逻辑，到终止逻辑完全完成，是有一个过程的 。

### 8.3 数据库查询与数据发送

MySQL 是“边读边发的”， 取数据和发数据的流程是这样的：
1. 获取一行，写到 net_buffer 中。这块内存的大小是由参数 net_buffer_length 定义的，默认是 16k。
2. 重复获取行，直到 net_buffer 写满，调用网络接口发出去。
3. 如果发送成功，就清空 net_buffer，然后继续取下一行，并写入 net_buffer。
4. 如果发送函数返回 EAGAIN 或 WSAEWOULDBLOCK，就表示本地网络栈（socket send buffer）写满了，进入等待。直到网络栈重新可写，再继续发送。

Sending Data：可能是处于执行器过程中的任意阶段。

Sending to Client：发送数据给客户端。

 

MySQL InnoDB 引擎内部使用改进的 LRU 算法，以应对全表扫描。

### 8.4 临时表

内存表：指使用 Memory 引擎的表，数据都保存在内存中。

临时表：可以使用各种引擎类型，使用 InnoDB 或 MyISAM 时，数据保存在磁盘上。

#### 8.4.1 特性

1. 建表语法是 create temporary table
2. 一个临时表只能被创建它的 session 访问，对其他线程不可见。
3. 临时表可以与普通表同名。
4. session A 内有同名的临时表和普通表的时候，show create 语句，以及增删改查语句访问的是临时表。
5. show tables 命令不显示临时表

#### 8.4.2 重名

创建临时表时，MySQL 要给这个 InnoDB 表创建一个 frm 文件保存表结构定义，还要有地方保存表数据。frm 文件放在临时文件目录下，文件名的后缀是.frm，前缀是“#sql{进程 id}\_{线程id}\_ 序列号”。

可以使用 select @@tmpdir 命令，来显示实例的临时文件目录。 

#### 8.4.3 临时表的主备复制

1. 在 binlog_format='row’的时候，临时表的操作不记录到 binlog 中。
2. 主库上的两个session创建了同名的临时表，binlog 同步到备库时，因为session 的线程 id 不一致，表名不会重复。

#### 8.4.4 内部临时表的使用

union 执行过程中，使用临时表保存子查询的数据；

group by 执行中构造一个带唯一索引的表，执行代价高。

1. 如果语句执行过程可以一边读数据，一边直接得到结果，是不需要额外内存的，否则就需要额外的内存，来保存中间结果；
2. join_buffer 是无序数组，sort_buffer 是有序数组，临时表是二维表结构；
3. 如果执行逻辑需要用到二维表特性，就会优先考虑使用临时表。比如我们的例子中，union 需要用到唯一索引约束， group by 还需要用到另外一个字段来存累积计数。 

### 8.5 Memory 引擎和 InnoDB 引擎

InnoDB 引擎把数据放在主键索引上，其他索引上保存的是主键 id。这种方式，我们称之为索引组织表（Index Organizied Table）。
Memory 引擎采用的是把数据单独存放，索引上保存数据位置的数据组织形式，我们称之为堆组织表（Heap Organizied Table）。 

#### 8.5.1 引擎对比

1. InnoDB 表的数据总是有序存放的，而内存表的数据就是按照写入顺序存放的；
2. 当数据文件有空洞的时候，InnoDB 表在插入新数据的时候，为了保证数据有序性，只能在固定的位置写入新值，而内存表找到空位就可以插入新值；
3. 数据位置发生变化的时候，InnoDB 表只需要修改主键索引，而内存表需要修改所有索引；
4. InnoDB 表用主键索引查询时需要走一次索引查找，用普通索引查询的时候，需要走两次索引查找。而内存表没有这个区别，所有索引的“地位”都是相同的。
5. InnoDB 支持变长数据类型，不同记录的长度可能不同；内存表不支持 Blob 和 Text 字段，并且即使定义了 varchar(N)，实际也当作 char(N)，也就是固定长度字符串来存储，因此内存表的每行数据长度相同。 

#### 8.5.2 锁和数据持久化

内存表不支持行锁，只支持表锁。

内存表在数据库重启时，内存表会被清空；主备同步时，binlog 中会写入 DELETE FROM table。

### 8.6 自增主键

#### 8.6.1 自增值的保存

- MyISAM 引擎的自增值保存在数据文件中。
- InnoDB 引擎的自增值：
  - MySQL 5.7 及之前的版本，自增值保存在内存中。每次重启后，第一次打开表会取 max(id) + 1；若重启前执行 delete 操作删除数据，自增值可能会变小。
  - MySQL 8.0 版本后，自增值的变更记录记录在了 redo log 中，重启的时候可以使用 redo log 恢复之前的值。 

#### 8.6.2 自增值的生成算法

从 auto_increment_offset（默认1）开始，以 auto_increment_increment（默认1）为步长，持续叠加，直到找到第一个大于 X 的值，作为新的自增值。

> 注意点：
>
> 在双 M 结构中，若要求双写，可以设置 auto_increment_increment=2，让两个库的自增主键分别是奇数和偶数。

#### 8.6.3 自增值不连续

- 唯一键冲突，插入失败；
- 事务回滚；
- 批量插入数据时，因插入的数据量不定，id 申请的策略是每次扩大 2 倍。

#### 8.6.4 自增锁

参数 innodb_autoinc_lock_mode，默认值 1。

0：锁的范围是语句级别；

1：插入1条数据的语句在申请后立即释放锁，批量插入数据的语句是语句级别的锁。

2：所有申请主键的动作都申请后立即释放。

> 注意点：
>
> 参数为 2 时，若批量插入数据，且 binlog_format=statement，会导致主备数据不一致。
>
> 批量插入数据，包含的语句类型是 insert … select、replace… select 和 load data 语句 

建议：参数设置为 2，binlog_format = rows。

### 8.7 复制表数据

#### 8.7.1 mysqldump 方法

将数据导出成一组 INSERT 语句：

```sql
mysqldump -h$host -P$port -u$user -add-locks --no-create-info --single-transactiob –set-gtid-purged=OFF db1 t --where="a>900" --result-file=/client_tmp/t.sql
```

主要参数含义如下：
1. –single-transaction 的作用是，在导出数据的时候不需要对表 db1.t 加表锁，而是使用 START TRANSACTION WITH CONSISTENT SNAPSHOT 的方法；
2. –add-locks 设置为 0，表示在输出的文件结果里，不增加" LOCK TABLES t WRITE;"；
3. –no-create-info 的意思是，不需要导出表结构；
4. –set-gtid-purged=off 表示的是，不输出跟 GTID 相关的信息；
5. –result-file 指定了输出文件的路径，其中 client 表示生成的文件是在客户端机器上的。 

如果希望一条 INSERT 语句只插入一行，可以使用参数：–skip-extended-insert。

然后，你可以通过下面这条命令，将这些 INSERT 语句放到 db2 库里去执行。

```sql
mysql -h127.0.0.1 -P13000 -uroot db2 -e "source /client_tmp/t.sql"
```

需要说明的是，source 并不是一条 SQL 语句，而是一个客户端命令。mysql 客户端执行这个命令的流程是这样的：

1. 打开文件，默认以分号为结尾读取一条条的 SQL 语句；
2. 将 SQL 语句发送到服务端执行。

也就是说，服务端执行的并不是这个“source t.sql"语句，而是 INSERT 语句。所以，不论是在慢查询日志（slow log），还是在 binlog，记录的都是这些要被真正执行的 INSERT 语句。 

#### 8.7.2 导出 CSV 文件

将结果导出成 csv 文件。

```sql
select * from db1.t where a>900 into outfile '/server_tmp/t.csv';
```

注意点：
1. 这条语句会将结果保存在服务端。如果你执行命令的客户端和 MySQL 服务端不在同一个机器上，客户端机器的临时目录下是不会生成 t.csv 文件的。
2. into outfile 指定了文件的生成位置（/server_tmp/），这个位置必须受参数 secure_file_priv 的限制。参数 secure_file_priv 的可选值和作用分别是：
  - 如果设置为 empty，表示不限制文件生成的位置，这是不安全的设置；
  - 如果设置为一个表示路径的字符串，就要求生成的文件只能放在这个指定的目录，或者它的子目录；
  - 如果设置为 NULL，就表示禁止在这个 MySQL 实例上执行 select … into outfile 操作。 
3. 这条命令不会帮你覆盖文件，因此你需要确保 /server_tmp/t.csv 这个文件不存在，否则执行语句时就会因为有同名文件的存在而报错。
4. 这条命令生成的文本文件中，原则上一个数据行对应文本文件的一行。但是，如果字段中包含换行符，在生成的文本中也会有换行符。不过类似换行符、制表符这类符号，前面都会跟上“\”这个转义符，这样就可以跟字段之间、数据行之间的分隔符区分开。

得到.csv 导出文件后，可以用下面的 load data 命令将数据导入到目标表 db2.t 中。 

```sql
load data infile '/server_tmp/t.csv' into table db2.t;
```

语句执行的完整流程：

1. 打开文件 /server_tmp/t.csv，以制表符 (\t) 作为字段间的分隔符，以换行符（\n）作为记录之间的分隔符，进行数据读取；
2. 启动事务。
3. 判断每一行的字段数与表 db2.t 是否相同：
4. 重复步骤 3，直到 /server_tmp/t.csv 整个文件读入完成，提交事务。 
5. 主库执行完成后，将 /server_tmp/t.csv 文件的内容直接写到 binlog 文件中。
6. 往 binlog 文件中写入语句 load data local infile ‘/tmp/SQL_LOAD_MB-1-0’ INTO TABLE `db2`.`t`。
7. 把这个 binlog 日志传到备库。
8. 备库的 apply 线程在执行这个事务日志时：
    a. 先将 binlog 中 t.csv 文件的内容读出来，写入到本地临时目录 /tmp/SQL_LOAD_MB-1-0 中；
    b. 再执行 load data 语句，往备库的 db2.t 表中插入跟主库相同的数据

 load data 命令有两种用法：
1. 不加“local”，是读取服务端的文件，这个文件必须在 secure_file_priv 指定的目录或子目录下；
2. 加上“local”，读取的是客户端的文件，只要 mysql 客户端有访问这个文件的权限即可。这时候，MySQL 客户端会先把本地文件传给服务端，然后执行上述的 load data 流程。 

> 注意点
>
> binlog 中使用的是 load data local，有以下两个原因：
>
> 1. 为了确保备库应用 binlog 正常。因为备库可能配置了 secure_file_priv=null，所以如果不用 local 的话，可能会导入失败，造成主备同步延迟。 
>
> 2. 使用 mysqlbinlog 工具解析 binlog 文件，并应用到目标库：
>
>    ```sql
>    mysqlbinlog $binlog_file | mysql -h$host -P$port -u$user -p$pwd
>    ```
>
>    把日志直接解析出来发给目标库执行。增加 local，就能让这个方法支持非本地的 $host。

可以同时导出表结构定义文件和 csv 数据文件，命令的使用方法如下：

```sql
mysqldump -h$host -P$port -u$user ---single-transaction --set-gtid-purged=OFF db1 t --where="a>900" --tab=$secure_file_priv
```



#### 8.7.3 物理拷贝方法

MySQL 5.6 版本引入了可传输表空间(transportable tablespace) 的方法，可以通过导出 + 导入表空间的方式，实现物理拷贝表的功能。
假设我们现在的目标是在 db1 库下，复制一个跟表 t 相同的表 r，具体的执行步骤如下：

1. 执行 create table r like t，创建一个相同表结构的空表；
2. 执行 alter table r discard tablespace，这时候 r.ibd 文件会被删除；
3. 执行 flush table t for export，这时候 db1 目录下会生成一个 t.cfg 文件；
4. 在 db1 目录下执行 cp t.cfg r.cfg; cp t.ibd r.ibd；这两个命令；
5. 执行 unlock tables，这时候 t.cfg 文件会被删除；
6. 执行 alter table r import tablespace，将这个 r.ibd 文件作为表 r 的新的表空间，由于这个文件的数据内容和 t.ibd 是相同的，所以表 r 中就有了和表 t 相同的数据。

至此，拷贝表数据的操作就完成了。这个流程的执行过程图如下： 

![物理拷贝表](/img/in-post/mysql/mysql-note/物理拷贝表.png)

### 8.8 用户权限

#### 8.8.1 全局权限

```sql
grant all privileges on *.* to 'ua'@'%' with grant option;

revoke all privileges on *.* from 'ua'@'%';
```

1. grant 命令对于全局权限，同时更新了磁盘（mysql.user 表）和内存（数组 acl_users）。命令完成后即时生效，接下来新创建的连接会使用新的权限。
2. 对于一个已经存在的连接，它的全局权限不受 grant 命令的影响。 

#### 8.8.2 db 权限

```sql
grant all privileges on db1.* to 'ua'@'%' with grant option;
```

grant 修改 db 权限的时候，是同时对磁盘（mysql.db 表）和内存（数组 acl_dbs）生效的。

每次判断用户的读写权限会遍历数组 acl_dbs，所以已经存在的连接会受影响。

#### 8.8.3 表权限和列权限

表权限定义存放在表 mysql.tables_priv 中，列权限定义存放在表 mysql.columns_priv 中。这两类权限，组合起来存放在内存的 hash 结构 column_priv_hash 中。 

```sql
grant all privileges on db1.t1 to 'ua'@'%' with grant option;
GRANT SELECT(id), INSERT (id,a) ON mydb.mytbl TO 'ua'@'%' with grant option;
```

#### 8.8.4 flush privileges

flush privileges 命令会清空 acl_users 数组，然后从 mysql.user 表中读取数据重新加载，重新构造一个 acl_users 数组。 

使用场景：当数据表中的权限数据跟内存中的权限数据不一致的时候，flush privileges 语句可以用来重建内存数据，达到一致状态。 如直接操作系统表。

### 8.9 分区表

#### 8.9.1 定义

```sql
CREATE TABLE `t` (
`ftime` datetime NOT NULL,
`c` int(11) DEFAULT NULL,
KEY (`ftime`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1
PARTITION BY RANGE (YEAR(ftime))
(PARTITION p_2017 VALUES LESS THAN (2017) ENGINE = InnoDB,
PARTITION p_2018 VALUES LESS THAN (2018) ENGINE = InnoDB,
PARTITION p 2019 VALUES LESS THAN (2019) ENGINE = InnoDB,
PARTITION p_others VALUES LESS THAN MAXVALUE ENGINE = InnoDB);
```

这个表包含了一个.frm 文件和 4 个.ibd 文件，每个分区对应一个.ibd 文件。

- 对于引擎层来说，这是 4 个表；
- 对于 Server 层来说，这是 1 个表。

#### 8.9.2 分区策略

通用分区策略（generic partitioning）：MyISAM 分区表使用的分区策略，每次访问分区都由 server 层控制。 从 5.7.17 开始标记弃用。MySQL 8.0 版本开始，不允许创建 MyISAM 分区表。

本地分区策略（native partitioning）：MySQL 5.7.9 开始，InnoDB 引擎引入。

#### 8.9.3 分区行为

1. MySQL 在第一次打开分区表的时候，需要访问所有的分区；
2. 在 server 层，认为这是同一张表，因此所有分区共用同一个 MDL 锁；
3. 在引擎层，认为这是不同的表，因此 MDL 锁之后的执行过程，会根据分区表规则，只访问必要的分区。

#### 8.9.4 分区表场景

- 对业务透明。

- 清理历史数据方便。速度快、对系统影响小。

  ```sql
  alter table t drop partition …
  ```

  

## 9 实例

### 9.1 查询的字符串长度大于字段长度的情况

```
> CREATE TABLE `table_a` (
`id` int(11) NOT NULL,
`b` varchar(10) DEFAULT NULL,
PRIMARY KEY (`id`),
KEY `b` (`b`)
) ENGINE=InnoDB;

> select * from table where b='1234567890abcd';
```

**（慢）**在这种情况下，MySQL 先截断 b，若匹配到 1234567890，则回表查，查询结果在 Server 层与 1234567890abcd 判断。

