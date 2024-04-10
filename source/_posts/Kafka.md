---
title: Kafka
date: 2022-09-15 18:06:03
categories: [Data, Integration, Kafka]
tags: [知识]
---
## 闲言碎语

没啥好说的，Kafka算是每个公司都会用到的东西了吧，虽然知道是消息队列，但具体怎么队列的，和其他的比如RabbitMQ有啥区别，确实不太懂。

<!--more-->

## 进程间的通信方式

消息队列（Message queue）是一种进程间通信或同一进程的不同线程间的通信方式，除了消息队列，进程间的通信方式（Inter-Process Communication，简称IPC）还有：无名管道（pipe）、高级管道（popen）、有名管道（named pipe）、信号量（semophore）、共享内存（shared memory）、套接字（socket）

### 无名管道

管道是一种半双工的通信方式，数据只能同一时间单向流动，而且只能在具有亲缘关系的进程间使用，进程间的亲缘关系同化成那个指的是父子进程关系。

#### 半双工

半双工（Half Duplex）是指数据可以在一个信号载体的两个方向上传输，但是不能同时传输。全双工，即信息可以在两个方向上同时传输。单工则只能在一个方向传输。

### 高级管道

讲一个程序当做一个新的进程在当前程序进程中启动，则它算是当前进程的紫禁城，这种方式称之为高级管道。

### 有名管道

也是半双工通信方式，但允许无情缘关系的进程间通信。

### 消息队列

消息队列是消息的链表，存放在内核中并由消息队列标识符标识。消息队列克服了信号传递信息量烧、管道只能承载无格式字节流以及缓冲区大小受限等缺点。

### 信号量

一个计数器，可以用来控制多个进程对共享资源的访问。常作为一个锁机制，防止某进程正在访问共享资源时，其他进程也访问该资源导致冲突。

### 共享内存

指的是映射一段能被其他进程所访问的内存，这段共享内存由一个进程创建，但多个进程都可以访问。共享内存是最快的进程间通信方式，它是针对其他进程间通信方式运行效率低而专门设计的。它往往与其他通信机制如信号量配合使用，来实现进程间的同步和通信。

### 套接字

可以用于不同机器间的进程通信。

## 消息队列

Message Queue（MQ），屏蔽底层复杂的通讯协议，定义了一套应用层更加简单的通讯协议

一个分布式系统中的两个模块之间的通讯要么是HTTP，要么是自己开发的（rcp）TCP

HTTP协议很难实现两端通讯，即A调用B同时B也可以调用A，想要实现的话AB都需要WebServer ，而实现TCP则更加复杂

MQ做的就是在这些协议至上构建一个更高层次的、更简单的生产者/消费者通讯模型，它定义了两个对象，发送数据的生产者和接收数据的消费者，并提供一个SDK来定义而这实现消息通讯而无视底层通讯协议

### 一些相关概念

#### HTTP协议

Hyper Text Transfer Protocal，超文本传输协议，是一个基于TCP/IP通信协议来传递数据的协议，它工作于客户端-服务端架构上，客户端通过URL向服务端（WebServer）发送请求，服务端接收到请求后，向客户端发送响应信息

HTTP是无连接，即限制每次连接只能处理一个请求，服务器处理完客户的请求并收到客户的应答后，即断开连接，采用这种方式可以节省传输时间

HTTP是媒体独立的，即只要客户端和服务器知道如何处理数据内容，任何类型的数据都可以通过HTTP发送，客户端及服务器指定使用合适的MIME-type内容类型

HTTP是无状态协议，指协议对于事务处理没有记忆能力，缺少状态意味着如果后续处理需要前面的信息，则它必须重传，这样可能导致每次连接传送的数据量增大，另一方面，在服务器不需要先前信息时它的应答就较快

#### MIME

Multipurpose Internet Main Extensions，描述消息内容类型的标准，用来标识文档、文件或字节流的性质和格式，浏览器通常使用MIME类型来确定如何处理URL，因此WebServer在响应头中添加正确的MIME类型非常重要，如果配置不正确，浏览器可能会无法解析文件内容，网站将无法正常工作，下载的文件也会被错误处理

MIME类型通用结构为type/subtype，常见的有

text/html -> 超文本标记语言文本

text/plain -> 普通文本

imge/gif -> GIF图形

