---
title: Zookeeper
date: 2023-08-24 18:06:03
categories: [Data, Integration, Zookeeper]
tags: [知识]
---
## 闲言碎语

对动物园管理员的了解一直停留在“这是一个管理Hadoop各种服务的服务“，但具体是怎么实现管理协调功能的，不太清楚，遂搞清楚一下

<!--more-->

## Apache Zookeeper

Zookeeper是一种分布式协调服务，用于管理大型主机。在分布式环境中协调和管理服务是一个复杂的过程，Zookeeper通过其简单的架构和API解决了这个问题，允许开发人员专注于核心应用程序逻辑，而不用担心应用程序的分布式特性

### 应用场景

1. 分布式协调组件：一旦某个节点发生改变，就会通知所有监听方改变自己的值
2. 分布式锁：依靠ZAB协议实现锁的强一致性
3. 无状态化实现：作为数据中心维护各种状态数据

### 搭建

``` bash
# 下载Zookeeper安装包
wget https://dlcdn.apache.org/zookeeper/zookeeper-3.9.0/apache-zookeeper-3.9.0-bin.tar.gz
# 解压缩
tar -zxvf apache-zookeeper-3.9.0.tar.gz
# 运行服务
cd apache-zookeeper-3.9.0-bin/bin
./zkServer.sh
# 提示Error: JAVA_HOME is not set and java could not be found in PATH.
# 没有安装Java，Zookeeper是Java实现，所以要先安装Java
sudo apt install default-jdk
# 安装默认JRE的话会安装JAVA 11
./zkServer.sh
# 提示../conf/zoo.cfg: No such file or directory
cd ../conf/
cp zoo_sample.cfg zoo.cfg  # 使用示例配置
cd ../bin/
./zkServer.sh start
# 检查是否成功启动
ps -ef|grep zookeeper

# 错误提示
# 提示Starting zookeeper ... FAILED TO START
./zkServer.sh start-foreground
# 查看启动失败原因
# Error: Could not find or load main class org.apache.zookeeper.server.quorum.QuorumPeerMain
# Caused by: java.lang.ClassNotFoundException: org.apache.zookeeper.server.quorum.QuorumPeerMain
# 下载错了包，要下载的是带有‘-bin’的包
wget https://www.apache.org/dyn/closer.lua/zookeeper/zookeeper-3.9.0/apache-zookeeper-3.9.0-bin.tar.gz
tar -zxvf apache-zookeeper-3.9.0-bin.tar.gz
# 提示gzip: stdin: not in gzip format
# 后来发现是链接问题，真正的下载链接要点进去
# 参考文章https://www.amwalle.com/more/working/20210516-gzip-stdin-not-in-gzip-format.html
```

### 常见命令

``` bash
./zkServer.sh status
# 查看服务状态

# status提示Error contacting service. It is probably not running.
# 说明还是没有启动成功
# ./zkServer.sh start-foreground查看原因
# Problem starting AdminServer on address 0.0.0.0, port 8080 and command URL /commands
# 看起来是有个Admin服务想绑定到8080端口但是被占用了
# vim ../conf/zoo.cfg
# 编辑配置文件，加上一行
# admin.serverPort=8081
# ./zkServer.sh start
# ./zkServer.sh status
# 提示Mode: standalone，启动成功

./zkServer.sh stop
# 停止服务状态

./zkCli.sh
# 进入zookeeper控制台
```

### 数据模型

Zookeeper内部保存数据的模型为树模型，数据存储基于节点，节点称为Znode，Znode的引用方式是路径引用，路径类似于文件路径

``` bash
ls /
# [zookeeper]
# 查看根节点

create /test1
ls /
# [test1, zookeeper]
# 创建一个新节点

create /test2
ls /
# [test2, test1, zookeeper]
create /test2/sub2 abc
get /test2/sub2
# abc
create /test2/sub1
ls /test2
# [sub2, sub1]
set /test2/sub1 bbc
get /test2/sub1
# bbc
# 存入数据并获取

ls -R /
# /
# /test1
# /test2
# /zookeeper
# /test2/sub1
# /test2/sub2
# /zookeeper/config
# /zookeeper/quota
# 递归查询所有子节点

delete /test1  # 删除节点，有根节点时无法删除
deleteall /test2  # 删除根节点及其子节点

delete -v 0 /test3  # 乐观锁删除
# 如果这个节点被修改，则不能删除，如果没有，则能够删除
# 并行时Znode可能被其他机器节点修改
```

