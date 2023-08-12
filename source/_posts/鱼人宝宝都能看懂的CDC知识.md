---
title: 鱼人宝宝都能看懂的CDC知识
date: 2022-08-25 17:06:03
categories: 知识
tags: Data
---
## 闲言碎语

在上家公司（代称N司）时，遇到一个情况：由于传统行业对数据资产的重视程度不够（也可能是安全意识较强），所有开发在数据库进行删除操作时，都是用的真删除，而非互联网公司常见的伪删除。这就造成，数据仓库每天同步全量数据的话，可能会发生主键丢失的情况，同时由于数据库问题以及不同系统间数据不通等各种问题，下游使用数据时就会出现准确性被质疑的情况。

在伪删除方案无法推动的情况下，数据侧考虑获取业务数据库的修改日志，来补齐这部分被删除的信息，所以就去研究了一下常见的方案。

## 啥叫CDC

CDC全称是Change Data Capture，即变更数据捕获，它是数据库领域常见的技术，主要用于捕获数据库的一些变更，然后把变更数据发送到下游。它的应用比较广，可以做一些数据同步、数据分发和数据采集，还可以做ETL。

## CDC的类型

业界主要分为两种：
1. 基于查询，客户端会通过SQL方式查询源库表变更数据，然后对外发送。这类技术是入侵式的，需要在数据源执行SQL语句，使用这种技术实现CDC会影响数据源的性能，通常需要扫描包含大量记录的整个表。常见工具有Sqoop/Kafka JDBC Source。
2. 基于日志，这也是业界广泛使用的一种方式，一般是通过binlog方式。变更的记录会写入binlog，解析binlog后会写入消息系统，或直接基于Flink CDC进行处理。这种技术是非入侵性的，不需要在数据源执行SQL语句，通过读取源数据库的日志文件以识别对源库表的创建/修改或删除数据。常见工具有Debezium/Canal/Maxwell。

## Debezium

当一个应用程序将数据写入到数据库时，变更会被记录在日志文件中，然后数据库的表才会被更新。对于MySQL来说，日志文件是binlog；对于PostgreSQL来说，是write-ahead-log；而对于MongoDB来说，是op日志。
Debezium有针对不同数据库的连接器，所以它能完全理解所有这些日志文件格式的艰巨工作。
Debezium可以读取日志文件，并产生一个通用的抽象事件到消息系统中，如Apache Kafka，其中会包含数据的变化。

<img src="/images/debezium.png" width="50%" height="50%">

## Canal

Canal是阿里开源的一款基于MySQL数据库的binlog增量订阅和消费组件，通过它可以订阅数据库的binlog日志，然后进行一些数据消费，如数据镜像、数据异构、数据索引、缓存更新等。
相对于消息队列，通过这种机制可以实现数据的有序化和一致性。

Canal模拟MySQL slave与MySQL master的交互协议，伪装自己是一个MySQL slave，向MySQL master发送dump协议。MySQL master收到dump请求，开始推送binlog增量日志给Canal。
Canal收到binlog增量日志后，就可以对这部分日志进行解析，获取主库的结构及数据变更。再发送到存储目的地。

<img src="/images/canal.png" width="50%" height="50%">

## 参考文章

[Debezium入门](https://access.redhat.com/documentation/zh-cn/red_hat_integration/2022.q3/html-single/getting_started_with_debezium/index#doc-wrapper)
[Canal入门](https://developer.aliyun.com/article/770496)