Kafka技术内幕：图文详解Kafka源码设计与实现.pdf
深入理解Kafka:核心设计与实践原理.pdf
网盘(优先)：00我的Java知识体系/Kafka/尚硅谷大数据技术之Kafka3.x
网盘：00【2020年6月、8月结课尚硅谷大数据开发】/08-kafka
图灵4期/1视频/11 分布式框架专题-分布式中间件_Rabbitmq_Rocketmq_Kafka/20-Kafka快速实战与基本原理详解-诸葛.mp4
kafka官网
http://kafka.apache.org/
 
官方文档
https://kafka.apache.org/documentation/
 
Kafka简介
kafka最初由linkedin公司开发，使用 scala语言编写
 
默认一个分区num.partitions=1，每个Partition只能被属于同一个Group的Consumer中的一个消费，而一个消费者可以消费多个Partition的数据。
相同消费者组内, 消费者数 > partition
 
如上图，向test发送消息：1，2， 3，4，5，6，7，8，9
只有C1能接收到消息，C2则不能接收到消息，即同一个partition内的消息只能被同一个组中的一个consumer消费。当消费者数量多于partition的数量时，多余的消费者空闲。
也就是说如果只有一个partition你在同一组启动多少个consumer都没用，partition的数量决定了此topic在同一组中被可被均衡的程度，例如partition=4，则可在同一组中被最多4个consumer均衡消费。
相同消费者组内, 消费者数 <= partition 
消费者数量2小于partition的数量3，此时，向test2发送消息1，2，3，4，5，6，7，8，9
C1接收到1，3，4，6，7，9
C2接收到2，5，8
此时P1、P2对对应C1，即多个partition对应一个消费者，C1接收到消息量是C2的两倍
然后，可以在g3组中再启动一个消费者，使得消费者数量为3等于topic2中partition的数量
不同消费者组内
 
如上图，向test2发送消息1，2，3，4，5，6，7，8，9
消息被g3组的消费者均分，g4组的消费者接收到了所有的消息。
g3组：
C1接收到了：2，5，8
C2接收到了：3，6，9
C3接收到了：1，4，7
g4组：
C1接收到了：1，2，3，4，5，6，7，8，9
启动多个组，则会使同一个消息被消费多次

默认保存7天log.retention.hours=168
https://spring.io/projects/spring-kafka#overview
 
项目中用了spring-boot:2.2.7.RELEASE、spring-kafka:2.4.3.RELEASE
 
消息队列的两种模式
（1）点对点模式（一对一，消费者主动拉取数据，消息收到后消息清除，类比考号）
消息生产者生产消息发送到Queue中，然后消息消费者从Queue中取出并且消费消息。
消息被消费以后，queue中不再有存储，所以消息消费者不可能消费到已经被消费的消息。Queue支持存在多个消费者，但是对一个消息而言，只会有一个消费者可以消费。
（2）发布/订阅模式（一对多，消费者消费数据之后不会清除消息，类比试卷）
消息生产者（发布）将消息发布到topic中，同时有多个消息消费者（订阅）消费该消息。和点对点方式不同，发布到topic的消息会被所有订阅者消费。

kafka的核心概念
1.4.0 Broker
一台kafka服务器就是一个broker。一个集群由多个broker组成。
1.4.1 Topic
	Topic 就是数据主题，kafka建议根据业务系统将不同的数据存放在不同的topic中！Kafka中的Topics总是多订阅者模式，一个topic可以拥有一个或者多个消费者来订阅它的数据。一个大的Topic可以分布式存储在多个kafka broker中！Topic可以类比为数据库中的库！

1.4.2 Partition（物理概念，几个分区几个目录）
	每个topic可以有多个分区，通过分区的设计，topic可以不断进行扩展！即一个Topic的多个分区分布式存储在多个broker!
此外通过分区还可以让一个topic被多个consumer进行消费！以达到并行处理！分区可以类比为数据库中的表！
kafka只保证按一个partition中的顺序将消息发给consumer，不保证一个topic的整体（多个partition间）的顺序。
1.4.3 Offset
	数据会按照时间顺序被不断第追加到分区的一个结构化的commit log中！每个分区中存储的记录都是有序的，且顺序不可变！