#### Znode

##### 内容

每个Znode中，包含了四个部分：
1. data：节点中的数据
2. acl：权限
   - c：create创建权限，允许在该节点下创建子节点
   - w：write更新权限，允许更新该节点的数据
   - r：read读取权限，允许读取该节点的内容以及子节点的列表信息
   - d：delete删除权限，允许删除该节点的子节点
   - a：admin管理权限，允许对该节点进行acl权限设置
3. stat：描述当前节点的元数据
4. child：当前节点的子节点

``` bash
# 注册当前会话的账号和密码
addauth digest username:pwd
# 创建节点并设置权限
create /test-node abcd auth:username:pwd:cdwra
# 在另一个会话中，必须先使用账号密码，才能拥有操作该节点的权限
get /test-node
# Insufficient permission : /test-node
# 另一个会话中访问不到
addauth digest username:pwd
get /test-node
# abcd
# 给这个会话添加权限后再进行访问

# 创建节点并设置权限后提示KeeperErrorCode = InvalidACL for /test-node
# 少写了关键词auth:
```

``` bash
[zk: localhost:2181(CONNECTED) 15] get -s /test2
null
cZxid = 0x4  # 创建节点的事务ID
ctime = Wed Aug 23 19:23:09 CST 2023  # 创建节点的时间
mZxid = 0x4  # 修改节点的事务ID
mtime = Wed Aug 23 19:23:09 CST 2023  # 修改节点的时间
pZxid = 0x7  # 添加和删除子节点的事务ID
cversion = 2  #
dataVersion = 0  # 节点内的数据版本，每更新一次版本+1
aclVersion = 0  # 节点内的权限版本
ephemeralOwner = 0x0  # 如果当前节点是临时节点，该值是当前节点所有者的session id，否则为0
dataLength = 0  # 节点数据的长度
numChildren = 2  # 子节点个数
[zk: localhost:2181(CONNECTED) 17] get -s /test2/sub1
abc
cZxid = 0x7
ctime = Wed Aug 23 19:23:19 CST 2023
mZxid = 0x7
mtime = Wed Aug 23 19:23:19 CST 2023
pZxid = 0x7
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 3
numChildren = 0
[zk: localhost:2181(CONNECTED) 16] get -s /test2/sub2
bbc
cZxid = 0x5
ctime = Wed Aug 23 19:23:11 CST 2023
mZxid = 0x8
mtime = Wed Aug 23 19:23:41 CST 2023
pZxid = 0x5
cversion = 0
dataVersion = 1
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 3
numChildren = 0
```

##### 类型

持久节点：创建出的节点在会话结束后依然存在，create path

持久序号（sequence）节点：创建出的节点，根据先后顺序，会在节点之后带上一个数值，越后执行数值越大，适用于分布式锁的应用场景，create -s path

临时（ephemeral）节点：会话结束后自动被删除的节点，通过这个特性可以实现服务注册与发现的效果，create -e path

临时序号节点：适用于临时的分布式锁，create -s -e path

容器（Container）节点：当容器中没有任何子节点，该容器节点会被zk定期删除，create -c path

TTL（Time-To-Live）节点：可以指定节点的到期时间，到期后被zk定期删除，只能通过系统配置zookeeper.extendedTypesEnabled=true开启，create -t ttl path

### 持久化机制

在恢复时先恢复快照文件中的数据到内存中，再用日志文件中的数据做增量恢复，以此来提高回复速度

#### 事务日志

log，zk把执行的命令以日志形式保存在dataLogDir指定的路径中，如果没有指定dataLogDir，则保存到dataDir路径中

#### 数据快照

snapshot，zk会在一定的时间间隔内做一次内存数据的快照，把该时刻的内存数据保存在快照文件中

### 分布式锁

分布式锁其实就是，控制分布式系统不同进程共同访问共享资源的一种锁的实现

如果不同的系统或同一个系统的不同主机之间共享了某个资源，往往需要互斥来防止彼此干扰，以保证一致性

#### 锁的种类

- 读锁：大家都可以读，要想上读锁的前提是之前没有写锁
- 写锁：只有得到写锁才能写，要想上写锁的前提是之前没有任何锁

数据正在进行写入的时候，是不能读取的；数据想要写入，不能正在被读取或写入

