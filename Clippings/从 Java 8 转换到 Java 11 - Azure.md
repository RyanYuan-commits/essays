---
title: 从 Java 8 转换到 Java 11 - Azure
source: https://learn.microsoft.com/zh-cn/java/openjdk/transition-from-java-8-to-java-11
created: 2025-08-29
description: 本文档提供了将 Java 应用程序从 Java 8 迁移到 Java 11 的实用指南，涵盖了使用工具识别和解决潜在问题，以及在 Azure 环境中优化性能和兼容性的建议。
finished: "false"
tag: clipper
updated: 2025-09-27 22:34:06
---
两个用于探查潜在问题的工具:

- jdeprscan: 可查看是否使用了已弃用或已删除的 API
- jdeps: 探索类使用了哪个内部 API, 可以在当前 JDK 继续使用哪个 API

|工具|Gradle 插件|Maven 插件|
|---|---|---|
|jdeps|[jdeps-gradle-plugin](https://github.com/kordamp/jdeps-gradle-plugin)|[Apache Maven JDeps 插件](https://maven.apache.org/plugins/maven-jdeps-plugin/index.html)|
|jdeprscan|[jdeprscan-gradle-plugin](https://github.com/kordamp/jdeprscan-gradle-plugin)|[Apache Maven JDeprScan 插件](https://maven.apache.org/plugins/maven-jdeprscan-plugin/index.html)|

## 1	使用 jdeprscan

```bash
jdeprscan --release 11 my-application.jar
```

调用案例:

```bash
jdeprscan --release 11 --class-path log4j-api-2.13.0.jar my-application.jar
error: cannot find class sun/misc/BASE64Encoder
class com/company/Util uses deprecated method java/lang/Double::<init>(D)V
```

这个输出告诉我们 `com.company.Util` 正在调用废弃的 API.

## 2	使用 jdeps

```bash
jdeps --jdk-internals --multi-release 11 --class-path log4j-core-2.13.0.jar my-application.jar

Util.class -> JDK removed internal API
Util.class -> jdk.base
Util.class -> jdk.unsupported
   com.company.Util        -> sun.misc.BASE64Encoder        JDK internal API (JDK removed internal API)
   com.company.Util        -> sun.misc.Unsafe               JDK internal API (jdk.unsupported)
   com.company.Util        -> sun.nio.ch.Util               JDK internal API (java.base)

Warning: JDK internal APIs are unsupported and private to JDK implementation that are
subject to be removed or changed incompatibly and could break your application.
Please modify your code to eliminate dependence on any JDK internal APIs.
For the most recent update on JDK internal API replacements, please check:
https://wiki.openjdk.java.net/display/JDK8/Java+Dependency+Analysis+Tool

JDK Internal API                         Suggested Replacement
----------------                         ---------------------
sun.misc.BASE64Encoder                   Use java.util.Base64 @since 1.8
sun.misc.Unsafe                          See http://openjdk.java.net/jeps/260
```

可以参考上面的输出来消除对 JDK 内部 API 的引用.

模块 jdk.unsupport 中的 API 也建议消除, 

```embed
title: "JEP 260: Encapsulate Most Internal APIs"
image: "https://openjdk.org/images/openjdk2.svg"
description: ""
url: "https://openjdk.org/jeps/260"
favicon: ""
aspectRatio: "27.12871287128713"
```
