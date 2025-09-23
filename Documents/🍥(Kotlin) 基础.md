---
type: Kotlin
sub-type: " 语法基础"
finished: "true"
---
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

布尔类型(Boolean) 只有两个值: `true` 和 `false`;

支持逻辑运算: `&&` (与), `||` (或), `!` (非);

```kotlin
val isActive = true
val hasPermission = false

if (isActive && !hasPermission) {
    println("活跃但无权限")
}
```

### 1.2 引用数据类型

#### 字符串类型

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

