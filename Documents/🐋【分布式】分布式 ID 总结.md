分布式 ID 是为了解决如何在多个不同的节点中，生成唯一的 ID 的问题
### 1 选型依据
除了全局唯一外，分布式 ID 系统至少要满足以下的要求：
- **高性能**：分布式 ID 的生成速度要快，对本地资源消耗要小。
- **高可用**：生成分布式 ID 的服务要保证可用性无限接近于 100%。
- **方便易用**：拿来即用，使用方便，快速接入！
除此之外，一个比较好的分布式 ID 还需要满足这些要求：
- **安全**：ID 中不包含敏感信息。
- **有序递增**：如果要把 ID 存放在数据库的话，ID 的有序性可以提升数据库写入速度。
- **有具体的业务含义**：生成的 ID 如果能有具体的业务含义，可以让定位问题以及开发更透明化（通过 ID 就能确定是哪个业务）。
- **独立部署**：也就是分布式系统单独有一个发号器服务，专门用来生成分布式 ID。这样就生成 ID 的服务可以和业务相关的服务解耦。不过，这样同样带来了网络调用消耗增加的问题。总的来说，如果需要用到分布式 ID 的场景比较多的话，独立部署的发号器服务还是很有必要的。
### 2 常见解决方案
#### 2.1 数据库方案
##### 主键递增方案
```sql
-- 创建库来存储分布式数据
CREATE TABLE `sequence_id` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
  `stub` char(10) NOT NULL DEFAULT '',
  PRIMARY KEY (`id`),
  UNIQUE KEY `stub` (`stub`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 获取分布式 ID
BEGIN;
REPLACE INTO sequence_id (stub) VALUES ('stub');
SELECT LAST_INSERT_ID();
COMMIT;
```
`stub` 字段无意义，只是为了占位，便于我们插入或者修改数据。并且，给 `stub` 字段创建了唯一索引，保证其唯一性。
在插入的时候，使用 `REPLACE` 关键字，保证每次都可以成功的获取到分布式 ID。
这种方式实现起来比较简单，且 ID 是自增的，但是它支持的并发量并不高，且存在数据库单点问题、ID 没有具体的业务含义，且存在一些安全性问题，比如根据订单 ID 的递增规律就能推算出每天的订单量。
##### 数据库号段模式
```sql
CREATE TABLE `sequence_id_generator` (
  `id` int(10) NOT NULL,
  `current_max_id` bigint(20) NOT NULL COMMENT '当前最大id',
  `step` int(10) NOT NULL COMMENT '号段的长度',
  `version` int(20) NOT NULL COMMENT '版本号',
  `biz_type` int(20) NOT NULL COMMENT '业务类型',
   PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```
数据库中的一行记录代表一个业务线下的 id，在获取的时候，通过 CAS 操作，尝试修改 current_max_id 字段，如果修改成功，就表明自己获取到了 `current_max_id ~ current_max_id+step` 这个范围内的 id。
```sql
UPDATE sequence_id_generator 
SET current_max_id=current_max_id+step , version=version+1 
WHERE version = 0 AND `biz_type` = 101
```
如果插入成功，标识自己获取到了这部分的 ID。
##### NoSQL 方案
以 Redis 举例：
```r
set sequence_id_biz_type 1
=> OK
incr sequence_id_biz_type
=> (integer) 2
get sequence_id_biz_type
=> "2"
```
这种方案性能不错并且生成的 ID 是有序递增的，同样也存在和数据库主键自增方案类似的缺点。
#### 2.2 算法方案
##### UUID 方案
UUID 是 Universally Unique Identifier（通用唯一标识符） 的缩写，它包含 32 个 16 进制数字，需要 128 位空间存储。
UUID 存在多种不同的版本，不同的版本生成 ID 基于的元素不同，举例几个版本：
- Version 1：基于时间戳 + 节点 ID（通常是设备 MAC 地址），可以保证全球唯一性，但是存在 MAC 地址泄露风险。
- Version 4：基于随机数，通常使用伪随机数生成器（PRNG）或加密安全随机数生成器（CSPRNG）来生成，存在碰撞的可能性。
JDK 提供的 UUID 的版本默认为 4：
```java
UUID uuid = UUID.randomUUID();
int version = uuid.version(); // 4
```
虽然，UUID 可以做到全局唯一性，但是，我们一般很少会使用它，比如使用 UUID 作为 MySQL 数据库主键的时候就非常不合适，数据库主键要尽量越短越好，而 UUID 的消耗的存储空间比较大（128 位），且UUID 是无顺序的，InnoDB 引擎下，数据库主键的无序性会严重影响数据库性能。
- **优点**：生成速度通常比较快、简单易用
- **缺点**：存储消耗空间大（32 个字符串，128 位）、 不安全（基于 MAC 地址生成 UUID 的算法会造成 MAC 地址泄露)、无序（非自增）、没有具体业务含义、需要解决重复 ID 问题（当机器时间不对的情况下，可能导致会产生重复 ID）
##### 雪花算法方案
Snowflake 是 Twitter 开源的分布式 ID 生成算法。Snowflake 由 64 bit 的二进制数字组成，这 64bit 的二进制被分成了几部分，每一部分存储的数据都有特定的含义：
![[雪花算法组成.png|center|900]]
- **sign**：符号位（标识正负），始终为 0，代表生成的 ID 为正数。
- **timestamp**：一共 41 位，用来表示时间戳，单位是毫秒，可以支撑 2 ^41 毫秒（约 69 年）
- **datacenter id + worker id**：一般来说，前 5 位表示机房 ID，后 5 位表示机器 ID，实际项目中可以根据实际情况调整，这样就可以区分不同集群/机房的节点。
- **sequence**：一共 12 位，用来表示序列号。 序列号为自增值，代表单台机器每毫秒能够产生的最大 ID 数(2^12 = 4096)，也就是说单台机器每毫秒最多可以生成 4096 个唯一 ID。
在实际项目中，我们一般也会对 Snowflake 算法进行改造，最常见的就是在 Snowflake 算法生成的 ID 中加入业务类型信息。
我们再来看看 Snowflake 算法的优缺点：
- **优点**：生成速度比较快、生成的 ID 有序递增、比较灵活（可以对 Snowflake 算法进行简单的改造比如加入业务 ID）
- **缺点**：ID 生成依赖时间，可能会出现时钟回拨的问题导致重复 ID 的产生；
  依赖机器 ID 对分布式环境不友好，当需要自动启停或增减机器时，固定的机器 ID 可能不够灵活。
