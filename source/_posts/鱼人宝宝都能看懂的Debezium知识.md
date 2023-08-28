---
title: 鱼人宝宝都能看懂的Debezium知识
date: 2023-08-27 17:49:03
categories: 知识
tags:  [Data, CDC, Debezium]
---
## 闲言碎语

针对当前最常用的开源CDC工具Debezium，详细了解后再试用一下

<!--more-->

## 架构

Debezium是一组分布式服务，用于捕获数据库的更改，以便应用程序看到这些更改并作出响应。Debezium在一个变更事件流中记录每个数据库表中所有行级别的变更，应用程序可以读取这些流，来按顺序查看这些变更事件

Debezium构建在Apache Kafka之上，并提供KafkaConnect兼容连接器

架构有三种形式

### 架构一：基于Kafka Connector部署

将Debezium当做Kafka的一个插件，将数据传输到Kafka中，再进行后续的处理，属于当前常用的架构

<img src="/images/debezium-1.png" width="50%" height="50%">

### 架构二：基于Debezium Server部署

做数据传输用，传输到目标数据库中，支持的目标数据库较少，不常用

<img src="/images/debezium-2.png" width="50%" height="50%">

### 架构三：嵌入式引擎部署

可以内置到其他应用中，例如Flink CDC

<img src="/images/debezium-3.png" width="50%" height="50%">

## 搭建

根据所需要监控的数据库，下载对应的Connector，然后启动Zookeeper和Kafka

``` bash
# 启动Zookeeper
./zkServer.sh start
# 启动Kafka
./kafka-server-start.sh -daemon ../config/server.properties
```

### 监控MySQL

首先要去开启MySQL的Binlog

``` bash
sudo vim /etc/my.cnf

# [mysqld]
# server-id=1
# log-bin=mysql-bin
# binlog_format=row

# 重启
sudo systemctl restart mysql

# 进入mysql后检查是否开启成功
> show variables like '%log_bin%';
+---------------------------------+-----------------------------+
| Variable_name                   | Value                       |
+---------------------------------+-----------------------------+
| log_bin                         | ON                          |
| log_bin_basename                | /var/lib/mysql/binlog       |
| log_bin_index                   | /var/lib/mysql/binlog.index |
| log_bin_trust_function_creators | OFF                         |
| log_bin_use_v1_row_events       | OFF                         |
| sql_log_bin                     | ON                          |
+---------------------------------+-----------------------------+
```

然后建立测试用的表

``` sql
CREATE DATABASE testdb;
USE testdb;
CREATE TABLE stu(id INT, name TEXT, age INT);
INSERT INTO stu VALUES(1, 'zs', 10);
```

安装debezium的MySQL Connector

``` bash
cd /xx/debezium/connector
wget https://repo1.maven.org/maven2/io/debezium/debezium-connector-mysql/1.7.1.Final/debezium-connector-mysql-1.7.1.Final-plugin.tar.gz
tar -zxvf debezium-connector-mysql-1.7.1.Final-plugin.tar.gz
# 获得文件夹 /xx/debezium/connector/debezium-connector-mysql/
```

将debezium-connector-mysql作为插件配置到Kafka中

``` bash
cd kafka/kafka_2.13-3.5.1/config/
vim connect-distributed.properties  # 连接器配置
# plugin.path=/xx/debezium/connector

# 修改配置后启动Kafka的连接器
./connect-distributed.sh -daemon ../config/connect-distributed.properties
# 日志为logs/connectDistributed.out

# 查看Kafka服务状态
curl -H "Accept:application/json" localhost:8083  # 8083为connector默认监听端口
# {"version":"3.5.1","commit":"2c6fb6c54472e90a","kafka_cluster_id":"Rh19dkgzT2KO9YIxZb6lOg"}
# 查看连接器状态
curl -H "Accept:application/json" localhost:8083/connectors/
# []
# 连接器为空，接下来需要注册一个连接器
```

注册连接器并查看数据