这个顺序是通过一个称之为offset的id来唯一标识！因此也可以认为offset是有序且不可变的！ 
在每一个消费者端，会唯一保存的元数据是offset（偏移量）,即消费在log中的位置.偏移量由消费者所控制。通常在读取记录后，消费者会以线性的方式增加偏移量，但是实际上，由于这个位置由消费者控制，所以消费者可以采用任何顺序来消费记录。例如，一个消费者可以重置到一个旧的偏移量，从而重新处理过去的数据；也可以跳过最近的记录，从"现在"开始消费。
这些细节说明Kafka 消费者是非常廉价的—消费者的增加和减少，对集群或者其他消费者没有多大的影响。比如，你可以使用命令行工具，对一些topic内容执行 tail操作，并不会影响已存在的消费者消费数据。
 

图1 Topic拓扑结构

 
图2 数据流
1.4.4 持久化
Kafka 集群保留所有发布的记录—无论他们是否已被消费—并通过一个可配置的参数——保留期限来控制。举个例子， 如果保留策略设置为2天，一条记录发布后两天内，可以随时被消费，两天过后这条记录会被清除并释放磁盘空间。
Kafka的性能和数据大小无关，所以长时间存储数据没有什么问题。
1.4.5 副本机制
日志的分区partition （分布）在Kafka集群的服务器上。每个服务器在处理数据和请求时，共享这些分区。每一个分区都会在已配置的服务器上进行备份，确保容错性。
每个分区都有一台 server 作为 “leader”，零台或者多台server作为 follwers 。leader server 处理一切对 partition （分区）的读写请求，而follwers只需被动的同步leader上的数据。当leader宕机了，followers 中的一台服务器会自动成为新的 leader。通过这种机制，既可以保证数据有多个副本，也实现了一个高可用的机制！
基于安全考虑，每个分区的Leader和follower一般会错在在不同的broker!
1.4.6 Producer
消息生产者，就是向kafka broker发消息的客户端。生产者负责将记录分配到topic的指定 partition（分区）中

1.4.7 Consumer
	消息消费者，向kafka broker取消息的客户端。每个消费者都要维护自己读取数据的offset。低版本0.9之前将offset保存在Zookeeper中，0.9及之后保存在Kafka的“__consumer_offsets”主题中。

1.4.8 Consumer Group 
每个消费者都会使用一个消费组名称来进行标识。同一个组中的不同的消费者实例，可以分布在多个进程或多个机器上！ 
如果所有的消费者实例在同一消费组中，消息记录会负载平衡到每一个消费者实例（单播）。即每个消费者可以同时读取一个topic的不同分区！
如果所有的消费者实例在不同的消费组中，每条消息记录会广播到所有的消费者进程(广播)。
如果需要实现广播，只要每个consumer有一个独立的组就可以了。要实现单播只要所有的consumer在同一个组。
一个topic可以有多个consumer group。topic的消息会复制（不是真的复制，是概念上的）到所有的CG，但每个partion只会把消息发给该CG中的一个consumer。
1.5 Kafka基础架构
 

kafka单机搭建
1.从官网http://kafka.apache.org/downloads下载
 
或在虚拟机中cd /usr/local
wget https://archive.apache.org/dist/kafka/1.1.0/kafka_2.11-1.1.0.tgz
2.解压：
tar -zxvf kafka_2.11-1.1.0.tgz
3.先启动ZooKeeper
cd /usr/local/zookeeper-3.4.14/bin
./zkServer.sh start
 
4.再后台启动Kafka
cd /usr/local/kafka_2.11-1.1.0/bin
./kafka-server-start.sh -daemon ../config/server.properties
5.查看启动日志
tail -f /usr/local/kafka_2.11-1.1.0/logs/server.log
kafka的基本操作
创建主题topic-test
bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic  topic-test
然后连接zookeeper，查看主题
cd  /usr/local/zookeeper-3.4.14
bin/zkCli.sh
ls /brokers/topics,如下图
 
