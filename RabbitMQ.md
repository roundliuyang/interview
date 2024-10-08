# RabbitMQ



## RabbitMQ 中的 Broker 是指什么？Cluster 又是指什么



- Broker ，是指一个或多个 erlang node 的逻辑分组，且 node 上运行着 RabbitMQ 应用程序

- Cluster ，是在 Broker 的基础之上，增加了 node 之间共享元数据的约束。



 **vhost 是什么？起什么作用？**

vhost 可以理解为虚拟 Broker ，即 mini-RabbitMQ server 。其内部均含有独立的 queue、exchange 和 binding 等，但最最重要的是，其拥有独立的权限系统，可以做到 vhost 范围的用户控制。当然，从 RabbitMQ 的全局角度，vhost 可以作为不同权限隔离的手段（一个典型的例子就是不同的应用可以跑在不同的 vhost 中）。

>这个，和 Tomcat、Nginx、Apache 的 vhost 是一样的概念





## RabbitMQ 概念里的 channel、exchange 和 queue 是什么



- queue 具有自己的erlang进程；
- exchange 内部实现为保存 binding关系的查找表；
- channel 是实际进行路由工作的实体，即负责按照routing_key 将 message 投递给 queue 。



由AMQP协议描述可知，channel 是真实TCP连接之上的虚拟连接，所有AMQP命令都是通过channel 发送的，且



**消息基于什么传输**

由于TCP连接的创建和销毁开销较大，且并发数受系统资源限制，会造成性能瓶颈。RabbitMQ 使用信道的方式来传输数据。信道是建立在真实的 TCP 连接内的虚拟连接，且每条 TCP连接上的信道数量没有限制。





 **能够在地理上分开的不同数据中心使用 RabbitMQ cluster 么**



不能。

- 第一，你无法控制所创建的 queue 实际分布在 cluster 里的哪个 node 上（一般使用 HAProxy + cluster 模型时都是这样），这可能会导致各种跨地域访问时的常见问题。
- 第二，Erlang 的 OTP 通信框架对延迟的容忍度有限，这可能会触发各种超时，导致业务疲于处理。
- 第三，在广域网上的连接失效问题将导致经典的“脑裂”问题，而 RabbitMQ 目前无法处理。（该问题主要是说 Mnesia）







## 如何确保消息正确地发送至 RabbitMQ



RabbitMQ 使用**发送方确认模式**，确保消息正确地发送到 RabbitMQ。

- 发送方确认模式：将信道设置成 confirm 模式（发送方确认模式），则所有在信道上发布的消息都会被指派一个唯一的 ID 。一旦消息被投递到目的队列后，或者消息被写入磁盘后（可持久化的消息），信道会发送一个确认给生产者（包含消息唯一ID）。如果 RabbitMQ 发生内部错误从而导致消息丢失，会发送一条 nack（not acknowledged，未确认）消息。
- 发送方确认模式是异步的，生产者应用程序在等待确认的同时，可以继续发送消息。当确认消息到达生产者应用程序，生产者应用程序的回调方法就会被触发来处理确认消息。





### **~~消息怎么路由~~**



从概念上来说，消息路由必须有三部分：交换器、路由、绑定。

- 生产者把消息发布到**交换器**上；

- **绑定**决定了消息如何从路由器路由到特定的队列；

  >如果一个路由绑定了两个队列，那么发送给该路由时，这两个队列都会增加一条消息。

- 消息最终到达**队列**，并被消费者接收。



常用的交换器主要分为一下三种：

- direct：如果路由键完全匹配，消息就被投递到相应的队列。
- fanout：如果交换器收到消息，将会广播到所有绑定的队列上。
- topic：可以使来自不同源头的消息能够到达同一个队列。 使用 topic 交换器时，可以使用通配符，比如：`“*”` 匹配特定位置的任意文本， `“.”` 把路由键分为了几部分，`“#”` 匹配所有规则等。特别注意：发往 topic 交换器的消息不能随意的设置选择键（routing_key），必须是由 `"."` 隔开的一系列的标识符组成。





## 如何确保消息接收方消费了消息



RabbitMQ 使用**接收方消息确认机制**，确保消息接收方消费了消息。

