---
type: Kotlin
sub-type: " 语法基础"
finished: "true"
---
[[🍓(Kotlin) String]]

[[🍸(Kotlin) 集合与数组]]

[[🛰️(Kotlin) 函数]]

[[🐨(Kotlin) 控制流]]

[[🎬(Kotlin) 面向对象]]

## 1 数据类型

### 1.1 基本数据类型

Kotlin 没有 Java 中 "原生类型 + 包装类" 区分, 而是通过可空性自动适配; 非空的基本类型编译后会对应到原生类型(如 `Int` -> `int`), 可空类型则对应 Java 的包装类 (如 `Int?` -> `Integer`), 避免手动拆箱装箱的问题.

#### 数值类型
| 类型       | 位数  | 取值范围                       | 说明与使用场景                                   |
| -------- | --- | -------------------------- | ----------------------------------------- |
| `Byte`   | 8   | `-128 ~ 127`               | 最小整数类型, 适合存储小范围数值 (如状态码)                  |
| `Short`  | 16  | `-32768 ~ 32767`           | 短整数, 较少单独使用, 多配合数组 / 缓冲区                  |
| `Int`    | 32  | `-2^31 ~ 2^31-1` (约 ±21 亿) | **默认整数类型** (如`val num = 10`默认为`Int`)      |
| `Long`   | 64  | `-2^63 ~ 2^63-1` (约 ±9e18) | 存储大整数, 需在数值后加`L`标识 (如`100L`)              |
| `Float`  | 32  | 单精度浮点数, 精度约 6-7 位有效数字      | 存储精度要求不高的浮点数 (如坐标), 加`f`标识 (`3.14f`)      |
| `Double` | 64  | 双精度浮点数, 精度约 15-17 位有效数字    | **默认浮点数类型** (如`3.14`默认为`Double`), 适合高精度场景 |

和 Java 不同, Kotlin 的小范围类型不能直接赋值给大范围类型 (如 `val a: Int = 1; val b: Long = a` 报错), 需要显示的进行转换 `val b = a.toLong()`;

#### 字符类型

字符类型(Char) 表示单个 Unicode 字符, 用单引号包裹;

与 Java 不同, Char 不是数值类型, 无法直接和整数比较(如 `'A' == 65` 报错), 需要通过 `toInt()` 转换;

#### 布尔类型

布尔类型(Boolean) 只有两个值 `true` 和 `false`, 常用于条件判断:
```kotlin
val isActive: Boolean = true
if (isActive) {
    println("Active")
}
```

### 1.2 引用数据类型

Kotlin 的引用数据类型包括字符串(String)、数组(Array)、集合(Collection)等. 它们的使用方式与 Java 类似, 但提供了更多的扩展函数和更简洁的语法.

#### 字符串类型

字符串(String) 是不可变的, 可以通过模板表达式插入变量或表达式:

```kotlin
val name = "Kotlin"
val message = "Hello, $name!"
println(message) // 输出: Hello, Kotlin!
```


#### 数组类型

数组(Array) 是固定大小的容器, 可以通过 `arrayOf` 或 `Array` 构造函数创建:

```kotlin
val numbers = arrayOf(1, 2, 3)
val squares = Array(5) { i -> i * i }
println(numbers.joinToString()) // 输出: 1, 2, 3
println(squares.joinToString()) // 输出: 0, 1, 4, 9, 16
```

#### 集合类型

集合(Collection) 包括列表(List)、集合(Set)、映射(Map), 提供了不可变和可变两种版本:
```kotlin
val immutableList = listOf(1, 2, 3)
val mutableList = mutableListOf(1, 2, 3)
mutableList.add(4)
println(mutableList) // 输出: [1, 2, 3, 4]
```

### 1.3 类型系统特点

Kotlin 的类型系统具有以下特点:
1. **可空性**: 通过 `?` 标记可空类型, 避免空指针异常 (NPE);
2. **智能类型转换**: 编译器会根据上下文自动转换类型, 无需显式转换;
3. **类型推断**: 声明变量时可以省略类型, 由编译器自动推断;

示例:
```kotlin
var name: String? = null
if (name != null) {
    println(name.length) // 智能类型转换, 无需显式 `name?.length`
}

val age = 25 // 类型推断为 Int
```

## 2 控制流

### 2.1 条件语句

Kotlin 的条件语句包括 `if` 和 `when`, 后者是更强大的分支选择工具:
```kotlin
val score = 85
val grade = when {
    score >= 90 -> "A"
    score >= 80 -> "B"
    score >= 70 -> "C"
    else -> "D"
}
println(grade) // 输出: B
```