或者使用kafka命令查看所有可用的主题
bin/kafka-topics.sh --list --zookeeper localhost:2181
先起消费端，9092默认配置，在server.properties中
 
新版本写法，具体到broker的实例：
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --consumer-property group.id=testGroup --consumer-property client.id=consumer-1 --topic topic-test
老版本写法：
#bin/kafka-console-consumer.sh --zookeeper localhost:2181 --topic topic-test --from-beginning
 
打开一个窗口，发送消息
bin/kafka-console-producer.sh --broker-list localhost:9092 --topic topic-test
出现箭头，输入内容,然后在消费端查看
查看消费组名
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --list
查看消费组的消费偏移量
./kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group testGroup
current-offset：当前消费组的已消费偏移量
log-end-offset：主题对应分区消息的结束偏移量(HW)
lag：当前消费组未消费的消息数
创建多个分区的主题：
bin/kafka‐topics.sh ‐‐create ‐‐zookeeper 192.168.137.130:2181 ‐‐replication‐factor 1 ‐‐partitions 2 ‐‐topic test1
查看下topic的情况
bin/kafka‐topics.sh ‐‐describe ‐‐zookeeper 192.168.137.130:2181 ‐‐topic test1
 
以下是输出内容的解释，第一行是所有分区的概要信息，之后的每一行表示每一个partition的信息。
Leader节点负责给定partition的所有读写请求。
Replicas 表示某个partition在哪几个broker上存在备份。不管这个几点是不是”leader“，甚至这个节点挂了，也会列出。
Isr是replicas的一个子集，它只列出当前还存活着的，并且已同步备份了该partition的节点。
kafka伪集群搭建
复制修改配置文件，主要修改了broker.id、监听端口号listeners（如listeners=PLAINTEXT://:9093）、数据文件位置log.dirs，启动多个实例，自动组建成集群
bin/kafka-server-start.sh -daemon config/server-1.properties
bin/kafka-server-start.sh -daemon config/server-2.properties
 
创建主题
bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 3 --partitions 1 --topic topic-replica-zhuge
 
查看创建的主题
bin/kafka-topics.sh --list --zookeeper localhost:2181
查看主题的详细信息
bin/kafka-topics.sh --describe --zookeeper localhost:2181 --topic topic-replica-zhuge
 
创建包含两个分区的主题
bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 3 --partitions 2 --topic topic-replica-zhuge2
查看主题topic-replica-zhuge2的详细信息
bin/kafka-topics.sh --describe --zookeeper localhost:2181 --topic topic-replica-zhuge2
 
kafka集群搭建
一、搭建过程
1.下载、上传/usr/local、解压
2.修改配置文件/usr/local/kafka_2.11-1.1.0/config/server.properties
a.配置broker的ID
b.打开监听端口，每台服务配置自已机器的IP
listeners=PLAINTEXT://192.168.174.147:9092
c.修改log的目录
log.dirs=/usr/local/kafka_2.11-1.1.0/kafka-logs
d.修改zookeeper.connect
zookeeper.connect=192.168.174.147:2181,192.168.174.148:2181,192.168.174.149:2181/kafka
　　这里配置的是zookeeper集群的IP和端口。
默认Kafka配置会使用ZooKeeper的/路径，如果有其他的应用也在使用ZooKeeper集群，查看ZooKeeper中数据可能会不直观，所以强烈建议指定一个路径，直接在 zookeeper.connect配置项中指定。
而且，需要手动在ZooKeeper中创建路径/kafka，使用如下命令连接到任意一台 ZooKeeper服务器
cd /usr/local/zookeeper-3.4.12/bin
zkCli.sh
然后输入：create /kafka 
这样就创建了路径，每次连接Kafka集群的时候（使用--zookeeper选项），也必须使用带路径的连接字符串。这个命令连接到zookeeper的任何一台机器执行，zookeeper会自动同步到其它集群中的机器。
3.在其它机器上进行配置
二、启动集群并创建topic
1.启动集群
在kafka的三台机器上分别启动Kafka，分别执行如下命令：
cd /usr/local/zookeeper-3.4.12/bin
./kafka-server-start.sh ../config/server.properties &
可以通过查看日志，或者检查进程状态，保证Kafka集群启动成功，启动的时候注意看错误，jps看到Kafka就表示服务已经启动了
2.创建一个名称为bj-replicated-topic5的Topic，5个分区，并且复制因子为3，执行如下命令：
bin/kafka-topics.sh --create --zookeeper 192.168.174.147:2181,192.168.174.148:2181,192.168.174.149:2181/kafka --replication-factor 3 --partitions 5 --topic bj-replicated-topic5
当然，这里也可以只指定一台zookeeper机器，因为集群之间会自动同步。如下所示
bin/kafka-topics.sh --create --zookeeper 192.168.174.147:2181/kafka --replication-factor 3 --partitions 5 --topic bj2-replicated-topic5
4.查看全部topic
bin/kafka-topics.sh -list -zookeeper 192.168.174.147:2181/kafka
5.查看某个topic的详细信息
bin/kafka-topics.sh --describe --zookeeper 192.168.174.147:2181/kafka --topic bj-replicated-topic5
或
bin/kafka-topics.sh --describe --zookeeper 192.168.174.147:2181,192.168.174.148:2181,192.168.174.149:2181/kafka --topic bj-replicated-topic5
 
