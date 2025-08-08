![[RocketMQ 领域模型.png]]
关键概念：
- `Topic`：主题，可以理解为类别和分类的概念；
- `MessageQueue`：消息队列，存储数据的一个容器，默认每个 `Topic` 下有四个队列来存储信息；
- `Message`：消息，真正携带信息的载体概念；
- `Producer`：生产者，负责发送消息；
- `Consumer`：消费者，负责消费消息；
- `Subscription`：订阅关系，消费者需要制定自己要消费哪个 `Topic` 下的消息。