### 2.2 循环语句

Kotlin 提供了 `for`、`while` 和 `do-while` 循环, 并支持区间表达式:
```kotlin
for (i in 1..5) {
    println(i) // 输出: 1, 2, 3, 4, 5
}

val items = listOf("apple", "banana", "cherry")
for (item in items) {
    println(item)
}
```

## 3 函数

### 3.1 函数定义

Kotlin 的函数定义非常简洁, 支持默认参数和命名参数:
```kotlin
fun greet(name: String = "Guest") {
    println("Hello, $name!")
}
greet() // 输出: Hello, Guest!
greet("Alice") // 输出: Hello, Alice!
```

### 3.2 高阶函数与 Lambda 表达式

高阶函数可以接收函数作为参数或返回函数, Lambda 表达式提供了简洁的函数定义方式:
```kotlin
val numbers = listOf(1, 2, 3, 4)
val doubled = numbers.map { it * 2 }
println(doubled) // 输出: [2, 4, 6, 8]
```

## 4 面向对象编程

### 4.1 类与对象

Kotlin 的类定义简洁, 支持主构造函数和次构造函数:
```kotlin
class Person(val name: String, var age: Int) {
    fun introduce() {
        println("Hi, I'm $name and I'm $age years old.")
    }
}
val person = Person("Alice", 25)
person.introduce() // 输出: Hi, I'm Alice and I'm 25 years old.
```

### 4.2 继承与接口

Kotlin 的继承使用 `:` 表示, 接口可以多继承:
```kotlin
open class Animal(val name: String) {
    open fun sound() {
        println("Animal sound")
    }
}

class Dog(name: String) : Animal(name) {
    override fun sound() {
        println("Woof")
    }
}

val dog = Dog("Buddy")
dog.sound() // 输出: Woof
```

## 5 其他特性

### 5.1 扩展函数

扩展函数可以为现有类添加新功能, 而无需修改类定义:
```kotlin
fun String.repeat(times: Int): String {
    return this.repeat(times)
}
println("Kotlin".repeat(3)) // 输出: KotlinKotlinKotlin
```

### 5.2 数据类

数据类用于存储数据, 自动生成 `toString`、`equals` 和 `hashCode` 方法:
```kotlin
data class User(val name: String, val age: Int)
val user = User("Alice", 25)
println(user) // 输出: User(name=Alice, age=25)
```

### 5.3 单例模式

单例模式通过 `object` 关键字实现:
```kotlin
object Singleton {
    val name = "Singleton"
    fun greet() {
        println("Hello from $name")
    }
}
Singleton.greet() // 输出: Hello from Singleton
```

## 6 空安全机制

### 6.1 可空类型与安全调用

Kotlin 的类型系统旨在消除空指针异常(NPE), 通过区分可空类型和非空类型:

```kotlin
var nonNullable: String = "Hello" // 非空类型, 不能赋值为 null
var nullable: String? = null      // 可空类型, 可以赋值为 null
```

安全调用操作符 `?.` 允许在可能为空的对象上安全调用方法:

```kotlin
val length = nullable?.length // 如果 nullable 为 null, 则 length 也为 null
```

### 6.2 Elvis 操作符

Elvis 操作符 `?:` 提供空值的默认替代值:

```kotlin
val name: String? = null
val displayName = name ?: "Unknown" // 如果 name 为 null, 则使用 "Unknown"
```

### 6.3 非空断言

非空断言操作符 `!!` 将任何值转换为非空类型, 如果值为 null 则抛出异常:

```kotlin
val value: String? = "Hello"
val length = value!!.length // 如果 value 为 null, 则抛出 NPE
```

**注意**: 应谨慎使用 `!!`, 它会破坏 Kotlin 的空安全机制.

### 6.4 安全类型转换

安全类型转换操作符 `as?` 在转换失败时返回 null 而不是抛出异常:

```kotlin
val obj: Any = "Hello"
val str: String? = obj as? String // 成功转换为 String
val num: Int? = obj as? Int       // 转换失败, 返回 null
```

## 7 协程

### 7.1 基本概念