　　第一行列出了这个topic的总体情况，如topic名称，分区数量，副本数量等。
　　第二行开始，每一行列出了一个分区的信息，如它是第几个分区，这个分区的leader是哪个broker，副本位于哪些broker，有哪些副本处理同步状态。

　　上面Leader、Replicas、Isr的含义如下：
　　a.Partition:分区
　　b.Leader:负责读写指定分区的节点
　　c.Replicas:复制该分区log的节点列表
　　d.Isr:"in-sync" replicas，当前活跃的副本列表（是一个子集），并且可能成为Leader
我们可以通过Kafka自带的bin/kafka-console-producer.sh和bin/kafka-console-consumer.sh脚本，来验证演示如果发布消息、消费消息。
6.在一个终端，启动Producer，并向我们上面创建的名称为bj-replicated-topic5的Topic中生产消息，执行如下脚本：
bin/kafka-console-producer.sh --broker-list 192.168.174.147:9092,192.168.174.148:9092,192.168.174.149:9092--topic bj-replicated-topic5
7.在另一个终端，启动Consumer，并订阅我们上面创建的名称为bj-replicated-topic5的Topic中生产的消息，执行如下脚本：
bin/kafka-console-consumer.sh --bootstrap-server 192.168.174.147:9092,192.168.174.148:9092,192.168.174.149:9092  --topic bj-replicated-topic5
８.修改topic

　　使用--alert原则上可以修改任何配置，以下列出了一些常用的修改选项：
　　a.改变分区数量,partitions只能改大，不能改小
bin/kafka-topics.sh --alter --zookeeper 192.168.174.147:2181/kafka --topic bj2-replicated-topic5 --partitions 6
b.增加、修改或者删除一个配置参数
bin/kafka-topics.sh --alter --zookeeper 192.168.174.147:2181/kafka --topic my_topic_name --config key=value
bin/kafka-topics.sh --alter --zookeeper 192.168.174.147:2181/kafka --topic my_topic_name --delete Config key
9.删除一个topic
bin/kafka-topics.sh --delete --zookeeper 192.168.174.147:2181/kafka --topic bj2-replicated-topic5
a.配置文件中必须delete.topic.enable=true，否则只会标记为删除，在对应的topic后面打一个-marked for deletion标识,而不是真正删除。
 
b.执行此脚本的时候，topic的数据会同时被删除。如果由于某些原因导致topic的数据不能完全删除（如其中一个broker down了）,此时topic只会被marked for deletion，而不会真正删除。此时创建同名的topic会有冲突。
10.停止集群
在kafka的三台机器上分别停止Kafka，分别执行如下命令：
bin/kafka-server-stop.sh
三、高可用
1.某个broker挂掉，本机器可重启
如果一个broker挂掉，且可以重启则处理步骤如下：
a.重启kafka进程
b.执行rebalance（由于已经设置配置项自动执行balance，因此此步骤一般可忽略）
详细分析见下面操作过程：
a.topic的情况
 
