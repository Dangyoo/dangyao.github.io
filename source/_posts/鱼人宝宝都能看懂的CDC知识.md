---
title: 鱼人宝宝都能看懂的CDC知识
date: 2022-08-25 17:06:03
categories: 知识
tags:  [Data, CDC, DataX, Debezium, Cannal, Binlog]
---
## 闲言碎语

在上家公司（代称N司）时，遇到一个情况：由于传统行业对数据资产的重视程度不够（也可能是安全意识较强），所有开发在数据库进行删除操作时，都是用的真删除，而非互联网公司常见的伪删除。这就造成，数据仓库每天同步全量数据的话，可能会发生主键丢失的情况，同时由于数据库设计以及不同系统间数据不通等各种问题，下游使用数据时就会出现准确性被质疑的情况。

在伪删除方案无法推动的情况下，数据侧考虑获取业务数据库的修改日志，来补齐这部分被删除的信息，所以就去研究了一下常见的方案。

<!--more-->

## 啥叫CDC

CDC全称是Change Data Capture，即变更数据捕获，它是数据库领域常见的技术，主要用于捕获数据库的一些变更，然后把变更数据发送到下游。它的应用比较广，可以做一些数据同步、数据分发和数据采集，还可以做ETL。

## CDC的类型

业界主要分为两种：
1. 基于查询，客户端会通过SQL方式查询源库表变更数据，然后对外发送。这类技术是入侵式的，需要在数据源执行SQL语句，使用这种技术实现CDC会影响数据源的性能，通常需要扫描包含大量记录的整个表。常见工具有Sqoop/DataX/Kafka JDBC Source。
2. 基于日志，这也是业界广泛使用的一种方式，一般是通过binlog方式。变更的记录会写入binlog，解析binlog后会写入消息系统，或直接基于Flink CDC进行处理。这种技术是非入侵性的，不需要在数据源执行SQL语句，通过读取源数据库的日志文件以识别对源库表的创建/修改或删除数据。常见工具有Debezium/Canal/Maxwell。

### DataX

DataX是阿里巴巴开源的一个异构数据源离线同步工具，实现包括 MySQL、Oracle、SqlServer、Postgre、HDFS、Hive、ADS、HBase、TableStore(OTS)、MaxCompute(ODPS)、DRDS等各种异构数据源之间高效的数据同步功能。

异构数据源离线同步指的是将源端数据同步到目的端，但是端与端的数据源类型种类繁多，在没有DataX之前，端与端的链路将组成一个复杂的网状结构，非常零散无法把同步核心逻辑抽象出来。

为了解决异构数据源同步问题，DataX将复杂的网状的同步链路变成了星型数据链路，DataX作为中间传输载体负责连接各种数据源。

所以，当需要接入一个新的数据源的时候，只需要将此数据源对接到DataX，就可以跟已有的数据源做到无缝数据同步。

DataX本身作为离线数据同步框架，采用Framework+plugin架构构建。将数据源读取和写入抽象成为Reader/Writer插件，纳入到整个同步框架中。

<img src="/images/datax1.png" width="50%" height="50%">

- Reader：数据采集模块，负责采集数据源的数据，将数据发送给Framework。
- Writer：数据写入模块，负责不断向Framework取数据，并将数据写入到目的端。
- Framework：它用于连接Reader和Writer，作为两者的数据传输通道，并处理缓冲、并发、数据转换等问题。

#### 核心模块

DataX完成单个数据同步的作业，我们把它称之为Job，DataX接收到一个Job之后，将启动一个进程来完成整个作业同步过程。

DataX Job启动后，会根据不同的源端切分策略，将Job切分成多个小的Task(子任务)，以便于并发执行。

切分多个Task之后，DataX Job会调用Scheduler模块，根据配置的并发数据量，将拆分成的Task重新组合，组装成TaskGroup（任务组）。每一个TaskGroup负责以一定的并发运行完毕分配好的所有Task，默认单个任务组的并发数量为5。