##### 读锁

zk如何上读锁
- 创建一个临时序号节点，节点的数据是read，表示是读锁
- 获取当前zk中序号比自己小的所有节点
- 判断最小节点是否是读锁
  - 如果不是读锁，则上锁失败，为最小节点设置监听，阻塞等待，zk的watch机制会在最小节点发生变化时通知当前节点，于是再执行第二步的流程
  - 如果是读锁，则上锁成功

##### 写锁

zk如何上写锁
- 创建一个临时序号节点，节点的数据是write，表示是写锁
- 获取zk中所有的子节点
- 判断自己是否是最小的节点
  - 如果是，则上写锁成功
  - 如果不是，说明前面还有锁，则上锁失败，监听最小节点，如果最小节点有变化，则回到第二步

##### 羊群效应

如果用上述的上锁方式，只要节点发生变化，就会出发其他节点的监听事件，这样的话对zk的压力非常大，调整成链式监听，可以解决该问题，即只监听自己序号的上一个节点，而不是监听最小节点

### Watch机制

可以将Watch理解为注册在特定Znode上的触发器，当这个Znode发生改变，即调用了create、delete、setData方法时，将会出发Znode上注册的对应事件，请求Watch的客户端会接收到异步通知

#### 具体操作

get -w /test客户端对/test这个Znode进行监听

当另一个客户端对被监听节点进行改变时，当前客户端会受到WatchedEvent

#### 交互过程

客户端调用getData方法，参数为true

服务端收到请求，返回节点数据，并且在对应的哈希表里插入被Watch的Znode路径，以及Watcher列表

当被Watch的Znode删除时，服务端会查找哈希表，找到对应Znode的所有Watcher，异步通知客户端，并且删除哈希表中对应的Key-Value

### 集群

Zookeeper追求的是一致性，但不保证强一致性（半数Follower返回ACK），而是保证顺序一致性（依靠事务ID的单调递增）

#### 集群角色

Zookeeper集群中的节点有三种角色
- Leader：处理集群的所有事务请求，集群中只有一个Leader
- Follower：只能处理读请求，可以参与Leader选举
- Observer：只负责读，提升集群的读性能，不参与Leader选举

#### 集群搭建

``` bash
# 1. 在/usr/local/zookeeper下创建四个文件，分别用来保存四个节点的数据
# 分别保存4个节点的myid，并赋值
/usr/local/zookeeper/zkdata/zk1 # echo 1 > myid
/usr/local/zookeeper/zkdata/zk2 # echo 2 > myid
/usr/local/zookeeper/zkdata/zk3 # echo 3 > myid
/usr/local/zookeeper/zkdata/zk4 # echo 4 > myid

# 2. 编写4个zoo.cfg
# 举例在zk2中，配置文件为zoo2.cfg
# dataDir = /usr/local/zookeeper/zkdata/zk2
# client_port = 2182
# 添加4个节点的通信端口和选举端口
# server.1=xxx.xxx.xxx.xxx:2001:3001
# server.2=xxx.xxx.xxx.xxx:2002:3002
# server.3=xxx.xxx.xxx.xxx:2003:3003
# server.4=xxx.xxx.xxx.xxx:2004:3004:observer  # 声明为Observer不参与选举

# 3. 启动集群
./zkServer.sh start ../conf/zoo1.cfg
./zkServer.sh start ../conf/zoo2.cfg
./zkServer.sh start ../conf/zoo3.cfg
./zkServer.sh start ../conf/zoo4.cfg

# 4. 查看节点角色
./zkServer.sh status ../conf/zoo1.cfg
...
```

#### 集群连接

``` bash
./bin/zkCli.sh -server xxx.xxx.xxx.xxx:2181,xxx.xxx.xxx.xxx:2182,xxx.xxx.xxx.xxx:2183
# 连接所有集群节点的client_port
```

### ZAB协议

ZAB（Zookeeper Atomic Broadcast）协议，解决了Zookeeper的崩溃恢复和主从数据同步的问题。

#### 状态

ZAB定义了四种节点状态
- Looking：选举状态
- Following：Follower节点所处的状态
- Leading：Leader节点所处的状态
- Observing：Observer节点所处的状态

#### 启动时的选举

Zookeeper集群的节点在上线时，会进入到Looking状态，当启动的节点大于1台时，就开始进入选举流程

