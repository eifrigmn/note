# Kafka

kafka是一个开源的流处理平台，为实时数据提供了一个统一、高吞吐量、低延迟的平台。其持久化的本质是一个“按照分布式事务日志架构的大规模发布/订阅消息队列”，此外，Kafka可以通过Kafka Connect连接到外部系统，用于数据的输入/输出。

## 消息系统

一个消息系统是负责将数据从一个应用传递到另外一个应用，应用只需关注于数据，无需关注数据在两个或多个应用间是如何进行传递的。分布式消息传递基于可靠的消息队列，在客户端应用和消息系统之间异步传递消息。

### 数据传递模式

有两种消息传递模式：`点对点`模式和`发布-订阅`模式，kafka使用`发布-订阅`模式。

## 结构

![kafka_struct](../assets/kfk_stucrt.png)

Kafka存储的消息来自任意多个`生产者 (Producer)`，数据从而被分配到不同的`分区 (Partition)`、不同的`Topic`下，在同一个分区内，这些消息被索引并连同时间戳存储在一起。`消费者 (Consumer)`可以从分区查询消息。

## 术语

+ Topic: 用来对消息进行分类，每个进入到Kafka的信息都会被发到一个Topic下

  Topic可以看做一个逻辑上的消息队列，生产者和消费者只需指定topic进行相应的生产/消费行为，无需关心数据存储于何方。

+ Broker: 用来实现数据存储的主机服务器

  Kafka集群包含一个或多个服务器，服务器节点称为broker。

  broker存储Topic的数据，broker数与Partition数量应该相同，如果不匹配，会导致Kafaka集群不均衡。

+ Partition: 每个Topic中的消息会被分为若干个Partition，以提高消息的处理效率

  Topic中的数据被分割成多个Partition进行物理存储，每个Partition中的数据是有序存储的，如果一个Topic有多个Partition，则消费数据时不能保证数据的顺序(理解为多个partition中的数据进行合并，不能保证消息进入topic时的顺序)。

+ Producer: 消息的生产者

  生产者将消息发送到对应的Topic中，broker收到生产者发送的消息后，将其以追加的方式存储到一个partition中，理想情况下，partition与broker应该是一一对应的，因此，生产者发送的消息，由kafka的load balance策略选择适当的broker进行存储。

+ Consumer: 消息的消费者

  顺序读取partition中的message并进行处理，通过offset来标记消息处理的进度。offset的维护可以是Consumer自身维护，也可以存于zookeeper中。

+ Consumer Group: 消息的消费群组

  每个Consumer属于一个特定的Consumer Group (没有指定的则属于默认的group)，

  各个Consumer可以组成一个组。

  当使用Kafka的High-level API时：

  + Partition中的每个message只能被同一个Consumer中的一个Consumer消费，此时kafka相当于一个消息队列

  + 不同组中的Consumer可以消费同一个partition中的消息，此时，kafka相当于一个广播系统。

## 消息投递可靠性

Kafka提供了三种消息投递模式：

### 最多一次

消息不会被重复发送，发送出去就认为数据传输成功，因此不能保证消息被成功投递到broker。

`request.require.acks=0`，生产者不等待broker的ack

### 最少一次

只有当所有节点都接收到消息时，才算投递成功，保证了数据的最可靠投递，但是可能存在数据的重复传输，对性能有影响。

`request.require.acks=-1`：生产者写partition leader成功且所有partition follower写成功后，才算成功。

### 精准一次

Master收到消息即认为消息投递成功，中和了可靠性和性能。

`request.require.acks=1`：生产者写partition leader成功后，broker就返回成功，不等待partition follower的反馈；

`request.require.acks=2`：生产者写partition leader成功，且其中一个partition follower成功后，broker返回成功，不等待剩余的partition follower的反馈。

## 消息消费的可靠性

Kafka提供"At least once"模型来保证Consumer消费消息的可靠性。即便：使用`offset`来维护消费的进度，消费者每消费成功一条消息，会对offset进行记录，下次消费时，会接着上次的位置继续消费。由于是"At least once"，因此，存在当由于异常情况导致Consumer未及时commit offset时，可能存在消息的重复消费情况。

## 应用场景

+ 日志收集
+ 消息系统
+ 流处理
+ 监测指标：如运营指标，用户活动跟踪等