``` bash
# 配置信息
{
	"name": "mysql-connector",  # 连接器名称
	"config": {
     "connector.class": "io.debezium.connector.mysql.MySqlConnector",  # 官方内置类
     "database.hostname": "localhost",
     "database.port": "3306",
     "database.user": "x",
     "database.password": "x",
     "database.server.id": "1",  # my.cnf中配置的server-id
     "database.server.name": "cr7-demo",
     "include.schema.changes": "true",
     "database.whitelist": "testdb",
     "table.whitlelist": "stu"
     "database.history.kafka.boostrap.servers": "localhost:9092",
     "database.history.kafka.topic": "cr7-schema-changes-inventory"  # 存储DDL表结构变化数据
  }
}
# 把以上配置信息写到了一个json文件中：my-connector.json

# 使用curl注册
curl -i -X POST -H "Content-Type:application/json" --data @my-connector.json localhost:8083/connectors
# HTTP/1.1 201 Created

# 检查连接器
curl -H "Accept:application/json" localhost:8083/connectors/
# ["mysql-connector"]
# 检查连接器运行状态
curl localhost:8083/connectors/mysql-connector/status -s | jq
# {
#   "name": "mysql-connector",
#   "config": {
#     "connector.class": "io.debezium.connector.mysql.MySqlConnector",
#     "database.user": "x",
#     "database.server.id": "1",
#     "database.history.kafka.bootstrap.servers": "xx.xx.xx.xx:9092",
#     "database.history.kafka.topic": "cr7-schema-changes-inventory",
#     "database.server.name": "cr7-demo",
#     "database.port": "3306",
#     "include.schema.changes": "true",
#     "database.hostname": "localhost",
#     "database.password": "x",
#     "table.whitlelist": "stu",
#     "name": "mysql-connector",
#     "database.whitelist": "testdb"
#   },
#   "tasks": [
#     {
#       "connector": "mysql-connector",
#       "task": 0
#     }
#   ],
#   "type": "source"
# }

# 用最新2.3.2Final的时候这一步报错 Error configuring an instance of KafkaSchemaHistory
# 没找到解决办法，后来改成了1.7.1Final版本

# 删除连接器
curl -X DELETE localhost:8083/connectors/test-mysql-connector

# 查看Kafka中的Topic
./kafka-topics.sh --list --bootstrap-server localhost:9092
# cr7-demo
# cr7-demo.testdb.stu
# cr7-schema-changes-inventory

# 查看Topic中的数据
./kafka-console-consumer.sh \
--bootstrap-server localhost:9092 \
--topic cr7-demo.testdb.stu \
--from-beginning

# {
#     "before":null,
#     "after":{
#         "id":1,
#         "name":"zs",
#         "age":10
#     },
#     "source":{
#         "version":"1.7.1.Final",
#         "connector":"mysql",
#         "name":"cr7-demo",
#         "ts_ms":1693128849580,
#         "snapshot":"last",
#         "db":"testdb",
#         "sequence":null,
#         "table":"stu",
#         "server_id":0,
#         "gtid":null,
#         "file":"binlog.000005",
#         "pos":1901,
#         "row":0,
#         "thread":null,
#         "query":null
#     },
#     "op":"r",
#     "ts_ms":1693128849582,
#     "transaction":null
# }

# 查看Topic中的数据
./kafka-console-consumer.sh \
--bootstrap-server localhost:9092 \
--topic cr7-demo \
--from-beginning

# {
#     "source":{
#         "version":"1.7.1.Final",
#         "connector":"mysql",
#         "name":"cr7-demo",
#         "ts_ms":1693128849464,
#         "snapshot":"true",
#         "db":"testdb",
#         "sequence":null,
#         "table":"stu",
#         "server_id":0,
#         "gtid":null,
#         "file":"binlog.000005",
#         "pos":1901,
#         "row":0,
#         "thread":null,
#         "query":null
#     },
#     "databaseName":"testdb",
#     "schemaName":null,
#     "ddl":"CREATE TABLE `stu` (\n  `id` int DEFAULT NULL,\n  `name` text,\n  `age` int DEFAULT NULL\n) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci",
#     "tableChanges":[
#         {
#             "type":"CREATE",
#             "id":"\"testdb\".\"stu\"",
#             "table":{
#                 "defaultCharsetName":"utf8mb4",
#                 "primaryKeyColumnNames":[
#                 ],
#                 "columns":[
#                     {
#                         "name":"id",
#                         "jdbcType":4,
#                         "nativeType":null,
#                         "typeName":"INT",
#                         "typeExpression":"INT",
#                         "charsetName":null,
#                         "length":null,
#                         "scale":null,
#                         "position":1,
#                         "optional":true,
#                         "autoIncremented":false,
#                         "generated":false
#                     },
#                     {
#                         "name":"name",
#                         "jdbcType":12,
#                         "nativeType":null,
#                         "typeName":"TEXT",
#                         "typeExpression":"TEXT",
#                         "charsetName":"utf8mb4",
#                         "length":null,
#                         "scale":null,
#                         "position":2,
#                         "optional":true,
#                         "autoIncremented":false,
#                         "generated":false
#                     },
#                     {
#                         "name":"age",
#                         "jdbcType":4,
#                         "nativeType":null,
#                         "typeName":"INT",
#                         "typeExpression":"INT",
#                         "charsetName":null,
#                         "length":null,
#                         "scale":null,
#                         "position":3,
#                         "optional":true,
#                         "autoIncremented":false,
#                         "generated":false
#                     }
#                 ]
#             }
#         }
#     ]
# }

# 查看Topic中的数据
./kafka-console-consumer.sh \
--bootstrap-server localhost:9092 \
--topic cr7-schema-changes-inventory \
--from-beginning

# {
#   "source" : {
#     "server" : "cr7-demo"
#   },
#   "position" : {
#     "ts_sec" : 1693128848,
#     "file" : "binlog.000005",
#     "pos" : 1901,
#     "snapshot" : true
#   },
#   "databaseName" : "",
#   "ddl" : "SET character_set_server=utf8mb4, collation_server=utf8mb4_0900_ai_ci",
#   "tableChanges" : [ ]
# }
# {
#   "source" : {
#     "server" : "cr7-demo"
#   },
#   "position" : {
#     "ts_sec" : 1693128849,
#     "file" : "binlog.000005",
#     "pos" : 1901,
#     "snapshot" : true
#   },
#   "databaseName" : "testdb",
#   "ddl" : "DROP TABLE IF EXISTS `testdb`.`stu`",
#   "tableChanges" : [ ]
# }
# {
#   "source" : {
#     "server" : "cr7-demo"
#   },
#   "position" : {
#     "ts_sec" : 1693128849,
#     "file" : "binlog.000005",
#     "pos" : 1901,
#     "snapshot" : true
#   },
#   "databaseName" : "testdb",
#   "ddl" : "DROP DATABASE IF EXISTS `testdb`",
#   "tableChanges" : [ ]
# }
# {
#   "source" : {
#     "server" : "cr7-demo"
#   },
#   "position" : {
#     "ts_sec" : 1693128849,
#     "file" : "binlog.000005",
#     "pos" : 1901,
#     "snapshot" : true
#   },
#   "databaseName" : "testdb",
#   "ddl" : "CREATE DATABASE `testdb` CHARSET utf8mb4 COLLATE utf8mb4_0900_ai_ci",
#   "tableChanges" : [ ]
# }
# {
#   "source" : {
#     "server" : "cr7-demo"
#   },
#   "position" : {
#     "ts_sec" : 1693128849,
#     "file" : "binlog.000005",
#     "pos" : 1901,
#     "snapshot" : true
#   },
#   "databaseName" : "testdb",
#   "ddl" : "USE `testdb`",
#   "tableChanges" : [ ]
# }
# {
#   "source" : {
#     "server" : "cr7-demo"
#   },
#   "position" : {
#     "ts_sec" : 1693128849,
#     "file" : "binlog.000005",
#     "pos" : 1901,
#     "snapshot" : true
#   },
#   "databaseName" : "testdb",
#   "ddl" : "CREATE TABLE `stu` (\n  `id` int DEFAULT NULL,\n  `name` text,\n  `age` int DEFAULT NULL\n) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci",
#   "tableChanges" : [ {
#     "type" : "CREATE",
#     "id" : "\"testdb\".\"stu\"",
#     "table" : {
#       "defaultCharsetName" : "utf8mb4",
#       "primaryKeyColumnNames" : [ ],
#       "columns" : [ {
#         "name" : "id",
#         "jdbcType" : 4,
#         "typeName" : "INT",
#         "typeExpression" : "INT",
#         "charsetName" : null,
#         "position" : 1,
#         "optional" : true,
#         "autoIncremented" : false,
#         "generated" : false
#       }, {
#         "name" : "name",
#         "jdbcType" : 12,
#         "typeName" : "TEXT",
#         "typeExpression" : "TEXT",
#         "charsetName" : "utf8mb4",
#         "position" : 2,
#         "optional" : true,
#         "autoIncremented" : false,
#         "generated" : false
#       }, {
#         "name" : "age",
#         "jdbcType" : 4,
#         "typeName" : "INT",
#         "typeExpression" : "INT",
#         "charsetName" : null,
#         "position" : 3,
#         "optional" : true,
#         "autoIncremented" : false,
#         "generated" : false
#       } ]
#     }
#   } ]
# }
```