集群中有3台机器，id为【11-13】，topic 有5个分区，每个分区3个副本，leader分别位于11，12，13，11，12中。
b.模拟机器down，kill掉进程
　　分区0的leader位于id=11的broker中，kill掉这台机器的kafka进程。
 
c.再次查看topic的情况
 
可以看到，分区0的leader已经移到id=12的机器上了，它的副本位于11，12，13这3台机器上，但处于同步状态的只有id=13和id=12这两台机器。分区3的leader已经移到id=13的机器上了，它的副本位于11，13，12这3台机器上，但处于同步状态的只有id=13和id=12这两台机器。
d.重启kafka进程
bin/kafka-server-start.sh config/server.properties &
e.再次查看状态
 
发现分区0的3个副本都已经处于同步状态，但leader依然为id=12的broker。
f.执行leader平衡
bin/kafka-preferred-replica-election.sh --zookeeper 10.255.34.76:2181/kafka
如果配置文件中新增了这行：
auto.leader.rebalance.enable=true
则此步骤不需要执行。
g.重新查看topic
 
此时分区0和分区3的leader已经回到了id=11的broker，一切恢复正常。
2.某个broker挂掉且无法重启，需要其它机器代替
当一个broker挂掉，需要换机器时，采用以下步骤：
a.将新机器kafka配置文件中的broker.id设置为与原机器一样
b.启动kafka，注意kafka保存数据的目录不会自动创建，需要手工创建
详细分析过程如下：
a.初始化机器，主要包括用户创建，kafka文件的复制等。
b.修改config/server.properties文件
注意，只需要修改一个配置broker.id，且此配置必须与挂掉的那台机器相同，因为kafka是通过broker.id来区分集群中的机器的。此处设为
broker.id=5
c.查看topic的当前状态
 
当前topic有3个分区，其中分区1的leader位于id=5的机器上。
d.关掉id=5的机器
　　kill -9 ** 用于模拟机器突然down
　　或者：
bin/kafka-server-stop.sh
　　用于正常关闭
e.查看topic的状态
 
可见，topic的分区0的leader已经迁移到了id=2的机器上，且处于同步的机器只有一个了。
f.启动新机器
nohup bin/kafka-server-start.sh config/server.properties
g.再看topic的状态
 
　id=5的机器也处于同步状态了，但还需要将leader恢复到这台机器上。
h.执行leader平衡
bin/kafka-preferred-replica-election.sh --zookeeper 192.168.172.98:2181/kafka
如果配置文件中
auto.leader.rebalance.enable=true
则此步骤不需要执行。
i.重新查看topic
 
所有内容都恢复了
四、扩容
将一台机器加入kafka集群很容易，只需要为它分配一个独立的broker id，然后启动它即可。但是这些新加入的机器上面并没有任何的分区数据，所以除非将现有数据移动这些机器上，否则它不会做任何工作，直到创建新topic。因此，当你往集群加入机器时，你应该将其它机器上的一部分数据往这台机器迁移。
五、数据迁移
　以下步骤用于将现有数据迁移到新的broker中，假设需要将test_topic与streaming_ma30_sdc的全部分区迁移到新的broker中（id 为6和7）
1.创建一个json文件，用于指定哪些topic将被迁移过去
cat topics-to-move.json
{"topics": [
 {"topic": "test_topic"},
 {"topic": "streaming_ma30_sdc"}
 ],
 "version":1
}
2.先generate迁移后的结果，检查一下是不是你要想的效果
bin/kafka-reassign-partitions.sh --zookeeper 192.168.172.98:2181/kafka --topics-to-move-json-file topics-to-move.json --broker-list "6,7" --generate
Current partition replica assignment
{"version":1,"partitions":[{"topic":"streaming_ma30_sdc","partition":2,"replicas":[2]},{"topic":"test_topic","partition":0,"replicas":[5,2]},{"topic":"test_topic","partition":2,"replicas":[3,4]},{"topic":"streaming_ma30_sdc","partition":1,"replicas":[5]},{"topic":"streaming_ma30_sdc","partition":0,"replicas":[4]},{"topic":"test_topic","partition":1,"replicas":[2,3]},{"topic":"streaming_ma30_sdc","partition":3,"replicas":[3]},{"topic":"streaming_ma30_sdc","partition":4,"replicas":[4]}]}
 
