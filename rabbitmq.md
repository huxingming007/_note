## MQ入门

### 消息队列简介

#### 消息队列的诞生

每个公司自己实现MQ，当订阅消息的时候，需要实现不一样的逻辑，因为协议、API不同。这个时候需要定义规范，比方说JDBC的规范（协议），每个数据库厂商自己去实现协议。06年，于是乎AMQP规范发布，跨语言和跨平台，促进了消息队列的繁荣。07年，rabbitmq是基于AMQP协议，使用erlang语言，作者擅长的开发语言，erlang天生适合分布式和高并发。

#### 为什么使用MQ

- 实现异步通信：提升响应速度
- 实现系统解耦：减少依赖
- 实现流量削峰：保护应用和数据库。先把流量承接下来，转换成MQ消息发送到消息队列服务器上，业务层就可以根据自己的消费速率去处理这些消息，处理之后再返回结果。

#### 使用MQ带来的问题

- 系统可用性降低：独立运行一个服务，MQ出现问题，就会导致请求失败
- 系统复杂性提高：需要了解MQ的模型和概念，需要考虑消息重复、消息丢失、数据一致性问题
- 总结：不能盲目的使用MQ

### rabbitmq简介

#### 基本特征

1. 高可靠，在性能和可靠性之间做出了权衡，包括持久性、发送应答（ack）、发送确认（confirm机制）、高可靠性。
2. 灵活的路由
3. 支持多客户端：java、Python、PHP、GO等等。并且支持多种协议。
4. 可靠的集群方案
5. 高可用队列：镜像队列
6. 权限管理：用户与虚拟机实现权限管理
7. 插件支持
8. 与spring集成：spring对AMQP进行封装

#### AMQP

rabbitmq实现了AMQP协议，所以它的工作模型也是基于AMQP的。

