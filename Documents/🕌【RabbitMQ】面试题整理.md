### RabbitMQ 架构设计
![[RabbitMQ 架构设计.png|900]]
- ==Broker==：RabbitMQ 的服务节点；
- ==Queue==：队列，是 RabbitMQ 的内部对象，用于存储消息，RabbitMQ 中的消息存储在队列之中。生产者投递消息到队列，消费者从队列中获取消息并且消费。多个消费者可以鼎御堂同一个队列，这时候消息会被平均分摊给多个消费者进行消费，而不是每个消费者都收到所有的消息进行消费。（注意：RabbitMQ 并不支持队列层面的广播消费，如果需要广播消费，可以采用一个交换机通过路由 key 绑定多个队列，有多个消费者来订阅这些队列的方式）；
- ==Exchange==：交换器，生产者将消息发送到 Exchange，由交换机将消息路由到一个或者多个队列中。如果路由不到，可以选择返回给生产者、直接丢弃或者做其他的处理；
- ==Routing Key==：路由 key，生产者发送消息到交换机的时候，一般都会指定一个 Routing Key，用来指定这个消息的路由规则，这个路由 key 需要与交换机类型和绑定键（Binding Key）联合使用才能最终生效，在交换机类型和绑定键固定的情况下，生产者可以在发送消息的时候通过执行 Routing Key 来决定消息流向哪里；
交换机和消息队列实际上是一种多对多的关系；类似关系数据库中的两张表，通过 BindingKey 做关联，在投递消息的时候，可以通过 Exchange 和 Routing Key 找到相应的队列；
### RabbitMQ 的交换机类型
- ==fanout 交换机==：扇出交换机，不需要判定 routing key，直接广播给所有的绑定的队列中；
- ==direct 交换机==：判定 routing key 是完全匹配的模式；
- ==topic 交换机==：判定 routing key 采用模糊匹配的方式；
- ==header 交换机==：绑定队列与交换机的时候指定一个键值对，当交换机在分发消息的时候会先解开消息体中的 header 数据，然后判断里面是否有所设置的键值对，如果发现匹配成功，才将消息分发到队列中；这种交换机在性能上相对较差，实际工作中很少会用到。
### RabbitMQ 的持久化机制
RabbitMQ 是基于内存实现的，当服务器宕机之后，内存中的数据就会消失；
1. 交换机的持久化：创建交换机的时候通过参数去指定；
2. 队列的持久化：创建队列的时候通过参数指定；
3. 消息的持久化：创建消息的时候通过参数指定
```java
channel.queueDeclare("persistent_queue", true, false, false, null); // 队列持久化

channel.basicPublish("exchange_name", "routing_key", MessageProperties.PERSISTENT_TEXT_PLAIN, "Persistent Message".getBytes()); // 消息持久化

channel.exchangeDeclare("persistent_exchange", "direct", true); // 交换机持久化
```
消息的持久化：RabbitMQ 使用一个机制叫 **WAL（Write-Ahead Logging）**，消息会先写入磁盘日志（rabbitmq_message_store），然后确认消息已被持久化。当消费者确认消息处理完毕时，RabbitMQ 可以根据策略删除磁盘中的消息。
### RabbitMQ 事务消息机制
RabbitMQ 的事务消息机制允许生产者在发送消息的过程中确保消息的可靠投递，但由于性能问题，事务机制通常不推荐使用，更常用的是 **Confirm 模式**。
RabbitMQ 支持 AMQP 协议的事务功能，生产者可以通过事务保证消息的可靠性。事务机制主要提供以下功能：
1. **事务开始**：txSelect() 方法开启事务。
2. **事务提交**：txCommit() 方法提交事务。
3. **事务回滚**：txRollback() 方法回滚事务。
```java
// 创建连接和通道
ConnectionFactory factory = new ConnectionFactory();
factory.setHost("localhost");
try (Connection connection = factory.newConnection();
     Channel channel = connection.createChannel()) {
    // 开启事务模式
    channel.txSelect();
    try {
        // 发送消息
        String queueName = "transaction_queue";
        channel.queueDeclare(queueName, true, false, false, null);
        String message = "Transaction Message";
        channel.basicPublish("", queueName, null, message.getBytes());
        System.out.println("Sent: '" + message + "'");

        // 提交事务
        channel.txCommit();
        System.out.println("Transaction committed.");
    } catch (Exception e) {
        // 回滚事务
        channel.txRollback();
        System.out.println("Transaction rolled back.");
    }
}
```
### RabbitMQ 如何保证消息的可靠性传输？
1. 使用事务消息
2. 使用消息确认机制
#### 发送方确认机制
- 将 Channel 设置为 Confirm 模式，每一条发送的消息都会有一个唯一 id；
- 消息投递成功，信道会发送一个 ACK 给生产者，包含了 id，回调 ConfirmCallback 接口；
- 如果发生错误导致消息丢失，发送 NACK 给生产者，回调 ReturnCallBack 接口；
- ACK 和 NACK 只有一个触发，且只会异步的触发一次。
```java
channel.confirmSelect(); // 开启发布确认模式
channel.basicPublish(exchange, routingKey, null, message.getBytes());

if (channel.waitForConfirms()) {
    System.out.println("消息发送成功！");
} else {
    System.out.println("消息发送失败！");
}
```
#### 接受方确认机制
- 声明队列的时候，指定 noakc=false，broker 等待消费者手动返回 ack 才会将消息删除；
- broker 的 ack 没有超时机制，只会判断链接是否断开，如果断开，消息会被重新发送。
```java
channel.basicConsume(queueName, false, (consumerTag, message) -> {
    try {
        System.out.println("接收到消息：" + new String(message.getBody()));
        channel.basicAck(message.getEnvelope().getDeliveryTag(), false); // 确认消息
    } catch (Exception e) {
        channel.basicNack(message.getEnvelope().getDeliveryTag(), false, true); // 拒绝并重新入队
    }
}, consumerTag -> {});
```
### RabbitMQ 的死信队列
**死信队列**是一个用于存储无法被正常消费的消息的特殊队列；当消息在原队列中由于某些原因无法被消费时，会被重新路由到死信队列中进行后续处理。
如果发生了下面的某种情况，消息会变为死信消息：
1. **消息被拒绝**（basicReject 或 basicNack），且 `requeue=false`：消息被消费者拒绝并未重新放回队列。
2. **消息过期**（TTL 到期）：消息的 TTL（存活时间）到了仍未被消费。
3. **队列达到最大长度**：队列的消息数量达到了预设的最大值。
死信队列的实现依赖于 RabbitMQ 的 **DLX（Dead Letter Exchange）**，即死信交换机。配置时需要将原队列绑定到一个死信交换机。
```java
Map<String, Object> args = new HashMap<>();
args.put("x-dead-letter-exchange", "dlx.exchange"); // 指定死信交换机
args.put("x-dead-letter-routing-key", "dlx.routingKey"); // 指定死信路由键
args.put("x-message-ttl", 60000); // 设置消息过期时间

// 声明原队列，绑定到死信交换机
channel.queueDeclare("normal.queue", true, false, false, args);

// 声明死信队列及绑定
channel.exchangeDeclare("dlx.exchange", "direct");
channel.queueDeclare("dead.queue", true, false, false, null);
channel.queueBind("dead.queue", "dlx.exchange", "dlx.routingKey");
```
使用场景：
- 消费失败的消息可以记录到死信队列中，后续进行日志分析或重新处理。
- 通过死信队列监控系统异常情况。
### RabbitMQ 延迟队列
**延迟队列**是指消息在发送到队列后，不立即被消费者消费，而是等待一段时间后才可以被消费。
RabbitMQ 并未直接支持延迟队列，需要通过以下两种方式实现：
通过 TTL + 死信队列：
1. 在消息队列上设置 TTL（消息存活时间）。
2. 消息过期后进入死信队列。
3. 死信队列中的消息路由到实际消费队列。
```java
// 配置延迟队列，绑定到死信交换机
Map<String, Object> args = new HashMap<>();
args.put("x-dead-letter-exchange", "dlx.exchange");
args.put("x-dead-letter-routing-key", "dlx.routingKey");
args.put("x-message-ttl", 10000); // 消息延迟 10 秒

channel.queueDeclare("delay.queue", true, false, false, args);

// 配置死信队列和消费队列
channel.exchangeDeclare("dlx.exchange", "direct");
channel.queueDeclare("real.queue", true, false, false, null);
channel.queueBind("real.queue", "dlx.exchange", "dlx.routingKey");
```

