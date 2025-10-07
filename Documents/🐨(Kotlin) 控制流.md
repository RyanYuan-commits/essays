---
created: 2025-09-27 22:34:06
updated: 2025-09-27 22:34:06
---
## 1 if 表达式

```kotlin
// 传统用法
var max = a 
if (a < b) max = b

// 使用 else 
var max: Int
if (a > b) {
    max = a
} else {
    max = b
}
 
// 作为表达式
val max = if (a > b) a else b
```

可以用 `in` 运算符来判断某个数字是否在指定区间内, 区间格式 `x..y`:

```kotlin
val x = 5
val y = 9
if (x in 1..8) {
	println("x 在区间内")
}
```

## 2 When 表达式

类似其他语言的 `switch` 操作符, `else` 同 `default`

```kotlin
when (i) {  
    1, 2 -> println("i is 1 or 2")
    else -> println("i is not 1 or 2")  
}
```

除了等于关系, Kotlin 的 `when` 表达式还支持其他关系的判断:

检测一个值是否在一个区间或者集合中:

```kotlin
when (x) {
    in 1..10 -> print("x is in the range")
    in validNumbers -> print("x is valid")
    !in 10..20 -> print("x is outside the range")
    else -> print("none of the above")
}
```

检验值是或者不是一个特定类型的值:

```kotlin
fun hasPrefix(x: Any) = when(x) {
    is String -> x.startsWith("prefix")
    else -> false
}
```

也可以放置布尔表达式:

```kotlin
when {
    x.isOdd() -> print("x is odd")
    x.isEven() -> print("x is even")
    else -> print("x is funny")
}
```

## 3 for 循环

```kotlin
// 对任何提供迭代起的对象进行遍历
for (item in collection) {
	print(item)
}

// 通过索引遍历
for (i in array.indices) {
    print(array[i])
}

// 同时比那里索引和元素
for ((index, value) in array.withIndex()) {
    println("the element at $index is $value")
}
```

## 4 while 与 doWhile 循环

```kotlin
while( 布尔表达式 ) {
  //循环内容
}

do {
   //代码语句
}while(布尔表达式);

```

## 5 返回和跳转

和 Java 相同, 有三种结构化的跳转表达式:

- return: 从直接包围它的函数或者匿名函数返回,
- break: 中支最直接包裹它的循环;
- continue: 继续下一次最直接包裹它的循环