- 接收方消息确认机制：消费者接收每一条消息后都必须进行确认（消息接收和消息确认是两个不同操作）。只有消费者确认了消息，RabbitMQ 才能安全地把消息从队列中删除。

这里并没有用到超时机制，RabbitMQ 仅通过 Consumer 的连接中断来确认是否需要重新发送消息。也就是说，只要连接不中断，RabbitMQ 给了 Consumer 足够长的时间来处理消息。

下面罗列几种特殊情况：

- 如果消费者接收到消息，在确认之前断开了连接或取消订阅，RabbitMQ 会认为消息没有被分发，然后重新分发给下一个订阅的消费者。（可能存在消息重复消费的隐患，需要根据 bizId 去重）
- 如果消费者接收到消息却没有确认消息，连接也未断开，则 RabbitMQ 认为该消费者繁忙，将不会给该消费者分发更多的消息。



### 如何避免消息重复投递或重复消费



在消息生产时，MQ内部针对每条生产者发送的消息生成一个 inner-msg-id ，作为去重和幂等的依据（消息投递失败并重传），避免重复的消息进入队列

在消费消息时，要求消息体中必须要有一个 bizId（对于同一业务全局唯一，如支付 ID、订单 ID、帖子 ID 等）作为去重和幂等的依据，避免同一条消息被重复消费。



### 消息如何分发



若该队列至少有一个消费者订阅，消息将以循环（round-robin）的方式发送给消费者。每条消息只会分发给一个订阅的消费者（前提是消费者能够正常处理消息并进行确认）。





### **~~RabbitMQ 有几种消费模式~~**



RabbitMQ 有 pull 和 push 两种消费模式。





## 为什么不应该对所有的 message 都使用持久化机制



首先，必然导致性能的下降，因为写磁盘比写 RAM 慢的多，message 的吞吐量可能有 10 倍的差距。

其次，message 的持久化机制用在 RabbitMQ 的内置 Cluster 方案时会出现“坑爹”问题

所以，是否要对 message 进行持久化，需要综合考虑性能需要，以及可能遇到的问题。若想达到 100,000 条/秒以上的消息吞吐量（单 RabbitMQ 服务器），则要么使用其他的方式来确保 message 的可靠 delivery ，要么使用非常快速的存储系统以支持全持久化（例如使用 SSD）。另外一种处理原则是：仅对关键消息作持久化处理（根据业务重要程度），且应该保证关键消息的量不会导致性能瓶颈。





### **message 被可靠持久化的条件**



为什么说保证 message 被可靠持久化的条件是 queue 和 exchange 具有 durable 属性，同时 message 具有 persistent 属性才行？

binding 关系可以表示为 exchange–binding–queue 。从文档中我们知道，若要求投递的 message 能够不丢失，要求 message 本身设置 persistent 属性，同时要求 exchange 和 queue 都设置 durable 属性。

- 其实这问题可以这么想，若 exchange 或 queue 未设置 durable 属性，则在其 crash 之后就会无法恢复，那么即使 message 设置了 persistent 属性，仍然存在 message 虽然能恢复但却无处容身的问题。
- 同理，若 message 本身未设置 persistent 属性，则 message 的持久化更无从谈起。





### **如何确保消息不丢失**



消息持久化的前提是：将交换器/队列的 durable 属性设置为 true ，表示交换器/队列是持久交换器/队列，在服务器崩溃或重启之后不需要重新创建交换器/队列（交换器/队列会自动创建）。

如果消息想要从 RabbitMQ 崩溃中恢复，那么消息必须：

- 在消息发布前，通过把它的 “投递模式” 选项设置为2（持久）来把消息标记成持久化
- 将消息发送到持久交换器
- 消息到达持久队列

RabbitMQ 确保持久性消息能从服务器重启中恢复的方式是，将它们写入磁盘上的一个持久化日志文件。

- 当发布一条持久性消息到持久交换器上时，RabbitMQ 会在消息提交到日志文件后才发送响应（如果消息路由到了非持久队列，它会自动从持久化日志中移除）。
- 一旦消费者从持久队列中消费了一条持久化消息，RabbitMQ 会在持久化日志中把这条消息标记为等待垃圾收集。如果持久化消息在被消费之前 RabbitMQ 重启，那么 RabbitMQ 会自动重建交换器和队列（以及绑定），并重播持久化日志文件中的消息到合适的队列或者交换器上。



