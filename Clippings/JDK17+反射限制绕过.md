---
title: JDK17+反射限制绕过
source: https://pankas.top/2023/12/05/jdk17-%E5%8F%8D%E5%B0%84%E9%99%90%E5%88%B6%E7%BB%95%E8%BF%87/
created: 2025-08-28
description: 本文详细介绍了JDK17及更高版本对Java核心API反射的强封装限制，并通过修改调用类的module属性使其与`Object`类的module一致，利用`sun.misc.Unsafe`类成功绕过这些限制的实践方法。
finished: "true"
tag: clipper
cover: https://www.loliapi.com/acg/pc/
updated: 2025-09-27 22:34:06
---
## 1	背景

JDK 17 启动了强封装, `java.*` 的非公共字段和方法都无法通过反射调用了, 但是 `sun.misc` 和 `sun.reflect` 包下的工具是可以被反射使用的, 可以利用其中的 Unsafe 来打破强封装的 module 限制.

## 2	原理

当我们给非 `public` 字段设置访问权限为 true 时, 会调用 `checkCanSetAccessible` 方法检查对应的类:

```java
@Override
@CallerSensitive
public void setAccessible(boolean flag) {
    AccessibleObject.checkPermission();
    if (flag) checkCanSetAccessible(Reflection.getCallerClass());
    setAccessible0(flag);
}
``` 

最终会调用到 `checkCanSetAccessible` 方法, 这个方法如果判断调用类的 module 和 `java.*` 下类的 module 相同, 就会赋予调用者修改权限:

```java
// java.lang.reflect.AccessibleObject#checkCanSetAccessible
private boolean checkCanSetAccessible(Class<?> caller,
                                          Class<?> declaringClass,
                                          boolean throwExceptionIfDenied) {
    if (caller == MethodHandle.class) {
        throw new IllegalCallerException();
    }

    Module callerModule = caller.getModule();
    Module declaringModule = declaringClass.getModule();

    if (callerModule == declaringModule) return true;
    // 判断 Module 是否相同
    if (callerModule == Object.class.getModule()) return true;
    if (!declaringModule.isNamed()) return true;

    // ......
}
```

根据这个原理, 我们可以通过 Unsafe 来修改当前类的 module 属性:

```java
// get Unsafe instance
Class unsafeClass = Class.forName("sun.misc.Unsafe");
Field field = unsafeClass.getDeclaredField("theUnsafe");
field.setAccessible(true);

// Set module
Unsafe unsafe = (Unsafe) field.get(null);
Module baseModule = Object.class.getModule();
Class currentClass = Main.class;
long addr = unsafe.objectFieldOffset(Class.class.getDeclaredField("module"));
unsafe.getAndSetObject(currentClass, addr, baseModule);
```