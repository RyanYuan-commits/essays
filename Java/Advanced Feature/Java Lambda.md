---
type: Java
sub-type: Advanced Features
---
Java Lambda 表达式是 JDK8 引入的特性, 提供了一种简洁的方式来表示匿名函数, 用于实现 **函数式接口**;

## 1	Main Features

Lambda 表达式没有名称, 可以直接作为参数传递;

显著减少了编写匿名内部类需要的样板代码;

它们只能用于实现只有一个抽象方法的接口(函数式接口);

使代码能够接受不同的函数作为参数, 支持函数式编程;

是 Java Stream API 中进行数据处理的核心机制.

## 2	Case

```java
import java.util.Arrays;
import java.util.Collections;
import java.util.Comparator;
import java.util.List;

public class LambdaDemonstration {

    public static void main(String[] args) {
        List<String> frameworks = Arrays.asList("Spring", "Hibernate", "Struts", "JSF", "MyBatis");

        // ----------------------------------------------------
        // 案例 1: 使用 Java 8 之前的匿名内部类进行排序
        // ----------------------------------------------------
        System.out.println("原始顺序: " + frameworks);

        // 这就是“旧方法”：为了定义一个简单的比较逻辑，
        // 我们需要创建一个完整的匿名内部类实例。
        // 这里的核心逻辑只有一行：s2.length() - s1.length()
        // 但被大量的样板代码包围着。
        Comparator<String> legacyComparator = new Comparator<String>() {
            @Override
            public int compare(String s1, String s2) {
                // 按字符串长度降序排序
                return s2.length() - s1.length();
            }
        };
        Collections.sort(frameworks, legacyComparator);
        System.out.println("使用匿名内部类排序后: " + frameworks);


        // ----------------------------------------------------
        // 案例 2: 使用 Lambda 表达式进行排序
        // ----------------------------------------------------
        // 重置列表以便对比
        frameworks = Arrays.asList("Spring", "Hibernate", "Struts", "JSF", "MyBatis");

        // 这就是“新方法”：使用 Lambda 表达式。
        // (s1, s2) -> s2.length() - s1.length() 这段代码本身
        // 就是对 Comparator 接口中 compare 方法的实现。
        // 它清晰地表达了“输入两个字符串，返回它们长度的差值”。
        // 编译器会根据 Collections.sort 的方法签名，自动推断出
        // 这个 Lambda 表达式就是 Comparator<String> 接口的一个实例。
        Collections.sort(frameworks, (s1, s2) -> s2.length() - s1.length());

        System.out.println("使用 Lambda 表达式排序后: " + frameworks);

        // ----------------------------------------------------
        // 案例 3: Lambda 在 forEach 中的应用
        // ----------------------------------------------------
        System.out.print("使用 Lambda 遍历输出: ");
        // forEach 方法接受一个 Consumer 函数式接口的实例
        // (framework) -> System.out.print(framework + " ") 就是一个 Consumer
        frameworks.forEach(framework -> System.out.print(framework + " "));
        System.out.println();
    }
}
```

## 3	Principle of Java Lambda

### 3.1	Comparison: Traditional Anoalymous Inner Class

```java
// Java 8 之前
Runnable r = new Runnable() {
    @Override
    public void run() {
        System.out.println("Hello, Anonymous Inner Class!");
    }
};
r.run();
```

每次使用匿名内部类, 编译器都会生成一个单独的 `.class` 文件, 这会导致类加载的开下, 并使最终的 JAR 包变大; 并且创建时, 都需要完整的类加载, 验证, 实例化过程, 相对较重, 并且语法也比较啰嗦.

Lambda 表达式的出现不仅解决了语法冗余的问题, 更重要的是在底层实现上进行了彻底的优化.

### 3.2	Core: `invokedynamic` instruction

#### 3.2.1	Brief Description of invokedynamic

`invokedynamic` 是 JDK7 引入的一条新的 JVM 字节码指令, 它最初是为了支持运行在 JVM 上的动态语言 (如 JRuby, Groovy) 而设计的; 与其他的调用指令, 如 `invokestatic`, `invokevirtual` 不同, `invokedynamic` 的方法调用的链接过程推迟到运行时.

当编译器遇到 Lamda 表达式时, 它并不会像匿名内部类那样立即生成一个新的类, 而是会执行如下操作:

1. **生成一个 "脱糖" 方法 (Desugaring)**: 编译器会将 Lambda 表达式的主体代码提取出来, 生成一个静态私有方法(如果 Lambda 捕获了实例变量, 则会生成一个实例私有方法), 这个方法包含了 Lambda 的具体逻辑, 方法名通常是 `lambda$..` 的形式.
2. **生成 `invokedynamic` 指令**: 在原来使用 Lambda 表达式的地方, 编译器会插入一条 `invokedynamic` 指令. 这条指令告诉 JVM: "这里需要一个实现了某个函数式接口的对象, 但具体如何创建这个对象, 请在运行时决定."

这个 "在运行时决定" 的过程, 由一个特殊的 Bootstrap Method 来完成对于 Lambda 表达式, 这个引导方法通常是 `LambdaMetafactory.metafactory()`.

#### 3.2.2	Execution Process

**首次执行**: 当 `invokedynamic` 指令第一次被执行时, JVM 会调用其指定的引导方法 `LambdaMetafactory.metafactory`;

**引导方法工作**: `LambdaMetafactory` 会接收到 JVM 传来的所有必要信息, 包括: 

- Lambda 表达式要实现的函数式接口是什么, 如 `Runnable`.
- 接口中需要实现的方法签名是什么, 如 `run()`.
- 编译器生成的那个私有静态/实例方法(即 Lambda 的主体代码)的方法句柄。

**动态生成类并创建实例**: `LambdaMetafactory` 会在内存中**动态地生成一个类**. 这个类实现了目标函数式接口, 并且其接口方法的实现就是去调用编译器之前生成的那个私有方法. 然后, 它会创建一个该类的实例.

**链接调用点**: 引导方法返回一个 `CallSite` 对象, 这个对象会和 `invokedynamic` 所在的位置进行永久链接. `CallSite` 内部包含了创建 Lambda 实例的逻辑.

**后续执行**: 之后再次执行到这条 `invokedynamic` 指令时, JVM 不会再调用引导方法, 而是直接使用已经链接好的 `CallSite` 来快速创建或返回 Lambda 实例. 对于没有捕获任何外部变量的 Lambda, `LambdaMetafactory` 足够智能, 可以返回一个**单例实例**, 从而达到极高的性能.

### 3.3	Why Final of Effectively Finally

当 Lambda 表达式被创建时, 它并不是获取了外部局部变量的内存地址, 而是像拍快照一样, **复制并持有了那个变量当时的值**. 定义 Lambda 的方法执行完后, 其局部变量就会被销毁. 但 Lambda 对象本身可能还存活(比如在另一个线程里). 如果 Lambda 持有的是变量的引用, 那么它将指向一块被回收的内存, 导致程序崩溃. 因此, 只能复制值.

---

既然 Lambda 内部持有的是一个值的 "快照", 如果外部的原始变量还能被修改, 那么 "快照" 里的值就和外面的新值不一致了. 这会导致代码的行为和程序员的直觉严重冲突, 产生难以调试的 Bug. Lambda 经常被用于多线程, 如果一个变量可以被主线程修改, 同时又被 Lambda 所在的子线程读取, 就会产生数据竞争, 导致结果完全不可预测.yi