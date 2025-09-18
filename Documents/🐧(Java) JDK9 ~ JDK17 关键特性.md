---
type: Java
sub-type: release
finished: "false"
source: https://www.oracle.com/java/technologies/javase/17-relnote-issues.html#NewFeature
---
## [1 Records Classes](https://openjdk.org/jeps/395)

Java 新的名义类型, 作为不可变数据的透明载体, 提供简单清晰的方式来定义不可变的数据结构.

定义 `record` 类型后, 会自动拥有: 构造函数, 所有字段的 `get` 和 `set` 方法, `equals` 和 `hashCode` 方法, `toString` 方法.

该特性在 JDK14 中首次预览, 在 JDK16 中转正.
### 1.1 Record 类型语法

`record` 类型的基本语法:

```java
public record Person(String name, int age) {}
```

上面的代码相当于:

```java
public final class Person {
    private final String name;
    private final int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public String name() {
        return name;
    }

    public int age() {
        return age;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Person person = (Person) o;
        return age == person.age && Objects.equals(name, person.name);
    }

    @Override
    public int hashCode() {
        return Objects.hash(name, age);
    }

    @Override
    public String toString() {
        return "Person{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}
```

### 1.2 Record 类型的优点

#### 减少样板代码的编写

不需要再手动的编写各种各样的样板方法, 甚至也无需使用 Lombok.

#### 默认的不可变性

Record 被设计为不可变的 (immutable), 为了保证这种不可变性, 我们保证引用类型的不可变性, 使用可变集合或者其他可变类型可能会破坏 Record 的不可变性.

解决方式有复制为不可变类型和防御性复制;

使用不可变类型, 从根本上维护不可变性:

```java
import java.util.Collections;
import java.util.List;

public record Person(String name, List<String> hobbies) {
    public Person {
        hobbies = Collections.unmodifiableList(hobbies);
    }
}
```

防御性复制, 保证原对象的修改不会影响到 Record 中的数据:

```java
import java.util.ArrayList;
import java.util.List;

public record Person(String name, List<String> hobbies) {
    public Person {
        hobbies = new ArrayList<>(hobbies);
    }
}
```

#### 增强代码的可读性

被 Record 修饰表示这是一个纯粹的数据载体, 不包含任何的额外行为.

### 1.3 Record 高级特性

#### 自定义构造函数

可以在 Record 中自定义构造函数, 来进行额外的验证或者处理:

```java
public record Person(String name, int age) {
    public Person {
        if (age < 0) {
            throw new IllegalArgumentException("Age cannot be negative");
        }
    }
}
```

编译器生成的构造方法:

```java
public Person(java.lang.String name, int age) {
	// 这是你的紧凑构造器代码块
	if (age < 0) {
		throw new java.lang.IllegalArgumentException("Age cannot be negative");
	}
	
	// 编译器自动添加的赋值代码
	this.name = name;
	this.age = age;
}
```

#### 自定义方法

Record 也可以包含自定义方法, 但由于记录的本质的简单的数据载体, 尽量还是减少这种方法保持代码的清晰.

#### 嵌套记录

记录也可以嵌套在其他的类或者记录中, 来构造更复杂的结构:

```java
public record Address(String street, String city) {}  
  
public record Person(String name, int age, Address address) {}
```


## 参考文章

```embed
title: "Java Language Updates"
image: ""
description: "This section summarizes the updated language features in Java SE 9 and subsequent releases."
url: "https://docs.oracle.com/en/java/javase/17/language/java-language-changes-release.html#GUID-6459681C-6881-45D8-B0DB-395D1BD6DB9B"
favicon: ""
```

```embed
title: "Java 9 新特性概览"
image: "https://oss.javaguide.cn/java-guide-blog/image-20210816083417616.png"
description: "Java 9 发布于 2017 年 9 月 21 日 。作为 Java 8 之后 3 年半才发布的新版本，Java 9 带来了很多重大的变化其中最重要的改动是 Java 平台模块系统的引入，其他还有诸如集合、Stream 流……。 你可以在 Archived OpenJDK General-Availability Releases 上下载自己需要的 ..."
url: "https://javaguide.cn/java/new-features/java9.html"
favicon: ""
aspectRatio: "60.07604562737643"
```