每一个Task都由TaskGroup负责启动，Task启动后，会固定启动Reader->Channel->Writer的线程来完成任务同步工作。

DataX作业运行完成之后，Job监控并等待多个TaskGroup模块任务完成，等待所有TaskGroup任务完成后Job成功退出。否则，异常退出。

<img src="/images/datax2.png" width="50%" height="50%">

#### 调度流程

举例来说，用户提交了一个DataX作业，并且配置了20个并发，目的是将一个100张分表的MySQL数据同步到ODPS里面。 DataX的调度决策思路是：

1. DataX Job根据分库分表切分成了100个Task。
2. 根据20个并发，DataX计算共需要分配4个TaskGroup。
3. 4个TaskGroup平分切分好的100个Task，每一个TaskGroup负责以5个并发共计运行25个Task。

#### 优化

常用的优化参数有：Channel（通道）/SplitPk（切片）/BatchSize（批数据大小）

- Channel - 通道，并发量，该设置对传输效率影响较为明显，设置为1时，即没有并发，此时同步速度均为8.9M/s，将该设置调高之后，速率明显倍增，但增大到一定程度后，瓶颈就转到其他配置了。
- SplitPk - 切片，MySQLReader进行数据抽取时，如果制定SplitPk，表示用户希望使用SplitPk代表的字段进行数据分片，DataX因此会启动并发任务进行数据同步，这样可以大大提高数据同步的效能。推荐SplitPk使用表主键，因为表主键通常情况下比较均匀，因此切分出来的分片也不容易出现数据热点。目前SplitPk仅支持整型数据切分，不支持浮点/字符串/日期等其他类型。如果用户指定其他非支持类型，MySQLReader将报错。如果SplitPk不填写，DataX视作使用单通道同步该表数据，并发需要与Channel设置配合。
- BatchSize - 批数据大小，一次性批量提交的记录数大小，该值可以极大减少DataX与MySQL的网络交互次数，并提升整体吞吐量，默认为1024MB，过大可能会造成DataX运行进程OOM。

#### 优缺点

Datax的优势非常明显：
- 首先，部署非常简单，无论是在物理机上或者虚拟机上，只要网络通畅，便可进行数据同步，给实施人员带来了极大的便利，不受标准数据同步产品部署的条框限制
- 再者，它是开源产品，几乎没有成本
但是，它的缺点也是显而易见的：
- 开源工具更多的是提供基础能力，并不具备任务管理，进度跟踪、校验等等一系列的功能，只能使用者自己通过脚本或者表格记录
- 单机部署，不提供分布式方案，需要通过调度系统解决

### Debezium

当一个应用程序将数据写入到数据库时，变更会被记录在日志文件中，然后数据库的表才会被更新。对于MySQL来说，日志文件是binlog；对于PostgreSQL来说，是write-ahead-log；而对于MongoDB来说，是op日志。
Debezium有针对不同数据库的连接器，所以它能完全理解所有这些日志文件格式的艰巨工作。
Debezium可以读取日志文件，并产生一个通用的抽象事件到消息系统中，如Apache Kafka，其中会包含数据的变化。

<img src="/images/debezium.png" width="50%" height="50%">

### Canal

Canal是阿里开源的一款基于MySQL数据库的binlog增量订阅和消费组件，通过它可以订阅数据库的binlog日志，然后进行一些数据消费，如数据镜像、数据异构、数据索引、缓存更新等。
相对于消息队列，通过这种机制可以实现数据的有序化和一致性。

Canal模拟MySQL slave与MySQL master的交互协议，伪装自己是一个MySQL slave，向MySQL master发送dump协议。MySQL master收到dump请求，开始推送binlog增量日志给Canal。
Canal收到binlog增量日志后，就可以对这部分日志进行解析，获取主库的结构及数据变更。再发送到存储目的地。

<img src="/images/canal.png" width="50%" height="50%">