#### RCP

Remote Copy，远程复制，是在Unix操作系统中用于在计算机之间远程复制一个或多个文件的命令，通过TCP/IP协议传输，已经被更安全的协议和命令所渠道，如SCP（基于安全Shell的安全副本）和SFTP（简单文件传输协议）

#### TCP/IP

Transmission Control Protocol/Internet Protocol，传输控制协议/网际协议，定义了电子设备如何连入因特网，以及数据如何在它们之间传输

TCP/IP中包含一系列用于处理数据通信的协议，如TCP用于应用程序之间的通信，UDP（用户数据报协议）用于应用程序之间的简单通信，IP用于计算机之间的通信，ICMP（因特网消息控制协议）是针对错误和状态的通信，DHCP（动态主机配置协议）是针对动态寻址的协议

TCP负责将应用程序的数据分割并装入IP包，然后在到达的时候重新组合它们，而IP则负责将包发送给接受者

#### SDK

Software Development Kit，软件开发工具包，即可用于开发面向特定平台软件应用程序的工具包，类似于Python中的TensorFlow包，就是谷歌提供的TensorFlow对应的SDK

### 消息队列的分类

#### 有Broker的MQ

有服务器作为Broker，所有消息都通过它中转，生产者把消息发送给Broker就结束了自己的任务，Broker着把消息主动推送给消费者或等待消费者主动轮询

##### 重Topic

Kafka、ActiveMQ（JMS）属于重Topic类型，生产者会发送key和数据到Broker，由Broker比较key之后决定给哪个消费者消费

在这种模式下，一个topc往往是一个较大的概念，甚至一个系统中可能只有一个topic，topic某种意义上就是queue

虽然架构一致，但Kafka的性能比ActiveMQ高很多，所以这种类型的MQ只有Kafka一种备选方案

RocketMQ是阿里基于Kafka重写的，保证消息必答而牺牲了性能，Kafka单机写入在百万条/秒，而RocketMQ则在7万条/秒，更适用于业务处理，而Kafka更适用于日志处理

##### 轻Topic

RabbitMQ（AMQP）属于轻Topic类型，生产者发送key和数据，消费者定义订阅的队列，Broker收到数据之后会通过一定的逻辑计算出key对应的队列，然后把数据交个队列

这种模式下解耦了key和queue，这种架构中queue是非常轻量级的，消费者只关心自己的queue，生产者不用关心数据最终给谁只要指定key就行，中间的映射层在AMQP中称为Exchange交换机

AMQP中有四种Exchange，分别是
- Direct exchange：key等于queue
- Fanout exchange：无视key，给所有queue都来一份
- Topic exchange：key可以模糊匹配queue
- Headers exchange：无视key，通过查看消息头部元数据来决定发给哪个queue

#### 无Broker的MQ

ZeroMQ被设计成了一个库，而非中间件，更加轻量和灵活

节点之间通讯的消息都发送到彼此的队列中，每个节点都既是生产者优势消费者，ZeroMQ做的就是封装出一套类似于Socket的API来完成发送、读取数据

### 消息队列的优势

1. 解耦：允许你独立地扩展或修改两边的处理过程，只要确保它们遵守同样的接口约束；
2. 可恢复性：系统的一部分组件失效时，不会影响到整个系统；
3. 缓冲：有助于控制和优化数据流经过系统的速度，解决生产消息和消费消息的处理速度不一致情况；
4. 灵活性&峰值处理能力：削峰，能够使关键组件顶住突发的访问压力，而不会因为突发的超负荷请求而完全崩溃；
5. 异步通信：提供异步处理机制，允许用户把消息放入队列，而不去处理，在想要去处理的时候再去处理。

### 消息队列的两种模式

1. 点对点模式
一对一，消费者主动拉取数据，消息收到后消息清除。
消息生产者生产消息发送到队列中，然后消息消费者从队列中取出并且消费消息。消息被消费后，队列中不再有存储，所以消息消费者不可能消费到已经被消费的消息。
2. 发布/订阅模式
一对多，消费者消费数据之后不会清除消息。 消息生产者将消息发布到topic（队列）中，同时有多个信息消费者（订阅）消费该消息，根据消费方式不同又分为两种：
   - 消费者主动订阅（Kafka的模式），消费者的消费速度由消费者自己定义，但需要轮询topic中是否有新消息；
   - 队列主动推送（RabbitMQ的模式）。

