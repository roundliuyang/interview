# Kafka



## Apache Kafka 是什么



Kafka 是基于**发布与订阅**的**消息系统**。它最初由 LinkedIn 公司开发，之后成为 Apache 项目的一部分。Kafka 是一个分布式的，可分区的，冗余备份的持久性的日志服务。它主要用于处理活跃的流式数据。

在大数据系统中，常常会碰到一个问题，整个大数据是由各个子系统组成，数据需要在各个子系统中高性能、低延迟的不停流转。传统的企业消息系统并不是非常适合大规模的数据处理。为了同时搞定在线应用（消息）和离线应用（数据文件、日志），Kafka 就出现了。Kafka 可以起到两个作用：

- 降低系统组网复杂度。
- 降低编程复杂度，各个子系统不在是相互协商接口，各个子系统类似插口插在插座上，Kafka 承担高速**数据总线**的作用。



### **Kafka 的主要特点**



1. 同时为发布和订阅提供高吞吐量。据了解，Kafka 每秒可以生产约 25 万消息（50MB），每秒处理 55 万消息（110MB）。
2. 可进行持久化操作。将消息持久化到磁盘，因此可用于批量消费，例如 ETL ，以及实时应用程序。通过将数据持久化到硬盘，以及replication ，可以防止数据丢失。
3. 分布式系统，易于向外扩展。所有的 Producer、Broker 和Consumer 都会有多个，均为分布式的。并且，无需停机即可扩展机器。
4. 消息被处理的状态是在 Consumer 端维护，而不是由 Broker 端维护。当失败时，能自动平衡。
5. 支持 online 和 offline 的场景。





### ~~**聊聊 Kafka 的设计要点**~~





## Kafka 的架构是怎么样的



![Kafka 架构图](Kafka.assets/ac883ce247c1ff31c7cd4244392dcaed)

Kafka 的整体架构非常简单，是分布式架构，Producer、Broker 和Consumer 都可以有多个。





## **Kafka 为什么要将 Topic 进行分区**



