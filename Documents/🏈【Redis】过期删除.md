### 1 Redis 过期删除相关命令
#### 1.1 配置 Key 的过期时间
设置 key 过期时间的命令一共有 4 个：
- `expire <key> <n>`：设置 key 在 n 秒后过期，比如 expire key 100 表示设置 key 在 100 秒后过期；
- `pexpire <key> <n>`：设置 key 在 n 毫秒后过期，比如 pexpire key2 100000 表示设置 key2 在 100000 毫秒（100 秒）后过期。
- `expireat <key> <n>`：设置 key 在某个时间戳（精确到秒）之后过期，比如 expireat key3 1655654400 表示 key3 在时间戳 1655654400 后过期（精确到秒）；
- `pexpireat <key> <n>`：设置 key 在某个时间戳（精确到毫秒）之后过期，比如 pexpireat key4 1655654400000 表示 key4 在时间戳 1655654400000 后过期（精确到毫秒）
在设置字符串时，也可以同时对 key 设置过期时间，共有 3 种命令：
- `set <key> <value> ex <n>` ：设置键值对的时候，同时指定过期时间（精确到秒）；
- `set <key> <value> px <n>` ：设置键值对的时候，同时指定过期时间（精确到毫秒）；
- `setex <key> <n> <valule>` ：设置键值对的时候，同时指定过期时间（精确到秒）。
#### 1.2 查看 Key 的过期时间
想查看某个 key 剩余的存活时间，可以使用 `TTL <key>` 命令。
```c
# 设置键值对的时候，同时指定过期时间位 60 秒
> setex key1 60 value1
OK

# 查看 key1 过期时间还剩多少
> ttl key1
(integer) 56
> ttl key1
(integer) 52
```
#### 1.3 取消 Key 的过期时间
取消 key 的过期时间，则可以使用 `PERSIST <key>` 命令。
```c
# 取消 key1 的过期时间
> persist key1
(integer) 1

# 使用完 persist 命令之后，
# 查下 key1 的存活时间结果是 -1，表明 key1 永不过期 
> ttl key1 
(integer) -1
```
### 2 过期删除实现原理
#### 2.1 如何判断 key 已经过期？
每当一个 Key 被设置了过期时间的时候，Redis 会把该 key 带上过期时间存储到一个**过期字典**中，也就是说「过期字典」保存了数据库中所有 key 的过期时间，过期字典存储在 redisDb 结构中，如下：
```c
typedef struct redisDb {
    dict *dict;    /* 数据库键空间，存放着所有的键值对 */
    dict *expires; /* 过期字典 */
    ....
} redisDb;
```
过期字典的 key 是指向某个对象的指针，value 是一个 longlong 类型的整数，存储了 key 的过期时间，redisDb 的结构如下图所示：
![[redis 过期字典图例.png|700]]
### 3 常见的过期删除策略
#### 3.1 定时删除
定时删除策略的做法是，在设置 key 的过期时间时，同时创建一个定时事件，当时间到达时，由事件处理器自动执行 key 的删除操作。
- 定时删除策略的**优点**：可以保证过期 key 会被尽快删除，也就是内存可以被尽快地释放。因此，定时删除对内存是最友好的。
- 定时删除策略的**缺点**： 在过期 key 比较多的情况下，删除过期 key 可能会占用相当一部分 CPU 时间，在内存不紧张但 CPU 时间紧张的情况下，将 CPU 时间用于删除和当前任务无关的过期键上，无疑会对服务器的响应时间和吞吐量造成影响。所以，定时删除策略对 CPU 不友好。
#### 3.2 惰性删除
惰性删除策略的做法是，**不主动删除过期键，每次从数据库访问 key 时，都检测 key 是否过期，如果过期则删除该 key。**
- 惰性删除策略的**优点**：因为每次访问时，才会检查 key 是否过期，所以此策略只会使用很少的系统资源，因此，惰性删除策略对 CPU 时间最友好。
- 惰性删除策略的**缺点**：如果一个 key 已经过期，而这个 key 又仍然保留在数据库中，那么只要这个过期 key 一直没有被访问，它所占用的内存就不会释放，造成了一定的内存空间浪费。
#### 3.3 定期删除
定期删除策略的做法是，**每隔一段时间「随机」从数据库中取出一定数量的 key 进行检查，并删除其中的过期key。**
定期删除策略的**优点**：
- 通过限制删除操作执行的时长和频率，来减少删除操作对 CPU 的影响，同时也能删除一部分过期的数据减少了过期键对空间的无效占用。
定期删除策略的**缺点**：
- 内存清理方面没有定时删除效果好，同时没有惰性删除使用的系统资源少。
- 难以确定删除操作执行的时长和频率。如果执行的太频繁，定期删除策略变得和定时删除策略一样，对CPU不友好；如果执行的太少，那又和惰性删除一样了，过期 key 占用的内存不会及时得到释放。
### 4 Redis 的过期删除策略
三种过期删除策略，每一种都有优缺点，仅使用某一个策略都不能满足实际需求，Redis 将惰性删除和定期删除两种策略配合使用，以求在合理使用 CPU 时间和避免内存浪费之间取得平衡。
#### 4.1 Redis 定期删除
定期删除策略的做法是，每隔一段时间「随机」从数据库中取出一定数量的 key 进行检查，并删除其中的过期 key。
Redis 采用了一种 **非阻塞的定期删除策略**，通过定时任务来清除过期的键，而 **不会阻塞主线程或影响主要的读写操作**。
Redis 会每隔 100 毫秒触发一次定期删除任务，也就是频率为每秒十次，这个值可以在 `redis.conf` 中配置，但如果配置过高的话，会使得 redis 在闲置的时候占用更多的内存，如果不是对删除延迟要求极低的环境中，还是使用默认的 10 为好，如果要修改也最好不要超过 100。
```
# By default "hz" is set to 10. Raising the value will use more CPU when
# Redis is idle, but at the same time will make Redis more responsive when
# there are many keys expiring at the same time, and timeouts may be
# handled with more precision.
#
# The range is between 1 and 500, however a value over 100 is usually not
# a good idea. Most users should use the default of 10 and raise this up to
# 100 only in environments where very low latency is required.
hz 10
```
定期删除的实现在 `expire.c` 文件下的 `activeExpireCycle` 函数中，
```c
void activeExpireCycle(int type) {
    unsigned long
    effort = server.active_expire_effort-1, /* Rescale from 0 to 9. */
    config_keys_per_loop = ACTIVE_EXPIRE_CYCLE_KEYS_PER_LOOP + ACTIVE_EXPIRE_CYCLE_KEYS_PER_LOOP/4*effort;
    // 其他逻辑	
}
```
`config_keys_per_loop` 变量就是每次循环时检查的参数个数，它的基础值为 `ACTIVE_EXPIRE_CYCLE_KEYS_PER_LOOP`(常量, 20)，redis 在定期删除的过程中，会对每个数据库（db）执行检查和清除操作，每个数据库每次检查的键数量最多为 `config_keys_per_loop`。
具体的删除逻辑在这个方法的一个 for 循环中：
```c
for (j = 0; j < dbs_per_call && timelimit_exit == 0; j++) {
	expireScanData data; // 存储扫描结果，包括当前数据库信息。
	redisDb *db = server.db+(current_db % server.dbnum); // 当前处理的数据库对象
	data.db = db;
	current_db++;
	do {
		unsigned long num, slots;
		iteration++;

		/* 过期时间的键的数量为 0，跳出循环，查询下一个数据库 */
		if ((num = dictSize(db->expires)) == 0) {
			db->avg_ttl = 0;
			break;
		}
		slots = dictSlots(db->expires);
		data.now = mstime();

		/*如果哈希表的填充率低于 1%，则跳出循环，以避免在无效情况下进行过多的扫描。 */
		if (slots > DICT_HT_INITIAL_SIZE &&
			(num*100/slots < 1)) break;

		data.sampled = 0;
		data.expired = 0;
		data.ttl_sum = 0;
		data.ttl_samples = 0;
		/* num 在上面被配置为过期键的总数量，后面这个值将会在搜索的时候使用，确保它不超过 config_keys_per_loop。 */
		if (num > config_keys_per_loop)
			num = config_keys_per_loop;
		/* 设置最大桶数量为当前处理数量的 20 倍，以提高扫描效率。 */    
		long max_buckets = num*20;
		long checked_buckets = 0;

		while (data.sampled < num && checked_buckets < max_buckets) {
			// 【1】查找 key
			db->expires_cursor = dictScan(db->expires, db->expires_cursor,
										  expireScanCallback, &data);
			checked_buckets++;
		}
		/* 更新全局统计数据，记录已处理和样本总数。 */
		total_expired += data.expired;
		total_sampled += data.sampled;

		/* 更新平均 TTL */
		if (data.ttl_samples) {
			long long avg_ttl = data.ttl_sum / data.ttl_samples;
			if (db->avg_ttl == 0) db->avg_ttl = avg_ttl;
			db->avg_ttl = (db->avg_ttl/50)*49 + (avg_ttl/50);
		}

		/* 每处理 16 次迭代后，检查是否超过时间限制，如果超过，则退出。 */
		if ((iteration & 0xf) == 0) { /* check once every 16 iterations. */
			elapsed = ustime()-start;
			if (elapsed > timelimit) {
				timelimit_exit = 1;
				server.stat_expired_time_cap_reached_count++;
				break;
			}
		}
	/* 【2】检查是否需要继续扫描 */
	} while (data.sampled == 0 ||  (data.expired * 100 / data.sampled) > config_cycle_acceptable_stale);
}
```
我们来关注上面打了标记的重点部分：
【1】这里是一个迭代过期字典（dict）的方法，名为 `dictScan`，它的作用是遍历元素，当遍历到非空 bucket 的时候，就调用传入回调函数 `expireScanCallback`，这个函数的作用是检查 `TTL`，如果键已过期就尝试删除这个键；同时会将 `data.sampled` 的值做一个自增，当这个值达到 `ACTIVE_EXPIRE_CYCLE_KEYS_PER_LOOP1` 也会结束循环，`checked_buckets` 是为了防止空桶过多，对最多遍历的桶数做了一个限制，而 `num` 限制的是最多遍历非空桶的个数。
【2】当前迭代中没有键被扫描到，`data.sampled` 仍然是初始值 `0`，此时清除的效果可能无法达到预期，继续扫描本库；如果本次循环清除的元素 `data.expired` 占遍历的总元素 `data.sampled` 的占比超过了 `config_cycle_acceptable_stale`，说明当前库中的过期键过多，仍需要继续清除；当然，如果清除的时间超过限制时间，当前循环也会退出。该内层循环的设计确保 Redis 能够尽可能多地清理过期的键，尤其是在采样结果不理想的情况下。只有在没有更多的键可以清理或当前的清理效果达到一定标准时，才会跳出这个循环，转向下一个数据库。
#### 4.2 Redis 的惰性删除
Redis 中有一组查找 key 的函数 —— lookupKey() 函数族，会在执行 write 和 read 操作之前，通过这个函数来查找 key。
lookupKey()函数的注释中提到，当调用这个函数的时候，会产生这样的效果：
```
1. A key gets expired if it reached it's TTL.  
2. The key's last access time is updated.  
3. The global keys hits/misses stats are updated (reported in INFO).  
4. If keyspace notifications are enabled, a "keymiss" notification is fired.
```
第一条就提到，如果 key 已经过期，这个 key 会被清除，这个效果是通过调用 `expireIfNeeded` 函数实现的，redis 的惰性删除策略就在这个方法中。
```cpp
robj *lookupKey(redisDb *db, robj *key, int flags) {
    dictEntry *de = dictFind(db->dict,key->ptr);
    robj *val = NULL;
    if (de) {
        val = dictGetVal(de);
	    ...
	    // 针对不同的 flag 标志位来修改 expire_flags
        if (expireIfNeeded(db, key, expire_flags)) {
            /* The key is no longer valid. */
            val = NULL;
        }
    }
    if (val) {
	    // 查找到了 key，做一些善后的工作
    }
    return val;
}
```
`expireIfNeeded()`方法
```c
int expireIfNeeded(redisDb *db, robj *key, int flags) {
    if (server.lazy_expire_disabled) return 0;
    if (!keyIsExpired(db,key)) return 0;
	// 针对主从的处理，从服务器不需要自己处理 key 的删除等操作，而是等待主服务器的通知；
	// 且从服务器复制主服务器的操作的时候，不需要检查 key 是否已经过期。
    if (server.masterhost != NULL) {
        if (server.current_client && (server.current_client->flags & CLIENT_MASTER)) return 0;
        if (!(flags & EXPIRE_FORCE_DELETE_EXPIRED)) return 1;
    }

    /* 针对不需要淘汰 key 的情况做一些处理 */
	    ......

    /* Delete the key */
    deleteExpiredKeyAndPropagate(db,key);
    if (static_key) {
        decrRefCount(key);
    }
    return 1;
}
```
在 `expireIfNeeded` 中，调用了 `deleteExpiredKeyAndPropagate`，这个方法的命名中，除了 `delete` 还提供了 `propagate`（传播），如果 Redis 在主节点（master）检测到某个 key 已过期并将其删除，为了保持与从节点（replicas）的数据一致性，主节点会将该删除操作传播给从节点，通知它们也删除该过期 key。
这也是为什么 `expireIfNeeded` 方法中，从服务器不需要检查 KEY 是否过期。
```cpp
void deleteExpiredKeyAndPropagate(redisDb *db, robj *keyobj) {  
    mstime_t expire_latency;  
    latencyStartMonitor(expire_latency);  
    dbGenericDelete(db,keyobj,server.lazyfree_lazy_expire,DB_FLAG_KEY_EXPIRED);  
    latencyEndMonitor(expire_latency);  
    latencyAddSampleIfNeeded("expire-del",expire_latency);  
    notifyKeyspaceEvent(NOTIFY_EXPIRED,"expired",keyobj,db->id);  
    signalModifiedKey(NULL, db, keyobj);  
    propagateDeletion(db,keyobj,server.lazyfree_lazy_expire);  
    server.stat_expiredkeys++;  
}
```
这个函数的执行流程是这样的：
• **删除过期 key**：调用 `dbGenericDelete` 删除指定数据库中的过期 key。如果 `server.lazyfree_lazy_expire` 为真，Redis 会采用懒删除的方式延迟释放内存。
• **统计延迟**：使用 `latencyStartMonitor` 和 `latencyEndMonitor` 监控和记录删除过期 key 的延迟，并通过 `latencyAddSampleIfNeeded` 添加到延迟监控采样中，帮助分析 Redis 系统的性能。
• **通知事件**：调用 `notifyKeyspaceEvent` 发送 expired 事件，用于通知其他模块或订阅系统的客户端该 key 已过期。
• **传播删除操作**：调用 `propagateDeletion` 将删除操作传播到其他从节点或记录到 AOF 持久化文件中。
• **更新统计信息**：通过 `server.stat_expiredkeys++` 增加过期键的统计计数，用于监控和统计 Redis 系统中发生的过期删除操作的次数。
与上面的定期删除不同，惰性删除会阻塞主进程，所以 Redis 提供了 `lazyfree_lazy_expire` 参数来让我们可以选择将惰性删除设置为非阻塞的方式：
```c
lazyfree-lazy-expire yes // 默认为 no
```
