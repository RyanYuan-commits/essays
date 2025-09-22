## 1 字符串基础

和 Java 相同, Kotlin 的字符串也是不可变的;

Kotlin 的字符串可以通过 `[]` 语法来方便的获取到字符串中的某个字符; 也可以通过 for 循环来遍历;

```kotlin
val name = "Kotlin"  
for (c in name) {  
    println(c)  
}
```

Kotlin 和 Python 相同, 支持使用 `"""` 包裹的多行字符串, 并且提供了 API 来删除多余的空白:

```kotlin
val multiLine = """
   |Line 1
   |Line 2
   |Line 3
""".trimMargin() // 去除多余的前置空格

println(multiLine)
```

## 2 字符串模板

```kotlin
val i = 1  
val str = "i = ${i}"  
println(str) // i = 1
```

在字符串中使用字面值

```kotlin
val price = """
${'$'}9.99
"""
println(price)  // $9.99
```

