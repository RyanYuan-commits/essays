---
created: 2025-09-27 22:34:06
updated: 2025-09-27 22:34:06
---
## 1 什么是字符串常量池？

JVM 为了提高性能和减少内存开销, 在实例化字符串常量的时候, 为字符串开辟了一个字符串常量池.

底层是 C++ 的 `StringTable`(类似于 `HashTable`), 保存的是字符串对象的引用.

## 2 字符串常量池的位置

JDK 1.6 之前, 方法区实现为永久代, 运行时常量池存储在永久代中, 运行时常量池包含字符串常量池.

JDK 1.7 已经逐步开始去除永久代, 字符串常量池和静态变量被移动到了堆内存中.

JDK 1.8 之后, 永久代被移除, 取而代之的是元空间, 而字符串常量池仍然位于堆内存之中.

![[JDK中线程共享区域的演进.png]]

## 3 三种字符串操作

### 3.1 操作一: 直接赋值字符串

```java
String s = "ryan";
```

![[直接赋值字符串演示.png|800]]

s 指向的是字符串常量池中的引用, 在执行这个语句时:

- 如果字符串常量池中有相同内容的字符串, 直接返回字符串常量池的引用;
- 如果没有, 则会在字符串常量池中创建, 并且返回这个引用. 

这种方式创建的字符串对象只会存在于常量池中. 
### 3.2 操作二: new String

例如: `String s = new String("a");`

![[使用 new 关键字来创建字符串.png|800]]

- 如果字符串常量池中存在这个字符串, 直接去堆内存中创建一个字符串对象；
- 如果不存在, ==会先在字符串常量池中创建一个字符串对象==, 再去堆内存中创建字符串对象. 

这两个对象唯一的关联就是, 它们的 `value` 属性指向的是**相同的 char 数组** (JDK 9+ 改为了 byte 数组). 

```java
public class Main {
    public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException {  
        String s1 = "abc"; 
        String s2 = new String("abc");
  
        Class<String> stringClass = String.class;
        Field value = stringClass.getDeclaredField("value");
        value.setAccessible(true);
  
        Object value1 = value.get(s1);
        Object value2 = value.get(s2);
  
        System.out.println(value1 == value2);
        System.out.println(s1 == s2);
    }
}
```

使用 `"abc"` 和 `new String("abc")` 创建字符串, 并比较它们的地址和 value 的地址, 输出为:

```java
true
false
```

### 3.3 操作三: intern 方法

`intern` 是一个 `native` 方法, 它的作用是当前将当前字符串对象放入字符串常量池中, 并返回池中已存在的相同字符串的引用. 

在 JDK6 中, 字符串常量池是位于永久代, 而非堆区域, 执行 `intern()` 方法时: 

- 首先检查字符串常量池中是否已经存在与当前字符串内容相同的字符串常量. 如果有, 直接返回常量池中该字符串对象的引用. 
- 如果不存在相同内容的字符串, JVM 会在**永久代**中创建一个新的字符串实例, 并让字符串常量池的一个表项指向这个新创建在方法区的实例, 然后返回这个新实例的引用. 

从 JDK7 开始, 字符串常量池被移动到了堆区域中. 此时执行过程变为: 

- 同样先检查字符串常量池中是否存在与当前字符串内容相同的字符串常量. 若存在, 返回常量池中该字符串对象的引用. 
- 当常量池中不存在相同内容的字符串时, 不会再新建一个实例, 而是让字符串常量池的一个表项**直接指向**堆区中已存在的这个字符串对象 (因为对象本身就在堆里), 然后返回该对象的引用. 

因为这个原因, 下面的这段代码在两个 JDK 版本中会得到不同的结果: 

```java
String s = new String("ab");
s.intern();
String s2 = "ab";
System.out.println(s2 == s); //JDK6: false, JDK7: true
```

在 JDK6 中, s2 指向的是常量池中新建的对象, 而 s 指向的是堆区域中的对象; 在 JDK7 及之后, s 指向的是堆区域中的对象, s2 也通过常量池来指向了这个对象. 

## 4 编译器对于字符串拼接操作的优化
### 4.1 字符串和字符串的拼接

如 `String s = "a" + "b" + "c";`, 编译器会将其优化为: `String s = "abc";`. 
### 4.2 字符串变量的拼接

```java
String a = "a";
String b = "b";
String c = "c";
String s = a + b + c;
```

上面的语句会被优化为

```java
String s = new StringBuilder().append(a).append(b).append(c).toString();
```

我们如果在循环中使用 + 来拼接字符串的话, idea 会这样提醒: 

![[在循环中使用 + 来拼接字符串.png|600]]

编译后, 这个语句执行方式等同于下面的这段代码, 每次循环中都会创建一个 `StringBuilder` 对象:

```java
for (int i = 0; i < 10; i++) {
	t = new StringBuilder(t).append(s).toString;
}
```

正常的写法是这样的:

```java
StringBuilder t = new StringBuilder("world");  
for (int i = 0; i < 10; i++) {  
    t.append(s);  
}
```

还有一种情况: 

```java
String a = "1";
String b = "2";
String c = "3";

String s = new StringBuilder("1").append(a).append(b).append(c).toString();
```

在这种情况下, IDE 会建议写成: `String s = "1" + a + b + c;` 来提升可读性.