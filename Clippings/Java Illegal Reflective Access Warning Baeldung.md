---
title: Java 非法反射访问警告 | Baeldung
source: https://www.baeldung.com/java-illegal-reflective-access
created: 2025-08-29
description: 本文详细阐述了 Java 模块化系统的强封装机制如何导致非法反射访问警告，并提供了通过模块声明、命令行选项或运行时方法解决这些警告的具体方案。
finished: "true"
tag: clipper
cover: https://source.unsplash.com/random/800x600/?nature,beautiful
updated: 2025-09-27 22:34:06
---
主题: 模块系统和反射机制之间的关系;

## 1 模块系统与反射机制

### 1.1 底层模型

模块系统的目标之一是强封装性, 包含可读性和可访问性:

- 可读性: 粗略的概念, 涉及一个模块是否依赖于另一个模块;
- 可访问性: 更加 精细的概念, 关注一个类能否访问到另一个类的字段或方法, 通过类边界, 包边界和模块边界来实现;

![[类边界, 包边界和模块边界.png|700]]

这两条规则的关系是: 可读性优先, 可访问性是建立在可读性之上的, 例如, 如果一个类是公共的但未导出, 可读性将阻塞进一步的使用, 而如果一个非公共类位于导出的包中, 可读性将允许其通过, 但是可访问性会拒绝它.

可读性的配置方式: 模块声明中的 `requires`, 命令行 `--add-reads`, 调用 `Module.addReads` 方法;

封装性的配置方式: 模块声明中的 `opens`(提供方), 命令行 `--add-opens`(使用方), 或者 `Module.addOpens` 方法;

在使用反射的时候, 会自动在两个模块建立可读性权利, 出现问题, 也是因为可访问性的原因.

### 1.2 不同的反射使用场景

模块类型:

![[Java 模块分类.png|700]]
发生非法反射访问的三种经典场景:

![[发生非反射访问的三种经典场景.png|700]]

深度反射指的是, 通过调用 `setAccessible(flag)` 方法, 通过反射 API 获取非公共成员的访问权限;

当通过一个命名模块访问另一个命名模块但是喜欢,会遇到 `IllegalAccessException` 或者 `InaccessibleObjectException` 异常, 同样的, 从未命名模块发昂问应用程序模块的时候, 也会出现相同的错误.

然而, 当未命名模块通过反射访问平台模块的时候, 会遇到 `IllegalAccessException` 异常或者 `WARNING`, 这种信息源于宽松的强封装机制.

### 1.3 宽松的强封装机制

在 JDK9 之前, 很多第三方库通过反射 API 实现核心功能, 比如 Hessian2 通过获取 `Constructor.getParameterTypes` 方法来获取构造函数中的参数;

而模块系统的强封装规则, 会使这些代码失效, 为了更加平滑的迁移, Java 使用的是一种宽松的强封装机制;

该机制通过启动参数 `--illegal-access` 控制, 该参数仅在未命名模块访问平台模块时生效, 有四种取值:

- `permit` (默认值): 允许非法反射访问, 但会发出警告; 这是 JDK9 的默认行为, 用于平滑迁移;
- `warn`: 与 `permit` 类似, 但每次非法访问都会发出警告, 而不是每个调用点只警告一次;
- `debug`: 在 `warn` 的基础上, 还会打印堆栈跟踪, 帮助定位问题来源;
- `deny`: 完全禁止非法反射访问, 直接抛出 `InaccessibleObjectException` 异常; 这是未来版本的默认行为.

从 JDK9 开始, `permit` 成为默认模式, 在 JDK16 中, `deny` 变成默认模式, 自 JDK17 开始, 该参数被完全移除.

## 2 修复非法访问

在 Java 模块中, 需要 open 包才能允许深度反射;

### 2.1 在 module-info 中

如果是代码作者, 可以在 module-info 文件中开放包:

```java
module baeldung.reflected {
    opens com.baeldung.reflected.opened;
}
```

为了谨慎, 可以使用限定开放:

```java
module baeldung.reflected {
    opens com.baeldung.reflected.internal to baeldung.intermedium;
}
```

也可以开放整个模块(内部不能再使用 open 指令):

```java
open module baeldung.reflected {
    // don't use opens directive
}
```

### 2.2 命令行方式

非代码作者可以通过添加 `-add-opens` 启动参数来实现:

```
--add-opens java.base/java.lang=baeldung.reflecting.named
```

如果要想所有的未命名模块添加权限, 则使用:

```
java --add-opens java.base/java.lang=ALL-UNNAMED
```

### 2.3 运行时操作

如果无法修改模块声明或命令行参数, 可以在运行时通过反射API来开放包. Java提供了`Module`类的`addOpens`方法来实现这一点:

```java
// 获取目标类所在的模块
Module targetModule = Class.forName("java.lang.String").getModule();
// 获取当前类所在的模块
Module currentModule = MyClass.class.getModule();
// 动态开放包
Method addOpensMethod = Module.class.getDeclaredMethod(
    "addOpens", String.class, Module.class);
addOpensMethod.setAccessible(true);
addOpensMethod.invoke(targetModule, "java.lang", currentModule);
```

在Java 9+中, 可以更直接地使用 `addOpens` 方法: 

```java
// 对于Java 9及以上版本
Module baseModule = Object.class.getModule();
baseModule.addOpens("java.lang", MyClass.class.getModule());
```

### 2.4 最佳实践

1. **逐步迁移**: 从`permit`开始，逐步修复警告，最终切换到`deny`模式;
2. **精确开放**: 只开放必要的包给必要的模块，避免使用`ALL-UNNAMED`;
3. **避免深度反射**: 尽量使用公共API，减少对反射的依赖;
4. **测试验证**: 在不同`--illegal-access`模式下充分测试.


