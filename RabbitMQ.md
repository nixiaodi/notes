# RabbitMQ

## 1 概述

### 1.1 RabbitMQ简介

RabbitMQ队列基于AMQP协议使用Erlang语言开发实现，支持多客户端类型如Java、Ruby、Go、PHP等。其余比较流行的消息队列中间件，相对的还有RocketMQ、ActiveMQ

MQ最大的三个特点就是异步、削峰、解耦，最简单的概括就是存储数据的容器，与MySQL和Redis数据库类似，区别在于自身实现特点决定了应用场景

![img](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/16ddc976f3098e32)

### 1.2 AMQP协议

可以简单理解为一套消息传递的标准协议，例如HTTP、HTTPS协议都有自身的规则，AMQP协议基本模型

![img](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/16ddc976f4f1bd33)

| 组件         | 描述                                                         |
| :----------- | :----------------------------------------------------------- |
| Publisher    | 生产者，应用客户端用于向服务端发送消息                       |
| Connection   | 连接，生产者与消费者客户端都需要Connection连接消息应用服务端。Connection连接创建、销毁成本太高，衍生轻量级逻辑连接Channel |
| Channel      | 轻量级逻辑连接，也称之为信道。Channel之间完全隔离，线程安全  |
| Broker       | 消息应用服务主体，客户端写代码不会涉及，相当于一个逻辑上的概念 |
| Virtual Host | 相当于namespace，多用户时每个用户可以在自己分配的Virtual Host区域操作 |
| Exchange     | 消息交换器，生产者不直接与队列耦合，通过交换器进行消息转发   |
| Binding      | 绑定关系，消息交换器与队列之间绑定的关系，通过与生产者传递消息携带的RoutingKey比较得知消息路由转发到哪个绑定队列 |
| Queue        | 队列，消息最后储存的位置                                     |
| Consumer     | 消费者，直接通过与队列耦合进行消费，这与生产者具备一定区别   |

相关概念:

#### 1.2.1 消息(Message)

`消息`一般可以包含2个部分：`消息体`和`消息属性`。`消息体`（也成payload）一般是带有业务逻辑结构的数据，比如Json字符串；`消息属性`用于表述消息，比如消息对应的Exchange名称和RoutingKey

#### 1.2.2 队列(Queue)

`队列`在RabbitMQ中用于`存储消息`,生产者所发送的消息最终被投递到队列中，消费者从队列获取消息并消费

- RabbitMQ中消息只存储在队列中，Exchange只做路由、不存储消息

- 所有队列都绑定到默认交换机exchange=""，并且绑定时使用的routingKey=queueName

  ![img](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/1722d5a23685a99a)

#### 1.2.3 路由键(RoutingKey)

生产者在发送消息时，需要指定Exchange和`RoutingKey`，并且队列和Exchange之间的绑定关系也通过RoutingKey来标识

#### 1.2.4 绑定(Binding)

`绑定`可以理解为一个三元组<queueName, exchangeName, bindingKey>，表示将queue用bindingKey绑定到exchange，该exchange收到消息时，会根据exchange类型和消息的routingKey将消息路由到符合的队列

- 也可以将交换机绑定到交换机

![img](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/1722d3ca9c56576a)

#### 1.2.5 节点(Broker)

`节点`是一个RabbitMQ服务器实例，一个集群中有多个节点，这些节点可以在多台服务器上，也可以在一台服务器的不同端口

#### 1.2.6 虚拟主机(Virtual Host)

`虚拟主机`是共享相同身份认证和加密环境的独立服务器域。不同虚拟主机之间是**相互隔离**的，拥有独立的交换机、队列、绑定等

- 可以将`虚拟主机与节点之间的关系`理解为`数据库与mysql之间的关系`，一台mysql服务上有不同的数据库

#### 1.2.7 连接(Connection)

`Connection`是客户端与服务器之间建立的一个网络连接

#### 1.2.8 信道(Channel)

`Channel`可以理解为一种轻量连接，多个channel共享一个Connection,由于创建和销毁Connection的开销大，所以在Connection内部建立Channel

- 不同的Channel之间`互相隔离`
- Channel是`双向`的数据通道
- RabbitMQ几乎所有操作均是`通过Channel完成`，包括发送消息、创建队列、交换机等、消费消息。先创建Connection、然后创建Channel，再使用Channel执行操作

![img](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/1722d4077af160e3)

#### 1.2.9 生产者(Provider)

`生产者`就是发送消息的一方。

- 个人理解本质上就是一个channel，channel执行了basicPublish的API

#### 1.2.10 消费者(Consumer)

`消费者`就是接收消息的一方。

