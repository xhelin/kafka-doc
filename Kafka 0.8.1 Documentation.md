# 1. 开始上手
## 1.1 介绍
Kafka is a distributed, partitioned, replicated commit log service. It provides the functionality of a messaging system, but with a unique design.

Kafka是一个分布式，分区，带副本的commit log服务系统。它提供的功能有些类似于消息系统，但是在设计上却与大多数消息系统不同。

What does all that mean?

究竟是怎么一回事？

First let's review some basic messaging terminology:

首先，让我们复习一些基本的消息术语：

* Kafka maintains feeds of messages in categories called topics.
* We'll call processes that publish messages to a Kafka topic producers.
* We'll call processes that subscribe to topics and process the feed of published messages consumers..
* Kafka is run as a cluster comprised of one or more servers each of which is called a broker.


* Kafka保留的每条消息的分类被称为**主题**（topic）
* 发布相关topic的消息的进程被称为**生产者**（producer）
* 订阅并处理特定topic的消息的进程被称为**消费者**（consumer）
* Kafka集群中的每一个服务进程被称为**代理者**（broker）

So, at a high level, producers send messages over the network to the Kafka cluster which in turn serves them up to consumers like this:

所以站在系统高层的角度来看，producers通过网络发送消息给Kafka集群，Kafka集群又将消息发送给consumers，如下图：

![Kafka High level](http://kafka.apache.org/images/producer_consumer.png)

Communication between the clients and the servers is done with a simple, high-performance, language agnostic TCP protocol. We provide a Java client for Kafka, but clients are available in many languages.

客户端与服务器端的通信采用的是一个简单，高性能，且非特定语言的TCP层之上的协议。我们实现了Kafka的Java客户端，但是很多其他语言的客户端在这儿可以获取到。

### 主题和日志

Let's first dive into the high-level abstraction Kafka provides—the topic.

让我们先研究一下Kafka的高层抽象之--主题。

A topic is a category or feed name to which messages are published. For each topic, the Kafka cluster maintains a partitioned log that looks like this:

一个主题是一个消息流的分类或名称。Kafka集群对于每个主题的日志都保存了一系列分区，像这样：

![anatomy of a topic](http://kafka.apache.org/images/log_anatomy.png)

Each partition is an ordered, immutable sequence of messages that is continually appended to—a commit log. The messages in the partitions are each assigned a sequential id number called the offset that uniquely identifies each message within the partition.

每个分区都是一个有序的，不可变（已有的），且不断追加的消息序列 -- 像commit log。这写分区中的每一个消息都被附上了一个序列号（sequential id number）称为**偏移值(offset)**，并且每条日志的偏移值都是（分区内）唯一的。

The Kafka cluster retains all published messages—whether or not they have been consumed—for a configurable period of time. For example if the log retention is set to two days, then for the two days after a message is published it is available for consumption, after which it will be discarded to free up space. Kafka's performance is effectively constant with respect to data size so retaining lots of data is not a problem.

Kafka集群保留一段时间内（可配置）所有发布的消息--不管消息是否已经被消费。举个例子，如果日志被设定为保留2天，那么当一条日志发布之后的两天内都是可以被消费的，再之后就会被清理掉。Kafka的性能和要保存的数据多少不相关因此即使要保存很大量的数据也不会是个问题。

In fact the only metadata retained on a per-consumer basis is the position of the consumer in in the log, called the "offset". This offset is controlled by the consumer: normally a consumer will advance its offset linearly as it reads messages, but in fact the position is controlled by the consumer and it can consume messages in any order it likes. For example a consumer can reset to an older offset to reprocess.

事实上每个消费者要保存的元信息仅仅只有消息的位置，也就是**偏移值(offset)**，这个偏移值是由消费者保存的：通常情况下消费者在读取消息的同时都会自动将偏移值往前线性移动，但事实上偏移的位置是由消费者来控制的，因此消费者可以按照它想要的任何顺序来进行数据的读取。例如，消费者可以重新将偏移值设成一个老的值来重新消费。

This combination of features means that Kafka consumers are very cheap—they can come and go without much impact on the cluster or on other consumers. For example, you can use our command line tools to "tail" the contents of any topic without changing what is consumed by any existing consumers.

这样的特性组合意味着Kafka的消费者变得非常的轻巧--他们可以随时取数据随时又停止，这样不会影响到整个集群以及其他的消费者。比如你可以使用我们提供的命令行工具来 tail 一个主题的新增的数据而不需要改变任何已经存在的消费者。

The partitions in the log serve several purposes. First, they allow the log to scale beyond a size that will fit on a single server. Each individual partition must fit on the servers that host it, but a topic may have many partitions so it can handle an arbitrary amount of data. Second they act as the unit of parallelism—more on that in a bit.

日志的分区主要由几个目的。第一，这样做可以使得日志的规模超过单台机器能够容纳的量。每个独立的分区必须适合它所在的服务，但是一个主题可以拥有很多很多的分区，这样就能够处理任意量的日志。第二，他们（分区）扮演了并行处理的最小单元 -- more on that in a bit（怎么翻译）。