## Kafka

Kafka最初由Linkedin公司开发，是一个分布式的、支持分区（partition）的、多副本（replica）的，基于Zookeeper协调的发布/订阅模式的消息系统，它最大的特性就是可以实时处理大量数据以满足各种需求场景，如基于Hadoop的批处理系统、低延迟的实时系统、Storm/Flink流式处理引擎、Web/Nginx日志、访问日志、消息服务等，用Scala编写。

### 使用场景

- 日志收集：用Kafka来收集各种服务的Log，通过Kafka以统一接口服务的方式开放给各种消费者
- 消息系统：解耦生产者和消费者，缓存消息
- 用户活动跟踪：记录Web用户或者App用户的各种活动，如浏览网页、搜索、点击等，这些活动信息被各个服务器发布到Kafka的Topic中，然后消费者通过订阅这些Topic来做实时的监控分析，或者装载到Hadoop进行离线分析和挖掘
- 运营指标：Kafka也经常用来记录运营监控数据，包括收集各种分布式应用的数据，生产各种操作的集中反馈，例如报警或警告

### 搭建

``` bash
# 下载包，基于Scala2.13构建的最新3.5.1版本
wget https://downloads.apache.org/kafka/3.5.1/kafka_2.13-3.5.1.tgz
tar -zxvf kafka_2.13-3.5.1.tgz
cd kafka_2.13-3.5.1/config/
# 编辑服务端配置文件
vim server.properties
# listeners=PLAINTEXT://localhost:9092  # 服务器地址和监听端口
# log.dirs=xxx  存储日志文件的地址
# zookeeper.connet=localhost:2181  # 所需连接的Zookeeper端口
# 后台启动
cd ../bin
./kafka-server-start.sh -daemon ../config/server.properties
# 检查是否启动成功
ps -aux | grep server.properties
```

### 基本架构

<img src="/images/kafka.png" width="50%" height="50%">

#### Broker

一台Kafka服务器就是一个Broker，一个集群由多个Broker组成，每个Broker可以容纳多个Topic

#### Producer

消息生产者，将消息push到Kafka集群中的Broker

``` bash
# 启动一个Producer，向Topic-test发送消息
./kafka-console-producer.sh --broker-list localhost:9092 --topic test
```

#### Consumer

消息消费者，从Kafka集群中pull消息，消费消息

``` bash
# 启动一个Consumer，消费Topic-test的消息
# 从最后一条消息的偏移量+1开始消费
./kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test
# 从头开始消费
./kafka-console-consumer.sh \
--bootstrap-server localhost:9092 \
--from-beginning \
--topic test
```

#### Topic

消息的类别或者主题，逻辑上可以理解为队列。Producer只关注push消息到哪个Topic，Consumer只关注订阅了哪个Topic

``` bash
# 创建Topic-test，只有一个分区，备份也是一个
./kafka-topics.sh --create \
--bootstrap-server localhost:9092 \
--replication-factor 1 \
--partitions 1 \
--topic test
# Created topic test.
# 列出所有Topic
./kafka-topics.sh --list --bootstrap-server localhost:9092
# test
# 查看test主题的信息
./kafka-topics.sh --describe --topic test --bootstrap-server localhost:9092
# Topic: test     TopicId: VjYQwUnxSquIDuoHR3HbsQ PartitionCount: 1       ReplicationFactor: 1    Configs: 
# Topic: test     Partition: 0    Leader: 0       Replicas: 0     Isr: 0

# 3.0以下的版本，Topic交给Zookeeper管理
# 创建Topic-test，只有一个分区，备份也是一个
./kafka-topics.sh --create 
--zookeeper localhost:2181 \
--replication-factor 1 \
--partitions 1 \
--topic test
# 列出所有Topic
./kafka-topics.sh --list --zookeeper localhost:2181
```

#### Consumer Group

消费者组，由一到多个Consumer组成，每个Consumer都属于一个Consumer Group，消费者组在逻辑上是一个订阅者。

消费者组内每个消费者负责消费不同分区的数据，一个分区只能由一个组内消费者消费，消费者组之间互不影响。即每条消息只能被Consumer Group中的一个Consumer消费，但是可以被多个Consumer Group组消费，这样就实现了单播和多播。

