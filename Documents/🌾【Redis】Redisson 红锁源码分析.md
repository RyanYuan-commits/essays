# 获取 Redisson 锁
通过下面的方式来获取 Redisson 实现的分布式锁：
```java
RLock redissonLock = redisson.getLock("RedisLock");
```
其具体的实现方式为：
>[!info] org.redisson.Redisson.getLock(String name)
```java
@Override  
public RLock getLock(String name) {  
    return new RedissonLock(commandExecutor, name);  
}
```

>[!info] org.redisson.RedissonLock：构造方法
```java
public RedissonLock(CommandAsyncExecutor commandExecutor, String name) {  
    super(commandExecutor, name);  
    this.commandExecutor = commandExecutor;  
    this.internalLockLeaseTime = getServiceManager().getCfg().getLockWatchdogTimeout();  
    this.pubSub = commandExecutor.getConnectionManager().getSubscribeService().getLockPubSub();  
}
```
上面的构造方法中，分别为这三个成员变量进行了赋值：
- `commandExecutor`：用于执行异步命令。这个变量通常用于与 Redis 服务器进行通信，发送和接收命令。
- `internalLockLeaseTime`：表示锁的租约时间（即锁的有效时间）。这个时间通常用于防止死锁，当锁持有时间超过这个值时，锁会被自动释放。
- `pubSub`：用于处理锁的发布和订阅消息。在分布式锁的实现中，通常会使用发布/订阅模式来通知其他节点锁的状态变化。
其中锁的租约时间默认为 `private long lockWatchdogTimeout = 30 * 1000;`，也即 30s；通过 Redisson 构造出来的锁，其默认的存活时间为 30s。
# 上锁操作
调用锁的 `lock()` 方法即可实现上锁的操作：
```java
redissonLock.lock();
```

>[!info] org.redisson.RedissonLock.lock() 上锁方法
```java
@Override  
public void lock() {  
    try {  
        lock(-1, null, false);  
    } catch (InterruptedException e) {  
        throw new IllegalStateException();  
    }  
}
```

```java
    private void lock(long leaseTime, TimeUnit unit, boolean interruptibly) throws InterruptedException {  
        long threadId = Thread.currentThread().getId();  
        Long ttl = tryAcquire(-1, leaseTime, unit, threadId);  
        
        // 成功获取到了锁
        if (ttl == null) {  
            return;  
        }  
  
        CompletableFuture<RedissonLockEntry> future = subscribe(threadId);  
        pubSub.timeout(future);  
        RedissonLockEntry entry;  
        
        if (interruptibly) {  
            entry = commandExecutor.getInterrupted(future);  
        } else {  
            entry = commandExecutor.get(future);  
        }  
  
        try {  
            //  循环等待锁
        } finally {  
            unsubscribe(entry, threadId);  
        }  
    }
```

>[!info] org.redisson.RedissonLock.tryAcquireAsync() 异步获取锁
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

>[!info] org.redisson.RedissonLock.tryLockInnerAsync() 具体获取锁的方法
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
如果 `redis.call('exists', KEYS[1]) == 0`：（当前分布式锁没有被任何线程持有） 或者 `redis.call('hexists', KEYS[1], ARGV[2]) == 1` 当前锁被该线程持有：
	`redis.call('hincrby', KEYS[1], ARGV[2], 1)`：将锁的重入次数加一（如果为空会创建，然后置为 1）。
	`redis.call('pexpire', KEYS[1], ARGV[1]);`：设置当前锁的过期时间。
	`return nil;`：返回空
如果当前锁被其他线程持有的话
	`return redis.call('pttl', KEYS[1]);`：返回当前的锁的存活时间，单位为毫秒（pttl）

>[!info] org.redisson.RedissonLock.scheduleExpirationRenewal()：watch dog 机制
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
如果没有配置过期时间，会自动触发续约的操作，在源码中初始化了一个定时器，dely的时间是 internalLockLeaseTime / 3。
自动续期的 Lua 脚本的源码是这样的：
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
# 解锁方法
```java
protected RFuture<Boolean> unlockInnerAsync(long threadId, String requestId, int timeout) {  
    return evalWriteSyncedAsync(getRawName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,  
                          "local val = redis.call('get', KEYS[3]); " +  
                                "if val ~= false then " +  
                                    "return tonumber(val);" +  
                                "end; " +  
                                "if (redis.call('hexists', KEYS[1], ARGV[3]) == 0) then " +  
                                    "return nil;" +  
                                "end; " +  
                                "local counter = redis.call('hincrby', KEYS[1], ARGV[3], -1); " +  
                                "if (counter > 0) then " +  
                                    "redis.call('pexpire', KEYS[1], ARGV[2]); " +  
                                    "redis.call('set', KEYS[3], 0, 'px', ARGV[5]); " +  
                                    "return 0; " +  
                                "else " +  
                                    "redis.call('del', KEYS[1]); " +  
                                    "redis.call(ARGV[4], KEYS[2], ARGV[1]); " +  
                                    "redis.call('set', KEYS[3], 1, 'px', ARGV[5]); " +  
                                    "return 1; " +  
                                "end; ",  
                            Arrays.asList(getRawName(), getChannelName(), getUnlockLatchName(requestId)),  
                            LockPubSub.UNLOCK_MESSAGE, internalLockLeaseTime,  
                            getLockName(threadId), getSubscribeService().getPublishCommand(), timeout);  
}
```