- 个人理解本质上是一个channel中basicConsume的对应的回调函数

### 1.3 AMQP交互流程

HTTP协议连接创建、销毁过程划分为三次握手、四次挥手，那么在AMQP协议中的生产者、消费者客户端与消息应用服务端交互是怎样的一个流程设计？

生产者、消费者与服务端连接创建、销毁命令保持一致，但是中间逻辑流程涉及到的命令肯定不同；所以将整个生产 -- 消费流程划分为三个部分，连接创建与销毁、生产消息、消费消息

#### 1.3.1 连接创建销毁

连接的创建销毁包含四个模块的通信，首先是连接创建开启，然后信道创建开启，最后则是信道关闭、连接关闭。从流程中可以更加理解Channel是基于Connection的逻辑连接，必须依赖于连接Connection。其余的步骤则是命令--确认命令的格式，确保客户端、服务端通信正常

![img](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/16ddc976f4d40e5f)

#### 1.3.2 生产消息

消息生产重点在于服务端对于这个操作是没有任何响应的，这里就是一个造成消息丢失的环节，当服务端和客户端之间网络连接出现波动等问题导致连接不可用关闭，这时传输的消息就可能丢失

![img](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/16ddc976f4c74ba1)

#### 1.3.3 消息消费

消息消费的重点在于确认机制，即消息由服务端推送至客户端并不会立即删除，而是变更消息状态标志，等待客户端反馈确认结果后在执行相对应的逻辑。当然反馈确认的结果可能有多种，操作也就有多种

![img](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/16ddc976f4ba37d2)

## 2 组件

### 2.1 Exchange

为什么要在生产者和队列之间提出交换机的概念？

- 如果生产者和队列直接耦合，当生产者客户端需要将一条消息发送到多个队列，则需要进行多次操作。多次操作不仅仅意味着连接性能损耗，虽然Channel是轻量级，并且意味着客户端复杂度上升
- 提出交换机概念，生产者只需要提前绑定设置好其与队列关系，发送消息时携带路由转发规则，后续逻辑交给服务端处理即可

#### 2.1.1 交换机参数

以Java Channel接口API来说明创建交换机的主要参数

```java
 Exchange.DeclareOk exchangeDeclare(String exchange, String type, boolean durable, boolean autoDelete, boolean internal, Map<String, Object> arguments) throws IOException;
```

- `exchange`：交换机名称
- `type`：交换机类型
- `durable`：是否持久化，取值为true时，broker重启后交换机仍然存在
- `autoDelete`：是否自动删除，有一个队列与该交换机绑定过，在该交换机所有绑定关系都被解绑时，交换机会被MQ自动删除
- `internal`：是否是内置的，客户端无法直接将消息发送给内置交换机，只能通过其它交换机将消息路由到内置交换机
- arguments

#### 2.1.2 交换器类型

不同类型的交换器针对不同场景应用而设计，如将消息路由至全部绑定队列选用fanout，将消息路由至指定routingKey队列选用direct、将消息路由至模糊匹配routingKey则选用topic

![img](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/16df17a99feea9f0)

| 类型   | 路由匹配规则            | 备注                                                         |
| ------ | ----------------------- | ------------------------------------------------------------ |
| fanout | 消息路由到所有绑定队列  | 交换器队列绑定无需binding，消息发送无需routingKey            |
| direct | routingKey与binding比较 | 一致性校验路由                                               |
| topic  | routingKey与binding比较 | 模糊匹配，binding/routingKey单词层次间用`"."` 标识，binding绑定关系支持`"*"`与`"#"`两种模糊规则。如com.message.log.* |

| 模糊规则 | 规则                                                         |
| -------- | ------------------------------------------------------------ |
| *        | 代表一层，如绑定关系binding为 log.* 则与 routingKey 为 log.error 或 log.info 的消息匹配 |
| #        | 代表0或多层，如绑定关系binding为 log.# 则与 routingKey 为 log 或 log.message.info 的消息匹配 |

##### 2.1.2.1 广播模式(fanout)

当一个交换机是广播模式时，所有绑定到该交换机上的队列(或交换机)都能收到路由的消息，不论bingdingKey的取值

- 对`队列`而言，无论队列和交换机绑定时`bindingkey`取值是什么，也不论发送者发送消息时的`routingkey`是什么，该队列都可以收到广播交换机上所有的消息

- 对于绑定到广播交换机上的`交换机`而言同样如此

  假设Exchange1是广播交换机，Exchange2是direct交换机，队列Queue1、Queue2绑定到Exchange2交换机。那么Exchange2可以收到Exchange1上所有消息，但是队列Queue1和Queue2只能收到routingKey（生产者设定）与bindingKey一致的消息

  ![img](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/172312883fd523c2)

