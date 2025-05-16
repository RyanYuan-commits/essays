string 是最基本的 k-v 结构，k 是唯一标识，v 是具体的值，值并不局限于字符串，也可以是数字，value 最多可以容纳的数据长度是 `512M`。
### 1 内部实现
String 类型的底层的数据结构实现主要是 int 和 SDS（简单动态字符串）。
#### 1.1 对比 C 字符串
SDS 和我们认识的 C 字符串不太一样，之所以没有使用 C 语言的字符串表示，因为相比于 C 的原生字符串：
- **有字段存储长度**：SDS 获取字符串长度的时间复杂度是 O(1)。因为 C 语言的字符串并不记录自身长度，所以获取长度的复杂度为 O(n)；
  而 SDS 结构里用 `len` 属性记录了字符串长度，所以复杂度为 `O(1)`。
- **依赖 len 判断结尾**：SDS 不仅可以保存文本数据，还可以保存二进制数据。因为 `SDS` 使用 `len` 属性的值而不是空字符来判断字符串是否结束，并且 SDS 的所有 API 都会以处理二进制的方式来处理 SDS 存放在 `buf[]` 数组里的数据。所以 SDS 不光能存放文本数据，而且能保存图片、音频、视频、压缩文件这样的二进制数据。
- **安全性好**：Redis 的 SDS API 是安全的，拼接字符串不会造成缓冲区溢出。因为 SDS 在拼接字符串之前会检查 SDS 空间是否满足要求，如果空间不够会自动扩容，所以不会导致缓冲区溢出的问题。
#### 1.2 内部编码
字符串对象的内部编码（encoding）有 3 种 ：int、raw和 embstr。
![[redis 字符串的编码.png|800]]
- 如果一个字符串对象保存的是整数值，并且这个整数值可以用 `long` 类型来表示，那么字符串对象会将整数值保存在字符串对象结构的 `ptr` 属性里面，并将字符串对象的编码设置为 `int`。
- 如果字符串对象保存的是一个字符串，并且这个字符申的长度小于等于 32 字节，那么字符串对象将使用一个简单动态字符串（SDS）来保存这个字符串，并将对象的编码设置为`embstr`， `embstr`编码是专门用于保存短字符串的一种优化编码方式：
- 如果字符串对象保存的是一个字符串，并且这个字符串的长度大于 32 字节，那么字符串对象将使用一个简单动态字符串（SDS）来保存这个字符串，并将对象的编码设置为`raw`：
#### 1.3 embstr 优化
`embstr`和`raw`编码都会使用`SDS`来保存值，但不同之处在于`embstr`会通过一次内存分配函数来分配一块连续的内存空间来保存`redisObject`和`SDS`，而`raw`编码会通过调用两次内存分配函数来分别分配两块空间来保存`redisObject`和`SDS`。
这样做会有很多好处：
- `embstr`编码将创建字符串对象所需的内存分配次数从 `raw` 编码的两次降低为一次；
- 释放 `embstr`编码的字符串对象同样只需要调用一次内存释放函数；
- 因为`embstr`编码的字符串对象的所有数据都保存在一块连续的内存里面可以更好的利用 CPU 缓存提升性能。
但是 embstr 也有缺点的：
- 如果字符串的长度增加需要重新分配内存时，整个 redisObject 和 SDS 都需要重新分配空间，所以 **embstr编码的字符串对象实际上是只读的**，redis没有为embstr编码的字符串对象编写任何相应的修改程序。当我们对embstr编码的字符串对象执行任何修改命令（例如append）时，程序会先将对象的编码从 embstr 转换成 raw，然后再执行修改命令。
### 2 常用指令
#### 2.1 普通字符串的基本操作
```c
# 设置 key-value 类型的值
> SET name lin
OK

# 根据 key 获得对应的 value
> GET name
"lin"

# 判断某个 key 是否存在
> EXISTS name
(integer) 1

# 返回 key 所储存的字符串值的长度
> STRLEN name
(integer) 3

# 删除某个 key 对应的值
> DEL name
(integer) 1
```
#### 2.2 批量设置
```c
# 批量设置 key-value 类型的值
> MSET key1 value1 key2 value2 
OK

# 批量获取多个 key 对应的 value
> MGET key1 key2 
1) "value1"
2) "value2"
```
#### 2.3 计数器
当字符串内容为整数的时候可以使用
```c
# 设置 key-value 类型的值
> SET number 0
OK

# 将 key 中储存的数字值加 1
> INCR number
(integer) 1

# 将key中存储的数字值加 10
> INCRBY number 10
(integer) 11

# 将 key 中储存的数字值减 1
> DECR number
(integer) 10

# 将key中存储的数字值减 10
> DECRBY number 10
(integer) 0
```
#### 2.4 设置过期时间
默认为永不过期
```c
# 设置 key 在 60 秒后过期（该方法是针对已经存在的key设置过期时间）
> EXPIRE name  60 
(integer) 1
# 查看数据还有多久过期
> TTL name 
(integer) 51

# 设置 key-value 类型的值，并设置该key的过期时间为 60 秒
> SET key  value EX 60
OK
> SETEX key 60 value
OK
```
#### 2.5 不存在则创建
```c
# 不存在就插入（not exists）
>SETNX key value
(integer) 1
```
### 3 应用场景
#### 3.1 缓存对象
使用 string 来缓存对象有两种方式：
- 直接缓存整个对象的 JSON，命令例子： `SET user:1 '{"name":"xiaolin", "age":18}'`。
- 采用将 key 进行分离为 user:ID:属性，采用 MSET 存储，用 MGET 获取各属性值，命令例子： `MSET user:1:name xiaolin user:1:age 18 user:2:name xiaomei user:2:age 20`。
#### 3.2 常规计数
因为 Redis 处理命令是单线程，所以执行命令的过程是原子的。因此 String 数据类型适合计数场景，比如计算访问次数、点赞、转发等等。
比如计算文章的阅读量：
```c
# 初始化文章的阅读量
> SET aritcle:readcount:1001 0
OK
# 阅读量 + 1
> INCR aritcle:readcount:1001
(integer) 1
# 阅读量 + 1
> INCR aritcle:readcount:1001
(integer) 2
# 阅读量 + 1
> INCR aritcle:readcount:1001
(integer) 3
# 获取对应文章的阅读量
> GET aritcle:readcount:1001
"3"
```
#### 3.3 分布式锁
SET 命令有个 NX 参数可以实现「key不存在才插入」，可以用它来实现分布式锁：
- 如果 key 不存在，则显示插入成功，可以用来表示加锁成功；
- 如果 key 存在，则会显示插入失败，可以用来表示加锁失败。
=> [[⛸️【Redis】分布式锁]]
#### 3.4 共享 Session 信息
通常我们在开发后台管理系统时，会使用 Session 来保存用户的会话(登录)状态，这些 Session 信息会被保存在服务器端，但这只适用于单系统应用，如果是分布式系统此模式将不再适用。
例如用户一的 Session 信息被存储在服务器一，但第二次访问时用户一被分配到服务器二，这个时候服务器并没有用户一的 Session 信息，就会出现需要重复登录的问题，问题在于分布式系统每次会把请求随机分配到不同的服务器。因此，我们需要借助 Redis 对这些 Session 信息进行统一的存储和管理，这样无论请求发送到那台服务器，服务器都会去同一个 Redis 获取相关的 Session 信息，这样就解决了分布式系统下 Session 存储的问题。