## Binlog
MySQL日志主要包括错误日志、查询日志、慢查询日志、事务日志、二进制日志几大类。其中比较重要的就是二进制日志binlog（归档日志）、事务日志redo log（重做日志）和undo log（回滚日志）。

- redo log（重做日志）是InnoDB存储引擎独有的，它让MySQL有了崩溃恢复的能力。当MySQL实例挂了或者宕机了，重启的时候InnoDB存储引擎会使用rede log日志恢复数据，保证事务的持久性和完整性。

- binlog（归档日志）是逻辑日志，记录内容是语句的原始逻辑，属于MySQL Server层。所有的存储引擎只要发生了数据更新，都会产生binlog日志。MySQL数据库的数据备份、主备、主主、住从都离不开binlog，需要依赖binlog来同步数据，保证数据一致性。Binlog是记录所有数据库表结构变更以及表数据修改的二进制日志，不会记录SELECT和SHOW这类操作。Binlog日志是以事件形式记录，还包含语句所执行的消耗时间。

- 想要保证事务的原子性，就需要在发生异常时，对已经执行的操作进行回滚，在MySQL中恢复机制是通过undo log（回滚日志）实现的，所有事务进行的修改都会先被记录到这个回滚日志，然后再执行其他相关的操作。如果执行过程中遇到异常的话，我们直接利用回滚日志中的信息将数据回滚到修改之前的样子。并且，回滚日志会先于数据持久化到磁盘上。这样就保证了即使遇到数据库突然宕机等情况，当用户再次启动数据库的时候，数据库还能够通过查询回滚日志来回滚将之前未完成的事务。

### 使用场景

1. 主从复制：在主库中开启Binlog功能，这样主库就可以把Binlog传递给从库，从库拿到Binlog后实现数据恢复达到主从数据一致性。 
2. 数据恢复：通过mysqlbinlog工具来恢复数据。

<img src="/images/binlog.png" width="50%" height="50%">

### 记录模式

Binlog文件名默认为“主机名_binlog-序列号”格式，例如oak_binlog-000001，也可以在配置文件中指定名称。文件记录模式有STATEMENT、ROW和MIXED三种，具体含义如下：
1. STATEMENT（statement-based replication, SBR） - 每一条被修改数据的SQL都会记录到master的Binlog中，slave在复制的时候SQL进程会解析成和原来master端执行过的相同的SQL再次执行。简称SQL语句复制。比如执行一条update T set update_time = now() where id = 1，记录内容如下：
   - 优点：日志量小，减少磁盘IO，提升存储和恢复速度
   - 缺点：在某些情况下会导致主从数据不一致，比如last_insert_id()、now()等函数
2. ROW（row-based replication, RBR） - 日志中会记录每一行数据被修改的情况，然后在slave端对相同的数据进行修改。同一句SQL，记录内容如下：
   - 优点：能清楚记录每一个行数据的修改细节，能完全实现主从数据同步和数据的恢复 
   - 缺点：批量操作，会产生大量的日志，尤其是alter table会让日志暴涨
3. MIXED（mixed-based replication, MBR） - 以上两种模式的混合使用，一般会使用STATEMENT模式保存binlog，对于STATEMENT模式无法复制的操作使用ROW模式保存binlog，MySQL会根据执行的SQL语句选择写入模式。例如上述SQL如果记录未STATMENT模式会由于now()造成数据不一致，所以会记录未MIXED格式。

### 写入机制

binlog的写入时机为事务执行过程中，先把日志写到binlog cache，事务提交的时候再把binlog cache写到binlog文件中（实际先会写入page cache，然后再由fsync写入binlog文件）。

因为一个事务的binlog不能被拆开，无论这个事务多大，也要确保一次性写入，所以系统会给每个线程分配一块内存作为binlog cache。可以通过binlog_cache_size参数控制单线程binlog_cache大小，如果存储内容超过了这个参数，就要暂存到磁盘。

<img src="/images/binlog1.png" width="50%" height="50%">

