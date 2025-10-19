---
created: 2025-09-27 22:34:06
updated: 2025-09-27 22:34:06
---

闭包是一个**可以作为变量传递的代码块**: 

```groovy
// 这是一个闭包，它接受一个参数并打印它
def myClosure = { name ->
    println("Hello, ${name}")
}

// 像调用函数一样调用它
myClosure("Groovy") // 输出: Hello, Groovy
```

---

```groovy
// 传统写法
someMethod(arg1, arg2, { closureBody })

// Groovy 语法糖写法
someMethod(arg1, arg2) { closureBody }
```

在 Groovy 中, 当函数的最后一个参数是闭包, 可以将闭包写在括号外面;

所以在 Gradle 构建脚本中类似 `dependencies { ... }` 和 `tasks { ... }` 的形式, 其实是传递了闭包的函数调用.

---

```groovy
// 传统写法，显式参数
def printNumber = { number -> println("Number: ${number}") }

// 简写，使用隐式参数 'it'
def printNumberIt = { println("Number: ${it}") }

printNumberIt(42) // 输出: Number: 42
```

如果闭包只有一个参数时, 可以省略参数和 `->` 符号, 这个唯一的参数会被隐式的命名为 `it`;

比如, 在 `subprojects` 中, 可以通过 `it.name` 来获取当前配置项目的名称.

---

```groovy
class Printer {
    void print(Closure c) {
	    // 指定闭包的委托对象是 this
        c.delegate = this
        c.resolveStrategy = Closure.DELEGATE_FIRST
        c.call()
    }
    
    void printLine(String s) {
        println("--- ${s} ---")
    }
}

def printer = new Printer()

printer.print {
    printLine("User Info:")
    printLine("Name: ${name}")
    printLine("Age: ${age}")
}
```

闭包可以通过 `c.delegate = xxx` 来指定委托对象;

在闭包内部, 如果你调用一个方法或者访问一个属性, Groovy 会首先尝试在闭包的委托对象上查找;

当在 Gradle 构建脚本中写 `dependencies { ... }` 时, 闭包委托对象的类型是 `DependencyHandler`, 所以你在 `{ ... }` 可以直接的调用 `implemention`, `testImplementions` 等方法.

