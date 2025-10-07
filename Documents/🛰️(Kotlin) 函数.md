---
created: 2025-09-27 22:34:06
updated: 2025-09-27 22:34:06
---
Kotlin 和 Python 一样, 都有 "函数是一等公民" 的概念, 这意味着函数可以:

- 存储在变量或者数据结构中;
- 作为参数传递给其他函数;
- 作为另一个函数的返回值.

## 1 函数基础知识

### 1.1 函数声明

Kotlin 的函数声明语法

```kotlin
fun functionName(parameter1: Type1, parameter2: Type2): ReturnType { ... }

// 案例
fun add(a: Int, b: Int): Int {
    return a + b
}
```

### 1.2 Unit 返回类型

```kotlin
fun printSum(a: Int, b: Int): Unit {
    println("Sum is ${a + b}")
}

// `Unit` 返回类型可以省略
fun printSumConcise(a: Int, b: Int) {
    println("Sum is ${a + b}")
}
```

如果一个函数不返回任何有意义的值, 它的返回类型是 `Unit`:

```kotlin
public actual object Unit {  
    override fun toString(): String = "kotlin.Unit"  
}
```

```java
interface AsyncTask<T> {  
    T execute();  
}  
  
public static void main(String[] args) {  
    AsyncTask<Void> asyncTask = () -> null;  
    System.out.println(asyncTask.execute().toString()); // NPE Error
}
```

这在处理泛型的时候很有帮助, 在 Java 中, 如果要处理带和不带返回值的情况, 可以声明返回值为 `Void`, 还需要显示的写 `return null`; `Void` 仅作为一个标识, 此时函数返回的仍然是 `null`, 需要手动处理, 否则会出现 NPE.

```kotlin
interface AsyncTask<T> {  
    fun execute(): T  
}  
  
class LogTask: AsyncTask<Unit> {  
    override fun execute() {  
        println("do log")  
    }  
}
```

而在 Kotlin 中, 可以选用更简单的方式.

