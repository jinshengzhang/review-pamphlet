# Kafka

## 0.kafka的概念相关的问题？

1. kafka是一个分布式的、可分区的、可复制的提交日志服务，复制是核心，保证了了可用性和持久性

## 1. kafka 新旧API的特点？

[Kafka 0.9 新消费者API](https://www.cnblogs.com/admln/p/5446361.html)

## 2. kafka 新版API auto.offset.reset 的含义

### 2.1 earliest

当各分区下有已提交的offset时，从提交的offset开始消费；无提交的offset时，从头开始消费

### 2.2 latest

当各分区下有已提交的offset时，从提交的offset开始消费；无提交的offset时，消费新产生的该分区下的数据

### 2.3 none

topic各分区都存在已提交的offset时，从offset后开始消费；只要有一个分区不存在已提交的offset，则抛出异常

[Kafka auto.offset.reset值详解](https://blog.csdn.net/lishuangzhe7047/article/details/74530417)

## 3. kafka enable.auto.commit 的含义

是否由客户端自动提交offset

## 4. kafka 版本的演进？

[kafka各个版本特性预览介绍](https://blog.csdn.net/hesi9555/article/details/70237744)

### 0.8.2 之前

1. kafka删除topic的功能存在bug
2. comsumer定期提交已经消费的kafka消息的offset位置到zookeeper中保存，对zookeeper而言，每次写操作代价是很昂贵的，而且zookeeper集群是不能扩展写能力的

### 0.8.2

1. 可以把comsumer提交的offset记录在compacted topic（__comsumer_offsets）中，该topic设置最高级别的持久化保证，即ack=-1
    1. __consumer_offsets由一个三元组< comsumer group, topic, partiotion> 组成的key和offset值组成，在内存也维持一个最新的视图view，所以读取很快。
    2. kafka可以频繁的对offset做检查点checkpoint，即使每消费一条消息提交一次offset。
    3. 在0.8.1中，已经实验性的加入这个功能，0.8.2中可以广泛使用.

### 0.9

1. 安全方面改进(在0.9之前，Kafka安全方面的考虑几乎为0，在进行外网传输时，只好通过Linux的防火墙、或其他网络安全方面进行配置。相信这一点，让很多用户在考虑使用Kafka进行外网消息交互时有些担心。)
    1. 客户端连接borker使用SSL或SASL进行验证
    2. borker连接ZooKeeper进行权限管理
    3. 数据传输进行加密（需要考虑性能方面的影响）
    4. 客户端读、写操作可以进行授权管理
    5. 可以对外部的可插拔模块的进行授权管理
2. Kafka Connect 它可以和外部系统、数据集建立一个数据流的连接，实现数据的输入、输出。有以下特性：
    1. 使用了一个通用的框架，可以在这个框架上非常方便的开发、管理Kafka Connect接口
    2. 支持分布式模式或单机模式进行运行
    3. 支持REST接口，可以通过REST API提交、管理 Kafka Connect集群
    4. offset自动管理
3. 新的Comsumer API
    1. 新的Comsumer API不再有high-level、low-level之分了，而是自己维护offset 这样做的好处是避免应用出现异常时，数据未消费成功但Position已经提交，导致消息未消费的情况发生
    2. Kafka可以自行维护Offset、消费者的Position。也可以开发者自己来维护Offset，实现相关的业务需求。
    3. 消费时，可以只消费指定的Partitions
    4. 可以使用外部存储记录Offset，如数据库之类的
    5. 可以使用多线程进行消费

### 0.10

1. 机架感知以便隔离副本 跨域多个机架或者可用区域，提高弹性和可用性
2. 所有消息包含了时间戳字段
3. Kafka Consumer Max Records，在Kafka 0.9.0.0，开发者们在新consumer上使用poll()函数的时候是几乎无法控制返回消息的条数。不过值得高兴的是，此版本的Kafka引入了max.poll.records参数，允许开发者控制返回消息的条数。

## 5. kafka 的leader 选举机制是怎样实现的以及各个版本的实现？

1. leader是对应partition的概念，每个partition都有一个leader。
2. 客户端生产消费消息都是只跟leader交互 (实现上简单。)
3. ISR（In-Sync Replicas）直译就是跟上leader的副本
4. ![image](http://static.lovedata.net/jpg/2018/5/29/8e14780235f97056c60785dce43f18e7.jpg)
    1. High watermark（高水位线）以下简称HW，表示消息被leader和ISR内的follow都确认commit写入本地log，所以在HW位置以下的消息都可以被消费（不会丢失）
    2. Log end offset（日志结束位置）以下简称LEO，表示消息的最后位置。LEO>=HW，一般会有没提交的部分。
5. 副本会有单独的线程（ReplicaFetcherThread），去从leader上去拉去消息同步。当follower的HW赶上leader的，就会保持或加入到 **ISR** 列表里，就说明此follower满足上述最基本的原则（跟上leader进度）。ISR列表存在zookeeper上。
    1. replica.lag.max.messages 落后的消息个数 （Kafka 0.10.0 移除了落后消息个数参数（replica.lag.max.messages），原因是这个值不好把控，需要经验值，不用的业务服务器环境，这个值可能不同，不然会频繁的移除加入ISR列表） [Kafka副本管理—— 为何去掉replica.lag.max.messages参数 - huxihx - 博客园](http://www.cnblogs.com/huxi2b/p/5903354.html)
    2. replica.lag.time.max.ms 多长时间没有发送FetchQuest请求拉去leader数据
6. producer的ack参数选择，取优先考虑可靠性，还是优先考虑高并发。以下不用的参数，会导致有可能丢消息。
    - 0表示纯异步，不等待，写进socket buffer就继续。
    - 1表示leader写进server的本地log，就返回，不等待follower的回应。
    - -1相当于all，表示等待follower回应再继续发消息。保证了ISR列表里至少有一个replica，数据就不会丢失，最高的保证级别。
7. 参考
    1. [kafka的HA设计 - 简书](https://www.jianshu.com/p/83066b4df739)
    2. [kafka的leader选举过程 - 简书](https://www.jianshu.com/p/c987b5e055b0)
8. unclean.leader.election.enable 默认是true，表示允许不在ISR列表的follower，选举为leader（最坏的打算，可能丢消息）

## 6. kafka 的 rebalance 是怎样的？

## 7. kafka中的offset状态，以及high.watermark是什么意思

![image](http://static.lovedata.net/jpg/2018/5/25/c2fa3b250b6512a80279e8140b1421d7.jpg)

例如，在下图中，消费者的位置在偏移6，其最后的提交的偏移1.
当分区重新分配给组中的另外一个使用者时，初始位置设置为最后一个已提交的偏移量。如果上面例子中的消费者突然崩溃了，那么接管的组成员将从偏移量1开始消费。在这种情况下，它必须重新处理消息直到崩溃消费者的位置6.

该图还显示了日志中的另外两个重要位置。日志结束偏移量是写入日志的最后一条消息的偏移量。高水印是成功复制到所有日志副本的最后一条消息的偏移量。从消费者的角度来看，主要知道的是，你只能读取高水印。这防止消费者读取稍后可能丢失的未复制数据。

## 8. Kafak本身提供的新的组协调协议是怎样的机制？

## 9. kafka 使用场景？

- 日志收集：一个公司可以用Kafka可以收集各种服务的log，通过kafka以统一接口服务的方式开放给各种consumer，例如hadoop、Hbase、Solr等。
- 消息系统：解耦和生产者和消费者、缓存消息等。
- 用户活动跟踪：Kafka经常被用来记录web用户或者app用户的各种活动，如浏览网页、搜索、点击等活动，这些活动信息被各个服务器发布到kafka的topic中，然后订阅者通过订阅这些topic来做实时的监控分析，或者装载到hadoop、数据仓库中做离线分析和挖掘。
- 运营指标：Kafka也经常用来记录运营监控数据。包括收集各种分布式应用的数据，生产各种操作的集中反馈，比如报警和报告。
- 流式处理：比如spark streaming和storm
- 事件源

[Kafka史上最详细原理总结 | 静水流深](http://www.thinkyixia.com/2017/10/25/kafka-2/)

## 10. partition和replica默认分配到哪个broker的策略?

- 将所有N Broker和待分配的i个Partition排序.
- 将第i个Partition分配到第(i mod n)个Broker上.
- 将第i个Partition的第j个副本分配到第((i + j) mod n)个Broker上.

## 11. kafka zookeeper中存储结构

1. ![image](http://static.lovedata.net/jpg/2018/5/29/e579c3897235853981bb911ef3328e4e.jpg)
2. ![image](http://static.lovedata.net/jpg/2018/5/29/58462246b8030bb67d3a633305cfe12b.jpg)
3. ![image](http://static.lovedata.net/jpg/2018/5/29/dc69269178701fdeae11e3388340176e.jpg)

## 12. 如果Zookeeper宕机了，kafka还能用吗？

## 13. kafka 发送消息的三种方式

>调用send()方法发送ProducerRecored对象， send()方法返回一个包含RecordMetadata的Future对象

1. 发送并忘记  直接send消息
2. 同步发送，调用send() 返回一个Futrue对象，调用get()方法进行等待，可能抛出异常，有可重试异常和不可重试异常（如数据太大）
3. 异步发送，调用send()方法，指定一个回调函数，服务器返回相应的时候调用该函数 (实现producer.Callback的onComplete方法)

## 14. kafka生产者有哪些重要的配置？

### 14.1 acks

acks 指定了必须要多少个分区副本收到消息，生产者才会认为消息写入是成功的。

1. acks=0 不会等待任何来自服务器的响应，可能会丢消息，但是又更大的吞吐量
2. acks=1 只要首领leader收到消息，就会收到成功响应（如果leader节点异常了，一个没有收到消息的节点成为新leader，还是会丢失）
3. acks=all 只有参与复制的节点全部收到消息，才会收到成功响应  延迟较高

### 14.2 buffer.memory

生产者内存缓冲区大小

### 14.3  compression.type

压缩方式 snapy gzip lz4

### 14.4 retries

重试次数

### 14.5 client.id

任意字符串，识别消息来源 用于日志和配额指标

### 14.6 max.block.ms

如果缓冲区已满之后send阻塞的时间，如果达到后抛异常

### 14.7 max.in.flight.requests.per.connection

指定生产者收到服务响应之前可以发送多少消息，值越大，内存占用越大，吞吐量越高，设置为1保证消息有序发送写入

## 15. kafka消息的顺序性保证是怎样的

>银行存款取款场景下，顺序很重要

1. 可以保证一个分区的消息是有序的
2. 如果retries为非零， max.in.flight.requests.per.connection>1,如果一个消息社保，第二批次成功，第一次重试成功后，那么顺序就乱了
3. 一般设置retries>0,把max.in.flight.requests.per.connection设置为1，保证有序

## 16.什么是Avro？

1. Avro 是一种与语言无关的序列化格式，通过schema定义，schema使用json描述，数据被序列为二进制或者json（一般二进制），schema内嵌在数据文件里， **兼容新旧版本**
2. 使用schema注册表来生产者注册schema，消费者获取schema，使用Confluent Schema Registry注册表

## 17. kafka主题增加分区后，原来的路由到分区A的数据，还会路由到A吗？

不会，回路由到其他分区，要想不变，就是在创建主题的时候，把分区规划好，而且永远不要增加新分区

## 18. kafka分区的方式

1. 键值为null，并且使用默认分区，分区器使用轮训（Round Robin）均衡分布到各个分区上
2. 兼职不是null，并且使用默认分区，使用kafka对键进行散列（这里散列使用所有分区，包括不可用的，可能会发生错误）
3. 自定义分区

## 19. 什么时候发生重新分配reblance？

在主题发生变化时，比如管理员添加了新的分区，会发生。
分区所有权从一个消费者转移到另一个消费者，这样的行为成为再均衡，给消费者群组带来了高可用性和伸缩性
弊端： 消费者群组一段时间不能读取消息。
消费者向群组协调器broker发送心跳维持所有权关系，在轮询消息和提交偏移量的时候发送心跳。
消费者必须持续的轮询向kafka请求数据，否则会被认为已经死掉，导致重新分配哦。

## 20. KafkaConsumer在订阅数据后退出了不关闭会有什么后果？

如果不管，网络连接和socket也不会关闭，就不能立即出发再均衡，要等待协调器发现心跳没了才确认他死亡了，这样就需要更长的时间，导致群组在一段时间内无法读取消息

## 21. 消费者线程安全问题？

同一个群组，无法让一个线程运行多个消费者，也无法让多个线程安全的共享一个消费者，按照规则，一个消费者使用一个线程。如果要多线程使用ExecutorServcie启动多个线程，让每个线程运行自己的消费者。

## 22.消费者的重要配置？

1. fetch.min.bytes 从服务器获取的最小字节数，如果数据不够，不会马上返回。针对于数据量不大的情况下，避免频繁的网络连接
2. fetch.max.wait.ms 等待broker数据的超时时间，与上面配置配套。不能老等是吧
3. max.parition.fetch.bytes 服务器从每个分区返回给消费者最大字节数  默认值1MB,要比max.message.size大，否则消费者就无法读取了，如果太大也不行，可能消费者线程一下处理不完，导致以为自己挂掉了，要么就要该打session超时时间了
4. session.timeout.ms 指定消费者被认为死亡之前可以与服务器断开连接的时间，hearbeat.interval.ms指定pool想协调器发送心跳的频率，一般比timeout.ms小，一般为他的三分之一
5. [enableautocommit-的含义](#3-kafka-enableautocommit-的含义)

## 23. 消费者偏移量自动提交的方式？

1. 自动提交 设置 anable.auto.commit=true,每 auto.commit.interval.ms 控制提交偏移量，默认5s，也是轮询里进行，每次轮询判断是否该提交了，无法完全避免消息被重复处理， **因为他并不能知道哪一条消息被处理掉了**
2. 手动提交偏移量  anable.auto.commit=false 使用commitSync()提交（提交poll最新的偏移量） 需要确保处理完成消息后调用该方法
    1. 缺点：阻塞应用程序
3. 异步提交 commitAsync() 失败后不会重试，因为有可能有其他更大的偏移量已经提交了 支持一个回调，记录。
    1. 可以使用一个单调递增序列号维护异步提交的顺序，失败后判断是否相等， 相等则可以重试
4. 同步和异步组合提交 在正常的时候使用异步提交，在最后异步使用同步提交
5. 提交特定偏移量，commitAync(map(topicparition,offsetandmetada)) 指定的偏移量
6. **使用ConsumerReblanceListener来锦亭 分区再分配时间 有 revoke和assign方法实现**

## 24. kafka如何退出

在ShutdownHook调用 consumer.wakeup（）方法，该方法在调用后，consumer调用poll的时候会抛出WakeupException

## 25.新版API如何消费指定的分区

使用partitionsFor("topic") 获取某个主题所有的分区，并且从土偶哦 consumer.assin(topicparitions) 分配分区，不能获取新的分区通知。

## 26.kafka高可用如何保证数据不丢失不重复消费？

[Spark Streaming和Kafka整合保证数据零丢失 - FelixZh - 博客园](https://www.cnblogs.com/felixzh/p/6371253.html)

## 27. kafka控制器的选举方式？

1. kafka通过zk的临时节点选举控制器，在节点加入集群或者退出通知控制器，控制器负责加入或者离开集群式进行分区的首领选举，使用 epoch避免脑裂（两个节点都认为自己是当前的控制器：通过controller epoch 的新旧来判断）

## 28. kafka broker 如何把消息发送给客户端

1. 客户端请求首先罗到分区首领上
2. 首领接到请求后首先判断请求是否有效：指定偏移是否存在  否则返回一个错误
3. kafka使用  **零复制** 技术，直接从文件（linux文件缓冲区） 里发送到网络，不是使用缓冲区，避免了字节复制，也不需要内存缓冲区