消费者组的存在提高了同一个Topic的消费能力

``` bash
# 创建一个消费者组中的消费者
./kafka-console-consumer.sh --bootstrap-server localhost:9092 \
--consumer-property group.id=testGroup \
--topic test
# 查看消费者组的信息
./kafka-consumer-groups.sh --bootstrap-server localhost:9092 --list
# testGroup
# 查看某个消费者组的信息
./kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
--describe --group testGroup
# GROUP           TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID                                           HOST            CLIENT-ID
# testGroup       test            0          5               5               0               console-consumer-fe47d017-2097-439e-9534-f0bf4b409345 /127.0.0.1      console-consumer
# 
# CURRENT-OFFSET：该TOPIC被该消费者组消费的最新OFFSET
# LOG-END-OFFSET：该TOPIC最新消息OFFSET
# LAG：LOG-END-OFFSET - CURRENT-OFFSET：多少消息没被消费
```
单播 - 同一条消息只能被同一个消费者组中的一个消费者消费

多播 - 同一条消息能被不同消费者组中的不同消费者消费

#### Partition

负载均衡与扩展性考虑，一个Topic可以分为多个Partition，物理存储在Kafka集群中的多个Broker上。可靠性上考虑，每个Partition都会有备份Replica

Kafka只能在Partition范围内保证消息的局部顺序性，而不能在同一个Topic中的多个Partition中保证总的消费顺序性

Consumer会定期将自己消费分区的offset提交给Kafka内置的topic：__consumer_offsets，提交过去的时候，key是Consumer Group+topic+分区号，value是当前的offset值，Kafka会定期清理topic里的消息，最后就保留最新的那条数据

因为__consumer_offsets可能会接收高并发的请求，Kafka默认分配50个分区（可以通过offsets.topic.num.partition设置）

通过公式（ hash(consumerGroupId)%__consumer_offsets的分区数 ）可以计算Consumer消费的offset要提交到__consumer_offsets的哪个分区

``` bash
./kafka-topics.sh --create 
--bootstrap-server localhost:9092 \
--replication-factor 1 \
--partitions 2 \  # 声明多个分区
--topic test1
# 创建的保存Log的文件夹名为test1-0\test1-1
# .index存储的是数据offset索引
# .timeindex存储的是时间索引
# .log存储的是具体数据
```

#### Replica

Partition的副本，为了保证集群中的某个节点发生故障时，该节点上的Partition数据不会丢失，且Kafka仍能继续工作，所以Kafka提供了副本机制，一个Topic的每个Partition都有若干个副本，一个Leader和若干个Follower

``` bash
./kafka-topics.sh --create 
--bootstrap-server localhost:9092 \
--replication-factor 3 \  # 声明3个副本
--partitions 2 \  # 声明多个分区
--topic test2
# ./kafka-topics.sh --describe --topic test2 --bootstrap-server localhost:9092
# Topic：test2		partitionCount: 2		ReplicationFactor: 3		configs:
# Topic: test2		Partition: 0            Leader: 2		Replicas: 2,0,1		Isr: 2,0,1
# Topic: test2          Partition: 1		Leader: 0		Replicas: 0,1,2         Isr: 0,1,2
```

#### Leader

Replica的主角色，Producer与Consumer只跟Leader交互

#### Follower

Replica的从角色，实时从Leader中同步数据，保持和Leader数据的同步。Leader发生故障时，某个Follower会变成新的Leader

#### Controller

Kafka集群中的其中一台服务器，用来进行Leader election以及各种Failover（故障转移），每个Broker启动时会向Zookeeper创建一个临时序号节点，序号最小的节点将作为集群的Controller
- 当某个分区的Leader副本出现故障时，由Controller负责为该分区选举新的Leader
- 当检测到某个分区的ISR集合发生变化时，由Controller负责通知所有的Broker更新其元数据信息
- 当使用Kafka-topics.sh脚本为某个Topic增加分区数量时，由Controller负责让新分区被其他节点感知到

#### Zookeeper

Kafka通过Zookeeper存储集群的meta等信息（0.9版本之后消费者的offset信息粗处在Kafka系统中）

