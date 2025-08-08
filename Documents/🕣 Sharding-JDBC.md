## 1 基本概念
### 1.1 基本介绍
![[sharding sphere.png|900]]
Apache ShardingSphere 是一款 **分布式的数据库生态系统**， 可以将任意数据库转换为分布式数据库，并通过数据分片、弹性伸缩、加密等能力对原有数据库进行增强。

Apache ShardingSphere 设计哲学为 Database Plus，旨在构建异构数据库上层的标准和生态。 它关注如何充分合理地利用数据库的计算和存储能力，而并非实现一个全新的数据库。 它站在数据库的上层视角，关注它们之间的协作多于数据库自身。

是一套开源的分布式数据库中间件解决方案，有两个成熟的产品：Sharding-JDBC 和 Sharding-Proxy：[sharding-sphere 中文官网](https://shardingsphere.apache.org/index_zh.html)
- Sharding-JDBC 定位为轻量级 Java 框架，在 Java 的 JDBC 层提供的额外服务。
- Sharding-Proxy 定位为透明化的数据库代理端，通过实现数据库二进制协议，对异构语言提供支持。
## 2 Sharding-JDBC
[GitHub 项目链接 sharding-demo](https://github.com/KkQ36/sharding-demo)：使用 sharding-jdbc 实现分库分表的案例代码
### 2.1 基本介绍
Sharding-JDBC 定位为轻量级 Java 框架，在 Java 的 JDBC 层提供的额外服务。
它使用客户端直连数据库，以 jar 包的形式提供服务，无需额外的部署和依赖，可以理解为增强版的 JDBC，完全兼容 JDBC 和各种 ORM 框架。
sharding-jdbc 并不是用来做分库分表的，而是在分库分表之后，处理拆分后需要解决的问题。
它主要提供了两个重要的功能：数据分片 和 读写分离。
### 2.2 实现水平拆分
#### 2.2.1 库表结构
```sql
-- 先创建两个库  
create database course_db;  
  
create database course_db_1;  
  
-- 在两个库中分别创建四个结构相同的表  
use course_db_1;  
use course_db_2;  
create table course_1  
(  
    id     bigint(20) primary key,  
    name   varchar(50),  
    status TINYINT  
);  
  
create table course_2  
(  
    id     bigint(20) primary key,  
    name   varchar(50),  
    status TINYINT  
);  
  
create table course_3  
(  
    id     bigint(20) primary key,  
    name   varchar(50),  
    status TINYINT  
);  
  
create table course_4  
(  
    id     bigint(20) primary key,  
    name   varchar(50),  
    status TINYINT  
);
```
#### 2.2.2 application.yaml 配置
```yaml
server:  
  port: 8080  
  
spring:  
  datasource:  
    driver-class-name: org.apache.shardingsphere.driver.ShardingSphereDriver  
    url: jdbc:shardingsphere:classpath:sharding/sharding-jdbc-dev.yaml
```
#### 2.2.3 sharding-jdbc-dev.yaml 配置
```yaml
mode:  
  # 运行模式类型。可选配置：内存模式 Memory、单机模式 Standalone、集群模式 Cluster - 目前为单机模式  
  type: Standalone  
  
dataSources:  
  ds_0:  
    dataSourceClassName: com.zaxxer.hikari.HikariDataSource  
    driverClassName: com.mysql.cj.jdbc.Driver  
    jdbcUrl: jdbc:mysql://127.0.0.1:3306/course_db?useUnicode=true&characterEncoding=utf8&autoReconnect=true&zeroDateTimeBehavior=convertToNull&serverTimezone=UTC&useSSL=true  
    username: root  
    password: my-secret-pw  
    connectionTimeoutMilliseconds: 30000  
    idleTimeoutMilliseconds: 60000  
    maxLifetimeMilliseconds: 1800000  
    maxPoolSize: 15  
    minPoolSize: 5  
  ds_1:  
    dataSourceClassName: com.zaxxer.hikari.HikariDataSource  
    driverClassName: com.mysql.cj.jdbc.Driver  
    jdbcUrl: jdbc:mysql://127.0.0.1:3306/course_db_2?useUnicode=true&characterEncoding=utf8&autoReconnect=true&zeroDateTimeBehavior=convertToNull&serverTimezone=UTC&useSSL=true  
    username: root  
    password: my-secret-pw  
    connectionTimeoutMilliseconds: 30000  
    idleTimeoutMilliseconds: 60000  
    maxLifetimeMilliseconds: 1800000  
    maxPoolSize: 15  
    minPoolSize: 5  
  
rules:  
  - !SHARDING  
    # 库的路由  
    defaultDatabaseStrategy:  
      standard:  
        shardingColumn: id  
        shardingAlgorithmName: database_inline  
    # 表的路由  
    tables:  
      course:  
        actualDataNodes: ds_$->{0..1}.course_$->{1..4}  
        tableStrategy:  
          standard:  
            shardingColumn: id  
            shardingAlgorithmName: course_inline  
    # 路由算法  
    shardingAlgorithms:  
      # 库-路由算法 2是两个库，库的数量。库的数量用哈希模2来计算。  
      database_inline:  
        type: INLINE  
        props:  
          algorithm-expression: ds_$->{Math.abs(id.hashCode()) % 2}  
      # 表-路由算法  
      course_inline:  
        type: INLINE  
        props:  
          algorithm-expression: course_$->{(id.hashCode() ^ (id.hashCode()) >>> 16) & (4 - 1)}  
  
props:  
  # 是否在日志中打印 SQL。  
  # 打印 SQL 可以帮助开发者快速定位系统问题。日志内容包含：逻辑 SQL，真实 SQL 和 SQL 解析结果。  
  # 如果开启配置，日志将使用 Topic ShardingSphere-SQL，日志级别是 INFO。 false  sql-show: true  
  # 是否在日志中打印简单风格的 SQL。false  
  sql-simple: true  
  # 用于设置任务处理线程池的大小。每个 ShardingSphereDataSource 使用一个独立的线程池，同一个 JVM 的不同数据源不共享线程池。  
  executor-size: 20  
  # 查询请求在每个数据库实例中所能使用的最大连接数。1  
  max-connections-size-per-query: 1  
  # 在程序启动和更新时，是否检查分片元数据的结构一致性。  
  check-table-metadata-enabled: false  
  # 在程序启动和更新时，是否检查重复表。false  
  check-duplicate-table-enabled: false
```
### 2.3 实现垂直拆分
垂直拆分，就是配置专库专表，也就是配置不同的表名映射到不同的库，这里只演示专库专标如何配置

初始化表结构
```sql
create database user_db;  
  
use user_db;  
  
create table user_0 (  
                        id bigint primary key,  
                        name varchar(50),  
                        age int  
);
```

其余配置相同，`sharding-jdbc-dev.yaml` 需要做改动：
```yaml


mode:  
  # 运行模式类型。可选配置：内存模式 Memory、单机模式 Standalone、集群模式 Cluster - 目前为单机模式  
  type: Standalone  
  
dataSources:  
  ds_0:  
    dataSourceClassName: com.zaxxer.hikari.HikariDataSource  
    driverClassName: com.mysql.cj.jdbc.Driver  
    jdbcUrl: jdbc:mysql://127.0.0.1:3306/user_db?useUnicode=true&characterEncoding=utf8&autoReconnect=true&zeroDateTimeBehavior=convertToNull&serverTimezone=UTC&useSSL=true  
    username: root  
    password: my-secret-pw  
    connectionTimeoutMilliseconds: 30000  
    idleTimeoutMilliseconds: 60000  
    maxLifetimeMilliseconds: 1800000  
    maxPoolSize: 15  
    minPoolSize: 5  
  
rules:  
  - !SHARDING  
    # 库的路由  
    defaultDatabaseStrategy:  
      standard:  
        shardingColumn: id  
        shardingAlgorithmName: database_inline  
    # 表的路由  
    tables:  
      user:  
        actualDataNodes: ds_0.user_0  
        tableStrategy:  
          standard:  
            shardingColumn: id  
            shardingAlgorithmName: course_inline  
    # 路由算法  
    shardingAlgorithms:  
      # 库-路由算法  
      database_inline:  
        type: INLINE  
        props:  
          algorithm-expression: ds_${id * 0}  
      # 表-路由算法  
      course_inline:  
        type: INLINE  
        props:  
          algorithm-expression: user_${id * 0}
```
上面的配置中，我们将表名为 `course` 的表名都引导到同一个库表中，如果需要其他的库表，只需要添加表名配置即可。
### 2.4 实现读写分离
#### 2.4.1 MySQL 主从配置
其使用的表结构与水平拆分的表结构相同，
> [!NOTE] 注意
>  <font color="#d99694">主从复制属于增量复制，在开启之前，务必保证主库和从库的数据已经完全相同；先将主库锁定，再手动进行主从的第一次同步。</font>

**主库配置**
```ini
[mysqld]
# 开启日志
log-bin = mysql-bin
# 设置服务id，主从需要有不同的 id
server-id = 1
# 设置需要同步的数据库
binlog-do-db = course_db
# 屏蔽系统库同步
binlog-ignore-db = mysql 
binlog-ignore-db = information_schema
binlog-ignore-db = performance_schema
```

**从库配置**
```ini
# 开启日志
log-bin = mysql-bin
# 设置服务id，主从需要有不同的 id
server-id = 2
# 设置需要同步的数据库
replicate_wild_do_table = course_db.%
# 屏蔽系统库同步
replicate_wild_ignore_table = mysql.%
replicate_wild_ignore_table = information_schema.%
replicate_wild_ignore_table = performance_schema.%
```

**确认主节点的文件名以及位点**
```sql
# 确认文件名以及位点
SHOW MASTER status;
```
![[Pasted image 20240919212548.png]]
**主从同步配置**
```sql
# 切换到从服务器
mysql -udb_sync -p123456

# 先停止同步
STOP SLAVE;

# 修改从库指向主库，使用上面得到的位点
CHANGE MASTER TO
master_host = 'localhost',
master_user = 'root',
master_password = 'my-secret-pw',
master_log_file = 'mysql-bin.000003',
master_log_pos = 2433;

# 启动同步
START SLAVE;

# 查看从库相关状态，
SHOW SLAVE status;
```
当使用 `SHOW SLAVE status;` 查看从库状态的时候，字段 `Slave_IO_Running` 和 `Slave_SQL_Running` 都为 Yes 的时候，则说明成功了。
如果进行上述配置之前从库有主库指向的话，需要先执行清空指令。
```sql
STOP REPLICA FOR CHANNEL '';  
RESET SLAVE all;
```
#### 2.4.2 Sharding-JDBC 配置
**运行模式及数据源配置**
```yaml
mode:  
  # 运行模式类型。可选配置：内存模式 Memory、单机模式 Standalone、集群模式 Cluster - 目前为单机模式  
  type: Standalone  
  
dataSources:  
  write_ds_0:  
    dataSourceClassName: com.zaxxer.hikari.HikariDataSource  
    driverClassName: com.mysql.cj.jdbc.Driver  
    jdbcUrl: jdbc:mysql://127.0.0.1:3306/course_db?useUnicode=true&characterEncoding=utf8&autoReconnect=true&zeroDateTimeBehavior=convertToNull&serverTimezone=UTC&useSSL=true  
    username: root  
    password: my-secret-pw  
    connectionTimeoutMilliseconds: 30000  
    idleTimeoutMilliseconds: 60000  
    maxLifetimeMilliseconds: 1800000  
    maxPoolSize: 15  
    minPoolSize: 5  
  read_ds_0:  
    dataSourceClassName: com.zaxxer.hikari.HikariDataSource  
    driverClassName: com.mysql.cj.jdbc.Driver  
    jdbcUrl: jdbc:mysql://127.0.0.1:3307/course_db?useUnicode=true&characterEncoding=utf8&autoReconnect=true&zeroDateTimeBehavior=convertToNull&serverTimezone=UTC&useSSL=true  
    username: root  
    password: my-secret-pw  
    connectionTimeoutMilliseconds: 30000  
    idleTimeoutMilliseconds: 60000  
    maxLifetimeMilliseconds: 1800000  
    maxPoolSize: 15  
    minPoolSize: 5
```

**主从配置**
这个配置需要写在上面，当我们启用主从的时候，其实就相当于对上面的数据源进行包装，让它们重新组合成了一个新的数据源。
这个新的数据源将遵守主写从读的规定。
```yaml
rules:  
- !READWRITE_SPLITTING  
  dataSources:  
    readwrite_ds_0:  
      writeDataSourceName: write_ds_0  
      readDataSourceNames:  
        - read_ds_0  
      transactionalReadQueryStrategy: PRIMARY  
      loadBalancerName: round  
  loadBalancers:  
    round:  
      type: ROUND_ROBIN
```

**路由配置**
```yaml
- !SHARDING  
  tables:  
    course:  
      actualDataNodes: readwrite_ds_0.course_$->{1..4}  
      tableStrategy:  
        standard:  
          shardingColumn: id  
          shardingAlgorithmName: course_inline  
  defaultDatabaseStrategy:  
    standard:  
      shardingColumn: id  
      shardingAlgorithmName: database_inline  
  shardingAlgorithms:  
    database_inline:  
      type: INLINE  
      props:  
        # 通过 id 确定路由到主库的具体表  
        algorithm-expression: readwrite_ds_${id * 0}  
    course_inline:  
      type: INLINE  
      props:  
        # 通过 id 确定路由到哪个分表  
        algorithm-expression: course_$->{(id.hashCode() ^ (id.hashCode()) >>> 16) & (4 - 1)}  # 分表路由
```
### 2.5 测试
#### 2.5.1 对于水平分库分表的测试
horizontal-sub-demo 模块
`com.ryan.demo.controller.CourseController`
```java
@RestController  
@RequestMapping("/course")  
public class CourseController {  
  
    @Resource  
    private CourseService service;  
  
    @PostMapping("/add")  
    public void add(Long id){  
        Course course = new Course();  
        course.setId(id);  
        course.setName("java");  
        course.setStatus(0);  
        service.save(course);  
    }  
  
    @GetMapping("/all")  
    public List<Course> getAll() {  
        return service.list();  
    }  
  
}
```
验证不同的 ID 会被索引到不同的库表。
#### 2.5.2 对于垂直分库分表的测试
vertical-sub-demo 模块
`com.ryan.demo.controller.UserController`
```java
@RestController  
@RequestMapping("/user")  
public class UserController {  
  
    @Resource  
    private UserService userService;  
  
    @PostMapping("/add")  
    public void add(Long id) {  
        User user = new User();  
        user.setId(id);  
        user.setName("ryan");  
        user.setAge(18);  
        userService.save(user);  
    }  
  
}
```
验证其索引到单一的库表。
#### 2.5.3 对于读写分离的测试
模块：read-write-separation
`com.ryan.demo.controller.CourseController`
```java
@RestController  
@RequestMapping("/course")  
public class CourseController {  
  
    @Resource  
    private CourseService service;  
  
    @PostMapping("/write")  
    public void writeCourse(Long id) {  
        Course course = new Course(id, "测试读写分离", 1);  
        service.save(course);  
    }  
  
    @GetMapping("/read")  
    public List<Course> readCourse() {  
        LambdaQueryWrapper<Course> wrapper = new LambdaQueryWrapper<>();  
        wrapper.eq(Course::getId, 123);  
        return service.list(wrapper);  
    }  
  
}
```
验证读操作到达读表（从库），写操作到达写表（主库）。