节点1生成自己的选票，选票格式为（myid | zXid），其中myid为配置的节点id，zXid为节点的事务（增删改）执行次数，刚上线时，zXid=0，此时节点1生成的选票为（1 | 0）

节点2生成自己的选票（2 | 0）

节点1和节点2分别将自己的选票发送给对方，此时两个节点中都存在两张选票：（1 | 0）、（2 | 0）

每个节点依次比较所有选票的zXid、myid，将较大的那张选票投入到投票箱中，这个步骤结束后，两个节点的投票箱中存在1张选票，选票为（2 | 0）

由于配置文件中定义了参与选举的节点数为3，所以只有当投票箱中有节点选票过半（3/2 = 1.5），即获得两张选票时，才能选出Leader，于是开始第二轮投票

第二轮开始，每个节点会将手上较大的选票发送给对方，而非第一轮时自己的选票，发送后两个节点中都存在两张选票：（2 | 0）、（2 | 0），其中一张是自己生成的，另一张是对方投递过来的

经过比较两张选票，将较大的投入到投票箱中，这个步骤结束后，两个节点的投票箱中都存在2张选票，选票为（2 | 0）、（2 | 0）

myid为2的节点获票过半，则被选为Leader

此时启动节点3，如果集群已经产生Leader，则节点3会自动将自己作为Follower

#### 崩溃时的选举

Leader建立完成后，会周期性向Follwer发送心跳（Ping命令，没有内容的socket）

当Leader崩溃后，Follower发现socket通道已关闭，于是Follwer会进入到Looking状态，重新回到Leader选举状态，此时集群不能对外提供服务

#### 数据同步

客户端向服务端写数据时，数据会首先传给Leader节点，由Leader节点处理数据的写入

Leader节点先把数据写到自己的数据文件中，并给自己返回一个ACK

Leader把数据发送给所有的Follower

Follwer将数据写到本地数据文件中，并给Leader返回一个ACK

Leader收到半数以上的ACK后向所有的Follower发送Commit

Follwer收到Commit后把数据文件中的数据写到内存中

#### NIO和BIO

BIO，Block-IO是一种同步阻塞的通信模式，NOI，即Non-Block IO，是一种非阻塞同步的通信模式

除此之外还有AOI，Asynchronous IO，异步非阻塞通信模式

BIO适用于连接数目较小且固定的架构，这种方式对服务器资源要求较高，并发局限于应用中

NIO适用于连接数目多且连接比较短的架构，并发局限于应用中

AIO适用于连接数目多且连接比较长的架构，充分调用OS参与并发操作

##### NIO

用于被客户端连接的2181端口，使用的是NIO模式与客户端建立连接

客户端开启Watch时，也是NIO，等待Zookeeper服务器的回调

##### BIO

集群在选举时，多个节点之间的投票通信使用的是BIO

## CAP理论

一个分布式系统最多只能满足一致性（Consistency）、可用性（Availability）和分区容错性（Partition tolerance）这三项中的两项

### 一致性

Consistency，指的是all nodes see the same data at the same time，即更新操作成功并返回客户端完成后，所有节点在同一时间的数据完全一致

### 可用性

Availability，指的是Reads and writes always succeed，即服务一致可用，且是正常相应时间

### 分区容错性

Partition tolerance，指的是the system continues to operate despite arbitrary message loss or failure of part of the system，即分布式系统在遇到某节点或网络分区故障的时候，仍然能够对外提供满足一致性或可用性的服务

## BASE理论

BASE理论是CAP理论的延伸，核心思想是即使无法做到强一致性（Strong Consistency），但应用可以采用适合的方法达到最终一致性（Eventual Consistency）

### 基本可用

Basically Available，指的是分布式系统在出现故障的时候，允许损失部分可用性，保证核心可用

### 软状态

Soft State，指的是允许系统存在中间状态，而该中间状态不会影响系统整体可用性。分布式存储中一般一份数据至少会有三个副本，允许不同节点间副本同步的延迟就是软状态的体现

### 最终一致性

Eventual Consistency，指的是系统中的所有数据副本经过一定时间后，最终能够达到一致的状态

## Curator

Curator是Netflix公司开源的一套Zookeeper客户端框架，封装了大部分Zookeeper的功能，比如Leader选举、分布式锁等，减少了技术人员再使用Zookeeper时的底层细节开发工作

在Java开发时引入依赖即可调用对应的方法进行操作