``` bash
# 进入Zookeeper客户端下
./zkCli.sh
ls /
# [admin, brokers, cluster, config, consumers, 
# controller, controller_epoch, feature, 
# isr_change_notification, latest_producer_id_block, log_dir_event_notification]
# 说明已经成功连接Zookeeper服务
ls /brokers
# [ids, seqid, topics]
ls /brokers/ids
# [0]
```

### 搭建集群

修改server.properties中的配置项

broker.id，listeners，log.dir

启动所有节点

### 配置

#### ACK

Kafka服务端接收到Producer消息之后，返回已接收到的消息元数据

配置ack = 0时，Producer只要把消息发送出去，Kafka就会返回ACK

配置ack = 1时，Producer发送消息到Leader之后，Leader把消息写入到本地文件中，然后返回ACK

配置ack = -1或all时，Producer发送消息到Leader之后，Leader把消息写入到本地文件，且数据被同步到（min.insync.replicas=1）台Follower中后，才会返回ACK

#### Batch

Producer处会生成一个缓冲区（Buffer Memory 默认32M），消息先进入缓冲区中，另外存在一个本地线程去缓冲区中拉取一定大小（Batch Size 默认16K）的数据，缓冲区数据小于16K时，每隔一定时间（Linger ms 默认10ms）拉取剩余数据，发送到Kafka中

#### Offset

Consumer消费消息时，会去Kafka服务端的Topic中拉取（poll）一定长度的消息到Consumer所在服务器，如果设置为自动提交offset，则在消息拉取回来后，不管消息是否已经被消费，就会将offset提交到Kafka服务端，也就是说，自动提交是有可能造成消息丢失的

另一种设置是手动提交offset，指的是Consumer消费完信息后再进行提交，细节又分为两种
- 手动同步提交：消费线程在offset提交前会一直阻塞
- 手动异步提交：消费线程不会因为offset提交未完成而阻塞

### 核心机制

#### ISR

In-Sync Replicas，指所有与Leader保持一定程度（可配置时间/准确度）同步的副本（包括Leader自己），Leader负责维护和跟踪ISR集合中所有Follower的之后状态，当某个Follower滞后太多或失效时，Leader会把它从ISR集合中剔除，当Leader失效时，只有在ISR集合中的副本才有资格被选举为新的Leader

#### Rebalance

前提是消费者没有指明分区消费，此时当消费者组里的消费者和分区关系发生变化时，就会触发Rebalance机制，来重新调整消费者和分区的关系
- Range：通过公式来计算哪个消费者消费哪个分区
- 轮询：轮流消费
- Sticky：先确保原有消费关系不变，再进行调整

#### HW和LEO

High Watermark，高水位，取一个Partition对应的ISR中最小的LEO（LOG-END-OFFSET）作为HW，Consumer最多只能消费到HW所在的位置。另外每个Replica都有HW，Leader和Follower各自负责更新自己的HW状态。对于Leader新写入的消息，Consumer不能立刻消费，Leader会等待该消息被所有ISR中的Replica同步后更新HW，之后Consumer才能进行消费。这样就保证了如果Leader所在的Broker失效，消息仍然能从新选举出的Leader中获得

### JAVA使用Kafka

#### 引入依赖

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>org.example</groupId>
  <artifactId>Kafka</artifactId>
  <version>1.0-SNAPSHOT</version>

  <properties>
    <maven.compiler.source>8</maven.compiler.source>
    <maven.compiler.target>8</maven.compiler.target>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  </properties>

  <dependencies>
    <dependency>
      <groupId>org.apache.kafka</groupId>
      <artifactId>kafka-clients</artifactId>
      <version>2.6.3</version>
      <!-->在SpringBoot中的依赖为<-->
      <!--><groupId>org.springframework.kafka</groupId><-->
      <!--><artifactId>spring-kafka</artifactId><-->
    </dependency>
    <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>slf4j-simple</artifactId>
        <version>1.7.30</version>
        <scope>compile</scope>
    </dependency>
    <!-->这个包用来解决Failed to load class "org.slf4j.impl.StaticLoggerBinder"报错<-->
  </dependencies>

</project>

<!-->import org.apache.kafka.clients<-->
```

#### Producer

``` java
package cn.dy;

import org.apache.kafka.clients.producer.*;
import org.apache.kafka.common.serialization.StringSerializer;

import java.util.Properties;
import java.util.concurrent.ExecutionException;