Proposed partition reassignment configuration
{"version":1,"partitions":[{"topic":"test_topic","partition":0,"replicas":[7,6]},{"topic":"streaming_ma30_sdc","partition":2,"replicas":[7]},{"topic":"test_topic","partition":2,"replicas":[7,6]},{"topic":"streaming_ma30_sdc","partition":1,"replicas":[6]},{"topic":"test_topic","partition":1,"replicas":[6,7]},{"topic":"streaming_ma30_sdc","partition":0,"replicas":[7]},{"topic":"streaming_ma30_sdc","partition":4,"replicas":[7]},{"topic":"streaming_ma30_sdc","partition":3,"replicas":[6]}]}
分别列出了当前的状态以及迁移后的状态。
把这2个json分别保存下来，第一个用来万一需要roll back的时候使用，第二个用来执行迁移。
3.执行迁移
bin/kafka-reassign-partitions.sh --zookeeper 192.168.172.98:2181/kafka --reassignment-json-file expand-cluster-reassignment.json --execute
其中expand-cluster-reassignment.json为保存上面第二段json的文件。
4.查看迁移过程
bin/kafka-reassign-partitions.sh --zookeeper 192.168.172.98:2181/kafka --reassignment-json-file expand-cluster-reassignment.json --verify
Status of partition reassignment:
Reassignment of partition [streaming_ma30_sdc,0] is still in progress
Reassignment of partition [streaming_ma30_sdc,4] is still in progress
Reassignment of partition [test_topic,2] completed successfully
Reassignment of partition [test_topic,0] completed successfully
Reassignment of partition [streaming_ma30_sdc,3] is still in progress
Reassignment of partition [streaming_ma30_sdc,1] is still in progress
Reassignment of partition [test_topic,1] completed successfully
Reassignment of partition [streaming_ma30_sdc,2] is still in progress
5.当所有迁移的完成后，查看一下结果是不是你想要的
bin/kafka-topics.sh --describe --zookeeper 192.168.172.111:2181/kafka --topic test_topic
    Topic:test_topic        PartitionCount:3        ReplicationFactor:2     Configs:
    Topic: test_topic       Partition: 0    Leader: 7       Replicas: 7,6   Isr: 6,7
    Topic: test_topic       Partition: 1    Leader: 6       Replicas: 6,7   Isr: 6,7
    Topic: test_topic       Partition: 2    Leader: 7       Replicas: 7,6   Isr: 6,7
完成

　　以上步骤将整个topic迁移，也可以只迁移其中一个或者多个分区。
以下将test_topic的分区1移到broker id为2，3的机器,分区2移到broker id为4,5的机器。
【其实还是整个topic迁移好一点，不然准备迁移文件会很麻烦】
1.准备迁移配置文件 cat custom-reassignment.json
{"version":1,"partitions":[{"topic":"test_topic","partition":1,"replicas":[2,3]},{"topic":"test_topic","partition":2,"replicas":[4,5]}]}
2.执行迁移
bin/kafka-reassign-partitions.sh --zookeeper 192.168.172.98:2181/kafka --reassignment-json-file custom-reassignment.json --execute
3.查看迁移过程
bin/kafka-reassign-partitions.sh --zookeeper 192.168.172.98:2181/kafka --reassignment-json-file custom-reassignment.json --verify
4.查看迁移结果
bin/kafka-topics.sh --describe --zookeeper 192.168.172.111:2181/kafka --topic test_topic
六、leader的平衡
当一个broker down掉时，所有本来将它作为leader的分区会被将leader转移到其它broker。这意味着当这个broker重启时，它将不再担任何分区的leader，kafka的client也不会从这个broker来读取消息，导致资源的浪费。
　　为了避免这种情况的发生，kafka增加了一个标记：优先副本（preferred replicas）。如果一个分区有3个副本，且这3个副本的优先级别分别为1，5，9，则1会作为leader。为了使kafka集群恢复默认的leader，需要运行以下命令：
bin/kafka-preferred-replica-election.sh --zookeeper 192.168.172.98:2181/kafka
　　或者可以设置以下配置项，leader 会自动执行balance：
auto.leader.rebalance.enable=true
　　这配置默认即为空，但需要经过一段时间后才会触发，约半小时。