## 什么是死信队列



DLX，Dead-Letter-Exchange。利用 DLX ，当消息在一个队列中变成死信（dead message）之后，它能被重新 publish 到另一个 Exchange ，这个 Exchange 就是DLX。消息变成死信一向有一下几种情况：

- 消息被拒绝（basic.reject / basic.nack）并且 `requeue=false` 。
- 消息 TTL 过期（参考：RabbitMQ之TTL（Time-To-Live 过期时间））。
- 队列达到最大长度。



**“dead letter”queue 的用途**

当消息被 RabbitMQ server 投递到 consumer 后，但 consumer 却通过 `Basic.Reject` 进行了拒绝时（同时设置 `requeue=false`），那么该消息会被放入 “dead letter” queue 中。该 queue 可用于排查 message 被 reject 或 undeliver 的原因。



**Basic.Reject 的用法是什么**

该信令可用于 consumer 对收到的 message 进行 reject 。

- 若在该信令中设置 `requeue=true` ，则当 RabbitMQ server 收到该拒绝信令后，会将该 message 重新发送到下一个处于 consume 状态的 consumer 处（理论上仍可能将该消息发送给当前 consumer）。
- 若设置 `requeue=false` ，则 RabbitMQ server 在收到拒绝信令后，将直接将该 message 从 queue 中移除

另外一种移除 queue 中 message 的小技巧是，consumer 回复 `Basic.Ack` 但不对获取到的 message 做任何处理。而 `Basic.Nack`是对 Basic.Reject 的扩展，以支持一次拒绝多条 message 的能力。





## RabbitMQ 如何实现高可用



RabbitMQ的高可用，是基于主从做高可用性的。它有三种模式：

- 单机模式
- 普通集群模式
- 镜像集群模式



### **单机模式**

单机模式，就是启动单个RabbitMQ节点，一般用于本地开发或者测试环境。实际生产环境下，基本不会使用。



### **普通集群模式（无高可用性）**

普通集群模式下，不同的节点之间只会相互同步元数据，不同步具体的消息，若需要具体消息时需要转发到源节点(1)

![「RabbitMQ」高级使用：集群及haproxy+keepalived高可用负载均衡](RabbitMQ.assets/3238_90_4biYKXXCobMSIYPn)

为什么不直接把队列的内容（消息）在所有节点上复制一份？
主要是出于存储和同步数据的网络开销的考虑，如果所有节点都存储相同的数据，就无法达到线性地增加性能和存储容量的目的（堆机器）。

- 假如生产者连接的是节点3，要将消息通过交换机A路由到队列1，最终消息还是会转发到节点1上存储，因为队列1的内容只在节点1上。
- 同理，如果消费者连接是节点 2，要从队列 1上拉取消息，消息会从节点1 转发到节点2。其它节点起到一个路由的作用，类似于指针。

普通集群模式不能保证队列的高可用性，因为队列内容不会复制。如果节点失效将导致相关队列不可用。



- 普通集群模式，意思就是在多台机器上启动多个RabbitMQ实例，每个机器启动一个。你**创建的 queue，只会放在一个 RabbitMQ 实例上**，但是每个实例都同步queue的**元数据**（元数据可以认为是 queue 的一些配置信息，通过元数据，可以找到 queue 所在实例）。
- 你消费的时候，实际上如果连接到了另外一个实例，那么那个实例会从 queue 所在实例上拉取数据过来。

![架构图](RabbitMQ.assets/75d36ed17c91932e28b5eeba681ad8ec)

这种方式确实很麻烦，也不怎么好，**没做到所谓的分布式**，就是个普通集群。因为这导致你要么消费者每次随机连接一个实例然后拉取数据，要么固定连接那个 queue 所在实例消费数据，前者有**数据拉取的开销**，后者导致**单实例性能瓶颈**。

而且如果那个放 queue 的实例宕机了，会导致接下来其他实例就无法从那个实例拉取，如果你**开启了消息持久化**，让 RabbitMQ 落地存储消息的话，**消息不一定会丢**，得等这个实例恢复了，然后才可以继续从这个 queue 拉取数据。