/**
 * 创建一个Kafka的生产者类
 */
public class MyProducer {

    private final static String TOPIC_NAME = "my-topic";  // 声明要生产的主题名称

    public static void main(String[] args) throws ExecutionException, InterruptedException {
        // 设置参数
        Properties props = new Properties();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "43.153.211.186:9092");
        // 把发送的Key从字符串序列化为字节数组
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        // 把发送消息的Value从字符串序列化为字节数组
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());

        // 创建生产者，传入参数
        Producer<String, String> producer = new KafkaProducer<>(props);

        // 创建消息，Key决定了发往哪个分区，Value是具体发送的消息内容
        ProducerRecord<String, String> producerRecord = new ProducerRecord<>(TOPIC_NAME, "myKey", "HelloWallet");
        
        // 同步发送消息，获取消息元数据并输出
        try {
            RecordMetadata recordMetadata = producer.send(producerRecord).get();  // 使用get方法获取服务端返回的ACK，如果没有返回则会阻塞
            System.out.println("同步发送结果：" + "topic - " + recordMetadata.topic() +
                    " partition - " + recordMetadata.partition() +
                    " offset - " + recordMetadata.offset());
        } catch (InterruptedException e){  // 用try-catch驳货异常
            e.printStackTrace();
            Thread.sleep(1000);  // 重试间隔
            try {
                RecordMetadata recordMetadata = producer.send(producerRecord).get();
            } catch (Exception e1) {
                // 重试失败，进行其他处理
            }
        } catch (ExecutionException e) {
            e.printStackTrace();
        }

        // 异步发送消息，异步容易产生消息丢失，真正使用时同步用得更多
        producer.send(producerRecord, new Callback() {
            @Override
            public void onCompletion(RecordMetadata recordMetadata, Exception e) {
                if (e != null) {
                    System.out.println("消息发送失败：" + e.getStackTrace());
                }
                if (recordMetadata != null) {
                    System.out.println("异步发送结果：" + "topic - " + recordMetadata.topic() +
                            " partition - " + recordMetadata.partition() +
                            " offset - " + recordMetadata.offset());
                }
            }
        });

        Thread.sleep(10000L);  // 主线程关闭不会打印出来，所以延迟关闭
    }
}

/* 执行后报错could not be established. Broker may not be available.
使用命令sudo lsof -i:9092可以看到localhost:9092 (LISTEN)
先检查防火墙是否开启了9092端口
然后修改server.properties中的
listeners=PLAINTEXT://0.0.0.0:9092
表示监听所有IP的访问
此时重启Kafka
使用命令sudo lsof -i:9092可以看到*:9092 (LISTEN)
*/
```

#### Consumer

``` java
package cn.dy;

import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.common.serialization.StringDeserializer;

import java.time.Duration;
import java.util.Arrays;
import java.util.Properties;

public class MyConsumer {
    private static final String TOPIC_NAME = "my-topic";  // 想要消费的Topic名称
    private static final String CONSUMER_GROUP_NAME = "testGroup";  // 所属的消费者组名称

    public static void main(String[] args) {
        Properties props = new Properties();  // 配置信息
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "43.153.211.186:9092");  // 服务IP
        props.put(ConsumerConfig.GROUP_ID_CONFIG, CONSUMER_GROUP_NAME);  // 消费者组
        //props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");  // 是否自动提交offset
        /*默认设置为true，也可以声明自动提交offset的时间间隔
        * props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "true");
        * props.put(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG, "1000");*/

        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        /*当消费主体的是一个新消费者组，或者指定offset的消费方式
        * latest：只消费自己启动之后发送到主题的消息
        * earliest：首次启动时从头开始消费，下次启动根据offset继续消费*/
        //consumer.seekToBeginning 配置则会每次都从头开始消费

        //props.put(ConsumerConfig.HEARTBEAT_INTERVAL_MS_CONFIG, 1000);  // Consumer给broker发送心跳的间隔时间
        //props.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, 10*1000);
        /*Kafka如果超过这个时间没有收到消费者的心跳，就会把消费者踢出消费者组，进行rebalance，将分区分配给其他消费者*/