##### 2.1.2.2 direct模式

direct类型交换机会将消息路由到`bindingkey与routingkey完全一致`的队列上

- 一个队列可以使用不同的routingkey绑定到交换机多次，用于接收多种消息，比如对于消息日志，日志分不同级别，queue1可以配置成接收debug、warning、info、error四种消息，queue2可以配置成只接收error消息

  ![img](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/172312d84f209c13)

##### 2.1.2.3 topic模式

当`routingkey`中使用`点号`分割时，bindingkey可以表现为类似正则表达式的形式，此时队列可以接收到符合bindingkey规则的多个routingkey的消息，而不用像direct模式一样进行多次绑定

- 生产者发送消息时，使用的routingkey是完整的，且每个单词间用.分割
- bindingkey中可以使用特殊字符`*`和`#`，`*`用来匹配一个单词，#用来匹配0或多个单词

对于routingKey为`quick.orange.rabbit`、`lazy.orange.elephant`、`quick.orange.fox`、`lazy.brown.fox`、`lazy.pink.rabbit`，Q1可以收到前3个routingKey的消息，Q2可以收到第1、2、4、5个routingKey的消息

![img](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/17231468a983ba69)

### 2.2 队列

以Java Channel接口API来说明创建队列的主要参数

```java
public Queue.DeclareOk queueDeclare(String queue, boolean durable, boolean exclusive, boolean autoDelete, Map<String, Object> arguments)
```

- `queue`：队列名

- `durable`：是否是持久化的。取值为true时，broker重启队列也存在。需要注意的是`队列持久化`不代表`消息持久化`

  项目中使用了RabbitMQ Cluster，且使用了持久化队列，在RabbitMQ Cluster中，队列只存在于一个broker上，当该broker不可访问(宕机、发生网络分区等原因)时，且没有镜像队列来保证高可用性，会使得队列不可用

  由于业务上可以容忍短时间内一些数据丢失，这时候重新再重连时进行队列的删除、重新创建等操作，不过由于队列设置了持久化，会使得剩余broker同样存有队列元数据，导致无法执行删除操作，后考虑将队列durable属性设为false，这样队列所在节点宕机后，其它节点上也不存在队列，可以重新创建队列，保证一定程度可用性

- `exclusive`：是否是排他的；取值为true时，队列仅对首次声明它的Connection可见。包含以下含义：

  - 队列存在时，后续执行声明队列的Connection无法访问队列
  - 占有队列的Connection内Channel都可以访问队列
  - 当占有队列的Connection断开时，队列将被自动删除

- `autoDelete`：是否是自动删除的；当队列被消费者连接过，之后`所有消费者连接`都断开时，队列将会被自动删除。包含以下含义：

  - 当一个队列被创建了，还没有被消费时，队列是不会被删除的
  - 所有消费连接断开时，队列会被删除，即便现在还有生产者在发送消息，队列已经不存在了

- `arguments`

**需要注意的是，每个队列都绑定了默认direct模式交换机，且bindkey为队列名，需要直接往队列发送消息的话，可以设置`exchange=""`、`routingKey=queueName`**

### 2.3 绑定

绑定可以分为`队列绑定到交换机`和`交换机绑定到交换机`

#### 2.3.1 队列绑定到交换机

以Java Channel接口API来说明队列绑定到交换机的主要参数：

```java
public Queue.BindOk queueBind(String queue, String exchange,String routingKey, Map<String, Object> arguments)
```

- `queue`：队列名
- `exchange`：交换机名
- `routingKey`：称为bindingKey更加合适

#### 2.3.2 交换机绑定到交换机

以Java Channel接口API来说明交换机绑定到交换机的主要参数：

```java
public Exchange.BindOk exchangeBind(String destination, String source,String routingKey, Map<String, Object> arguments)
```

`destination`：可以类比队列绑定的参数，用`消息传递的目的方`更好理解

`source`：被绑定的交换机，用`消息传递的来源`更好理解。

`routingKey`：称为bindingKey更加合适。 被发送到`source交换机`的消息，如果消息的`routingkey`匹配`destination交换机`的`bindingkey`或者说`source交换机`是广播模式，那么消息将被转发到`destination交换机`

### 2.4 其他概念

#### 2.4.1 consumerTag

`consumerTag`是`消费者唯一标识`，用于区分不同的消费者。

- 实际测试，同一channel里面不同consumer的consumerTag也不同

#### 2.4.2 deliveryTag

`deliveryTag`是消息被投递给消费者时的唯一编号，该参数最常用与消费者ack(acknowledge)消息