协程(Coroutines)是 Kotlin 提供的轻量级线程, 用于简化异步编程:

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    launch { // 启动新协程
        delay(1000L) // 非阻塞延迟 1 秒
        println("World!") // 延迟后打印
    }
    println("Hello") // 主协程继续执行
}
// 输出:
// Hello
// World!
```

### 7.2 协程作用域

协程作用域定义了协程的生命周期:

```kotlin
fun main() = runBlocking { // 创建阻塞协程作用域
    launch { // 在 runBlocking 作用域中启动协程
        delay(200L)
        println("Task from runBlocking")
    }
    
    coroutineScope { // 创建新的协程作用域
        launch {
            delay(500L)
            println("Task from nested scope")
        }
        delay(100L)
        println("Task from coroutine scope")
    }
    
    println("End of runBlocking")
}
// 输出顺序:
// Task from coroutine scope
// Task from runBlocking
// Task from nested scope
// End of runBlocking
```

### 7.3 挂起函数

挂起函数(suspend function)是可以暂停执行并稍后恢复的函数:

```kotlin
suspend fun fetchData(): String {
    delay(1000L) // 挂起协程, 不阻塞线程
    return "Data loaded"
}

fun main() = runBlocking {
    val result = fetchData() // 调用挂起函数
    println(result)
}
```

### 7.4 协程上下文与调度器

协程上下文包含调度器, 控制协程在哪个线程上执行:

```kotlin
fun main() = runBlocking {
    launch(Dispatchers.Default) { // 使用默认调度器(适合CPU密集型任务)
        println("Default: ${Thread.currentThread().name}")
    }
    
    launch(Dispatchers.IO) { // 使用IO调度器(适合IO操作)
        println("IO: ${Thread.currentThread().name}")
    }
    
    launch(Dispatchers.Main.immediate) { // 使用Main调度器(适合UI操作, 仅在UI环境可用)
        println("Main: ${Thread.currentThread().name}")
    }
}
```

## 8 Kotlin 与 Java 互操作

### 8.1 调用 Java 代码

Kotlin 可以直接调用 Java 代码, 但需注意空安全处理:

```kotlin
// Java 类
public class JavaClass {
    public String getData() {
        return "Data from Java";
    }
}

// Kotlin 代码
fun useJavaClass() {
    val javaObj = JavaClass()
    val data = javaObj.data // 直接调用 Java 方法
    println(data)
}
```

### 8.2 平台类型

从 Java 代码返回的类型在 Kotlin 中被视为"平台类型", 编译器不确定其可空性:

```kotlin
// Java 方法
public String getStringOrNull() {
    return Math.random() > 0.5 ? "Value" : null;
}

// Kotlin 代码
fun usePlatformType() {
    val value = getStringOrNull() // 平台类型, 可能为 null
    // 需要开发者自行处理可能的空值
    println(value?.length ?: "Value is null")
}
```

### 8.3 注解互操作

Kotlin 提供特殊注解以优化与 Java 的互操作:

```kotlin
// 指定 Java 调用时的方法名
@JvmName("processItems")
fun List<String>.process() {
    // 处理逻辑
}

// 生成静态方法
@JvmStatic
fun staticMethod() {
    println("Static method")
}

// 生成重载方法
@JvmOverloads
fun greet(name: String = "Guest") {
    println("Hello, $name!")
}
```

### 8.4 SAM 转换

Kotlin 支持 Java 的 SAM (Single Abstract Method) 转换, 简化函数式接口的使用:

```kotlin
// Java 接口
public interface Runnable {
    void run();
}

// Kotlin 代码
fun useRunnable() {
    val runnable = Runnable { println("Running") } // SAM 转换
    Thread(runnable).start()
}
```


字符串(String) 用双引号表示, 支持模板表达式;

```kotlin
val name = "Kotlin"
val message = "Hello, $name! Length: ${name.length}"
```

支持多行字符串 (三重引号):

```kotlin
val multiLine = """
    |Line 1
    |Line 2
    |Line 3
""".trimMargin()
```

#### 数组类型

数组(Array) 用 `arrayOf()` 创建, 类型参数化;

```kotlin
val numbers = arrayOf(1, 2, 3, 4, 5)
val strings = arrayOf("a", "b", "c")

// 基本类型数组 (避免装箱)
val intArray = intArrayOf(1, 2, 3)
val charArray = charArrayOf('a', 'b', 'c')
```

## 2 变量与常量

### 2.1 变量声明

使用 `var` 声明可变变量, `val` 声明不可变变量(推荐);

```kotlin
var count = 10  // 类型推断为 Int
count = 20     // 可以重新赋值

val pi = 3.14  // 类型推断为 Double
// pi = 3.15   // 编译错误: val 不可重新赋值
```

### 2.2 类型注解

可以显式指定类型:

```kotlin
val name: String = "Kotlin"
var age: Int = 25
val price: Double = 99.99
```

### 2.3 延迟初始化

使用 `lateinit` 延迟初始化非空变量:

```kotlin
lateinit var userService: UserService

