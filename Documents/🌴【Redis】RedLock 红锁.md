# RedLock 是怎么产生的？
有时在特殊情况下，例如在故障期间，**多个客户端可以同时持有锁** 是完全没问题的。如果是这种情况，您可以使用基于复制的解决方案。否则，我们建议实施本文档中描述的解决方案。
Thread1 首先获取锁成功，将键值对写入 redis 的 master 节点，在 redis 将该键值对同步到 slave 节点之前，master发生了故障；
redis 触发故障转移，其中一个 slave 升级为新的 master，此时新上位的 master 并不包含线程1写入的键值对，因此线程⒉尝试获取锁也可以成功拿到锁，此时相当于有两个线程获取到了锁，可能会导致各种预期之外的情况发生。
如果加的是排它独占锁，同一时间只能有一个建redis锁成功并持有锁，严禁出现2个以上的请求线程拿到锁，此时采用单点的方式就不可行了。
# RedLock 算法设计理念
Redis之父提出了 RedLock 算法解决上面这个一锁被多建的问题，方案也是基于(set加锁、Lua脚本解锁）进行改良的，所以 redis 之父 Antirez 只描述了差异的地方。
大致方案如下：假设我们有 N 个 Redis 主节点，例如 N= 5 这些节点是完全独立的，我们不使用复制或任何其他隐式协调系统，为了取到锁客户端执行以下操作：
1. 获取当前时间，以毫秒为单位;
2. 依次尝试从5个实例，使用相同的 key 和随机值（例如 UUID）获取锁。当向 Redis 请求获取锁时，客户端应该设置一个超时时间，这个超时时间应该小于锁的失效时间。
   例如你的锁自动失效时间为 10 秒，则超时时间应该在 5-50 毫秒之间。这样可以防止客户端在试图与一个宕机的 Redis 节点对话时长时间处于阻塞状态。如果一个实例不可用，客户端应该尽快尝试去另外一个Redis实例请求获取锁;
3. 客户端通过当前时间减去步骤 1 记录的时间来计算获取锁使用的时间。当且仅当从大多数 (N/2+1，这里是3个节点) 的Redis节点都取到锁，并且获取锁使用的时间小于锁失效时间时，锁才算获取成功;
4. 如果取到了锁，其真正有效时间等于初始有效时间减去获取锁所使用的时间（步骤3计算的结果）。
5. 如果由于某些原因未能获得锁（无法在至少 N/2＋1个Redis实例获取锁、或获取锁的时间超过了有效时间)，**客户端应该在所有的 Redis 实例上进行解锁**（即便某些Redis实例根本就没有加锁成功，防止某些节点获取到锁但是客户端没有得到响应而导致接下来的一段时间不能被重新获取锁）。
该方案为了解决数据不一致的问题，直接舍弃了异步复制只使用 `master` 节点，同时由于舍弃了 `slave`，为了保证可用性，引入了N个节点
客户端只有在满足下面的这两个条件时，才能认为是加锁成功。
- 条件1:客户端从超过半数（大于等于 N/2+1）的Redis实例上成功获取到了锁;
- 条件2:客户端获取锁的总耗时没有超过锁的有效时间。
# Redisson 提供的 RedLock 实现
「Redisson 官网」[https://redisson.org](https://redisson.org/)
「GitHub」[https://github.com/redisson/redisson/wiki](https://github.com/redisson/redisson/wiki)
## 使用 Redisson 提供的 RedLock
```xml
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson</artifactId>
    <version>3.13.4</version>
</dependency>
```

>[!info] 配置 yaml
```yaml
spring:  
  application:  
    name: redis-distributed-lock  
  data:  
    redis:  
      host: 127.0.0.1  
      port: 6379  
server:  
  port: 8083
```

>[!info] InventoryService
```java
@Service  
@Slf4j  
public class InventoryService implements IInventoryService {  
  
    @Resource  
    RedissonClient redisson;  
  
    @Value("${server.port}")  
    String port;  
  
    @PostConstruct  
    void init() {  
        redisson.getBucket("inventory01").set(5000);  
    }  
  
    @Override  
    public String sale() {  
        String resMessgae = "";  
        RLock redissonLock = redisson.getLock("RedisLock");  
        redissonLock.lock();  
        try {  
            // 1 抢锁成功，查询库存信息  
            String result = redisson.getBucket("inventory01").get().toString();  
            // 2 判断库存书否足够  
            int inventoryNum = result == null ? 0 : Integer.parseInt(result);  
            // 3 扣减库存，每次减少一个库存  
            if (inventoryNum > 0) {  
                redisson.getBucket("inventory01").set(--inventoryNum);  
                resMessgae = "成功卖出一个商品，库存剩余：" + inventoryNum + "\t" + "，服务端口号：" + port;  
                log.info(resMessgae);  
            } else {  
                resMessgae = "商品已售罄。" + "\t" + "，服务端口号：" + port;  
                log.info(resMessgae);  
            }  
        } finally {  
            // 改进点，只能删除属于自己的key，不能删除别人的  
            if(redissonLock.isLocked() && redissonLock.isHeldByCurrentThread()) {  
                redissonLock.unlock();  
            }  
        }  
        return resMessgae;  
    }  
  
}
```
