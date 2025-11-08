---
created: 2025-09-27 22:34:06
updated: 2025-09-27 22:34:06
---
Kotlin 的空安全机制是其最显著的特征之一, 它旨在消除代码中的空指针异常. 在 Kotlin 中, 类型系统明确区分了可以持有 null 的引用和不能持有 null 的引用.

## 1 空安全基础知识

### 1.1 可空类型与非空类型

在 Kotlin 中, 默认情况下类型是非空的, 如果需要允许为 null, 必须显式使用 `?` 标记:

```kotlin
var a: String = "abc" // 非空类型, 不能赋值为 null
a = null // 编译错误

var b: String? = "abc" // 可空类型, 可以赋值为 null
b = null // 正确
```

### 1.2 安全调用操作符 `?.`

安全调用操作符允许我们在对象可能为 null 的情况下安全地访问其属性或方法:

```kotlin
val a: String? = null
println(a?.length) // 输出: null, 不会抛出 NPE

// 链式调用
val user: User? = getUser()
val streetLength = user?.address?.street?.length
```

### 1.3 Elvis 操作符 `?:`

Elvis 操作符用于提供 null 时的默认值:

```kotlin
val name: String? = null
val userName = name ?: "Unknown" // 如果 name 为 null, 则使用 "Unknown"

// 也可以用于 return 和 throw
fun getName(): String {
    val name: String? = null
    return name ?: throw IllegalArgumentException("Name required")
}
```

## 2 空安全高级特性

### 2.1 非空断言 `!!`

非空断言将任何值转换为非空类型, 如果值为 null, 则抛出 NPE:

```kotlin
val a: String? = null
val b: String = a!! // 抛出 NullPointerException
```

### 2.2 安全类型转换 `as?`

安全类型转换在转换失败时返回 null 而不是抛出异常:

```kotlin
val a: Any? = "123"
val b: Int? = a as? Int // 转换失败, b 为 null
```

### 2.3 let 函数与空安全

`let` 函数与安全调用操作符结合使用, 可以优雅地处理可空对象:

```kotlin
val a: String? = null
a?.let { 
    // 这段代码只有在 a 不为 null 时才会执行
    println(it.length) // it 是 a 的非空值
}
```

### 2.4 智能类型转换

Kotlin 编译器会跟踪条件检查, 自动进行类型转换:

```kotlin
fun getStringLength(obj: Any): Int? {
    if (obj is String) {
        // obj 在此分支中自动转换为 String 类型
        return obj.length
    }
    return null
}
```

## 3 空安全最佳实践

### 3.1 设计非空接口

```kotlin
// 不好的设计
fun process(data: Data?) {
    if (data != null) {
        // 处理数据
    }
}

// 好的设计
fun process(data: Data) {
    // 直接处理数据, 不需要 null 检查
}

// 如果确实需要处理 null 情况, 使用重载
fun process(data: Data?) {
    data?.let { process(it) }
}
```

### 3.2 使用数据类避免 null

```kotlin
// 不好的设计
data class User(val name: String?, val age: Int?)

// 好的设计
data class User(val name: String, val age: Int)
// 或者使用默认值
data class User(val name: String = "", val age: Int = 0)
```

### 3.3 避免过度使用非空断言

```kotlin
		// 不好的做法
val user: User? = getUser()
val name = user!!.name // 危险, 可能抛出 NPE

// 好的做法
val user: User? = getUser()
val name = user?.name ?: "Unknown" // 安全
```

### 3.4 使用 lateinit 处理延迟初始化

```kotlin
class MainActivity : AppCompatActivity() {
    private lateinit var adapter: RecyclerView.Adapter<*>

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        adapter = MyAdapter() // 延迟初始化
    }
}
```

## 4 空安全与函数式编程

### 4.1 使用 Optional 模式

```kotlin
fun findUser(id: Int): Optional<User> {
    val user = database.findById(id)
    return Optional.ofNullable(user)
}

// 使用
val user = findUser(123)
val name = user.map { it.name }.orElse("Unknown")
```

### 4.2 使用 Result 类型处理错误

```kotlin
sealed class Result<out T> {
    data class Success<out T>(val value: T) : Result<T>()
    data class Error(val exception: Exception) : Result<Nothing>()
}

fun divide(a: Int, b: Int): Result<Int> {
    return try {
        Result.Success(a / b)
    } catch (e: Exception) {
        Result.Error(e)
    }
}

// 使用
when (val result = divide(10, 0)) {
    is Result.Success -> println("Result: ${result.value}")
    is Result.Error -> println("Error: ${result.exception.message}")
}
```