![](https://tva1.sinaimg.cn/large/006y8mN6ly1g8a5v69j40j31ck0git9u.jpg)

##### channel

消费者和生产者与MQ是建立了TCP长连接，在这条连接上面创建channel用于发送和消费消息，避免了TCP连接创建和释放的性能开销。并且channel在原生的rabbit api最重要的编程接口，创建交换机、队列、绑定关系、发送消息、消费消息。

##### exchange

rabbitmq不会出现消息直接发送到队列的情况，因为有exchange的存在，用于实现消息的灵活路由。交换机是一个绑定列表，用来匹配绑定关系。

##### vhost虚拟主机

隔离业务系统，不用单独安装一个rabbitmq，使用vhost，它可以认为一个小型的rabbitmq服务器，每个vhost都有独立的交换机、队列、绑定关系、权限用户。

##### 路由方式

直连、topic、广播fanout。

![](https://tva1.sinaimg.cn/large/006y8mN6ly1g8a625haf4j31ra0su439.jpg)

### 基本使用

Exclusive：是否是排他队列。排他队列只能在声明它的connection中使用，适用于RPC场景，定义一个回调队列，不会被其他消费者消费走。

声明队列的参数：durable、exclusive、其他属性

~~~java
        // 通过队列属性设置消息过期时间
        Map<String, Object> argss = new HashMap<String, Object>();
        argss.put("x-message-ttl",6000);
        // 声明队列（默认交换机AMQP default，Direct）
        // String queue, boolean durable, boolean exclusive, boolean autoDelete, Map<String, Object> arguments
        channel.queueDeclare("TEST_TTL_QUEUE", false, false, false, argss);
~~~

![](https://tva1.sinaimg.cn/large/006y8mN6ly1g8a6gi9b54j30qc088q3l.jpg)

声明消息属性basicproperties

~~~java
        Map<String, Object> headers = new HashMap<String, Object>();
        headers.put("name", "gupao");
        headers.put("level", "top");

        AMQP.BasicProperties properties = new AMQP.BasicProperties.Builder()
                .deliveryMode(2)   // 2代表持久化
                .contentEncoding("UTF-8")  // 编码
                .expiration("10000")  // TTL，过期时间
                .headers(headers) // 自定义属性
                .priority(5) // 优先级，默认为5，配合队列的 x-max-priority 属性使用
                .messageId(UUID.randomUUID().toString())
                .build();

        String msg = "Hello world, Rabbit MQ";
        // 声明队列（默认交换机AMQP default，Direct）
        // String queue, boolean durable, boolean exclusive, boolean autoDelete, Map<String, Object> arguments
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);

        // 发送消息
        // String exchange, String routingKey, BasicProperties props, byte[] body
        channel.basicPublish("", "qq", properties, msg.getBytes());
~~~

![](https://tva1.sinaimg.cn/large/006y8mN6ly1g8a6t5r4acj30ri08674s.jpg)

## rabbitmq进阶知识

### TTL

通过队列进行设置，通过对每条消息进行设置。

### 死信队列

队列在创建的时候，可以指定死信交换机。

什么情况下，消息会变成死信？

- 消息过期
- 队列长度超过最大限度
- 消息拒绝或者nack并且未设置重回队列

### 延迟队列

场景比如：未付款的订单，15分钟之后关闭。rabbitmq本身是不支持延迟队列，方案如下：

1. 存数据库，定时任务扫描。

2. 利用rabbitmq的死信队列

   利用队列方式定义TTL，梯度很多的情况，比如1分钟、2分钟、15分钟，需要定义非常多队列。

   给每个消息定义TTL，发到同一队列中，如果TTL不一样，会发送阻塞，比如第一条消息TTL30分钟，先到达队列，第二条消息TTL15分钟，后到达队列，第二条消息先到期，无法出列，无法投递到死信交换机中，因为它被第一条消息阻塞了。

3. 利用rabbitmq插件实现

### 服务端流控

rabbitmq生成速度远远大于消费者消费的速度，会产生大量的消息堆积，占用系统资源，导致机器性能降低。我们要控制服务端接收消息的数量。怎么做呢？

队列有两个控制长度的属性：存储最大消息数，存储最大消息容量。超过限制会丢弃队头的消息。

内存控制：启动时，会去检测rabbitmq占用40百分之以上，MQ会主动抛出一个内存警告并阻塞所有连接。内存阈值可以通过rabbitmq.conf进行调整。

磁盘控制：当磁盘空间低于指定的值时，控制消息的发布。

### 消费者限流

默认情况下，队列中有多少数据，会有多少数据快速的发送到消费者，消费者本地进行缓存，可能会出现OOM或者占用CPU影响其他业务或者消费者宕机，这些消息可能会进行丢失。

消费者的处理能力有限，我们希望没有消费完，不要进行推送给我。

QOS

~~~java
 // 非自动确认消息的前提下，如果一定数目的消息（通过基于consume或者channel设置Qos的值）未被确认前，不进行消费新的消息。
        channel.basicQos(10);
        channel.basicConsume(QUEUE_NAME, false, consumer);
~~~

## spring amqp

MessageListenerContainerFactory--MessageListenerContainer--MessageListener

container理解为listener的容器，一个container只要一个listener，但是可以生成多个线程使用相同的messagelistener同时消费消息。

转换器messageconvertor：消息传输会序列化成转成byte[]，消费者再反序列化。默认使用SimpleMessageConverter进行序列化，它是采用了JDK序列化方式（体积大，效率差），当然也是可以自己设置的。

~~~java
@Bean
public RabbitTemplate rabbitTemplate(final ConnectionFactory connectionFactory) { 
  final RabbitTemplate rabbitTemplate = new RabbitTemplate(connectionFactory); 
  rabbitTemplate.setMessageConverter(new Jackson2JsonMessageConverter()); // 先转化为json，再转化成byte[]数组
  return rabbitTemplate;
}
~~~

## rabbitmq可靠性投递

### 消息发送到rabbitmq服务器

生产者不确认消息有没有发送到broker。服务确认机制：第一种是事务模式；第二种是确认模式。

事务模式：channel.txSelect() channel.txCommit。消息没有发送完毕，不能发送下一条消息，阻塞的，榨干rabbitmq的性能，不建议大家在生产环境使用。

~~~java
      try {
            channel.txSelect();
            // 发送消息
            // String exchange, String routingKey, BasicProperties props, byte[] body
            channel.basicPublish("", QUEUE_NAME, null, (msg).getBytes());
            channel.txCommit();
            System.out.println("消息发送成功");
        } catch (Exception e) {
            channel.txRollback();
            System.out.println("消息已经回滚");
        }
~~~

confirm模式：普通确认模式、批量确认模式（发送异常，需要对所有消息进行重发）、异步确认模式（rabbitTemplate对channel进行了封装，叫做confirmCallback）。

~~~java
        //异步监听确认和未确认的消息
        channel.addConfirmListener(new ConfirmListener() {
            public void handleNack(long deliveryTag, boolean multiple) throws IOException {
                System.out.println("Broker未确认消息，标识：" + deliveryTag);
            }
            public void handleAck(long deliveryTag, boolean multiple) throws IOException {
                // 如果true表示批量执行了deliveryTag这个值以前（小于deliveryTag的）的所有消息，如果为false的话表示单条确认
                System.out.println(String.format("Broker已确认消息，标识：%d，多个消息：%b", deliveryTag, multiple));
            }
        });
~~~

### 消息从交换机路由到队列

交换机到队列。什么情况下，消息会无法路由到正确的队列？可能因为路由键错误，或者队列不存在。**一种方法是让服务端重发给生产者，一种是让交换机路由到备份交换机。**

补充：队列可以指定死信交换机，交换机可以指定备份交换机。

~~~java
rabbitTemplate.setMandatory(true); // 如果设置为false，消息会丢弃不会返回给消费者
rabbitTemplate.setReturnCallback(new RabbitTemplate.ReturnCallback(){
public void returnedMessage(Message message, int replyCode,
String replyText, String exchange, String routingKey){
    System.out.println("回发的消息:"); 
    System.out.println("replyCode: "+replyCode); 
    System.out.println("replyText: "+replyText); 
    System.out.println("exchange: "+exchange); 
    System.out.println("routingKey: "+routingKey);
} });
~~~

### 消息在队列存储

队列持久化、交换机持久化、消息持久化、集群（高可用消息队列：镜像队列）

### 消息投递到消费者

提供了消费者的消息确认机制message acknowledgement，消费者可以自动或者手动的发送ACK给服务端。没有收到ACK的消息，等消费者断开连接后，rabbitMQ会把这条消息发送给其他消费者。如果没有其他消费者，消费者重启后会重新消费这条消息，重复执行业务逻辑。

~~~java
spring.rabbitmq.listener.direct.acknowledge-mode=manual 
spring.rabbitmq.listener.simple.acknowledge-mode=manual
~~~

NONE：自动ACK

MANUAL：手动ACK

AUTO：如果方法未抛出异常，则发送ack

~~~java
public class SecondConsumer {
@RabbitHandler
public void process(String msgContent,Channel channel, Message message) throws IOException {
System.out.println("Second Queue received msg : " + msgContent );
channel.basicAck(message.getMessageProperties().getDeliveryTag(), false); }
}
~~~

Basic.reject拒绝单条，basic.nack可以进行批量拒绝。requeue如果为true可以重新存入队列中。

### 消费者回调

消费者成功消费或者失败消费，生产者是不知情的，如果能让生产者之情呢？

1. 调用生产者的API，耦合。
2. 发送响应到生产者

### 补偿机制

生产者迟迟没有得到消费者的响应消息。重发呗。消息落库+定时任务来实现。

![image-20191025165616109](https://tva1.sinaimg.cn/large/006y8mN6ly1g8ajt421o8j316q0m6gsk.jpg)

### 消息幂等性

只能在消费端进行控制。对每一条消息生成一个唯一业务ID，做下重复控制。不过有些程序天然都具有幂等性，比方说唯一主键约束，一些update语句，或者加一个version号，实现乐观锁，用Redis实现幂等。

### 最终一致性

如果多次重发，消息都没办法进行处理。只能人肉处理咯。

### 消息顺序性

多个消费者消费同一个队列，肯定会有顺序的问题。消费者业务里面自己进行处理，比方说给定义一个字段parentId，parentId没有处理完，不能处理本消息。

## 集群和高可用

集群用于实现高可用和负载均衡。

高可用：一台挂了，用另外一台进行服务

负载均衡：单台服务器处理能力有限，可以分发给其他服务器

rabbitmq有两种集群模式：普通集群模式和境像队列集群模式

erlang天生具备分布式的特性，做集群不需要引入ZK或者数据库来实现数据同步。

### 节点类型

磁盘节点：将元数据（包括交换机属性、队列属性、绑定关系、vhost）放到磁盘，集群中必须有一个节点是磁盘节点，用于持久化元数据。

内存节点：将元数据放到内存中。持久化消息会存在磁盘。

### 普通集群模式

这是默认的集群方式。不同节点只会同步元数据，不会同步消息。为什么？性能消耗和数据存储（加入一个节点，对扩大消息存储没什么帮助）。这种模式，不会保证队列的高可用，因为队列里的消息不会复制，如果节点失效将导致此队列不可用。

### 境像集群

消息内容会进行同步，副作用是系统性能降低，节点过多的情况下同步的代价比较大。

### 高可用

![image-20191025172218248](https://tva1.sinaimg.cn/large/006y8mN6ly1g8akk7f01oj311w0u07c9.jpg)

## 实践经验总结

到底在消费者创建还是在生产者创建？

交换机和队列，实际上是作为资源，由运维管理员创建。

仍然在代码中定义，重复创建是不报错的。

生产者资源申请单、消费者资源申请单

命名规范

调用封装

重要的消息：信息落库+定时任务

生产环境运维监控：ZABBIX关注磁盘、内存、连接数

日志跟踪，使用插件，或者spring cloud sleuth

减少连接数：批量发送消息，不要一条一条的发，创建连接和释放连接会有一定的开销

把MQ集群瘫痪作为后背，极端情况下，rabbitmq会发生整个集群宕机，A服务发送的消息无法送到B服务，这个时候补偿job开始工作，定期从A服务批量拉取消息提供给B服务（A在数据库中一定有变动的数据，把这些变动的数据转化成message，B一定要做好幂等性），**做好这套后备方案是非常重要的，因为我们无法确保中间件的百分之100可用。**

### rabbitmq监控

rest api的方式进行监控。

1. 监控是否能够接收TCP连接并且能够成功创建channel；
2. 监控消息能否成功路由到queue；
3. 监控配置文件是否被修改
4. 监控集群状态
5. 监控堆积的消息数（重要）,堆积严重的话及时增强下游的处理能力（加机器、加线程），绘制热点图，把消息流向和流速全部表达出来，一眼就能知道哪些消息有压力。

## 与其他MQ的区别

### rocketMQ

**顺序消费**

rocketmq消息发送是默认采用轮训的方式，发送到不同的queue中，如果发送到同一条queue里面，mq就能保证顺序消费，这个时候就需要MessageQueueSelector这个类。[参考](https://www.cnblogs.com/happyflyingpig/p/8245297.html)

~~~java
// 生产端
SendResult sendResult = producer.send(msgToBroker,
						new MessageQueueSelector() {
							@Override
							public MessageQueue select(List<MessageQueue> mgs,
									Message msg, Object arg) {
								Integer id = (Integer) arg;
								int index = id % mgs.size();
								return mgs.get(index);
							}
						}, 1);// 订单创建、订单支付、订单完成。1可以表示为订单的id，相同订单id的消息可以发送到同一个队列中。
				System.out.println(sendResult);
// 消费端
consumer.registerMessageListener(new MessageListenerOrderly());// 保证M1消费完再消费M2，会出现以下问题
// 遇到消息失败的消息，无法跳过，当前队列消费暂停
~~~

**消息重试**

通过查看源码，消息消费的状态，有两种，一个是成功，一个是失败&稍后重试。[参考](https://blog.csdn.net/zhanglianhai555/article/details/77162208?ref=myrecommend)

~~~java
public enum ConsumeConcurrentlyStatus {
    /**
     * Success consumption
     */
    CONSUME_SUCCESS,
    /**
     * Failure consumption,later try to consume
     */
    RECONSUME_LATER;// 策略：如果消费失败，那么1秒后再次消费，如果在失败，5秒后再次消费。。。知道2H后如果消费还失败，那么该条消息就会终止发送给消费者了
}
~~~

按照上文的策略，其实我们并不需要有这么多的失败重试，我们只需要三次而已，还没成功，我们希望把这条消息存储起来并采用另一种方式处理，而且希望rocketmq不要再重试呢，因为重试解决不了问题了！

~~~java
/**
 * Consumer，订阅消息
 */
public class Consumer {

	public static void main(String[] args) throws InterruptedException, MQClientException {
		DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("group_name");
		consumer.setNamesrvAddr("192.168.2.222:9876;192.168.2.223:9876");
		consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);
		consumer.subscribe("TopicTest", "*");

		consumer.registerMessageListener(new MessageListenerConcurrently() {
			@Override
			public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
				try {
					MessageExt msg = msgs.get(0);
					String msgbody = new String(msg.getBody(), "utf-8");
					System.out.println(msgbody + " Receive New Messages: " + msgs);
					if (msgbody.equals("HelloWorld - RocketMQ4")) {
						System.out.println("======错误=======");
						int a = 1 / 0;
					}
				} catch (Exception e) {
					e.printStackTrace();
					if (msgs.get(0).getReconsumeTimes() == 3) {
						// 该条消息可以存储到DB或者LOG日志中，或其他处理方式
						return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;// 成功
					} else {
						return ConsumeConcurrentlyStatus.RECONSUME_LATER;// 重试
					}
				}
				return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
			}
		});

		consumer.start();
		System.out.println("Consumer Started.");
	}
}
~~~

还有一种情况是：超时，当消费者1取走消息后，阻塞住了，没有给MQ返回成功或稍后重试。当消费者1挂掉之后，这些消息还是会被消费者2取走。

**事务消息**

mq第一阶段发送prepared消息，会拿到消息的地址，第二阶段执行本地事务，第三阶段通过第一个阶段拿到的地址去访问消息，并修改消息的状态已确认。如果确认消息发送失败了怎么办？mq会定期扫描消息集群中的事务消息，如果发现了prepared消息，它会向消息发送端确认，发送端的事务有没有执行成功，rocketmq会根据发送端设置的策略来决定是回滚还是继续发送确认消息，这样就保证了消息发送与本地事务同时成功或同时失败。

![](https://tva1.sinaimg.cn/large/006y8mN6gy1g9cm58z2oqj30hc096wew.jpg)