Kafka Tool 可视化管理kafka集群服务器
 
报错处理
1.OpenJDK 64-Bit Server VM warning: INFO: os::commit_memory(0x00000000c0000000, 1073741824, 0) failed; error='Out of memory' (errno=12)
https://www.cnblogs.com/liuyupen/p/11189363.html

使用docker 安装kafka时启动失败
 
查看报错日志
# docker logs --since 30m 71846a96e514
 Excluding KAFKA_HOME from broker config
 [Configuring] 'port' in '/opt/kafka/config/server.properties'
 [Configuring] 'advertised.listeners' in '/opt/kafka/config/server.properties'
 [Configuring] 'broker.id' in '/opt/kafka/config/server.properties'
 Excluding KAFKA_VERSION from broker config
 [Configuring] 'listeners' in '/opt/kafka/config/server.properties'
 [Configuring] 'zookeeper.connect' in '/opt/kafka/config/server.properties'
 [Configuring] 'log.dirs' in '/opt/kafka/config/server.properties'
 OpenJDK 64-Bit Server VM warning: INFO: os::commit_memory(0x00000000c0000000, 1073741824, 0) failed; error='Out of memory' (errno=12)

查看内存使用情况
# free -m

 
创建swapfile
# dd  if=/dev/zero  of=swapfile  bs=1024  count=500000  count=空间大小 of空间名字
将swapfile设置为swap空间
# mkswap swapfile
启用交换空间，这个操作有点类似于mount操作
# swapon  swapfile （删除交换空间 swapoff swapfile）
至此增加交换空间的操作结束了，可以使用free命令查看swap空间大小是否发生变化
 
kafka 安装成功~
 

2.在启动kafka的时候报错：count not reserve enough space for 1048576KB object heap
https://blog.csdn.net/yy756127197/article/details/78213498
解决办法：
1.查看下本机的JDK版本，cmd 下运行 java -version
2.找到使用的jdk路径，的bin路径下双击jvisualvm.exe
 
3.进入kafka安装路径下文本打开启动文件kafka-server-start.bat
 
3./usr/local/kafka_2.11-1.1.0/bin/kafka-run-class.sh: line 271: exec: java: not found
 
Kafka的默认/usr/bin/java路径与我们实际的$JAVA_HOME/bin/java路径不一致导致的
有两种修改方式
1.修改我们的实际路径(太麻烦，而且可能会引起其它配置的变化，有些配置中直接使用JAVA_HOME的实际路径，没有使用环境变量)
2.设置一个软连接就可以了
echo $JAVA_HOME
查看JAVA_HOME路径,source /etc/profile
 
建立软链接
ln -s /usr/local/java/jdk1.8.0_202/bin/java /usr/bin/java

4.kafka tool 报 unable to connect broker
使用kafka tool 访问 kafka 时，需要在windows 的host文件里面加上你kafka的主机名，文件路径在/windows/system32/drivers/etc，因为kafka tool是通过主机名来访问的。
===B站尚硅谷Kafka教程===
https://www.bilibili.com/video/BV1a4411B7V9?p=3&spm_id_from=pageDriver
配套资料https://my.oschina.net/jallenkwong/blog/4449224#h2_3
网盘：00我的Java知识体系/Kafka/ B站尚硅谷Kafka教程
