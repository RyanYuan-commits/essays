Redis 发布订阅（Pub/Sub）是一种消息传递模式，允许发送者（发布者）向特定频道（Channel）发送消息，而无需知道哪些接收者（订阅者）存在。订阅者可以订阅一个或多个频道，实时接收消息。这种机制实现了发送者与接收者之间的解耦，是构建实时应用、消息队列和分布式系统的基础。
### 1 核心概念
- **频道（Channel）**：消息的逻辑分组，发布者和订阅者通过频道名称进行关联。
- **发布者（Publisher）**：向频道发送消息的客户端。
- **订阅者（Subscriber）**：订阅频道并接收消息的客户端。
- **模式匹配（Pattern Matching）**：支持使用通配符（如 `*`）订阅多个频道。
### 2 基本命令
#### 2.1 订阅频道
```bash
SUBSCRIBE channel1 [channel2 ...]  # 订阅一个或多个频道
PSUBSCRIBE pattern1 [pattern2 ...]  # 订阅匹配模式的频道（如 news.*）
```
#### 2.2 发布消息
```bash
PUBLISH channel "message"  # 向指定频道发送消息
```
#### 2.3）取消订阅
```bash
UNSUBSCRIBE [channel1 ...]  # 取消订阅指定频道
PUNSUBSCRIBE [pattern1 ...]  # 取消订阅匹配模式的频道
```
### 3 消息格式
订阅者接收到的消息包含三个部分：
- **消息类型**：`message`（普通订阅）或 `pmessage`（模式匹配订阅）。
- **频道名称**：消息来源的频道。
- **消息内容**：发布者发送的实际内容。
### 4 示例演示
订阅者：
```bash
redis> SUBSCRIBE channel:news
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "channel:news"
3) (integer) 1  # 已成功订阅 1 个频道
```
发布者：
```bash
redis> PUBLISH channel:news "Breaking: Redis 7.0 released!"
(integer) 1  # 1 个订阅者收到消息
```
订阅者收到的消息：
```bash
1) "message"
2) "channel:news"
3) "Breaking: Redis 7.0 released!"
```
### 5 模式匹配示例
#### 5.1 订阅所有以 `channel` 开头的频道
```bash
redis> PSUBSCRIBE channel:*
```
#### 5.2 发布消息到不同频道
```bash
redis> PUBLISH channel:news "News update"
redis> PUBLISH channel:sports "Game result"
```
#### 5.3 订阅者收到消息（模式匹配）
```bash
1) "pmessage"
2) "channel:*"  # 匹配的模式
3) "channel:news"  # 实际频道
4) "News update"

5) "pmessage"
6) "channel:*"
7) "channel:sports"
8) "Game result"
```
### 6 发布订阅的特点
#### 6.1 优点
- **解耦**：发布者和订阅者无需知道彼此的存在。
- **实时性**：消息立即传递给所有订阅者。
- **多播支持**：一条消息可同时发送给多个订阅者。
- **简单易用**：基于 Redis 原生命令，无需额外配置。
#### 6.2 缺点
- **不可靠**：消息不会持久化，若订阅者离线，消息丢失。
- **无回溯**：订阅者只能接收订阅后发布的消息。
- **性能开销**：大量消息可能影响 Redis 性能。
- **不支持消息确认**：无法得知订阅者是否成功处理消息。
### 7 与专业消息队列的对比

| **特性**               | **Redis Pub/Sub**               | **RabbitMQ/Kafka**             |
|------------------------|---------------------------------|--------------------------------|
| 消息持久化             | 不支持                          | 支持                          |
| 可靠性                 | 低（消息可能丢失）              | 高（支持确认机制）            |
| 消息回溯               | 不支持                          | 支持（可重复消费）            |
| 吞吐量                 | 中等                            | 高（Kafka 尤其适合海量数据）  |
| 消息顺序               | 不保证                          | 可保证（分区内）              |
| 复杂路由               | 简单（基于频道）                | 复杂（支持多种交换器和绑定）  |
| 适用场景               | 实时通知、简单广播              | 可靠消息传递、海量数据处理    |
