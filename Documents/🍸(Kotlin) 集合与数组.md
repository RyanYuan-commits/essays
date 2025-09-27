---
type: Kotlin
sub-type: 语法基础
finished: "true"
---
## 1 数组

Kotlin 有类似 Java 的对象数组 `Array<T>`, 使用 `arrayOf()` 创建, 对应着 Java 中的 `String[]`, `Integer[]` 等:

```kotlin
val arr: Array<Int> = arrayOf(1, 2, 3, 4, 5)
```

而 Java 中的原生类型数组, 比如 `int[]`, `double[]` 等, 在 Kotlin 中需要特殊声明, 通过对应的 `xxxArrayOf()` 函数来创建, 创建出来的数组类型为 `xxxArray`, 比如 `intArrayOf()` => `IntArray`.

## 2 集合

### 2.1 默认不可变性

Kotlin 默认创建的集合是 Read-Only 的:

```kotlin
val numbers: List<Int> = listOf(1, 2, 3)
// numbers.add(4) // <-- 编译错误！'add' 方法不存在
```

可变集合需要特殊的声明:

```kotlin
val mutableNumbers: MutableList<Int> = mutableListOf(1, 2, 3) mutableNumbers.add(4) // OK! 
println(mutableNumbers) // 输出: [1, 2, 3, 4]
```


## 2.2 常见集合的创建与使用

#### List

创建只读 List: `listOf()`;

创建可变 List: `mutableListOf()`

处理空值: `listOfNotNull()` 会自动过滤掉传入的 `null` 元素.

```kotlin
val fruits = listOf("apple", "banana", "cherry") // 只读
val mutableFruits = mutableListOf("apple", "banana") // 可变
mutableFruits.add("cherry")

// 获取元素
println(fruits[0]) // 使用下标访问，比 .get(0) 更简洁
println(fruits.first())
println(fruits.last())
```

#### Set

创建只读 Set: `setOf()`;

创建可变 Set: `mutableSetOf()`;

```kotlin
val fruits = setOf("apple", "banana", "cherry") // 只读
val mutableFruits = mutableSetOf("apple", "banana") // 可变
mutableFruits.add("cherry")

// 检查元素是否存在
println("apple" in fruits) // 输出: true
println("orange" in fruits) // 输出: false
```

#### Map

创建只读 Map: `mapOf()`;

创建可变 Map: `mutableMapOf()`;

```kotlin
val capitals = mapOf("France" to "Paris", "Japan" to "Tokyo") // 只读
val mutableCapitals = mutableMapOf("France" to "Paris") // 可变
mutableCapitals["Japan"] = "Tokyo"

// 获取元素
println(capitals["France"]) // 输出: Paris
println(mutableCapitals["Japan"]) // 输出: Tokyo

// 遍历键值对
for ((country, capital) in capitals) {
    println("The capital of $country is $capital")
}
```

## 3 函数式 API

Kotlin 将 Java 中的函数式 API 直接内置到了集合类型中:

```kotlin
val numbers = listOf(1, -2, 3, -4, 5)

// 1. 遍历 forEach
numbers.forEach { println(it) }

// 2. 筛选 filter (返回新 List)
val positives = numbers.filter { it > 0 } // 'it' 是 lambda 中的单个元素
println(positives) // 输出: [1, 3, 5]

// 3. 转换 map (返回新 List)
val squared = numbers.map { it * it }
println(squared) // 输出: [1, 4, 9, 16, 25]

// 4. 查找 find (或 firstOrNull)
val firstEven = numbers.find { it % 2 == 0 } // 返回第一个满足条件的元素，或 null
println(firstEven) // 输出: -2

// 5. 检查 any / all / none
val hasNegative = numbers.any { it < 0 } // 是否有任意元素满足条件? -> true
val allPositive = numbers.all { it > 0 } // 是否所有元素都满足条件? -> false

// 6. 分组 groupBy (返回 Map)
val words = listOf("apple", "banana", "ant", "ball")
val groupedByFirstChar = words.groupBy { it.first() }
println(groupedByFirstChar) // 输出: {a=[apple, ant], b=[banana, ball]}

// 7. 打平 flatMap
val nestedList = listOf(listOf(1, 2), listOf(3, 4))
val flatList = nestedList.flatMap { it }
println(flatList) // 输出: [1, 2, 3, 4]
```