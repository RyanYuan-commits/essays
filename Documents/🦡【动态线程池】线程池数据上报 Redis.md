# 项目结构
- 项目地址：[https://github.com/KkQ36/dynamic-thread-pool/tree/20241022-pook-redis-register](https://github.com/KkQ36/dynamic-thread-pool/tree/20241022-pook-redis-register)
- 分支：`20241022-pook-redis-register`
>[!info] dynamic-thread-pool-starter
>添加 Redis 作为注册中心，添加上报功能
```
├── dynamic-thread-pool-starter
│   ├── pom.xml
│   └── src
│       ├── main
│       │   ├── java
│       │   │   └── com
│       │   │       └── ryan
│       │   │           └── sdk
│       │   │               ├── config
│       │   │               │   ├── DynamicThreadPoolAutoConfig.java
│       │   │               │   └── DynamicThreadPoolAutoProperties.java
│       │   │               ├── domain
│       │   │               │   └── pool
│       │   │               │       ├── commons
│       │   │               │       │   └── RegistryEnums.java
│       │   │               │       ├── model
│       │   │               │       │   └── entity
│       │   │               │       │       └── ThreadPoolConfigEntity.java
│       │   │               │       └── service
│       │   │               │           ├── DynamicThreadPoolService.java
│       │   │               │           └── IDynamicThreadPoolService.java
│       │   │               ├── registry
│       │   │               │   ├── IRegistry.java
│       │   │               │   └── redis
│       │   │               │       └── RedisRegistry.java
│       │   │               └── trigger
│       │   │                   └── job
│       │   │                       ├── IJob.java
│       │   │                       └── SpringScheduledJobHandler.java
│       │   └── resources
│       │       └── META-INF
│       │           └── spring.factories
│       └── test
│           └── java
```

>[!info] dynamic-thread-pool-test
>暂未做改变
```
├── dynamic-thread-pool-test
│   ├── pom.xml
│   └── src
│       ├── main
│       │   ├── java
│       │   │   └── com
│       │   │       └── ryan
│       │   │           ├── Main80.java
│       │   │           └── config
│       │   │               ├── ThreadPoolConfig.java
│       │   │               └── ThreadPoolConfigProperties.java
│       │   └── resources
│       │       ├── application-dev.yml
│       │       └── application.yml
│       └── test
│           └── java
│               └── com
│                   └── ryan
│                       └── TestApi.java
└── pom.xml
```
# 要点记录
## 整合 Redisson
>[!info] pom.xml
```xml
<dependency>  
    <groupId>org.redisson</groupId>  
    <artifactId>redisson-spring-boot-starter</artifactId>  
    <version>${redisson.version}</version>  
</dependency>
```

>[!info] com.ryan.sdk.config.DynamicThreadPoolAutoConfig.redissonClient(DynamicThreadPoolAutoProperties properties)
>构造 Redisson 客户端
```java
@Bean("dynamicThreadRedissonClient")  
public RedissonClient redissonClient(DynamicThreadPoolAutoProperties properties) {  
    Config config = new Config();  
    config.useSingleServer()  
            .setAddress("redis://" + properties.getHost() + ":" + properties.getPort())  
            .setPassword(properties.getPassword())  
            .setConnectionPoolSize(properties.getPoolSize())  
            .setConnectionMinimumIdleSize(properties.getMinIdleSize())  
            .setIdleConnectionTimeout(properties.getIdleTimeout())  
            .setConnectTimeout(properties.getConnectTimeout())  
            .setRetryAttempts(properties.getRetryAttempts())  
            .setRetryInterval(properties.getRetryInterval())  
            .setPingConnectionInterval(properties.getPingInterval())  
            .setKeepAlive(properties.isKeepAlive());  
  
    RedissonClient redissonClient = Redisson.create(config);  
    log.info("动态线程池，注册器（redis）链接初始化完成。{} {} {}", properties.getHost(), properties.getPoolSize(), !redissonClient.isShutdown());  
    return redissonClient;  
}
```

>[!info] com.ryan.sdk.registry.redis.RedisRegistry
>使用 Redis 客户端执行操作
```java
public class RedisRegistry implements IRegistry {  
	@Resource
	private RedissonClient redissonClient;  
	
	// 执行 Redis 操作
}
```

## Spring 定时器实现
>[!info] com.ryan.sdk.trigger.job.SpringScheduledJobHandler
> 在 Bean 方法上加上 `@Scheduled` 注解：
```java
@Slf4j  
public class SpringScheduledJobHandler implements IJob {  
    @Override  
    @Scheduled(cron = "0/20 * * * * ?")  
    public void execReportThreadPoolList() {  
        // 。。。。。。
    }  
}
```

>[!info] com.ryan.sdk.config.DynamicThreadPoolAutoConfig
>在配置类上加上 @EnableScheduling，当在一个配置类上添加 @EnableScheduling 注解时，Spring 会扫描并管理带有 @Scheduled 注解的方法，使其能够按照指定的时间间隔或 cron 表达式执行。
```java
@Slf4j  
@Configuration  
@EnableScheduling  
@EnableConfigurationProperties(DynamicThreadPoolAutoProperties.class)  
public class DynamicThreadPoolAutoConfig
```

# 核心代码
>[!info] com.ryan.sdk.domain.pool.service.DynamicThreadPoolService
> 实际执行任务的类，这里只实现了 queryThreadPoolList。
```java
public class DynamicThreadPoolService implements IDynamicThreadPoolService {  
  
    private final String applicationName;  
    private final Map<String, ThreadPoolExecutor> threadPoolExecutorMap;  
  
    public DynamicThreadPoolService(String applicationName, Map<String, ThreadPoolExecutor> threadPoolExecutorMap) {  
        this.applicationName = applicationName;  
        this.threadPoolExecutorMap = threadPoolExecutorMap;  
    }  
    @Override  
    public List<ThreadPoolConfigEntity> queryThreadPoolList() {  
        Set<String> strings = threadPoolExecutorMap.keySet();  
        List<ThreadPoolConfigEntity> entities = new ArrayList<>(strings.size());  
        for (String string : strings) {  
            ThreadPoolExecutor threadPoolExecutor = threadPoolExecutorMap.get(string);  
            ThreadPoolConfigEntity entity = ThreadPoolConfigEntity.builder()  
                    .appName(applicationName)  
                    .poolSize(threadPoolExecutor.getPoolSize())  
                    .corePoolSize(threadPoolExecutor.getCorePoolSize())  
                    .activeCount(threadPoolExecutor.getActiveCount())  
                    .maximumPoolSize(threadPoolExecutor.getMaximumPoolSize())  
                    .queueSize(threadPoolExecutor.getQueue().size())  
                    .remainingCapacity(threadPoolExecutor.getQueue().remainingCapacity())  
                    .queueType(threadPoolExecutor.getQueue().getClass().getName())  
                    .build();  
            entities.add(entity);  
        }  
        return entities;  
    }  
  
    @Override  
    public ThreadPoolConfigEntity queryThreadPoolConfigByName(String threadPoolName) {  
        return null;  
    }  
  
    @Override  
    public void updateThreadPoolConfig(ThreadPoolConfigEntity threadPoolConfigEntity) {  
  
    }  
  
}
```

>[!info] com.ryan.sdk.trigger.job.SpringScheduledJobHandler.execReportThreadPoolList()
>定时任务方法
```java
    @Override  
    @Scheduled(cron = "0/20 * * * * ?")  
    public void execReportThreadPoolList() {  
        List<ThreadPoolConfigEntity> threadPoolConfigEntities = dynamicThreadPoolService.queryThreadPoolList();  
        registry.reportThreadPool(threadPoolConfigEntities);  
        log.info("动态线程池，上报线程池信息：{}", threadPoolConfigEntities);  
  
        for (ThreadPoolConfigEntity threadPoolConfigEntity : threadPoolConfigEntities) {  
            registry.reportThreadPoolConfigParameter(threadPoolConfigEntity);  
            log.info("动态线程池，上报线程池配置：{}", threadPoolConfigEntity);  
        }  
    }
```

>[!info] com.ryan.sdk.registry.redis.RedisRegistry
> 具体的上报方法的实现
```java
public class RedisRegistry implements IRegistry {  
    private final RedissonClient redissonClient;  
    public RedisRegistry(RedissonClient redissonClient) {  
        this.redissonClient = redissonClient;  
    }  
    @Override  
    public void reportThreadPool(List<ThreadPoolConfigEntity> threadPoolEntities) {  
        RList<ThreadPoolConfigEntity> list = redissonClient.getList(RegistryEnums.THREAD_POOL_CONFIG_LIST_KEY.getKey());  
        list.delete();  
        list.addAll(threadPoolEntities);  
    }  
    @Override  
    public void reportThreadPoolConfigParameter(ThreadPoolConfigEntity threadPoolConfigEntity) {  
        String cacheKey = RegistryEnums.THREAD_POOL_CONFIG_PARAMETER_LIST_KEY.getKey() + "_" + threadPoolConfigEntity.getAppName() + "_" + threadPoolConfigEntity.getThreadPoolName();  
        RBucket<ThreadPoolConfigEntity> bucket = redissonClient.getBucket(cacheKey);  
        bucket.set(threadPoolConfigEntity, Duration.ofDays(30));  
    }  
}
```
# 测试
还是直接启动 test 程序，观察控制台输出和 Redis  中的数据
**控制台输出**
```java
2024-10-22T12:31:40.064+08:00  INFO 29240 --- [   scheduling-1] c.r.s.t.job.SpringScheduledJobHandler    : 动态线程池，上报线程池信息：[ThreadPoolConfigEntity(appName=dynamic-thread-pool-test-app, threadPoolName=null, corePoolSize=20, maximumPoolSize=50, activeCount=0, poolSize=0, queueType=java.util.concurrent.LinkedBlockingQueue, queueSize=0, remainingCapacity=5000), ThreadPoolConfigEntity(appName=dynamic-thread-pool-test-app, threadPoolName=null, corePoolSize=20, maximumPoolSize=50, activeCount=0, poolSize=0, queueType=java.util.concurrent.LinkedBlockingQueue, queueSize=0, remainingCapacity=5000)]
2024-10-22T12:31:40.070+08:00  INFO 29240 --- [   scheduling-1] c.r.s.t.job.SpringScheduledJobHandler    : 动态线程池，上报线程池配置：ThreadPoolConfigEntity(appName=dynamic-thread-pool-test-app, threadPoolName=null, corePoolSize=20, maximumPoolSize=50, activeCount=0, poolSize=0, queueType=java.util.concurrent.LinkedBlockingQueue, queueSize=0, remainingCapacity=5000)
2024-10-22T12:31:40.072+08:00  INFO 29240 --- [   scheduling-1] c.r.s.t.job.SpringScheduledJobHandler    : 动态线程池，上报线程池配置：ThreadPoolConfigEntity(appName=dynamic-thread-pool-test-app, threadPoolName=null, corePoolSize=20, maximumPoolSize=50, activeCount=0, poolSize=0, queueType=java.util.concurrent.LinkedBlockingQueue, queueSize=0, remainingCapacity=5000)
```

**Redis 中的数据**
![[Pasted image 20241022123322.png]]