所以这个事儿就比较尴尬了，这就**没有什么所谓的高可用性**，**这方案主要是提高吞吐量的**，就是说让集群中多个节点来服务某个 queue 的读写操作。





### **镜像集群模式（高可用性）**

这种模式，才是所谓的 RabbitMQ 的高可用模式。跟普通集群模式不一样的是，在镜像集群模式下，你创建的 queue，无论元数据还是 queue 里的消息都会**存在于多个实例上**，就是说，每个 RabbitMQ 节点都有这个 queue 的一个**完整镜像**，包含 queue 的全部数据的意思。然后每次你写消息到 queue 的时候，都会自动把**消息同步**到多个实例的 queue 上。

镜像队列模式下，消息内容会在镜像节点间同步，可用性更高。不过也有一定的**副作用**，**系统性能会降低**，节点过多的情况下同步的代价比较大。镜像队列模式下，消息内容会在镜像节点间同步，可用性更高。不过也有一定的副作用，系统性能会降低，节点过多的情况下同步的代价比较大。

![架构图](RabbitMQ.assets/20a6c4d82a08becf7e8d913662b00357)

那么**如何开启这个镜像集群模式**呢？其实很简单，RabbitMQ 有很好的管理控制台，就是在后台新增一个策略，这个策略是**镜像集群模式的策略**，指定的时候是可以要求数据同步到所有节点的，也可以要求同步到指定数量的节点，再次创建 queue 的时候，应用这个策略，就会自动将数据同步到其他的节点上去了。

