
=> [Gradle官网](https://link.juejin.cn?target=https%3A%2F%2Fdocs.gradle.org%2Fcurrent%2Fuserguide%2Fuserguide.html "https://docs.gradle.org/current/userguide/userguide.html")
## 1 什么是 Gradle

Gradle 是一个自动化的构建工具, 是基于 Apache Ant 和 Apache Maven 的概念, 提供了更为灵活和强大的功能, 与 Maven 相比, Gradle 使用了一种基于 Groovy 的 特定领域语言 DSL 来声明项目设置, 这使得构建脚本更为简洁和易读, 同时, Gradle 也支持基于 Kotlin 语言的 Kotlin DSL, 提供了更多的选择.

Gradle 的主要特点和优势包括: 

- 灵活性: Gradle 允许你使用任何你喜欢的构建脚本语言, 并提供了强大的自定义任务类型的功能.
- 性能: Gradle 具有出色的构建性能, 特别是对于那些大型项目.它支持构建缓存、守护进程和并行执行等特性, 这些都有助于提高构建速度.
- 依赖管理: Gradle 提供了强大的依赖管理功能, 可以很容易地解决依赖冲突和版本问题.
- 多项目构建: Gradle 可以轻松地管理多个相关的项目, 使得构建过程更为高效.
- 插件系统: Gradle 的插件系统允许你轻松地扩展构建过程的功能, 以适应不同的项目需求.

## 2 常见使用方式

### 2.1 运行与打包

```bash
# Gradle 打包命令
gradle build # 生成的 jar: build/libs/xxx.jar
```

默认是普通 jar, 不能直接通过 `java -jar` 运行, 如果要生成 fat jar, 需要使用 Shadow 插件:

```groovy
plugins {
    id 'java'
    id 'application'
    id 'com.github.johnrengelman.shadow' version '8.1.1'
}

application {
    mainClass = 'com.example.App'
}
```

```bash
# 打包命令修改为
gradle shadowJar
```

### 2.2 查看依赖树

```bash
gradle dependencies
```

```sql
implementation - Implementation only dependencies for main.
\--- com.google.guava:guava:33.0.0-jre
     +--- ...
```

## 3 Gradle 插件

### 3.1 Gradle Plugins

Gradle 插件是一组用于拓展和定制 Gradle 构建功能的代码.

Gradle 本身是一个通用的构建工具, 它知道如何管理任务, 处理依赖和执行构建脚本; 但它并不知道如何编译 Java 代码, 打包 Web 应用或运行单元测试.

插件可以提供一些列预定义的 **任务**, **属性** 和 **配置**, 来协助完成特定类型的项目构建, 例如, 应用 `java` 插件后, 项目就自动拥有了 `compileJava`, `test`, `jar` 等任务.

### 3.2 核心功能

**核心作用**: 拓展 Gradle 的能力, 将 Gradle 从一个通用的任务执行器, 转变为一个能够处理特定语言, 特定框架, 特定打包格式的专业工具.

**添加任务**: 给项目注入一系列新的任务, 这些任务是完成特定构建步骤的最小单元.

**配置项目**: 插件根据项目的类型, 自动配置合理的默认值和约定, 来减少手动的配置.

- `java` 插件: 默认将 `src/main/java` 作为源代码目录, `src/test/java` 作为测试代码目录.
- `war` 插件: 默认将 `src/main/webapp` 作为 Web 资源目录.

**提供 DSL 拓展**: 插件可以在 `build.gradle` 文件中引入新的配置块和属性, 让用户通过声明式的方式来配置复杂的功能.

- `java` 插件: 提供了 `java { ... }` 配置块, 可以在其中设置 `sourceCompatibility` 等.
- `spring-boot` 插件: 提供了 `boorJar { ... }` 等配置块.