fun initialize() {
    userService = UserService()
}

fun useService() {
    if (::userService.isInitialized) {
        userService.doSomething()
    }
}
```

## 3 空安全

### 3.1 可空类型

类型后加 `?` 表示可空:

```kotlin
var nullableString: String? = null
var nonNullString: String = "hello" // 不能为 null
```

### 3.2 安全调用操作符

使用 `?.` 安全调用:

```kotlin
val length = nullableString?.length // 返回 Int?
```

### 3.3 Elvis 操作符

`?:` 提供默认值:

```kotlin
val safeLength = nullableString?.length ?: 0
```

### 3.4 非空断言

`!!` 强制转换为非空(慎用):

```kotlin
val forcedLength = nullableString!!.length // 可能抛出 NPE
```

## 4 控制流

### 4.1 条件表达式

`if` 可以作为表达式使用:

```kotlin
val max = if (a > b) a else b

val grade = if (score >= 90) "A" 
           else if (score >= 80) "B"
           else "C"
```

### 4.2 when 表达式

强大的模式匹配:

```kotlin
when (x) {
    1 -> println("One")
    2, 3 -> println("Two or Three")
    in 4..10 -> println("4 to 10")
    is String -> println("String")
    else -> println("Other")
}
```

### 4.3 循环

`for` 循环:

```kotlin
for (i in 1..5) { // 1,2,3,4,5
    println(i)
}

for (i in 1 until 5) { // 1,2,3,4
    println(i)
}

for (i in 5 downTo 1) { // 5,4,3,2,1
    println(i)
}

for (i in 1..10 step 2) { // 1,3,5,7,9
    println(i)
}
```

`while` 和 `do-while` 循环:

```kotlin
var i = 0
while (i < 5) {
    println(i)
    i++
}

do {
    println(i)
    i--
} while (i > 0)
```

## 5 函数

### 5.1 函数声明

```kotlin
fun greet(name: String): String {
    return "Hello, $name!"
}

// 单表达式函数
fun square(x: Int) = x * x
```

### 5.2 默认参数

```kotlin
fun connect(
    host: String, 
    port: Int = 8080,
    timeout: Int = 5000
) {
    // ...
}

connect("localhost") // 使用默认端口和超时
```

### 5.3 命名参数

```kotlin
connect(
    host = "example.com",
    timeout = 10000,
    port = 9090
)
```

### 5.4 扩展函数

为现有类添加新功能:

```kotlin
fun String.addExclamation(): String {
    return this + "!"
}

val excited = "Hello".addExclamation() // "Hello!"
```

## 6 类与对象

### 6.1 类声明

```kotlin
class Person(
    val name: String,   // 只读属性
    var age: Int        // 可变属性
) {
    init {
        println("Person created: $name")
    }
    
    fun greet() {
        println("Hello, I'm $name, $age years old")
    }
}
```

### 6.2 数据类

自动生成 `equals()`, `hashCode()`, `toString()`, `copy()`:

```kotlin
data class User(
    val id: Long,
    val name: String,
    val email: String
)

val user = User(1, "Alice", "alice@example.com")
val copied = user.copy(name = "Alice Smith")
```

### 6.3 伴生对象

替代 Java 的静态方法:

```kotlin
class MyClass {
    companion object {
        const val CONSTANT = "value"
        
        fun create(): MyClass = MyClass()
    }
}

val instance = MyClass.create()
val constant = MyClass.CONSTANT
```

## 7 集合

### 7.1 不可变集合

```kotlin
val list = listOf("a", "b", "c")      // List<String>
val set = setOf(1, 2, 3, 2, 1)        // Set<Int> (1,2,3)
val map = mapOf("key" to "value")     // Map<String, String>
```

### 7.2 可变集合

```kotlin
val mutableList = mutableListOf(1, 2, 3)
mutableList.add(4)

val mutableSet = mutableSetOf(1, 2, 3)
mutableSet.add(4)

val mutableMap = mutableMapOf("a" to 1)
mutableMap["b"] = 2
```

### 7.3 集合操作

丰富的操作函数:

```kotlin
val numbers = listOf(1, 2, 3, 4, 5)

val doubled = numbers.map { it * 2 }          // [2,4,6,8,10]
val even = numbers.filter { it % 2 == 0 }     // [2,4]
val sum = numbers.reduce { acc, i -> acc + i } // 15
val grouped = numbers.groupBy { it % 2 }      // {1=[1,3,5], 0=[2,4]}
```

