## 1 什么是字符串常量池？
JVM 为了提高性能和减少内存开销，在实例化字符串常量的时候，为字符串开辟了一个字符串常量池（String Constant Pool），类似于缓冲区。
字符串常量池的底层是 HotSpot 的 C++ 实现，它是一个 StringTable（类似于 Java 中的 HashTable），保存的本质上是 **字符串对象的引用**。
## 2 字符串常量池的位置
JDK 1.6 之前，方法区实现为永久代，运行时常量池存储在永久代中，运行时常量池包含字符串常量池。
JDK 1.7 已经逐步开始去除永久代，字符串常量池和静态变量被移动到了堆内存中。
JDK 1.8 之后，永久代被移除，取而代之的是元空间，而字符串常量池仍然位于堆内存之中。
![[JDK中线程共享区域的演进.png]]
## 3 三种字符串操作
本部分我们会讲解三种基础的字符串操作及其背后的原理，如果没有具体说明，则是默认根据 JDK 1.8 来作为案例的，字符串常量池位于堆内存。
### 3.1 操作一：直接赋值字符串
```java
String s = "ryan";
```
![[直接赋值字符串演示.png]]
s 指向的是字符串常量池中的引用，在执行这个语句的时候，JVM 会先判断字符串常量池中是否有这个对象
- 如果有，直接返回字符串常量池的引用；
- 如果没有，则会在字符串常量池中创建，并且返回这个引用。  
这种方式创建的字符串对象只会存在于常量池中。
### 3.2 操作二：使用 new 关键字来创建字符串
例如：String s = new String("a");
![[使用 new 关键字来创建字符串.png|1200]]
虚拟机栈中的局部变量 s 指向的是字符串常量池中的引用。
JVM 首先去检查字符串常量池中是否存在这个字符串：
- 如果存在，就直接去堆内存中创建一个字符串对象；
- 如果不存在，==会先在字符串常量池中创建一个字符串对象==，再去堆内存中创建字符串对象。
**这两个对象唯一的关联就是，它们的 value 属性指向的是相同的字符数组**（JDK 9+ 改为了 byte 数组）。

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
输出：
```java
true
false
```
### 3.3 编译器对于字符串拼接操作的优化
#### 3.3.1 字符串和字符串的拼接操作
对于 String s = "a" + "b" + "c"; 这样的语句，Java 编译器会对其进行优化，上面的句子其实就等价于 String s = "abc";。
#### 3.3.2 字符串变量和字符串变量的拼接操作
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

我们如果在循环中使用 + 来拼接字符串的话，idea 会这样提醒：
![[在循环中使用 + 来拼接字符串.png]]

编译后，这个语句执行方式等同于下面的这段代码：
![[在循环中使用 + 来拼接字符串编译后.png]]
每次循环中都会创建一个 StringBuilder 对象，也就是会存在大量的对象创建与销毁，在开发的时候一定要注意。

可以将 StringBuilder 的创建提到循环外：
```java
StringBuilder t = new StringBuilder("world");  
for (int i = 0; i < 10; i++) {  
    t.append(s);  
}
```

还有一种情况：
![[StringBuilder 优化可读性案例.png]]
在这种情况下，IDEA  会推荐我们写成：String s = "1" + a + b + c; 这和上面的方式是等价的，但是可读性大大提升。
### 3.4 intern 方法
String 中的 intern 方法是一个 native 方法，当调用 intern 的时候，它的作用是当前将当前字符串对象放入字符串常量池中，并返回池中已存在的相同字符串的引用。
在 JDK 1.6 中，字符串常量池是位于方法区（`PermGen`，永久代），而不是堆区。具体执行 `intern()` 方法时：
- **判断是否存在**：首先检查字符串常量池中是否已经存在与当前字符串内容相同的字符串常量。如果有，直接返回常量池中该字符串对象的引用。
- **不存在时的操作**：如果常量池中不存在相同内容的字符串，JVM 会在方法区中创建一个新的字符串实例，并让字符串常量池（`StringTable`）的一个表项指向这个新创建在方法区的实例，然后返回这个新实例的引用。

从 JDK 1.7 开始，字符串常量池被移动到了堆区域中。此时 `intern()` 方法的执行过程是：
- **判断是否存在**：同样先检查字符串常量池中是否存在与当前字符串内容相同的字符串常量。若存在，返回常量池中该字符串对象的引用。
- **不存在时的操作**：当常量池中不存在相同内容的字符串时，不会再新建一个实例，而是让字符串常量池（`StringTable`）的一个表项直接指向堆区中已存在的这个字符串对象（因为对象本身就在堆里），然后返回该对象的引用。

因为这个原因，下面的这段代码在两个 JDK 版本中会得到截然不同的结果：
```java
String s = new String("ab");
s.intern();
String s2 = "ab";
System.out.println(s2 == s);
```
在 JDK1.6 中，s2 指向的是常量池中新建的对象，而 s 指向的是堆区域中的对象。
在 JDK 1.7 及之后，s 指向的是堆区域中的对象，s2 也通过常量池来指向了这个对象。