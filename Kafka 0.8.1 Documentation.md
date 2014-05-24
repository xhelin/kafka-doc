# 1. 开始上手
## 1.1 介绍

Kafka是一个分布式，分区，带副本的commit log服务系统。它提供的功能有些类似于消息系统，但是在设计上却与大多数消息系统不同。
<!--Kafka is a distributed, partitioned, replicated commit log service. It provides the functionality of a messaging system, but with a unique design.-->
究竟是怎么一回事？
<!--What does all that mean?-->
首先，让我们复习一些基本的消息术语：
<!--First let's review some basic messaging terminology:-->
* Kafka保留的每条消息的分类被称为**主题**（topic）
* 发布相关topic的消息的进程被称为**生产者**（producer）
* 订阅并处理特定topic的消息的进程被称为**消费者**（consumer）
* Kafka集群中的每一个服务进程被称为**代理者**（broker）
<!--* Kafka maintains feeds of messages in categories called topics.-->
<!--* We'll call processes that publish messages to a Kafka topic producers.-->
<!--* We'll call processes that subscribe to topics and process the feed of published messages consumers.-->
<!--* Kafka is run as a cluster comprised of one or more servers each of which is called a broker.-->

所以站在系统高层的角度来看，producers通过网络发送消息给Kafka集群，Kafka集群又将消息发送给consumers，如下图：
<!--So, at a high level, producers send messages over the network to the Kafka cluster which in turn serves them up to consumers like this:-->