- 上图的write，是指把日志写入到文件系统的page cache，并没有把数据持久化硬盘，所以速度比较快。
- 上图的 fsync才是将数据库持久化到硬盘的操作。
- 
write和fsync的时机可以由参数sync_binlog控制，可以配置成0、1、N(N>1)。
- 设置成0时：表示每次提交事务都只会write，由系统自行判断什么时候执行fsync。
- 设置成1时：表示每次提交事务都会执行fsync，就和redo log日志刷盘流程一样。
- 设置成N时：表示每次提交事务都会write，但是积累N个事务后才fsync。

### 操作代码

``` SQL
-- binlog状态查看
show variables like 'log_bin';

-- 开启Binlog功能，需要需改my.cnf或my.ini配置文件，在mysqld下面增加log_bin=mysql_bin_log，重启MySQL服务
#log-bin=ON  
#log-bin-basename=mysqlbinlog  
binlog-format=ROW  
log-bin=mysqlbinlog

-- 执行开启语句
set global log_bin=mysqllogbin; 

-- 使用show binlog event命令
show binary logs;                      //等价于show master logs; 
show master status; 
show binlog events; 
show binlog events in 'mysqlbinlog.000001'\G;
结果:
     Log_name: mysql_bin.000001    //此条log存在那个文件中
        Pos: 174                  //log在bin-log中的开始位置 
     Event_type: Intvar          //log的类型信息 
        Server_id: 1            //可以查看配置中的server_id,表示log是那个服务器产生 
     End_log_pos: 202          //log在bin-log中的结束位置 
        Info: INSERT_ID=2     //log的一些备注信息，可以直观的看出进行了什么操作 
-- 可以用mysql自带的工具mysqlbinlog
mysqlbinlog "文件名" mysqlbinlog "文件名" > "文件名比如:test.sql" 

-- 使用binlog恢复数据
// 按指定时间恢复 
mysqlbinlog --start-datetime="2020-04-25 18:00:00" --stop-datetime="2020-04-26 00:00:00" mysqlbinlog.000002 | mysql -uroot -p1234 
// 按事件位置号恢复 
mysqlbinlog --start-position=154 --stop-position=957 mysqlbinlog.000002 | mysql -uroot -p1234
// mysqldump：定期全部备份数据库数据
// mysqlbinlog: 可以做增量备份和恢复操作

-- 删除binlog文件
purge binary logs to 'mysqlbinlog.000001'; // 删除指定文件 
purge binary logs before '2020-04-28 00:00:00'; // 删除指定时间之前的文件 
reset master; // 清除所有文件
// 可以通过设置expire_logs_days参数来启动自动清理功能
// 默认值为0表示没启用。设置为1表示超出1天binlog文件会自动删除掉
```

### Redo Log和Binlog的区别

- Redo Log是属于InnoDB引擎功能，Binlog是属于MySQL Server自带功能，并且是以二进制文件记录。
- Redo Log属于物理日志，记录该数据页更新状态内容，Binlog是逻辑日志，记录更新过程。
- Redo Log日志是循环写，日志空间大小是固定，Binlog是追加写入，写完一个写下一个，不会覆盖使用。
- Redo Log作为服务器异常宕机后事务数据自动恢复使用，Binlog可以作为主从复制和数据恢复使用。Binlog没有自动crash-safe能力。

crash-safe即在InnoDB 存储引擎中，事务提交过程中任何阶段，MySQL突然奔溃，重启后都能保证事务的完整性，已提交的数据不会丢失，未提交完整的数据会自动进行回滚。这个能力依赖的就是redo log和undo log两个日志

## 参考文章

[DataX入门](https://developer.aliyun.com/article/1045779)
[Debezium入门](https://access.redhat.com/documentation/zh-cn/red_hat_integration/2022.q3/html-single/getting_started_with_debezium/index#doc-wrapper)
[Canal入门](https://developer.aliyun.com/article/770496)