        //props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 500);  // 一次poll最大拉取消息的条数，根据消费速度来设置
        //props.put(ConsumerConfig.MAX_POLL_INTERVAL_MS_CONFIG, 30*1000);  // 两次poll间隔超过这个事件，Kafka认为消费能力过弱，将其踢出消费者组

        // 把收到的Key从字节数组转化为字符串
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        // 把发送消息的Value从字节数组转化为字符串
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());

        KafkaConsumer<String, String> consumer = new KafkaConsumer<String, String>(props);  // 创建消费者客户端
        consumer.subscribe(Arrays.asList(TOPIC_NAME));  // 订阅主题列表

        while (true) {
            ConsumerRecords<String, String> consumerRecords = consumer.poll(Duration.ofMillis(1000));  // 拉取消息的长轮询
            for (ConsumerRecord<String, String> consumerRecord: consumerRecords) {
                // 打印消息
                System.out.printf("收到消息：partition = %d， offset = %d， key = %s， value = %s%n",
                                  consumerRecord.partition(), consumerRecord.offset(), consumerRecord.key(), consumerRecord.value());
            }
        }
    }
}
```

### 线上问题优化

#### 防止消息丢失

Producer：ack设置为-1或all，确保足够多的副本都完成消息同步再返回ACK

Consumer：将自动提交改为手动提交，确保消息被消费后再更新offset

#### 防止消息重复消费

Producer：当Producer发送消息后，由于网络问题未收到Kafka服务端返回的ACK，则会进行重试，此时服务端就会造成消息重复，如果从Producer端考虑防止，则应该关闭重试机制，但为了确保消息不丢失，又不应该关闭重试，所以应该考虑在Consumer端进行解决

Consumer：当Consumer收到消息后，需要一个机制来确保当前消息与之前已经消费过的消息不重复，可以通过幂等性保证来解决，例如通过主键来判断重复性或使用Redis/Zookeeper的分布式锁（主流方案）

##### 幂等性

在数学中某一元运算为幂等时，其作用在任一元素两次后会和其作用一次的结果相同。在软件工程中，指的是函数/接口可以使用相同的参数重复执行，不影响系统的状态也不会对系统造成改变

幂等性保证需要满足三个条件：
- 请求唯一标识，即每一个请求必须有一个唯一标识； 
- 处理唯一标识，即每次处理完请求之后，必须有一个记录标识这个请求被处理过了
- 逻辑判断处理，即每次接收请求需要进行判断之前是否处理过，也就是根据请求唯一标识查询是否存在处理唯一标识

幂等性常见的实现方案有：
1. token机制：针对客户端重复连续多次点击的情况，提交接口需要通过token机制实现防止重复提交，主要流程为：
   a. 服务端提供生成请求token的接口，在存在幂等问题的业务执行前，向服务器请求获取token，服务器会把token保存到Redis中
   b. 调用业务接口请求时，把token携带过去，一般放在请求头中
   c. 服务器判断请求token是否存在于Redis中，存在则表示为第一次请求，这时把Redis中的token删除，继续执行业务；如果判断token不存在于Redis，则表示是重复操作，直接返回重复标记给客户端
2. 数据库索引唯一：往数据库表里插入数据的时候，利用数据库的唯一索引特性来保证唯一性
3. Redis实现：将唯一序列号作为key存入Redis，在请求处理前先查看key是否存在，不存在则表示为处理过，存在则表示已经处理过
4. 状态机：在业务处理中，通过业务流转状态控制请求的幂等
5. 分布式锁：[一篇文章](https://mp.weixin.qq.com/s?__biz=MzUyNzgyNzAwNg==&amp;mid=2247484334&amp;idx=1&amp;sn=4429f5810752012233bf8d5718181e2c&amp;scene=21#wechat_redirect)

#### 实现顺序消费

Producer：ack不能设置为0，关闭重试，使用同步发送，确保消息是顺序发送的

Topic：设置为单一分区，因为Kafka只确保单一分区中的消息有序

Consumer：由于是单一分区，此时同一个消费者组里只能有一个消费者，这个消费者消费到的消息是顺序的

#### 解决消息积压

Producer的生产速度超过Consumer消费速度，会导致消息积压，进而引发Kafka性能问题或服务崩溃，一般由以下方案：
1. 在一个消费者中启动多个线程进行消费，提升单一消费者的消费能力；
2. 在一个或多个服务器上启动多个消费者，提高消费能力；
3. 让一个消费者将积压Topic中的消息发送到另一个新的Topic中，新Topic设置多个分区和多个消费者

#### 实现延迟队列

业务场景：如果订单创建30分钟内没有付款，则需要取消订单。这个业务场景可以通过延时队列来实现
- 创建多个Topic，每个Topic表示延时的间隔
   - topic_5s：延时5s后需要执行操作的消费者需要消费的主题
  - topic_1m：延时1min后需要执行操作的消费者需要消费的主题
  - topic_30m：延时30min后需要执行操作的消费者需要消费的主题
  - Producer将消息发送到相应的Topic中，并带上消息的发送时间
- 消费者订阅相应的Topic，消费时轮询Topic中的消息
  - 如果消息的发送时间和当前消费时间之差超过预设值，则进行某些操作（如超过30min，则去数据库判断订单是否已付款，如未付款则标注这个订单已取消）
  - 如果未超过预设值，则不消费当前offset及之后的消息
  - 等待一定的时间之后，继续从上次消费截止的offset开始poll消息进行判断

### KafkaEagle监控平台

``` bash
# 下载
wget https://github.com/smartloli/kafka-eagle-bin/archive/v3.0.1.tar.gz
# 解压
tar -zvxf v3.0.1.tar.gz
cd kafka-eagle-bin-3.0.1/
tar -zxvf efak-web-3.0.1-bin.tar.gz 
mv efak-web-3.0.1 ../../kafka-eagle-web