正如我们在 [「聊聊 Kafka 的设计要点？」](http://svip.iocoder.cn/Kafka/Interview/#) 问题中所看到的，是为了负载均衡，从而能够水平拓展。

- Topic 只是逻辑概念，面向的是 Producer 和 Consumer ，而 Partition 则是物理概念。如果 Topic 不进行分区，而将 Topic 内的消息存储于一个 Broker，那么关于该 Topic 的所有读写请求都将由这一个 Broker 处理，吞吐量很容易陷入瓶颈，这显然是不符合高吞吐量应用场景的。
- 有了 Partition 概念以后，假设一个 Topic 被分为 10 个 Partitions ，Kafka 会根据一定的算法将 10 个 Partition 尽可能均匀的分布到不同的 Broker（服务器）上。
- 当 Producer 发布消息时，Producer 客户端可以采用 random、key-hash 及轮询等算法选定目标 Partition ，若不指定，Kafka 也将根据一定算法将其置于某一分区上。
- 当 Consumer 拉取消息时，Consumer 客户端可以采用 Range、轮询 等算法分配 Partition ，从而从不同的 Broker 拉取对应的 Partition 的 leader 分区。

所以，Partiton 机制可以极大的提高吞吐量，并且使得系统具备良好的水平扩展能力。





## Kafka 的应用场景有哪些



![Kafka 的应用场景](Kafka.assets/3636ff4bd554ee1dfcfb92448073b5b8)

1）消息队列

2）行为跟踪

3）元信息监控

4）日志收集

5）流处理

6）事件源

7）持久性日志（Commit Log）





## Kafka 消息发送和消费的简化流程是什么



![Kafka 消息发送和消费](Kafka.assets/456f3adab70714c0a1b1fdbd8a686732)

- 1、Producer ，根据指定的 partition 方法（round-robin、hash等），将消息发布到指定 Topic 的 Partition 里面。
- Kafka 集群，接收到 Producer 发过来的消息后，将其持久化到硬盘，并保留消息指定时长（可配置），而不关注消息是否被消费
- Consumer ，从 Kafka 集群 pull 数据，并控制获取消息的 offset 。至于消费的进度，可手动或者自动提交给 Kafka 集群。





###  **Producer 发送消息**



Producer 采用 push 模式将消息发布到 Broker，每条消息都被 append 到 Patition 中，属于顺序写磁盘（顺序写磁盘效率比随机写内存要高，保障 Kafka 吞吐率）。Producer 发送消息到 Broker 时，会根据分区算法选择将其存储到哪一个 Partition 。

其路由机制为：

1. 指定了 Partition ，则直接使用。
2. 未指定 Partition 但指定 key ，通过对 key 进行 hash 选出一个 Partition 。
3. Partition 和 key 都未指定，使用轮询选出一个 Partition 。



写入流程：

1. Producer 先从 ZooKeeper 的 `"/brokers/.../state"` 节点找到该 Partition 的 leader 。

   >注意噢，Producer 只和 Partition 的 leader 进行交互。

2. Producer 将消息发送给该 leader 。

3. leader 将消息写入本地 log 

4. followers 从 leader pull 消息，写入本地 log 后 leader 发送 ACK 。

5. leader 收到所有 ISR 中的 replica 的 ACK 后，增加 HW（high watermark ，最后 commit 的 offset） 并向 Producer 发送 ACK 。





### **Broker 存储消息**



物理上把 Topic 分成一个或多个 Patition，每个 Patition 物理上对应一个文件夹（该文件夹存储该 Patition 的所有消息和索引文件）。



### **Consumer 消费消息**



high-level Consumer API 提供了 consumer group 的语义，一个消息只能被 group 内的一个 Consumer 所消费，且 Consumer 消费消息时不关注 offset ，最后一个 offset 由 ZooKeeper 保存（下次消费时，该 group 中的 Consumer 将从 offset 记录的位置开始消费）。

注意：

- 1、如果消费线程大于 Patition 数量，则有些线程将收不到消息。
- 2、如果 Patition 数量大于消费线程数，则有些线程多收到多个 Patition 的消息。
- 3、如果一个线程消费多个 Patition，则无法保证你收到的消息的顺序，而一个 Patition 内的消息是有序的。



Consumer 采用 pull 模式从 Broker 中读取数据。

- push 模式，很难适应消费速率不同的消费者，因为消息发送速率是由 Broker 决定的。它的目标是尽可能以最快速度传递消息，但是这样很容易造成 Consumer 来不及处理消息，典型的表现就是拒绝服务以及网络拥塞。而 pull 模式，则可以根据 Consumer 的消费能力以适当的速率消费消息。
- 对于 Kafka 而言，pull 模式更合适，它可简化 Broker 的设计，Consumer 可自主控制消费消息的速率，同时 Consumer 可以自己控制消费方式——即可批量消费也可逐条消费，同时还能选择不同的提交方式从而实现不同的传输语义。





### **Kafka Consumer 是否可以消费指定的分区消息**



Consumer 消费消息时，向 Broker 发出“fetch”请求去消费特定分区的消息，Consumer 指定消息在日志中的偏移量(offset)，就可以消费从这个位置开始的消息，Consumer 拥有了 offset 的控制权，可以向后回滚去重新消费之前的消息，这是很有意义的。





## Kafka 的副本机制是怎么样的



Kafka 的副本机制，是多个 Broker 节点对其他节点的 Topic 分区的日志进行复制。当集群中的某个节点出现故障，访问故障节点的请求会被转移到其他正常节点(这一过程通常叫 Reblance)，Kafka 每个主题的每个分区都有一个主副本以及 0 个或者多个副本，副本保持和主副本的数据同步，当主副本出故障时就会被替代。

![副本机制](Kafka.assets/1f42986437f76c108007e86a51ae1287)

>注意哈，下面说的 Leader 指的是每个 Topic 的某个分区的 Leader ，而不是 Broker 集群中的【集群控制器】。



在 Kafka 中并不是所有的副本都能被拿来替代主副本，所以在 Kafka 的Leader 节点中维护着一个 ISR（In sync Replicas）集合，翻译过来也叫正在同步中集合，在这个集合中的需要满足两个条件:



1. 节点必须和 Zookeeper 保持连接。
2. 在同步的过程中这个副本不能落后主副本太多。

另外还有个 AR（Assigned Replicas）用来标识副本的全集，OSR 用来表示由于落后被剔除的副本集合，所以公式如下：

- ISR = Leader + 没有落后太多的副本。
- AR = OSR + ISR 。





## ZooKeeper 在 Kafka 中起到什么作用



在基于 Kafka 的分布式消息队列中，ZooKeeper 的作用有：

1. Broker 在 ZooKeeper 中的注册。

2. Topic 在 ZooKeeper 中的注册。

3. Consumer 在 ZooKeeper 中的注册。

4. Producer 负载均衡。

   >主要指的是，Producer 从 Zookeeper 拉取 Topic 元数据，从而能够将消息发送负载均衡到对应 Topic 的分区中。

5. Consumer 负载均衡。

6. 记录消费进度 Offset 。

   >Kafka 已推荐将 consumer 的 Offset 信息保存在 Kafka 内部的 Topic 中。

7. 记录 Partition 与 Consumer 的关系。





## Kafka 如何实现高可用



在 [「Kafka 的架构是怎么样的？」](http://svip.iocoder.cn/Kafka/Interview/#) 问题中，已经基本回答了这个问题。

[Kafka 集群](https://segmentfault.com/img/bVbcmpm?w=776&h=436)

- Zookeeper 部署 2N+1 节点，形成 Zookeeper 集群，保证高可用。

- Kafka Broker 部署集群。每个 Topic 的 Partition ，基于【副本机制】，在 Broker 集群中复制，形成 replica 副本，保证消息存储的可靠性。每个 replica 副本，都会选择出一个 leader 分区（Partition），提供给客户端（Producer 和 Consumer）进行读写。
- Kafka Producer 无需考虑集群，因为和业务服务部署在一起。Producer 从 Zookeeper 拉取到 Topic 的元数据后，选择对应的 Topic 的 leader 分区，进行消息发送写入。而 Broker 根据 Producer 的 `request.required.acks` 配置，是写入自己完成就响应给 Producer 成功，还是写入所有 Broker 完成再响应。这个，就是胖友自己对消息的可靠性的选择。
- Kafka Consumer 部署集群。每个 Consumer 分配其对应的 Topic Partition ，根据对应的分配策略。并且，Consumer 只从 leader 分区（Partition）拉取消息。另外，当有新的 Consumer 加入或者老的 Consumer 离开，都会将 Topic Partition 再均衡，重新分配给 Consumer 。

总的来说，Kafka 和 RocketMQ 的高可用方式是比较类似的，主要的差异在 Kafka Broker 的副本机制，和 RocketMQ Broker 的主从复制，两者的差异，以及差异带来的生产和消费不同。





## Kafka 是否会弄丢数据



**消费端弄丢了数据**



唯一可能导致消费者弄丢数据的情况，就是说，你消费到了这个消息，然后消费者那边自动提交了 offset ，让 Broker 以为你已经消费好了这个消息，但其实你才刚准备处理这个消息，你还没处理，你自己就挂了，此时这条消息就丢咯。

这不是跟 RabbitMQ 差不多吗，大家都知道 Kafka 会自动提交 offset ，那么只要关闭自动提交 offset ，在处理完之后自己手动提交 offset ，就可以保证数据不会丢。但是此时确实还是可能会有重复消费，比如你刚处理完，还没提交 offset ，结果自己挂了，此时肯定会重复消费一次，自己保证幂等性就好了。

>RocketMQ push 模式下，在确认消息被消费完成，才会提交 Offset 给 Broker 。



**Broker 弄丢了数据**



这块比较常见的一个场景，就是 Kafka 某个 Broker 宕机，然后重新选举 Partition 的 leader。大家想想，要是此时其他的 follower 刚好还有些数据没有同步，结果此时 leader 挂了，然后选举某个 follower 成 leader 之后，不就少了一些数据？这就丢了一些数据啊。

生产环境也遇到过，我们也是，之前 Partition 的 leader 机器宕机了，将 follower 切换为 leader 之后，就会发现说这个数据就丢了。

所以此时一般是要求起码设置如下 4 个参数：

- 给 Topic 设置 `replication.factor` 参数：这个值必须大于 1，要求每个 partition 必须有至少 2 个副本。

- 在 Kafka 服务端设置 `min.insync.replicas` 参数：这个值必须大于 1 ，这个是要求一个 leader 至少感知到有至少一个 follower 还跟自己保持联系，没掉队，这样才能确保 leader 挂了还有一个 follower 吧。

- 在 Producer 端设置 `acks=all`：这个是要求每条数据，必须是**写入所有 replica 之后，才能认为是写成功了**。

  >不过这个也不一定能够绝对保证，例如说，Broker 集群里，所有节点都挂了，只剩下一个节点。此时，`acks=all` 和 `acks=1` 就等价了。当然，也可以通过设置 `min.insync.replics` 参数，每次写入要求最小的同步副本数。
  >
  >这块也和朋友交流了下，他们金融场景下，`acks=all` 也是这么配置的。原因嘛，因为他们是金融场景呀。

- 在 Producer 端设置 `retries=MAX`（很大很大很大的一个值，无限次重试的意思）：这个是**要求一旦写入失败，就无限重试**，卡在这里了。



我们生产环境就是按照上述要求配置的，这样配置之后，至少在 Kafka broker 端就可以保证在 leader 所在 Broker 发生故障，进行 leader 切换时，数据不会丢失。



 **生产者会不会弄丢数据**



如果按照上述的思路设置了 `acks=all` ，一定不会丢，要求是，你的 leader 接收到消息，所有的 follower 都同步到了消息之后，才认为本次写成功了。如果没满足这个条件，生产者会自动不断的重试，重试无限次。





## Kafka 如何保证消息的顺序性



Kafka 本身，并不像 RocketMQ 一样，提供顺序性的消息。所以，提供的方案，都是相对有损的。如下：

>这里的顺序消息，我们更多指的是，单个 Partition 的消息，被顺序消费。

- 方式一，Consumer ，对每个 Partition 内部单线程消费，单线程吞吐量太低，一般不会用这个。

- 方式二，Consumer ，拉取到消息后，写到 N 个内存 queue，具有相同 key 的数据都到同一个内存 queue 。然后，对于 N 个线程，每个线程分别消费一个内存 queue 即可，这样就能保证顺序性。

  

## Kafka如何解决数据堆积

　**1. 监控和警报：**

　　实时监控Kafka集群的健康状态对于及时发现消息堆积问题至关重要。可以使用Kafka提供的监控工具或第三方监控工具进行监测，并设置警报机制，一旦发现消息堆积情况，及时采取措施进行处理。

　　**2. 扩展消费者数量：**

　　增加消费者的数量可以提高消息处理的并发性，从而减轻消息堆积的压力。可以通过增加消费者实例的数量或者增加消费者组的数量来实现。

　　**3. 提高消费者的消费能力：**

　　消费者的消费能力可能成为消息堆积的瓶颈。可以通过以下方式提高消费者的消费能力：

　　- 增加消费者的线程数量，使消费者能够并行地处理消息。

　　- 优化消费者的代码逻辑，减少处理消息的时间。

　　- 提高消费者的硬件配置，例如增加内存或CPU资源。

　　**4. 增加Kafka分区数量：**

　　如果消息堆积问题集中在某个特定的分区上，可以考虑增加该分区的数量。增加分区数量会增加消息的并行处理能力，减少单个分区的负载压力。

　　**5. 调整Kafka参数：**

　　通过调整Kafka的配置参数，可以优化消息的传递和处理效率。例如，可以调整以下参数：

　　- `max.poll.records`：每次拉取的最大消息数。

　　- `fetch.max.bytes`：每次拉取的最大字节数。

　　- `replica.fetch.max.bytes`：副本拉取的最大字节数。

　　**6. 数据清理和归档：**

　　对于已经处理完毕的消息，可以进行数据清理和归档，以减少磁盘空间的占用和提高整体性能。可以根据业务需求设置合适的数据保留

　　期限和清理策略。

　　**7. 避免生产者过载：**

　　如果生产者发送的消息量过大，可能会导致消费者无法及时处理，从而造成消息堆积。因此，需要合理设置生产者的发送速率，避免过度发送消息。