基于官方的插件：
RabbitMQ 提供了官方插件 **rabbitmq-delayed-message-exchange**，可以直接支持延迟队列。
```java
// 声明延迟交换机
Map<String, Object> args = new HashMap<>();
args.put("x-delayed-type", "direct");
channel.exchangeDeclare("delay.exchange", "x-delayed-message", true, false, args);

// 绑定延迟队列
channel.queueDeclare("delay.queue", true, false, false, null);
channel.queueBind("delay.queue", "delay.exchange", "delay.routingKey");

// 发送延迟消息
AMQP.BasicProperties.Builder props = new AMQP.BasicProperties.Builder();
props.headers(Map.of("x-delay", 10000)); // 设置延迟时间 10 秒
channel.basicPublish("delay.exchange", "delay.routingKey", props.build(), "延迟消息".getBytes());
```
### 如何确保消息不被重复消费？
产生消息重复消费可能有这么几种情况：
- 发送方因为操作失误导致同一条消息的多次发送；
- 接受方未返回 ACK，且断开连接，此时消息会被重新发送。
保证消息不被重新消费就是要保证消息的幂等性，可以使用雪花算法，给每一条消费分配一个唯一的 ID，通过关系型数据库的唯一性约束来确保幂等性；在发送方也可以通过 Redis 的 setnx、数据库等方式来确保消息不被重复发送；
### 如何保证消息的顺序消费
在生产者侧，保证消息顺序的发出；
可以为每一条消息指定一个 id，在消费者侧，判断这个 id 前一个消息是否有消费过，如果有消费该消息，如果没有，将消息存储到 redis；
在消费一个消息之后，检测一下 redis 中是否有这条消息的下一条消息，如果有，拿出来消费。