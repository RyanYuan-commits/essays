官方文档: https://docs.oracle.com/javase/9/

## [1 Convenience Factory Methods for Collections](https://openjdk.org/jeps/269)

方便的集合工厂方法, 用于 **只读** 集合的快速创建, 案例:

```java
// 新版本
List<String> list = List.of("A", "B", "C");
Set<String> set = Set.of("A", "B", "C");
Map<String, Integer> map = Map.of("A", 1, "B", 2, "C", 3);

// 旧版本
List<String> list = Collections.unmodifiableList(
	Arrays.asList("A", "B", "C")
);

```

需要注意创建的集合是只读的, 且不允许传入 null 值.

## [2  More Concurrency Updates](https://openjdk.org/jeps/266)

### 2.1 Stream API 增强

`Stream` 类新增方法: `takeWhile`, `dropWhile`, `ofNullable`, `iterate`(新重载).

`takeWhile(Predicate<? super T>)` 从头取元素, 遇到不符合的停止;

```java
Stream.of(1, 2, 3, 4, 5, 6)
      .takeWhile(n -> n < 4)
      .toList(); // [1, 2, 3]
```

`dropWhile(Predicate<? super T>)` 从头丢弃元素, 直到遇到第一个符合条件的;

```java
Stream.of(1, 2, 3, 4, 5, 6)
      .dropWhile(n -> n < 4)
      .toList(); // [4, 5, 6]
```

`ofNullable(T element)` 创建空的 `Stream` 或单个元素的 `Stream`;

```java
Person anna = null;
Stream<Person> personStream = Stream.ofNullable(anna); 
```

`iterate(...)`: 可以在参数中指定无限流的终止条件.

```java
Stream<Integer> fibonacci = Stream.iterate(0, 
    n -> n < 10, 
    n -> n + 1;
);
fibonacci.forEach(System.out::println);
```

## [3 Milling Project Coin](https://openjdk.org/jeps/213)

### 3.1 接口私有方法

允许在接口中定义 `private` 方法, 让接口的 `default` 和 `static` 复用这些方法.

```java
public interface MyService {

    default void doTaskA() {
        log("Task A running");
    }

    default void doTaskB() {
        log("Task B running");
    }

    private void log(String msg) { // Java 9 新增
        System.out.println("LOG: " + msg);
    }
}

```

### 3.2 try-with-resource 增强

JDK9 之前 `try-with-resources` 资源必须在 `try(...)` 中声明;

改进后, 只要资源是 `final` 或者 `effective final` 可以直接放到 `()` 中:

```java
// old
BufferedReader br = new BufferedReader(new FileReader(path));
try (BufferedReader br2 = br) { ... }

// new
BufferedReader br = new BufferedReader(new FileReader(path));
try (br) { ... }
```

## 4 [Process API Updates](https://openjdk.org/jeps/102)

在 JDK9 之前, 使用 `Runtime.exec` 或者 `ProcessBuilder` 操作外部进程非常复杂, 主要体现在: 获取 `PID` 不直接, 检查进程存活需要自己维护, 获取退出码复杂.

JDK 9 给 `Process` 和 `ProcessHandle` 增加了新的方法, 优化了使用体验:

`Process.pid` 直接获取进程 PID

```java
Process process = Runtime.getRuntime().exec("notepad");
long pid = process.pid();
System.out.println("PID: " + pid);
```

**`ProcessHandle`** 可以更全面地管理和监控进程:

```java
ProcessHandle handle = process.toHandle();
System.out.println("Is alive? " + handle.isAlive());
handle.onExit().thenAccept(h -> System.out.println("Process exited with " + h.exitValue()));
```

`ProcessHandle.allProcesses()`, `current()` 可以列出系统上所有进程, 或获取当前 `JVM` 进程:

```java
ProcessHandle.current().info().command().ifPresent(System.out::println);
```

## 5 Optional  API 增强

### 5.1 JDK8 Optional 常用方法
| 方法                                          | 作用                                    | 示例                                                   |
| ------------------------------------------- | ------------------------------------- | ---------------------------------------------------- |
| `Optional.of(value)`                        | 创建一个非空 Optional，若 value 为 null 会抛 NPE | `Optional.of("A")`                                   |
| `Optional.ofNullable(value)`                | 创建 Optional，可接受 null                  | `Optional.ofNullable(someNullable)`                  |
| `Optional.empty()`                          | 创建一个空 Optional                        | `Optional.empty()`                                   |
| `isPresent()`                               | 判断是否有值                                | `if(opt.isPresent()) ...`                            |
| `get()`                                     | 获取值（若空会抛 NoSuchElementException）      | `opt.get()`                                          |
| `orElse(T other)`                           | 值为空时返回其他值                             | `opt.orElse("B")`                                    |
| `orElseGet(Supplier<? extends T>)`          | 值为空时返回 Supplier 生成的值                  | `opt.orElseGet(() -> computeValue())`                |
| `orElseThrow()`                             | 值为空时抛异常（默认 NPE 或自定义）                  | `opt.orElseThrow(() -> new IllegalStateException())` |
| `ifPresent(Consumer<? super T>)`            | 值存在时执行消费函数                            | `opt.ifPresent(System.out::println)`                 |
| `map(Function<? super T, ? extends U>)`     | 转换值                                   | `opt.map(String::length)`                            |
| `flatMap(Function<? super T, Optional<U>>)` | 转换 Optional 嵌套                        | `opt.flatMap(s -> Optional.of(s.length()))`          |
| `filter(Predicate<? super T>)`              | 根据条件过滤                                | `opt.filter(s -> s.length() > 0)`                    |

### 5.2 JDK9 新增方法
|方法|作用|示例|
|---|---|---|
|`or(Supplier<? extends Optional<? extends T>>)`|当前 Optional 空时，返回备用 Optional|`opt.or(() -> Optional.of("B"))`|
|`stream()`|把 Optional 转为 Stream（0 或 1 个元素）|`opt.stream().forEach(System.out::println)`|

## [6 Javadoc Search](https://openjdk.org/jeps/225)

以前 Javadoc 很难搜索类或方法内容, 只能通过浏览目录或者自己用 IDE 搜索.

JDK9 做了如下的改进:

- 支持搜索框: Javadoc 页面增加搜索栏, 可以输入类名, 方法名快速搜索
- HTML5 支持: Javadoc 输出变成了 HTML5, 更现代化
- 附加标签增强: `@apiNote`, `@implSpec`, `@implNote` 等新的标签, 可以在方法说明中提供额外信息

```java
/**
 * Performs some operation.
 *
 * @apiNote This method is thread-safe.
 * @implSpec Implementation uses a lock-free algorithm.
 */
public void doSomething() { ... }
```

## [7 Stack-Walking API](https://openjdk.org/jeps/259)

在 JDK9 之前, 使用 `Thread.currentThread().getStackTrace()` 获取 `StackTrace` 的时候, 获取的是完整的栈数组, 性能开销大, 且无法灵活的过滤.

JDK9 引入了 `StackWalker`, 提供了惰性访问栈帧, 过滤栈帧, 获取调用者深度的能力:

```java
// 获取默认 StackWalker
StackWalker walker = StackWalker.getInstance();

// 遍历所有栈帧
walker.forEach(frame -> 
    System.out.println(frame.getClassName() + "." + frame.getMethodName())
);

// 获取调用者的直接类名
String callerClass = walker.getCallerClass().getName();
System.out.println("Caller: " + callerClass);
```

## 8 Java Platform Module System

