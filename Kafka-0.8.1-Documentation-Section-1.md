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

客户端与服务器端的通信采用的是一个简单，高性能，且非特定语言的TCP层之上的[协议](https://cwiki.apache.org/confluence/display/KAFKA/A+Guide+To+The+Kafka+Protocol)。我们实现了Kafka的Java客户端，但是很多[其他语言](https://cwiki.apache.org/confluence/display/KAFKA/Clients)的客户端在这儿可以获取到。
<!--Communication between the clients and the servers is done with a simple, high-performance, language agnostic [TCP protocol](https://cwiki.apache.org/confluence/display/KAFKA/A+Guide+To+The+Kafka+Protocol). We provide a Java client for Kafka, but clients are available in [many languages](https://cwiki.apache.org/confluence/display/KAFKA/Clients).-->

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

传统来看，消息有两种模型：[队列](http://en.wikipedia.org/wiki/Message_queue)和[发布-订阅](http://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern)。在队列模型中，一池子的消费者从一个server中读取消息，每条消息只被发给其中的一个消费者；在发布-订阅模型中，消息被广播给所有的消费者，Kafka提供了一个单消费者的抽象--消费者组，它可以同时兼容两种模型。
<!--Messaging traditionally has two models: [queuing](http://en.wikipedia.org/wiki/Message_queue) and [publish-subscribe](http://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern). In a queue, a pool of consumers may read from a server and each message goes to one of them; in publish-subscribe the message is broadcast to all consumers. Kafka offers a single consumer abstraction that generalizes both of these—the consumer group.-->
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
<!--A consumer instance sees messages in the order they are stored in the log.
For a topic with replication factor N, we will tolerate up to N-1 server failures without losing any messages committed to the log.-->
更多的细节参见文档的设计部分。
<!--More details on these guarantees are given in the design section of the documentation.-->

## 1.2 使用场景

以下是使用Kafka的几个普通的场景。如果想要了解更多场景请参看这篇[blog](http://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying)。
<!--Here is a description of a few of the popular use cases for Apache Kafka. For an overview of a number of these areas in action, see this [blog post](http://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying).-->

### 消息系统

Kafka可以替换传统的消息代理（message broker）。使用消息代理有很多原因(为了将处理消息和生产消息解耦开，缓存未被处理的消息等等)。对比大多数的消息系统，Kafka拥有更大的吞吐，内置分区，副本，以及错误容忍，这些特性都是选择Kafka作为大型可扩展消息处理应用的原因。
<!--Kafka works well as a replacement for a more traditional message broker. Message brokers are used for a variety of reasons (to decouple processing from data producers, to buffer unprocessed messages, etc). In comparison to most messaging systems Kafka has better throughput, built-in partitioning, replication, and fault-tolerance which makes it a good solution for large scale message processing applications.-->
从我们的经验角度来看，消息系统通常被用在相对吞吐量不大但是延迟要求低的场景中，并且通常都需要强健的持久化保障。
<!--In our experience messaging uses are often comparatively low-throughput, but may require low end-to-end latency and often depend on the strong durability guarantees Kafka provides.-->
在这个领域，Kafka和[ActiveMQ](http://activemq.apache.org/)或[RabbitMQ](https://www.rabbitmq.com/)这些传统消息系统进行比较。
<!--In this domain Kafka is comparable to traditional messaging systems such as [ActiveMQ](http://activemq.apache.org/) or [RabbitMQ](https://www.rabbitmq.com/).-->

### 网站活动追踪

最初Kafka被用来搭建网站用户活动追踪的管道，管道中是一系列实时的消息流。站点的活动信息（页面浏览，搜索，以及其他用户的活动）被发布成不同主题下的流。这些流可以被不同场景下的应用订阅，包括但不限于实时处理，实时监控，上载到hadoop，或是构建离线数据仓库。
<!--The original use case for Kafka was to be able to rebuild a user activity tracking pipeline as a set of real-time publish-subscribe feeds. This means site activity (page views, searches, or other actions users may take) is published to central topics with one topic per activity type. These feeds are available for subscription for a range of use cases including real-time processing, real-time monitoring, and loading into Hadoop or offline data warehousing systems for offline processing and reporting.-->
活动追踪消息的量通常都很大，因为用户的每一个行为都会产生消息。
<!--Activity tracking is often very high volume as many activity messages are generated for each user page view.-->

### 度量
Kafka经常被用在监控操作数据上。比如要将分布式应用的操作记录集中并导入聚合成一个流用来做统计分析。
<!--Kafka is often used for operational monitoring data. This involves aggregating statistics from distributed applications to produce centralized feeds of operational data.-->

### 日志收集
许多人将Kafka作为日志收集的一个解决方案。典型的日志收集是将物理的日志文件收集起来并且将它们导入到一个集中的地方（比如HDFS）。Kafka将文件细节抽象掉，提供了一个更为简洁的抽象（消息流）。这使得处理延迟变低，且能方便地支持多数据源和多消费者。对比Scribe 或者 Flume这些消息收集工具，Kafka提供了相同的性能，更强的持久化(副本)，以及端到端的低延迟。
<!--Many people use Kafka as a replacement for a log aggregation solution. Log aggregation typically collects physical log files off servers and puts them in a central place (a file server or HDFS perhaps) for processing. Kafka abstracts away the details of files and gives a cleaner abstraction of log or event data as a stream of messages. This allows for lower-latency processing and easier support for multiple data sources and distributed data consumption. In comparison to log-centric systems like Scribe or Flume, Kafka offers equally good performance, stronger durability guarantees due to replication, and much lower end-to-end latency.-->

### 流处理
许多用户将处理消息分成了多个阶段性的处理过程：原始数据被消费出来并聚合产生成一些新主题的流，被用作进一步的处理。举个例子，一个文章推荐处理的系统通常先是会将文章内容通过RSS抓取下来，发布到一个叫"article"的主题里；后续的处理会将内容进行规范，去重和清洗；最终的阶段会将内容和用户关联匹配上。整个过程定义了一个实时数据流动的图。[Storm](https://github.com/nathanmarz/storm) 和 [Samza](http://samza.incubator.apache.org/) 是这一类应用比较流行的实现。
<!--Many users end up doing stage-wise processing of data where data is consumed from topics of raw data and then aggregated, enriched, or otherwise transformed into new Kafka topics for further consumption. For example a processing flow for article recommendation might crawl article content from RSS feeds and publish it to an "articles" topic; further processing might help normalize or deduplicate this content to a topic of cleaned article content; a final stage might attempt to match this content to users. This creates a graph of real-time data flow out of the individual topics. [Storm](https://github.com/nathanmarz/storm) and [Samza](http://samza.incubator.apache.org/) are popular frameworks for implementing these kinds of transformations.-->

### Event Sourcing
Event sourcing是应用程序的一种设计范式，状态的改变会被记录成一个时间相关的序列。Kafka大容量的特性使得可以很好地支持这一类应用。
<!--Event sourcing is a style of application design where state changes are logged as a time-ordered sequence of records. Kafka's support for very large stored log data makes it an excellent backend for an application built in this style.-->

### Commit Log
Kafka可以作为分布式系统的commit log服务。log帮助数据在节点之间复制，并且帮助失效的节点进行数据的恢复。Kafka的[日志压缩](http://kafka.apache.org/documentation.html#compaction)特性在其中起到了帮助作用。Kafka在这个场景下类似于Apache的[BookKeeper](http://zookeeper.apache.org/bookkeeper/)项目.
<!--Kafka can serve as a kind of external commit-log for a distributed system. The log helps replicate data between nodes and acts as a re-syncing mechanism for failed nodes to restore their data. The [log compaction](http://kafka.apache.org/documentation.html#compaction) feature in Kafka helps support this usage. In this usage Kafka is similar to Apache [BookKeeper](http://zookeeper.apache.org/bookkeeper/) project.-->

## 1.3 迅速开始

这篇tutorial假定你从一个全新的环境开始，里面不包含历史Kafka或者Zookeeper数据。
<!--This tutorial assumes you are starting fresh and have no existing Kafka or ZooKeeper data.-->

### Step 1: 下载代码

[Download](https://www.apache.org/dyn/closer.cgi?path=/kafka/0.8.1/kafka_2.9.2-0.8.1.tgz) 0.8.1 版本并且解压.
<!--[Download](https://www.apache.org/dyn/closer.cgi?path=/kafka/0.8.1/kafka_2.9.2-0.8.1.tgz) the 0.8.1 release and un-tar it.-->

	> tar -xzf kafka_2.9.2-0.8.1.tgz
	> cd kafka_2.9.2-0.8.1

### Step 2: 启动服务

Kafka使用了Zookeeper，因此你需要先启动Zookeeper服务。你可以使用简易的打包好的脚本快速启动一个单点的Zookeeper实例。
<!--Kafka uses ZooKeeper so you need to first start a ZooKeeper server if you don't already have one. You can use the convenience script packaged with kafka to get a quick-and-dirty single-node ZooKeeper instance.-->

	> bin/zookeeper-server-start.sh config/zookeeper.properties
	[2013-04-22 15:01:37,495] INFO Reading configuration from: config/zookeeper.properties (org.apache.zookeeper.server.quorum.QuorumPeerConfig)
	...

现在启动Kafka服务：
<!--Now start the Kafka server:-->

	> bin/kafka-server-start.sh config/server.properties
	[2013-04-22 15:01:47,028] INFO Verifying properties (kafka.utils.VerifiableProperties)
	[2013-04-22 15:01:47,051] INFO Property socket.send.buffer.bytes is overridden to 1048576 (kafka.utils.VerifiableProperties)
	...

### Step 3: 创建一个主题

让我们创建一个叫做“test”的主题，其中只包含1个分区，1个副本。
<!--Let's create a topic named "test" with a single partition and only one replica:-->

	> bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test

现在我们可以使用命令列出所有的主题：
<!--We can now see that topic if we run the list topic command:-->

	> bin/kafka-topics.sh --list --zookeeper localhost:2181
	test

除了手动创建和配置新的主题外，你也可以让broker自动创建和配置新的主题。
<!--Alternatively, instead of manually creating topics you can also configure your brokers to auto-create topics when a non-existent topic is published to.-->

### Step 4: 发送一些消息

Kafka提供了一个命令行客户端用来从文件中或者stdin中读入数据并且发送到Kafka集群里。默认文件中的一行是一条消息。
<!--Kafka comes with a command line client that will take input from a file or from standard input and send it out as messages to the Kafka cluster. By default each line will be sent as a separate message.-->

启动producer，在控制台里键入一些字符发送到server。
<!--Run the producer and then type a few messages into the console to send to the server.-->
	
	> bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test 
	This is a message
	This is another message

### Step 5: 启动一个消费者

Kafka也提供一个命令行消费者程序，它会将消息dump到stdout。
<!--Kafka also has a command line consumer that will dump out messages to standard output.-->
	
	> bin/kafka-console-consumer.sh --zookeeper localhost:2181 --topic test --from-beginning
	This is a message
	This is another message

如果你在不同的终端运行上面两个程序，你将会看到从producer终端键入的消息会打印在consumer终端上。
<!--If you have each of the above commands running in a different terminal then you should now be able to type messages into the producer terminal and see them appear in the consumer terminal.-->
所有的这些命令行工具都有一些额外选项，你可以不加任何参数运行这些命令，它将打印出详细的使用信息。
<!--All of the command line tools have additional options; running the command with no arguments will display usage information documenting them in more detail.-->

### Step 6: 创建多server的集群

到现在为止，我们跑了例子是单broker的，这不是很有趣。虽然看起来单broker集群与多broker集群仅仅只是broker数量的不同，但是为了有一个切身的体验，现在让我们将集群的broker数量扩展到3（但是仍然是在一台机器上）。
<!--So far we have been running against a single broker, but that's no fun. For Kafka, a single broker is just a cluster of size one, so nothing much changes other than starting a few more broker instances. But just to get feel for it, let's expand our cluster to three nodes (still all on our local machine).-->
首先，我们创建另外两个broker的配置文件：
<!--First we make a config file for each of the brokers:-->
	
	> cp config/server.properties config/server-1.properties 
	> cp config/server.properties config/server-2.properties

现在编辑这三个文件，配置成下列参数：
<!--Now edit these new files and set the following properties:-->

	config/server-1.properties:
   		broker.id=1
   		port=9093
   		log.dir=/tmp/kafka-logs-1
 
	config/server-2.properties:
		broker.id=2
		port=9094
		log.dir=/tmp/kafka-logs-2

属性broker.id是集群中每个broker的独一无二的且永久的名字。我们必须重写端口和日志目录只是因为我们将所有broker跑在了一台机器上。
<!--The broker.id property is the unique and permanent name of each node in the cluster. We have to override the port and log directory only because we are running these all on the same machine and we want to keep the brokers from all trying to register on the same port or overwrite each others data.-->

我们已经启动了Zookeeper和一个broker，现在是要启动另外两个新broker的时候了：
<!--We already have Zookeeper and our single node started, so we just need to start the two new nodes:-->
	
	> bin/kafka-server-start.sh config/server-1.properties &
	...
	> bin/kafka-server-start.sh config/server-2.properties &
	...

现在创建一个有3个副本的主题：
<!--Now create a new topic with a replication factor of three:-->

	> bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 3 --partitions 1 --topic my-replicated-topic

好，现在我们有了一个集群，我们如何能得知每个broker在干嘛？可以使用一个描述topic的命令：
<!--Okay but now that we have a cluster how can we know which broker is doing what? To see that run the "describe topics" command:-->

	> bin/kafka-topics.sh --describe --zookeeper localhost:2181 --topic my-replicated-topic
	Topic:my-replicated-topic   PartitionCount:1    ReplicationFactor:3 Configs:
	        Topic: my-replicated-topic  Partition: 0    Leader: 1   Replicas: 1,2,0 Isr: 1,2,0

解释一下命令的输出。第一行给出了一个包含所有的分区的摘要，其他每一行包含一个独立的分区的信息。既然我们只有1个分区（译者注：原文有误），所以这里只有两行输出。
<!--Here is an explanation of output. The first line gives a summary of all the partitions, each additional line gives information about one partition. Since we have only two partitions for this topic there are only two lines.-->

* “leader” 是负责这个分区读和写的节点。每一个节点都会是随机一些分区的leader。
* “replicas” 是所有副本所在的节点，不论是否是leader，也不论是否现在存活。
* “isr” 是同步完毕的副本("in-sync"), 这里的节点是所有副本节点集合的子集，并且是现在存活的，且数据从leader处同步完毕。

<!--* "leader" is the node responsible for all reads and writes for the given partition. Each node will be the leader for a randomly selected portion of the partitions.
* "replicas" is the list of nodes that replicate the log for this partition regardless of whether they are the leader or even if they are currently alive.
* "isr" is the set of "in-sync" replicas. This is the subset of the replicas list that is currently alive and caught-up to the leader.-->

注意到在这个例子中，节点1是仅有的一个分区的leader。
<!--Note that in my example node 1 is the leader for the only partition of the topic.-->

我们可以对之前的那个主题运行同样的命令，看看会有什么输出：
<!--We can run the same command on the original topic we created to see where it is:-->

	> bin/kafka-topics.sh --describe --zookeeper localhost:2181 --topic test
	Topic:test  PartitionCount:1    ReplicationFactor:1 Configs:
	        Topic: test Partition: 0    Leader: 0   Replicas: 0 Isr: 0

毫无例外的，没有副本，leader是节点0，这个节点是我们之前创建的唯一一个server。
<!--So there is no surprise there—the original topic has no replicas and is on server 0, the only server in our cluster when we created it.-->

让我们来发布一些新主题的数据：
<!--Let's publish a few messages to our new topic:-->

	> bin/kafka-console-producer.sh --broker-list localhost:9092 --topic my-replicated-topic
	...
	my test message 1
	my test message 2
	^C

现在来消费这些数据：
<!--Now let's consume these messages:-->

	> bin/kafka-console-consumer.sh --zookeeper localhost:2181 --from-beginning --topic my-replicated-topic
	...
	my test message 1
	my test message 2
	^C

现在测试一下错误容忍。节点1现在是leader，我们来杀掉它。
<!--Now let's test out fault-tolerance. Broker 1 was acting as the leader so let's kill it:-->

	> ps | grep server-1.properties
	7564 ttys002    0:15.91 /System/Library/Frameworks/JavaVM.framework/Versions/1.6/Home/bin/java...
	> kill -9 7564

领导权被交由到了其中一个slave上，节点1现在不在(in-sync)的列表里面了。
<!--Leadership has switched to one of the slaves and node 1 is no longer in the in-sync replica set:-->

	> bin/kafka-topics.sh --describe --zookeeper localhost:218192 --topic my-replicated-topic
	Topic:my-replicated-topic   PartitionCount:1    ReplicationFactor:3 Configs:
    Topic: my-replicated-topic  Partition: 0    Leader: 2   Replicas: 1,2,0 Isr: 2,0

但是消息仍然可以被消费，即使原来的leader已经挂掉了。
<!--But the messages are still be available for consumption even though the leader that took the writes originally is down:-->

	> bin/kafka-console-consumer.sh --zookeeper localhost:2181 --from-beginning --topic my-replicated-topic
	...
	my test message 1
	my test message 2
	^C

## 1.4 生态圈

除了Kafka官方发布的版本外，周边还有许多丰富的围绕Kafka的生产工具。[这个页面](https://cwiki.apache.org/confluence/display/KAFKA/Ecosystem)里面列出了包括与流处理系统的集成，与hadoop系统的集成，Kafka的监控和部署等在内的工具。
<!--There are a plethora of tools that integrate with Kafka outside the main distribution. The [ecosystem page](https://cwiki.apache.org/confluence/display/KAFKA/Ecosystem) lists many of these, including stream processing systems, Hadoop integration, monitoring, and deployment tools.-->

## 1.5 版本升级

### 从 0.8.0 升级至 0.8.1

0.8.1完全兼容0.8.因此升级只需要一次操作一个broker，通过简单的关闭，升级，重启就完成了。
<!--0.8.1 is fully compatible with 0.8. The upgrade can be done one broker at a time by simply bringing it down, updating the code, and restarting it.-->

### 从 0.7 版本升级

0.8增加了副本功能，这是我们第一个不向后兼容的版本：我们对API做了大范围的改动，Zookeeper数据结构，协议，配置这些也都变了。因此从0.7升级到0.8.x需要一个[迁移工具](https://cwiki.apache.org/confluence/display/KAFKA/Migrating+from+0.7+to+0.8)。迁移可以在线完成。
<!--0.8, the release in which added replication, was our first backwards-incompatible release: major changes were made to the API, ZooKeeper data structures, and protocol, and configuration. The upgrade from 0.7 to 0.8.x requires a [special tool](https://cwiki.apache.org/confluence/display/KAFKA/Migrating+from+0.7+to+0.8) for migration. This migration can be done without downtime.-->