### 监控PostgreSQL

首先配置PostgreSQL

``` bash
psql -c "show config_file"
# 查找配置文件路径
# /etc/postgresql/13/main/postgresql.conf
sudo vim /etc/postgresql/13/main/postgresql.conf
# 修改listen_address = '*'
# 新增
# wal_level = logical
# max_wal_senders = 2
# max_replication_slots = 1

# 配置pg_hba.conf
sudo vim /etc/postgresql/13/main/pg_hba.conf
# 新增
# host all			all	0.0.0.0/0	scram-sha-256
# host replication	all	0.0.0.0.0	trust

# 重启数据库
sudo systemctl restart postgresql
```

然后建立测试用表

``` sql
! psql
CREATE DATABASE testdb OWNER x;
! psql -d testdb;
CREATE TABLE stu(id INT, name VARCHAR, age INT);
INSERT INTO stu VALUES(1, 'zs', 10);
```

安装PostgreSQL Connector

``` bash
wget https://repo1.maven.org/maven2/io/debezium/debezium-connector-postgres/1.7.1.Final/debezium-connector-postgres-1.7.1.Final-plugin.tar.gz
tar -zxvf debezium-connector-postgres-1.7.1.Final-plugin.tar.gz
cd debezium-connector-postgres/
```

注册连接器并查看数据