- 实际测试，deliveryTag唯一的范围是`channel`，而不是consumer。同一个channel的不同consumer收到消息时的deliveryTag不同

#### 2.4.3 correlationId

`correlationId`是请求唯一标识，可用于将RPC的请求和响应关联起来，需要客户端在生产消息时生成并并设置给消息属性

## 3 RabbitMQ特性

### 3.1 消费模式

RabbitMQ消费模式分为两种，`推模式`(basic.consume)和`拉模式`(basic.get)

- 推模式下channel被设置为投递模式，在消费者订阅队列之后，RabbitMQ会不停将消息推送给消费者，直到取消订阅（或者未ack消息数目达到上限）。
- 推模式适合`持续订阅`，拉模式适合订阅单条消息。
- 如果用在循环中basic.get来代替basic.consume，会严重影响RabbitMQ性能。推模式`吞吐量`远高于拉模式，一般都应采用`推模式`；

### 3.2 生产者确认

生产者如何确保发送的消息被投递到RabbitMQ中呢？

RabbitMQ提供两种方式实现生产者确认：

- 事务机制
- 发送方确认(publisher confirm)机制

#### 3.2.1 事务机制

事务相关操作涉及`事务开启`(channel.txSelect())、`事务提交`(channel.txCommit())和`事务回滚`(channel.txRollback())。 先进行事务开启，然后发送消息，最后事务提交，提交失败进行事务回滚。在执行事务开启和事务提交之后，RabbitMQ都会回复OK回调。`事务提交成功，则消息一定到达了RabbitMQ`

需要注意的是，`事务`对`性能`影响巨大，批量发送消息`开启事务`和`不开启事务`性能可以相差百倍

#### 3.2.2 发送方确认机制

发送方可以将`channel`设置为`confirm`模式（channel.confirmSelect()），生产者发送消息后，RabbitMQ需要对生产者发送的消息进行`confirm`，生产者可以等待RabbitMQ回应的confirm（channel.waitForConfirms()），如果等待confirm返回false，认为消息发送失败，此时可以重新发送

- 如果开启了持久化，消息落盘之后RabbitMQ才会confirm

发送方确认机制弥补了事务机制的缺陷，`提高了吞吐量`。 需要注意的是，如果每发送一个消息后，都等待confirm，那么吞吐量和事务机制间并没有太大差距。发送方确认机制优势在于`并不一定需要同步confirm`，可以使用`批量confirm`和`异步confirm`

- `批量confirm`：发送一批消息后，执行等待confirm函数
- `异步confirm`：提供回调函数，在收到RabbitMQ的confirm时执行回调，对于未能confirm的消息要进行重发

### 3.3 消费者确认

如果保证RabbitMQ发出的消息被消费者收到？或者说某个消费者因为机器负载较大不想处理消息，想由其它消费消费的话，该如何处理？

RabbitMQ提供了消费者确认机制，消费者执行`basic.consume`时可以设置`autoAck`

- `autoAck为true`时，当消息被`发送出去`（被写入到TCP中）时，消息就会被RabbitMQ从内存或磁盘删除（其实是先标记后删除）
   需要注意的是，这里不需要管消费者是否收到消息
- `autoAck为false`时，只有当消费者`手动ack`后，消息才会被MQ删除，在收到Consumer的nack或者reject时会将消息重新放入队列或者丢弃掉

### 3.4 消费者流控

RabbitMQ可以通过`channel.Qos()`设置`prefetchCount`或`prefetchSize`来控制Consumer接收消息的频率(流控应该是只对推模式起作用)

- `prefetchCount`：表示consumer`未ack消息的数目上限`，达到上限后consumer将无法消费消息，直至未ack消息数目降低。0表示没有上限
- `prefetchSize`：表示consumer`未ack消息的空间大小上限`，单位是B,0表示没有上限
- 两者均可以设置global参数，表示是否channel中每个消费者都进行共享流控，共享流控时，假设prefetchCount是10，那么每个消费者的prefetchCount都是10

### 3.5 持久化

RabbitMQ持久化分为`交换机持久化`、`队列持久化`和`消息持久化`

- `交换机持久化`：在声明交换机时，参数durable设置为true，borker重启后交换机仍存在
- `队列持久化`：在声明队列，参数durable设置为true，borker重启后队列仍存在
- `消息持久化`：消息持久化`首先要求队列持久化`，队列删除了消息也将丢失。在生产者发送消息时，可以通过设置BasicProperties中的`deliveryMode`=2，将投递模式设置为2，来进行消息持久化
- 需要注意的是如果所有消息都开启持久化，会严重影响RabbitMQ性能