# 配置
vim /etc/profile
# 增加KE_HOME和PATH配置
# export KE_HOME=../kafka-eagle-web
# export PATH=$PATH:$KE_HOME/bin
# ke配置文件修改
vim conf/system-config.properties
# Zookeeper相关配置
# efak.zk.cluster.alias=cluster1
# cluster1.zk.list=xxx.xxx.xxx.xxx:2181
# 数据库相关配置，配置为MySQL，在MySQL中创建对应的库
# 主要用来保存Eagle的元数据
# efak.driver=com.mysql.cj.jdbc.Driver
# efak.url=jdbc:mysql://...
# efak.username
# efak.password

# 启动
./ke.sh start

# 启动时报错The KE_HOME environment variable is not defined correctly.
# 需要重新加载系统配置
# source /etc/profile
# 再次启动报错The JAVA_HOME environment variable is not defined correctly.
# 需要在/etc/profile里配置JAVA_HOME
# 查询JAVA安装路径
# which java
# /usr/bin/java
# ls -lrt /usr/bin/java
# /usr/bin/java -> /etc/alternatives/java
# ls -lrt /etc/alternatives/java
# /etc/alternatives/java -> /usr/lib/jvm/java-11-openjdk-amd64/bin/java
# 设置JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
```

### 应用场景

### Flume对接Kafka

生产环境中，更多的是将数据生产到日志之中，从日志中获取数据，一般使用Flume。如果有多个消费者需要这个数据，则需要一个能够支持动态添加的中间件，也就是Kafka。使用Flume+Kafka，就可以实现一次采集，给多个消费者消费。

Flume不支持副本事件。于是，如果Flume代理的一个节点奔溃了，即使使用了可靠的文件管道方式，你也将丢失这些事件直到你恢复这些磁盘。如果你需要一个高可靠行的管道，那么使用Kafka是个更好的选择。

如果只是想把日志数据存储到HDFS，则可以Flume直接对接HDFS。

## Kafka和RabbitMQ的差异

1. RabbitMQ是一个分布式消息代理，从多个来源收集流式处理数据，然后将其路由到不同的目标进行处理。而Kafka是一个流式处理平台，用于构建实时数据管道和流失处理应用程序，功能要比RabbitMQ复杂；
2. RabbitMQ中生产者发送并监控消息是否达到目标消费者。而对于Kafka来说，无论消费者是否检索消息，生产者都会向队列发布消息；
3. RabbitMQ允许生产者使用优先队列升级某些消息，即不按照先入先出的顺序发送，而是按照优先级来处理消息。而Kafka平等对待这些消息；
4. RabbitMQ中生产者收到消费者发送的ACK后会将消息从队列中删除。而Kafka数据会一直保存到硬盘直到保留期限到期；
5. 在消息传输容量方面，Kafka优于RabbitMQ。
