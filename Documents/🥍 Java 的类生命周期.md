类的生命周期可以被分为七个阶段：
![[类的生命周期.png]]
## 加载阶段
加载阶段的第一步是类加载器根据类的全限定名，通过不同的渠道以二进制流的方式加载字节码信息，例如从本地文件中加载字节码文件、利用动态代理生成的字节码文件、通过网络传输的类等。
当类加载器加载完成类之后，Java 虚拟机会将字节码中的信息保存到内存的方法区中。
同时会生成一个 InstanceKlass 对象，保存类的全部信息，其中还包含实现特定的功能，比如实现堕胎的信息。
在堆中生成一个与方法区中的 InstanceKlass 类似的 java.lang.Class 类型的对象，通过这个对象，我们可以在 Java 代码中获取类信息以及存储的静态字段的数据，这也就是反射的基本原理。
## 连接阶段
连接阶段又可以分为以下的三个具体阶段：
- ==验证阶段==：验证内容是否符合《Java 虚拟机规范》
- ==准备阶段==：给静态变量赋初始值
- ==解析阶段==：将常量池中的符号引用替换为指向内存的直接引用
### 验证阶段
这个阶段是为了验证字节码文件是否符合《Java 虚拟机规范》中的约束，这个阶段一般不需要程序员的参与，一般包含一下的四个部分。
1. 文件格式的验证。
2. 元信息的验证，例如类必须有父类。
3. 验证字节码文件 执行指令的语义，比如方法中执行到一般突然转到其他方法中。
4. 符号引用的验证，例如是否访问了其他类的 private 方法等。
### 准备阶段
如果是 static 修饰的变量，无论开发者指定了什么值，在这个阶段给它赋值的都是这个类型的初始值，比如 int 就是 0。
但是当变量被 final 修饰的时候，会将开发者指定的值赋值给变量。
### 解析阶段
在解析的阶段会将符号引用转换成内存地址的引用，可以提高访问的速度。
## 初始化阶段
1. 初始化阶段会指定静态代码块中的代码，并且为静态变量赋值。
2. 初始化阶段会执行字节码文件中 **clinit** 部分的字节码指令，给 static 变量赋值和执行 static 代码块中的方法就是在这个阶段执行的。
```java
public class Main {
    public static int x = 1;
    static {
        x++;
    }
}
```

```java
 0 iconst_1
 1 putstatic #7 <org/example/Main.x : I> // 从操作数栈中获取内容，放置到变量中
 4 getstatic #7 <org/example/Main.x : I> // 将静态变量放置到操作数栈中
 // 这里的自增没有用到 iinc 1 by 1？这个指令是在对局部变量操作的时候使用的
 7 iconst_1
 8 iadd
 9 putstatic #7 <org/example/Main.x : I>
12 return
```
**以下的几种方式会导致类的初始化：**
- 访问一个类的静态变量和静态方法，注意变量是 final 修饰的时候，并不会触发初始化
- 调用 `Class.forName()`
- 通过 new 去创建一个对象的时候（ 类的实例化，属于使用阶段）
- 执行 main 方法的当前类
```java
public class Main {
    public static void main(String[] args) {
        System.out.println("A");
        new Main();
        new Main();
    }
    public Main() {
        System.out.println("B");
    }
    {
        System.out.println("C");
    }
    static {
        System.out.println("D");
    }
}
```
输出的顺序 DACBCB
首先执行 main 方法的类需要被加载，执行静态代码块中的方法，然后执行 main 方法中的部分，非静态代码块会在每次创建类的时候，在构造方法之前执行，所以每次创建类的时候都会输出 CB。
**类构造方法的执行逻辑：**
```java
 0 aload_0 // 将当前对象引用加载到操作数栈顶。
 1 invokespecial #21 <java/lang/Object.<init> : ()V> // 调用超类（java/lang/Object）的构造函数。这是每个类都会隐式调用的步骤，确保对象正确地初始化。
 4 getstatic #1 <java/lang/System.out : Ljava/io/PrintStream;> // 获取标准输出流对象
 7 ldc #24 <C> // 将索引为 24 的内容放到栈顶（C）
 9 invokevirtual #9 <java/io/PrintStream.println : (Ljava/lang/String;)V> // 调用PrintStream对象的println方法，打印操作数栈顶的字符串。
12 getstatic #1 <java/lang/System.out : Ljava/io/PrintStream;> // 获取输入流
15 ldc #26 <B> // 将索引为 26 的内容放到栈顶
17 invokevirtual #9 <java/io/PrintStream.println : (Ljava/lang/String;)V> // 输出内容
20 return
```
**在初始化阶段需要执行的逻辑：在 clinit 中**
```java
// 输出 D
0 getstatic #1 <java/lang/System.out : Ljava/io/PrintStream;>
3 ldc #9 <D>
5 invokevirtual #3 <java/io/PrintStream.println : (Ljava/lang/String;)V>
8 return
```

> **在以下的情况中，不会出现 clinit 方法：** 无静态代码块且没有静态变量的赋值语句 有静态变量的声明，但是没有赋值语句 静态变量的定义使用了 final 关键字，在准备阶段就已经初始化了

有两个需要特别注意的点：
**直接访问父类的静态变量，不会触发子类的初始化** **子类的初始化 clinit 调用之前，会优先调用父类的 clinit 方法**
```java
public class Main {
    public static void main(String[] args) {
        B b = new B(); // 实例化 b，需要执行初始化阶段
        System.out.println(B.a); // 10
    }
}
class A {
    static int a = 0;
    static {
        a = 1;
        System.out.println("父类的静态代码块");
    }
}

class B extends A {
    static {
        a = 10;
    }
}
```
而将 `B b = new B();` 去掉之后，就会直接访问父类的变量此时输出的是 1。