如果想要使用 Snowflake 算法的话，一般不需要你自己再造轮子。有很多基于 Snowflake 算法的开源实现比如美团的 Leaf、百度的 UidGenerator，并且这些开源实现对原有的 Snowflake 算法进行了优化，性能更优秀，还解决了 Snowflake 算法的时间回拨问题和依赖机器 ID 的问题。
#### 2.3 开源框架
##### UidGenerator
[UidGenerator](https://github.com/baidu/uid-generator) 是百度开源的一款基于 Snowflake 的唯一 ID 生成器，不过，UidGenerator 对 Snowflake(雪花算法)进行了改进，生成的唯一 ID 组成如下：
![[UidGenerator.png|center|800]]
- **sign**：符号位（标识正负），始终为 0，代表生成的 ID 为正数。
- **delta seconds**：当前时间，相对于时间基点 "2016-05-20" 的增量值，单位是秒，最多可支持约 8.7 年
- **worker id**：机器 id，最多可支持约 420w 次机器启动。内置实现为在启动时由数据库分配，默认分配策略为用后即弃。
- **sequence**：每秒下的并发序列，13 bits 可支持每秒 8192 个并发。
自 18 年后，UidGenerator 就基本没有再维护了......
##### Leaf
[Leaf](https://github.com/Meituan-Dianping/Leaf) 是美团开源的一个分布式 ID 解决方案 ，Leaf 提供了**号段模式**和 **Snowflake** 两种模式来生成分布式 ID，并且，它支持双号段，还解决了雪花 ID 系统时钟回拨问题。
双号段是为了解决在获取 DB 在获取号段的时候线程被阻塞的问题，会在一个号段消费到一定成都的时候就主动启动后台线程去获取下一个号段。
时钟问题的解决需要弱依赖于 Zookeeper，使用 Zookeeper 作为注册中心，通过在特定路径下读取和创建子节点来管理 workId。

![[Leaf 双号段模式.png|center|800]]
根据项目 README 介绍，在 4C8G VM 基础上，通过公司 RPC 方式调用，QPS 压测结果近 5w/s，TP999 1ms。
##### Tinyid
[Tinyid](https://github.com/didi/tinyid) 是滴滴开源的一款基于数据库号段模式的唯一 ID 生成器。
传统的数据库号段方案存在这样的问题：获取新号段的情况下，程序获取唯一 ID 的速度比较慢，需要保证 DB 高可用，这个是比较麻烦且耗费资源的。
相比于基于数据库号段模式的简单架构方案，Tinyid 方案主要做了下面这些优化：
- **双号段缓存**：为了避免在获取新号段的情况下，程序获取唯一 ID 的速度比较慢。 Tinyid 中的号段在用到一定程度的时候，就会去异步加载下一个号段，保证内存中始终有可用号段。
- **增加多 db 支持**：支持多个 DB，并且，每个 DB 都能生成唯一 ID，提高了可用性。
- **增加 tinyid-client**：纯本地操作，无 HTTP 请求消耗，性能和可用性都有很大提升。