| 操作方式              | 命令或步骤                                                   |
| --------------------- | ------------------------------------------------------------ |
| rabbitmqctl (Windows) | rabbitmqctl set_policy ha-all “^ha.” “{”“ha-mode”":"“all”"}" |
| HTTP API              | PUT /api/policies/%2f/ha-all {“pattern”:"^ha.", “definition”:{“ha-mode”:“all”}} |
| WebUI                 | 1、 avigate to Admin > Policies > Add / update a policy 2、 Name 输入：mirror_image 3、 Pattern输入：^ (代表匹配所有） 4、 Definition 点击 HA mode，右边输入：all 5、 Add policy |



## RabbitMQ 是否会弄丢数据





![弄丢消息的几种情况](RabbitMQ.assets/6303e69011255831c54d605250a6aa67)

​																						*弄丢消息的几种情况*



### **生产者弄丢了数据**



生产者将数据发送到 RabbitMQ 的时候，可能数据就在半路给搞丢了，因为网络问题啥的，都有可能。



**方案一：事务功能**

此时可以选择用RabbitMQ提供的【事务功能】，就是生产者**发送数据之前**开启RabbitMQ事务

`channel.txSelect`，然后发送消息，如果消息没有成功被RabbitMQ接收到，那么生产者会收到异常报错，此时就可以回滚事务`channel.txRollback`，然后重试发送消息；如果收到了消息，那么可以提交事务`channel.txCommit`。代码如下：

```java
// 开启事务
channel.txSelect
try {
    // 这里发送消息
} catch (Exception e) {
    channel.txRollback

    // 这里再次重发这条消息
}

// 提交事务
channel.txCommit
```

但是问题是，RabbitMQ 事务机制（同步）一搞，基本上**吞吐量会下来，因为太耗性能**。





**方案二：confirm 功能**

 所以一般来说，如果你要确保说写 RabbitMQ 的消息别丢，可以开启 【`confirm` 模式】，在生产者那里设置开启 `confirm` 模式之后，你每次写的消息都会分配一个唯一的 id，然后如果写入了 RabbitMQ 中，RabbitMQ 会给你回传一个 `ack` 消息，告诉你说这个消息 ok 了。如果 RabbitMQ 没能处理这个消息，会回调你的一个 `nack` 接口，告诉你这个消息接收失败，你可以重试。而且你可以结合这个机制自己在内存里维护每个消息 id 的状态，如果超过一定时间还没接收到这个消息的回调，那么你可以重发。



**对比总结**

事务机制和 `confirm` 机制最大的不同在于，**事务机制是同步的**，你提交一个事务之后会**阻塞**在那儿，但是 `confirm` 机制是**异步**的，你发送个消息之后就可以发送下一个消息，然后那个消息 RabbitMQ 接收了之后会异步回调你的一个接口通知你这个消息接收到了。

所以一般在生产者这块**避免数据丢失**，都是用 `confirm` 机制的。





###  **Broker 弄丢了数据**



就是Broker 自己弄丢了数据，这个你必须开启Broker 的持久化，就是消息写入之后会持久化到磁盘，哪怕是 Broker 自己挂了，**恢复之后会自动读取之前存储的数据**，一般数据不会丢。除非极其罕见的是，Broker 还没持久化，自己就挂了，**可能导致少量数据丢失**，但是这个概率较小。

设置持久化的两个步骤：

- 创建 queue 的时候将其设置为持久化。这样就可以保证 RabbitMQ 持久化 queue 的元数据，但是它是不会持久化 queue 里的数据的。
- 第二个是发送消息的时候将消息的 `deliveryMode` 设置为 2
  就是将消息设置为持久化的，此时 RabbitMQ 就会将消息持久化到磁盘上去。

必须要同时设置这两个持久化才行，Broker 哪怕是挂了，再次重启，也会从磁盘上重启恢复 queue，恢复这个 queue 里的数据。

注意，哪怕是你给 Broker 开启了持久化机制，也有一种可能，就是这个消息写到了 Broker 中，但是还没来得及持久化到磁盘上，结果不巧，此时 Broker 挂了，就会导致内存里的一点点数据丢失。

所以，持久化可以跟生产者那边的 `confirm` 机制配合起来，只有消息被持久化到磁盘之后，才会通知生产者 `ack` 了，所以哪怕是在持久化到磁盘之前，Broker 挂了，数据丢了，生产者收不到 `ack`，你也是可以自己重发的。





### **消费端弄丢了数据**



RabbitMQ如果丢失了数据，主要是因为你消费的时候，**刚消费到，还没处理，结果进程挂了**，比如重启了，那么就尴尬了，RabbitMQ 认为你都消费了，这数据就丢了。

这个时候得用 RabbitMQ 提供的 `ack` 机制，简单来说，就是你必须关闭 RabbitMQ 的自动 `ack`，可以通过一个 api 来调用就行，然后每次你自己代码里确保处理完的时候，再在程序里 `ack` 一把。这样的话，如果你还没处理完，不就没有 `ack` 了？那 RabbitMQ 就认为你还没处理完，这个时候 RabbitMQ 会把这个消费分配给别的 consumer 去处理，消息是不会丢的。



**总结**

![总结](RabbitMQ.assets/83213e2ac79cd8899b09a66a5cf71669)



## RabbitMQ 如何保证消息的顺序性



和 Kafka 与 RocketMQ 不同，Kafka 不存在类似类似 Topic 的概念，而是真正的一条一条队列，并且**每个队列可以被多个 Consumer 拉取消息**。这个，是非常大的一个差异。



 **来看看 RabbitMQ 顺序错乱的场景**：

一个 queue，多个 consumer。比如，生产者向 RabbitMQ 里发送了三条数据，顺序依次是 data1/data2/data3，压入的是 RabbitMQ 的一个内存队列。有三个消费者分别从 MQ 中消费这三条数据中的一条，结果消费者 2 先执行完操作，把 data2 存入数据库，然后是 data1/data3。这不明显乱了。😈 也就是说，乱序消费的问题。

![乱序](RabbitMQ.assets/29ea655479f826f9a6a5d9d005f46c7c)

 **解决方案**：



方案一， 拆分多个queue,每个queue一个consumer,就是多一些queue而已，确实是麻烦点。

>这个方式，有点模仿 Kafka 和 RocketMQ 中 Topic 的概念。例如说，原先一个 queue 叫 `"xxx"` ，那么多个 queue ，我们可以叫 `"xxx-01"`、`"xxx-02"` 等，相同前缀，不同后缀。

RabbitMQ 的问题是由于不同的消息都发送到了同一个 queue 中，多个消费者都消费同一个 queue 的消息。解决这个问题，我们可以给 RabbitMQ 创建多个 queue，每个消费者固定消费一个 queue 的消息，生产者发送消息的时候，同一个订单号的消息发送到同一个 queue 中，由于同一个 queue 的消息是一定会保证有序的，那么同一个订单号的消息就只会被一个消费者顺序消费，从而保证了消息的顺序性。

如下图是 RabbitMQ 保证消息顺序性的方案：

![img](RabbitMQ.assets/48cd8e82cc15e0287c952450ad7e41d0.png)





### 真的能保证顺序吗

目前很多资料显示RabbitMQ的消息能够保障顺序性，这是不正确的，或者说这个观点有很大的局限性。在不使用任何RabbitMQ的高级特性，也没有消息丢失、网络故障之类异常的情况发生，并且只有一个消费者的情况下，最好也只有一个生产者的情况下可以保证消息的顺序性。如果有多个生产者同时发送消息，无法确定消息到达Broker 的前后顺序，也就无法验证消息的顺序性。



**那么哪些情况下RabbitMQ的消息顺序性会被打破呢?下面介绍几种常见的情形。**

- 如果生产者使用了**事务机制**，在发送消息之后遇到异常进行了事务回滚，那么需要重新补偿发送这条消息，如果补偿发送是在另一个线程实现的，那么消息在生产者这个源头就出现了错序。
- 同样，如果启用publisher confirm时，**在发生超时、中断，又或者是收到RabbitMQ的Basic.Nack命令时，那么同样需要补偿发送，结果与事务机制一样会错序**。或者这种说法有些牵强，我们可以固执地认为消息的顺序性保障是从存入队列之后开始的，而不是在发送的时候开始的。
- 考虑另一种情形，如果生产者发送的消息设置了不同的**超时时间**，并且也设置了死信队列，整体上来说相当于一个延迟队列，那么消费者在消费这个延迟队列的时候，消息的顺序必然不会和生产者发送消息的顺序一致。
- ......

包括但不仅限于以上几种情形会使RabbitMQ消息错序。如果要保证消息的顺序性，需要业务方使用RabbitMQ之后做进一步的处理，比如在消息体内添加全局有序标识（类似Sequence ID）来实现。





## RabbitMQ消息堆积



### **消息堆积原因**

- 消息堆积即消息没及时被消费，是生产者生产消息速度快于消费者消费的速度导致的。
- 消费者消费慢可能是因为：本身逻辑耗费时间较长、阻塞了。



### **预防措施**

**生产者**

1. 减少发布频率
2. 考虑使用队列最大长度限制



**消费者**

默认情况下，rabbitmq消费者为单线程串行消费（org.springframework.amqp.rabbit.listener.SimpleMessageListenerContainer类的concurrentConsumers与txSize（对应prefetchCount）都是1），设置并发消费两个关键属性concurrentConsumers和prefetchCount。concurrentConsumers：设置的是对每个listener在初始化的时候设置的并发消费者的个数；prefetchCount：每次从broker里面取的待消费的消息的个数。





### **已出事故的解决措施**



#### **情况1：堆积的消息还需要使用**

**方案1：简单修复**

修复consumer的问题，让他恢复消费速度，然后等待几个小时消费完毕



**方案2：复杂修复**

临时紧急扩容了，具体操作步骤和思路如下：

1）先修复consumer的问题，确保其恢复消费速度
2）新建一个topic，partition是原来的10倍，临时建立好原先10倍或者20倍的queue数量
3）然后写一个临时的分发数据的consumer程序，这个程序部署上去消费积压的数据，消费之后不做耗时的处理，直接均匀轮询写入临时建立好的10倍数量的queue
4）接着临时征用10倍的机器来部署consumer，每一批consumer消费一个临时queue的数据
5）这种做法相当于是临时将queue资源和consumer资源扩大10倍，以正常的10倍速度来消费数据
6）等快速消费完积压数据之后，得恢复原先部署架构，重新用原先的consumer机器来消费消息



**方案3：Shovel**

在一筹莫展之时，不如试一下Shovel。当某个队列中的消息堆积严重时，比如超过某个设定的阈值，就可以通过Shovel将队列中的消息交给另一个集群。



#### **情况2：堆积的消息不需要使用**

删除消息即可。（可以在RabbitMQ控制台删除，或者使用命令）。