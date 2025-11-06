---
created: 2025-09-27 22:34:06
updated: 2025-09-27 22:34:06
---
## 1	类与构造函数

### 1.1	主构造函数

```kotlin
class Person(val id: Long, val name: String, var email: String) {

    // `init` 块是主构造函数的一部分，用于执行更复杂的初始化逻辑
    init {
        require(name.isNotBlank()) { "Name cannot be blank" }
        println("Created Person - ID: $id, Name: $name")
    }
}

// 使用主构造函数创建实例
val person = Person(1L, "Alice", "alice@example.com")
```

Kotlin 引入了主构造函数的概念, 它是类头的一部分, 通常用于声明和初始化属性; 这样声明的属性会自带`get` 和 `set` 方法;

### 1.2	次构造函数

```kotlin
class HttpRequest(val url: String, val method: String, val timeout: Int) {

    // 提供一个便利的次构造函数，使用默认的请求方法 "GET"
    constructor(url: String, timeout: Int) : this(url, "GET", timeout) {
        println("Secondary constructor called")
    }
}

val postRequest = HttpRequest("https://api.example.com/data", "POST", 5000)
val getRequest = HttpRequest("https://api.example.com/data", 3000) // 调用次构造函数
```

当需要提供多种创建对象方法的时候, 可以使用次构造函数; 如果类有主构造函数, 所有次构造函数都必须直接或间接地委托给主构造函数 (`this(...)`).

## 2	属性与字段

```kotlin
class Employee {
    var name: String = "Default Name" // 自动拥有 getter 和 setter
    val employeeId: String = generateId() // 只读属性，只有 getter

    private fun generateId(): String = "EMP-${System.currentTimeMillis()}"

    // 带自定义访问器的属性
    var department: String = "General"
        set(value) { // 自定义 setter
            println("Changing department to $value")
            // `field` 是一个特殊的标识符，指向属性的“幕后字段”
            field = value.trim() 
        }

    // 这是一个“计算属性”，它没有幕后字段，其值在每次访问时动态计算
    val description: String
        get() = "$name (ID: $employeeId) in $department"
}

val emp = Employee()
emp.department = "  Engineering  " // 调用自定义 setter
println(emp.description) // 调用自定义 getter
```

## 3	继承与多态

Kotlin 在继承方面采用了更安全的设计哲学 "默认关闭".

- 类和方法默认是 `final` 的, 如果要允许继承, 需要使用 `open` 标记;
	
- 重写父类的方法或者属性, 必须使用 `override` 关键字明确标注.

```kotlin
// 父类和其方法必须是 open 的，才能被继承和重写
open class Shape(val name: String) {
    open fun getArea(): Double = 0.0
}

open class Polygon(name: String, val sides: Int) : Shape(name) {
    // 重写父类方法
    override fun getArea(): Double {
        println("Area calculation not implemented for a generic polygon.")
        return super.getArea()
    }
}

class Rectangle(name: String, val width: Double, val height: Double) : Polygon(name, 4) {
    // 再次重写
    override fun getArea(): Double = width * height
}
```

## 4	接口与抽象类

```kotlin
// 接口可以有抽象属性和带默认实现的方法
interface Flyable {
    val maxAltitude: Int // 抽象属性，实现类必须提供
    fun fly()
    fun land() { // 带默认实现的方法
        println("Landing smoothly.")
    }
}

abstract class Bird(val name: String) {
    abstract fun sing()
}

class Eagle(name: String) : Bird(name), Flyable {
    override val maxAltitude: Int = 10000 // 实现接口的抽象属性

    override fun sing() {
        println("Screech!")
    }

    override fun fly() {
        println("Eagle soaring high at $maxAltitude feet.")
    }
}
```

## 5	 可见性修饰符

| 修饰符       | 范围     | 描述                      |
| --------- | ------ | ----------------------- |
| public    | 所有地方   | 所有地方可见                  |
| internal  | 模块内部   | 在同一个 Gradle/Maven 模块内可见 |
| protected | 类和子类   | 与 Java 类似               |
| private   | 类/文件内部 | 在生命它的类或文件内部可见           |

## 6	Kotlin 特殊类

### 6.1	数据类

为存储数据而生的类, 自动生成核心样板代码, 适合作为 DTO 类使用;

```kotlin
data class Book(val isbn: String, val title: String, val author: String)

fun main() {
    val book1 = Book("978-3-16-148410-0", "Kotlin in Action", "Dmitry Jemerov")
    val book2 = Book("978-3-16-148410-0", "Kotlin in Action", "Dmitry Jemerov")

    // 1. 自动 toString()
    println(book1) // -> Book(isbn=..., title=..., author=...)

    // 2. 自动 equals() & hashCode()
    println(book1 == book2) // -> true

    // 3. 强大的 copy() 方法，用于创建对象的不可变副本
    val book3 = book1.copy(title = "Kotlin in Action, Second Edition")
    println(book3)

    // 4. 解构声明 (Destructuring Declaration)
    val (isbn, title, _) = book1 // 使用 _ 忽略不需要的属性
    println("Title: $title, ISBN: $isbn")
}
```

### 6.2	密封类

创建一种可控的, 有限的类结构层次, 是枚举的超集.

```kotlin
sealed class NetworkState {
    object Loading : NetworkState()
    data class Success(val data: List<String>) : NetworkState()
    data class Error(val message: String, val code: Int) : NetworkState()
    object Idle : NetworkState()
}

fun handleState(state: NetworkState) {
    val message = when (state) {
        is NetworkState.Loading -> "Data is loading..."
        is NetworkState.Success -> "Success! Data size: ${state.data.size}"
        is NetworkState.Error -> "Error ${state.code}: ${state.message}"
        is NetworkState.Idle -> "Waiting to start."
    }
    println(message)
}
```

### 6.3	单例对象与伴生对象

#### 6.3.1	单例 Object

```kotlin
// 一个线程安全的单例
object AppSettings {
    val apiUrl = "https://api.myapp.com"
    var theme = "dark"
}

// 全局直接使用
println(AppSettings.apiUrl)
AppSettings.theme = "light"
```

与 Java 相比, 极大的简化了单例模式的实现.

#### 6.3.2	伴生对象

```kotlin
class Path private constructor(val fullPath: String) {
    companion object {
        const val SEPARATOR = "/" // 真正的编译期常量
        
        // 相当于 Java 的静态工厂方法
        fun from(userPath: String): Path {
            val sanitizedPath = userPath.trim()
            return Path(sanitizedPath)
        }
    }
}

// 调用方式类似静态成员
val separator = Path.SEPARATOR
val myPath = Path.from("/usr/local/bin ")
```

Kotlin 用于替代 Java `static` 成员的方案, 本质是一个与外部类紧密关联的单例对象; 伴生对象可以实现接口, 拥有自己的父类, 比 Java 的 `static` 更加灵活和强大.