``` bash
# 配置信息
{
	"name": "postgresql-connector",  # 连接器名称
	"config": {
     "connector.class": "io.debezium.connector.postgresql.PostgresConnector",  # 官方内置类
     "database.hostname": "localhost",
     "database.port": "5432",
     "database.user": "x",
     "database.password": "x",
     "database.dbname": "testdb",
     "database.server.name": "pgsqldemo",
     "plugin.name": "pgoutput"
  }
}
# 写入postgresql-connector.json文件中
# 重启Kafka连接器，这里不重启的话会报500 Failed to find any class that implements Connector错误

# 使用curl注册
curl -i -X POST -H "Content-Type:application/json" --data @postgresql-connector.json localhost:8083/connectors
# HTTP/1.1 201 Created

# 检查连接器
curl -H "Accept:application/json" localhost:8083/connectors/
# ["postgresql-connector","mysql-connector"]
# 检查连接器运行状态
curl localhost:8083/connectors/postgresql-connector/status -s | jq
# {
#   "name": "postgresql-connector",
#   "connector": {
#     "state": "RUNNING",
#     "worker_id": "127.0.1.1:8083"
#   },
#   "tasks": [
#     {
#       "id": 0,
#       "state": "RUNNING",
#       "worker_id": "127.0.1.1:8083"
#     }
#   ],
#   "type": "source"
# }

# 查看Kafka中的Topic
./kafka-topics.sh --list --bootstrap-server localhost:9092
# pgsqldemo.public.stu

# 查看Topic中的数据
./kafka-console-consumer.sh \
--bootstrap-server localhost:9092 \
--topic pgsqldemo.public.stu \
--from-beginning
# {
#     "before":null,
#     "after":{
#         "id":1,
#         "name":"zs",
#         "age":10
#     },
#     "source":{
#         "version":"1.7.1.Final",
#         "connector":"postgresql",
#         "name":"pgsqldemo",
#         "ts_ms":1693214757722,
#         "snapshot":"last",
#         "db":"testdb",
#         "sequence":"[null,\"23256472\"]",
#         "schema":"public",
#         "table":"stu",
#         "txId":516,
#         "lsn":23256472,
#         "xmin":null
#     },
#     "op":"r",
#     "ts_ms":1693214757727,
#     "transaction":null
# }
```

## 常见问题

snapshot.mode = initial，控制Debezium在首次监控某个表的时候会先同步所有的历史数据，如果在同步历史数据的过程中不能避免数据更新，则使用snapshot.locking.mode = minimal来给表加锁

默认一个表的监控数据会写入一个Topic中，支持通过topic routing设置同样表结构的表可以写入同一个Topic中