![Kafka High level](http://kafka.apache.org/images/producer_consumer.png)

客户端与服务器端的通信采用的是一个简单，高性能，且非特定语言的TCP层之上的协议。我们实现了Kafka的Java客户端，但是很多其他语言的客户端在这儿可以获取到。
<!--Communication between the clients and the servers is done with a simple, high-performance, language agnostic TCP protocol. We provide a Java client for Kafka, but clients are available in many languages.-->

### 主题和日志

让我们先来研究一下Kafka提供的高层抽象之--主题。
<!--Let's first dive into the high-level abstraction Kafka provides—the topic.-->
一个主题是一个消息流的分类或名称。Kafka集群对于每个主题的日志都保存了一系列分区，像这样：
<!--A topic is a category or feed name to which messages are published. For each topic, the Kafka cluster maintains a partitioned log that looks like this:-->

![anatomy of a topic](http://kafka.apache.org/images/log_anatomy.png)

每个分区都是一个有序的，不可变（已有的），且不断追加的消息序列 -- 像commit log。这写分区中的每一个消息都被附上了一个序列号（sequential id number）称为**偏移值(offset)**，并且每条日志的偏移值都是（分区内）唯一的。
<!--Each partition is an ordered, immutable sequence of messages that is continually appended to—a commit log. The messages in the partitions are each assigned a sequential id number called the offset that uniquely identifies each message within the partition.-->
Kafka集群保留一段时间内（可配置）所有发布的消息--不管消息是否已经被消费。举个例子，如果日志被设定为保留2天，那么当一条日志发布之后的两天内都是可以被消费的，再之后就会被清理掉。Kafka的性能和要保存的数据多少不相关因此即使要保存很大量的数据也不会是个问题。
<!--The Kafka cluster retains all published messages—whether or not they have been consumed—for a configurable period of time. For example if the log retention is set to two days, then for the two days after a message is published it is available for consumption, after which it will be discarded to free up space. Kafka's performance is effectively constant with respect to data size so retaining lots of data is not a problem.-->
事实上每个消费者在消费过程中要维护的元信息仅仅只有消息的位置，也就是**偏移值(offset)**，这个偏移值是由消费者控制的：通常情况下消费者在读取消息的同时都会将偏移值往前线性移动，但事实上因为偏移值由消费者控制，所以消费者可以按照它想要的任何顺序来进行数据的读取。例如，消费者可以重新将偏移值设成一个老的值来重新消费。
<!--In fact the only metadata retained on a per-consumer basis is the position of the consumer in in the log, called the "offset". This offset is controlled by the consumer: normally a consumer will advance its offset linearly as it reads messages, but in fact the position is controlled by the consumer and it can consume messages in any order it likes. For example a consumer can reset to an older offset to reprocess.-->
这样的特性组合意味着Kafka的消费者变得非常的轻巧--他们可以随时取数据随时又停止，这样不会影响到整个集群以及其他的消费者。比如你可以使用我们提供的命令行工具来 tail 一个主题新增的数据而不改变任何已经存在的消费者。
<!--This combination of features means that Kafka consumers are very cheap—they can come and go without much impact on the cluster or on other consumers. For example, you can use our command line tools to "tail" the contents of any topic without changing what is consumed by any existing consumers.-->
日志的分区主要由几个目的。第一，这样做可以使得日志的规模超过单台server能够容纳的量。每个独立的分区必须适合它所在的服务，但是一个主题可以拥有很多很多的分区，这样就能够处理任意量的日志。第二，他们（分区）扮演了并行处理的最小单元 -- more on that in a bit（怎么翻译？）。
<!--The partitions in the log serve several purposes. First, they allow the log to scale beyond a size that will fit on a single server. Each individual partition must fit on the servers that host it, but a topic may have many partitions so it can handle an arbitrary amount of data. Second they act as the unit of parallelism—more on that in a bit.-->

### 分布式

日志的分区分布在Kafka集群的server上，每台server管理着分区里的数据以及分区的读取请求。每一个分区都被复制了n份（可配置）到其他server，以保证系统的故障容忍性。
<!--The partitions of the log are distributed over the servers in the Kafka cluster with each server handling data and requests for a share of the partitions. Each partition is replicated across a configurable number of servers for fault tolerance.-->
每个分区都有一个server扮演"leader"(leader)的角色，0到多台server扮演"follower"。leader处理所有的读和写请求，follower被动地复制leader。如果leader挂了，其中一个follower会自动成为leader。每一台server对某些分区而言扮演着leader的角色，对于另外一些分区则扮演着follower的角色，这样集群的负载可以很好地均衡。
<!--Each partition has one server which acts as the "leader" and zero or more servers which act as "followers". The leader handles all read and write requests for the partition while the followers passively replicate the leader. If the leader fails, one of the followers will automatically become the new leader. Each server acts as a leader for some of its partitions and a follower for others so load is well balanced within the cluster.-->

### 生产者

生产者根据他们的处理逻辑发布特定的主题的数据。生产者负责选择消息被发到哪个partition上。这可以是一个round-robin的简单负载均衡策略，也可以是一个消息感知的分区策略（比如根据消息中的某个特定字段）。More on the use of partitioning in a second（怎么翻译？）.
<!--Producers publish data to the topics of their choice. The producer is responsible for choosing which message to assign to which partition within the topic. This can be done in a round-robin fashion simply to balance load or it can be done according to some semantic partition function (say based on some key in the message). More on the use of partitioning in a second.-->

### 消费者

传统来看，消息有两种模型：队列和发布-订阅。在队列模型中，一池子的消费者从一个server中读取消息，每条消息只被发给其中的一个消费者；在发布-订阅模型中，消息被广播给所有的消费者，Kafka提供了一个单消费者的抽象--消费者组，它可以同时兼容两种模型。
<!--Messaging traditionally has two models: queuing and publish-subscribe. In a queue, a pool of consumers may read from a server and each message goes to one of them; in publish-subscribe the message is broadcast to all consumers. Kafka offers a single consumer abstraction that generalizes both of these—the consumer group.-->
消费者用组名标识自己，每一条发布的消息会被传递给所有订阅了这个主题的消费者组，每个消费者组只会有一个消费者接收到这条消息。消费者组中的消费者实例可以是不同的进程或者干脆在不同的机器上。
<!--Consumers label themselves with a consumer group name, and each message published to a topic is delivered to one consumer instance within each subscribing consumer group. Consumer instances can be in separate processes or on separate machines.-->
如果所有的消费者都在同一个组下面，那么就和传统队列模型一样，消费者之间会进行负载均摊。
<!--If all the consumer instances have the same consumer group, then this works just like a traditional queue balancing load over the consumers.-->
如果所有的消费者都有不同的组名，那么就和发布-订阅模型是一样，所有的消息都会被广播给所有的消费者。
<!--If all the consumer instances have different consumer groups, then this works like publish-subscribe and all messages are broadcast to all consumers.-->
更通常的情况是，主题“拥有”一些消费者组，每一个消费者组都是一个逻辑上的订阅者，每一个组由多个消费者组成，这样可以方便进行水平扩展和获得错误容忍。这其实就是发布-订阅模型的语义，只是一个订阅者实际上是由一群消费者组成。
<!--More commonly, however, we have found that topics have a small number of consumer groups, one for each "logical subscriber". Each group is composed of many consumer instances for scalability and fault tolerance. This is nothing more than publish-subscribe semantics where the subscriber is cluster of consumers instead of a single process.-->
Kafka相比传统的消息队列系统，有着更为强的顺序保证。
<!--Kafka has stronger ordering guarantees than a traditional messaging system, too.-->
传统的消息队列按照一定顺序保存消息，如果多个消费者消费同一个队列，那么server会将消息按照他们保存的顺序进行发送。但是虽然如此，消息却被异步地传达到消费者们那里，因此消息在不同的消费者之间可能是乱序到达的。这意味着消息的顺序在并行消费的时候丢失了。消息系统通常采用一些手段绕过这一缺点，比如只允许一个进程消费一个队列，但是这就意味着无法进行并行消费。
<!--A traditional queue retains messages in-order on the server, and if multiple consumers consume from the queue then the server hands out messages in the order they are stored. However, although the server hands out messages in order, the messages are delivered asynchronously to consumers, so they may arrive out of order on different consumers. This effectively means the ordering of the messages is lost in the presence of parallel consumption. Messaging systems often work around this by having a notion of "exclusive consumer" that allows only one process to consume from a queue, but of course this means that there is no parallelism in processing.-->
Kafka做得更好。注意到一个主题拥有多个可并行的分区，Kafka能够做到同时保证顺序，又可以在消费者组之间进行负载均摊。要做到这一点，只需通过将单个分区只分配给一个消费者。这样就能保证单个分区内的数据是被顺序消费的。既然我们有多个分区，我们还是可以将负载均摊到多个消费者实例上。不过要注意一点，消费者的数量不可能比分区数量多。
<!--Kafka does it better. By having a notion of parallelism—the partition—within the topics, Kafka is able to provide both ordering guarantees and load balancing over a pool of consumer processes. This is achieved by assigning the partitions in the topic to the consumers in the consumer group so that each partition is consumed by exactly one consumer in the group. By doing this we ensure that the consumer is the only reader of that partition and consumes the data in order. Since there are many partitions this still balances the load over many consumer instances. Note however that there cannot be more consumer instances than partitions.-->
Kafka只保证分区内的数据是被顺序消费的，并不保证分区之间也如此。单个分区内部的顺序性结合自定义分区逻辑已经足够应付大多数的应用场景。然而如果你一定要求所有的消息都被顺序消费，可以定义这个主题只有一个分区来达到要求，虽然这意味着你只能有一个消费者进行消费。
<!--Kafka only provides a total order over messages within a partition, not between different partitions in a topic. Per-partition ordering combined with the ability to partition data by key is sufficient for most applications. However, if you require a total order over messages this can be achieved with a topic that has only one partition, though this will mean only one consumer process.-->

![consumer group](http://kafka.apache.org/images/consumer-groups.png)

图中，是一个两个server组成的集群，一共包含(P0-P3)4个分区（对某一个主题而言），有两个消费者组订阅了这个主题。组A有两个消费者，组B有4个消费者。
<!--A two server Kafka cluster hosting four partitions (P0-P3) with two consumer groups. Consumer group A has two consumer instances and group B has four. -->
### 保证
Kafka在高层视角上提供了以下几点保证：
<!--At a high-level Kafka gives the following guarantees:-->
同一个生产者生产到同一个分区的数据是和发送顺序一致的。这就是说，如果消息M1和消息M2被同一个生产者生产，而且M1在M2之前发送，那么M1的偏移值（offset）比M2要小。（译者注：M1和M2需要在同一个分区里）
<!--Messages sent by a producer to a particular topic partition will be appended in the order they are sent. That is, if a message M1 is sent by the same producer as a message M2, and M1 is sent first, then M1 will have a lower offset than M2 and appear earlier in the log.-->
一个消费者实例接收到的消息和他们被存储时的顺序是一致的。如果一个主题配置的副本个数是N，我们可以忍受N-1台server的失效而不用担心丢失数据。
A consumer instance sees messages in the order they are stored in the log.
For a topic with replication factor N, we will tolerate up to N-1 server failures without losing any messages committed to the log.
更多的细节参见文档的设计部分。
More details on these guarantees are given in the design section of the documentation.

