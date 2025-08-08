### 1 简单分布式锁实现
```java
public String sale() {
    String resMessgae = "";
    String key = "luojiaRedisLocak";
    String uuidValue = IdUtil.simpleUUID() + ":" + Thread.currentThread().getId();

    // 不用递归了，高并发容易出错，我们用自旋代替递归方法重试调用；也不用if，用while代替
    while (!stringRedisTemplate.opsForValue().setIfAbsent(key, uuidValue, 30L, TimeUnit.SECONDS)) {
        // 线程休眠20毫秒，进行递归重试
        try {
	        TimeUnit.MILLISECONDS.sleep(20);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
    }

    try {
        // 1 抢锁成功，查询库存信息
        String result = stringRedisTemplate.opsForValue().get("inventory01");
        // 2 判断库存书否足够
        Integer inventoryNum = result == null ? 0 : Integer.parseInt(result);
        // 3 扣减库存，每次减少一个库存
        if (inventoryNum > 0) {
            stringRedisTemplate.opsForValue().set("inventory01", String.valueOf(--inventoryNum));
            resMessgae = "成功卖出一个商品，库存剩余：" + inventoryNum + "\t" + "，服务端口号：" + port;
            log.info(resMessgae);
        } else {
            resMessgae = "商品已售罄。" + "\t" + "，服务端口号：" + port;
            log.info(resMessgae);
        }
    } finally {
        // Lua 脚本的Redis分布式锁调用
        String luaScript =
            "if redis.call('get',KEYS[1]) == ARGV[1] then " +
            "return redis.call('del',KEYS[1]) " +
            "else " +
            "return 0 " +
            "end";
        stringRedisTemplate.execute(new DefaultRedisScript(luaScript, Boolean.class), Arrays.asList(key), uuidValue);

    }
    return resMessgae;
}
```
### 2 Redisson 提供的分布式锁
#### 2.1 获取 Redisson 锁
```java
RLock redissonLock = redisson.getLock("RedisLock");
```
其具体的实现方式为：
```java
@Override  
public RLock getLock(String name) {  
    return new RedissonLock(commandExecutor, name);  
}

public RedissonLock(CommandAsyncExecutor commandExecutor, String name) {  
    super(commandExecutor, name);  
    this.commandExecutor = commandExecutor;  
    this.internalLockLeaseTime = getServiceManager().getCfg().getLockWatchdogTimeout();  
    this.pubSub = commandExecutor.getConnectionManager().getSubscribeService().getLockPubSub();  
}
```
上面的构造方法中，分别为这三个成员变量进行了赋值：
- `commandExecutor`：用于执行异步命令。这个变量通常用于与 Redis 服务器进行通信，发送和接收命令。
- `internalLockLeaseTime`：表示锁的租约时间（即锁的有效时间）。这个时间通常用于防止死锁，当锁持有时间超过这个值时，锁会被自动释放，默认为 `private long lockWatchdogTimeout = 30 * 1000;`，也即 30s，通过 Redisson 构造出来的锁，其默认的存活时间为 30s。
- `pubSub`：用于处理锁的发布和订阅消息，在分布式锁的实现中，通常会使用发布/订阅模式来通知其他节点锁的状态变化。
#### 2.2 上锁方法
##### lock 方法
```java
private void lock(long leaseTime, TimeUnit unit, boolean interruptibly) throws InterruptedException {  
	long threadId = Thread.currentThread().getId();  
	Long ttl = tryAcquire(-1, leaseTime, unit, threadId);  
	
	// 成功获取到了锁
	if (ttl == null) {  
		return;  
	}  

	// ......

	try {  
		// 循环等待锁
	} finally {  
		unsubscribe(entry, threadId);  
	}  
}
```
##### tryAcquireAsync() 异步获取锁
```java
private RFuture<Long> tryAcquireAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId) {  
    RFuture<Long> ttlRemainingFuture;  
    if (leaseTime > 0) {  
        ttlRemainingFuture = tryLockInnerAsync(waitTime, leaseTime, unit, threadId, RedisCommands.EVAL_LONG);  
    } else {  
		// 尝试获取锁
        ttlRemainingFuture = tryLockInnerAsync(waitTime, internalLockLeaseTime,  
                TimeUnit.MILLISECONDS, threadId, RedisCommands.EVAL_LONG);  
    }  
    CompletionStage<Long> s = handleNoSync(threadId, ttlRemainingFuture);  
    ttlRemainingFuture = new CompletableFutureWrapper<>(s);  
  
    CompletionStage<Long> f = ttlRemainingFuture.thenApply(ttlRemaining -> {  
        // 获取到了锁
        if (ttlRemaining == null) {  
            if (leaseTime > 0) {  
                internalLockLeaseTime = unit.toMillis(leaseTime);  
            } else {  
	            // 开启一个缓存续期的定时任务
                scheduleExpirationRenewal(threadId);  
            }  
        }  
        return ttlRemaining;  
    });  
    return new CompletableFutureWrapper<>(f);  
}
```
##### tryLockInnerAsync 具体的 Lua 脚本
```java
<T> RFuture<T> tryLockInnerAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId, RedisStrictCommand<T> command) {  
    return evalWriteSyncedAsync(getRawName(), LongCodec.INSTANCE, command,  
            "if ((redis.call('exists', KEYS[1]) == 0) " +  
					"or (redis.call('hexists', KEYS[1], ARGV[2]) == 1)) then " +  
                    "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +  
                    "redis.call('pexpire', KEYS[1], ARGV[1]); " +  
                    "return nil; " +  
                "end; " +  
                "return redis.call('pttl', KEYS[1]);",  
            Collections.singletonList(getRawName()), unit.toMillis(leaseTime), getLockName(threadId));  
}
```
其中的 `KEYS[1]` 为锁的名称，`ARGV[1]` 为释放时间，`ARGV[2]` 当前线程的一个唯一标识，标识这个锁的拥有者。
如果 `redis.call('exists', KEYS[1]) == 0`，表示当前分布式锁没有被任何线程持有或者 `redis.call('hexists', KEYS[1], ARGV[2]) == 1`，锁当前锁被该线程持有，会执行如下操作：
- `redis.call('hincrby', KEYS[1], ARGV[2], 1)`：将锁的重入次数加一（如果为空会创建，然后置为 1）。
- `redis.call('pexpire', KEYS[1], ARGV[1]);`：设置当前锁的过期时间。
- `return nil;`：返回空
如果当前锁被其他线程持有的话
- `return redis.call('pttl', KEYS[1]);`：返回当前的锁的存活时间，单位为毫秒（pttl）
#### 2.3 WatchDog 机制
```java
private void renewExpiration() {
    ExpirationEntry ee = EXPIRATION_RENEWAL_MAP.get(getEntryName());
    if (ee == null) {
        return;
    }

    Timeout task = commandExecutor.getConnectionManager().newTimeout(new TimerTask() {
        @Override
        public void run(Timeout timeout) throws Exception {
            ExpirationEntry ent = EXPIRATION_RENEWAL_MAP.get(getEntryName());
            if (ent == null) {
                return;
            }
            Long threadId = ent.getFirstThreadId();
            if (threadId == null) {
                return;
            }

            RFuture<Boolean> future = renewExpirationAsync(threadId);
            future.onComplete((res, e) -> {
                if (e != null) {
                    log.error("Can't update lock " + getName() + " expiration", e);
                    return;
                }

                if (res) {
                    // reschedule itself
                    renewExpiration();
                }
            });
        }
    }, internalLockLeaseTime / 3, TimeUnit.MILLISECONDS);

    ee.setTimeout(task);
}
```
如果没有配置过期时间，会自动触发续约的操作，在源码中初始化了一个定时器，dely的时间是 internalLockLeaseTime / 3，自动续期的 Lua 脚本的源码是这样的：
```java
protected RFuture<Boolean> renewExpirationAsync(long threadId) {
    return evalWriteAsync(getName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,
                          "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
                          "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                          "return 1; " +
                          "end; " +
                          "return 0;",
                          Collections.singletonList(getName()),
                          internalLockLeaseTime, getLockName(threadId));
}
```
如果锁属于当前 key 的话，重置其过期时间。
#### 2.4 解锁方法
```java
protected RFuture<Boolean> unlockInnerAsync(long threadId, String requestId, int timeout) {  
    return evalWriteSyncedAsync(getRawName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,  
				// 检查操作状态缓存，如果存在则直接返回之前的结果
				"local val = redis.call('get', KEYS[3]); " +  
				"if val ~= false then " +  
					"return tonumber(val);" +  
				"end; " +  
				// 验证锁的所有权，如果当前锁的持有者不是请求者，则返回 nil
				"if (redis.call('hexists', KEYS[1], ARGV[3]) == 0) then " +  
					"return nil;" +  
				"end; " +  
				// 递减锁计数器（允许多次释放，例如可重入锁）
				"local counter = redis.call('hincrby', KEYS[1], ARGV[3], -1); " +  
				// 如果计数器仍大于 0，表示锁仍被持有
				"if (counter > 0) then " + 
					// 刷新锁的过期时间
					"redis.call('pexpire', KEYS[1], ARGV[2]); " +  
					// 记录操作状态（0表示锁仍被持有）
					"redis.call('set', KEYS[3], 0, 'px', ARGV[5]); " +  
					"return 0; " +  
				"else " +  
					// 计数器为 0，释放锁
					"redis.call('del', KEYS[1]); " + 
					// 发布通知（例如通知其他等待者资源可用） 
					"redis.call(ARGV[4], KEYS[2], ARGV[1]); " + 
					// 记录操作状态（1表示锁已释放） 
					"redis.call('set', KEYS[3], 1, 'px', ARGV[5]); " +  
					"return 1; " +  
				"end; ",  
			Arrays.asList(getRawName(), getChannelName(), getUnlockLatchName(requestId)),  
			LockPubSub.UNLOCK_MESSAGE, internalLockLeaseTime,  
			getLockName(threadId), getSubscribeService().getPublishCommand(), timeout);  
}
```
### 3 RedLock 红锁
#### 3.1 RedLock 是怎么产生的？
有时在特殊情况下，例如在故障期间，**多个客户端可以同时持有锁** 是完全没问题的。如果是这种情况，您可以使用基于复制的解决方案。否则，我们建议实施本文档中描述的解决方案。
Thread1 首先获取锁成功，将键值对写入 redis 的 master 节点，在 redis 将该键值对同步到 slave 节点之前，master发生了故障；
redis 触发故障转移，其中一个 slave 升级为新的 master，此时新上位的 master 并不包含线程1写入的键值对，因此线程⒉尝试获取锁也可以成功拿到锁，此时相当于有两个线程获取到了锁，可能会导致各种预期之外的情况发生。
如果加的是排它独占锁，同一时间只能有一个建redis锁成功并持有锁，严禁出现2个以上的请求线程拿到锁，此时采用单点的方式就不可行了。
#### 3.2 RedLock 算法设计理念
Redis之父提出了 RedLock 算法解决上面这个一锁被多建的问题，方案也是基于(set加锁、Lua脚本解锁）进行改良的，所以 redis 之父 Antirez 只描述了差异的地方。大致方案如下：假设我们有 N 个 Redis 主节点，例如 N=5，这些节点是完全独立的，我们不使用复制或任何其他隐式协调系统，为了取到锁客户端执行以下操作：
1. 获取当前时间，以毫秒为单位;
2. 依次尝试从 5 个实例，使用相同的 key 和随机值（例如 UUID）获取锁。当向 Redis 请求获取锁时，客户端应该设置一个超时时间，这个超时时间应该小于锁的失效时间，例如你的锁自动失效时间为 10 秒，则超时时间应该在 5-50 毫秒之间。这样可以防止客户端在试图与一个宕机的 Redis 节点对话时长时间处于阻塞状态。如果一个实例不可用，客户端应该尽快尝试去另外一个Redis实例请求获取锁;
3. 客户端通过当前时间减去步骤 1 记录的时间来计算获取锁使用的时间。当且仅当从大多数 (N/2+1，这里是3个节点) 的Redis节点都取到锁，并且获取锁使用的时间小于锁失效时间时，锁才算获取成功;
4. 如果取到了锁，其真正有效时间等于初始有效时间减去获取锁所使用的时间（步骤3计算的结果）。
5. 如果由于某些原因未能获得锁（无法在至少 N/2＋1个Redis实例获取锁、或获取锁的时间超过了有效时间)，**客户端应该在所有的 Redis 实例上进行解锁**（即便某些Redis实例根本就没有加锁成功，防止某些节点获取到锁但是客户端没有得到响应而导致接下来的一段时间不能被重新获取锁）。
该方案为了解决数据不一致的问题，直接舍弃了异步复制只使用 `master` 节点，同时由于舍弃了 `slave`，为了保证可用性，引入了N个节点
客户端只有在满足下面的这两个条件时，才能认为是加锁成功。
- 条件1:客户端从超过半数（大于等于 N/2+1）的Redis实例上成功获取到了锁;
- 条件2:客户端获取锁的总耗时没有超过锁的有效时间。
#### 3.3 Redisson 提供的 RedLock 实现
「Redisson 官网」[https://redisson.org](https://redisson.org/)
「GitHub」[https://github.com/redisson/redisson/wiki](https://github.com/redisson/redisson/wiki)
##### 使用 Redisson 提供的 RedLock
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
