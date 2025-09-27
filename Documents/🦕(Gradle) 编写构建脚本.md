---
type: 工具
sub-type: Gradle
finished: "false"
---

![[Gradle构建.png|800]]
## 1 多项目构建的结构化

包含三个子项目的 Gradle 项目目录结构:

```
.
├── gradlew
├── gradlew.bat
├── settings.gradle(.kts)
├── sub-project-1
│   └── build.gradle(.kts)
├── sub-project-2
│   └── build.gradle(.kts)   
└── sub-project-3
    └── build.gradle(.kts)
```

其 setting.gradle 应该为:

```groovy
include('sub-project-1', 'sub-project-2', 'sub-project-3')
```

Gradle 社区对多项目构建结构有两种约定标准:

- [Multi-Project Builds using buildSrc](https://docs.gradle.org/current/userguide/sharing_build_logic_between_subprojects.html#sec:using_buildsrc)
- [Composite Builds including build-logic](https://docs.gradle.org/current/userguide/composite_builds.html#composite_builds)

在多项目中, 如果在根目录使用 `gradle test`, 会沿着项目层级执行所有具有此名称的任务, 如果在便利的任何子项目中没有具有此名称的任务, 则会报错;

而如果想要执行某一个具体的子项目的任务, 则需要通过 `gradle :sub-project-1:build` 来指定.

## 2 构建生命周期

![[Gradle Build Lifecycle.png|700]]

### 2.1 初始化阶段

在初始化阶段, Gradle 会检测参与构建的项目集合以及其包含的构建; 

Gradle 首先评估 `setting.gradle` 文件, 并实例化一个 `Setting` 对象;

然后 Gradle 会为构建中包含的每个项目实例化 `Project` 实例.

### 2.2 配置阶段

在配置阶段, Gradle 会向初始化阶段找到的项目添加任务和其他属性; 然后通过理解任务之间的依赖关系来构建任务图.

![[Gradle task graph.png|700]]

例如, 如果项目中包含了 `buildHtml`, `assembleDocs` 和 `createDocs` 并且它们依赖于前者, 则够看出来的任务图为:

```
buildHtml → assembleDocs → createDocs
```

任务图是一个有向无环图.
### 2.3 执行阶段

使用配置阶段生成的任务图来执行任务, 可以并行执行.

## 3 编写构建脚本

Gradle 会在配置阶段使用构建脚本来配置每个 `Project` 对象.
### 3.1 脚本结构

Gradle 脚本主要由两种元素构成:

- Statement 语句: 在初始化阶段或配置阶段立即执行的顶层表达式;
- Blocks 代码块: 传递给配置方法的嵌套代码块 (Groovy 闭包或 Kotlin Lambda), 这些代码块将设置并应用于 Gradle 对象, 例如 `project`, `pluginManagement`, `dependencyResolutionManagement`, `repositories` 或者 `dependencies`.

Gradle 脚本基于 Groovy 的动态闭包实现, Gradle 会将方法/属性调用动态委托给目标对象. 
=> [[🌪️(Groovy) 闭包]]

```groovy
buildscript {
    repositories {
        maven { url "xxx" }
    }
}
```

调用 `Project#buildscript()` 函数, 传入的是一个闭包 `{ repositories { ... } }`;

这个闭包会委托给 `RepositoryHandler` 实例, 执行 `maven` 方法;

`maven` 方法接收一个闭包, 委托给 `MavenArtifactRepository` 实例, 来执行 `url("xxx")` 方法.

### 3.2 变量

**局部变量**: 使用 `def` 关键字声明局部变量, 局部变量仅在声明它的作用域内可见.

```groovy
def dest = 'dest'

tasks.register('copy', Copy) {
    from 'source'
    into dest
}
```

---

**额外属性**: Gradle 为增强对象, 提供了对于存储用户自定义数据的额外属性;

```groovy
ext {
    springVersion = "3.1.0.RELEASE"
    emailNotification = "build@master.org"
}
```

额外属性依附于其所属对象, 如 `Project`, 与局部变量不同, 额外属性具有更广阔的作用域, 可以在所属对象可见的任何位置访问它们, 包括从子项目访问其父项目的属性.

### 3.3 案例

```groovy
plugins {   
    id 'application'
}

repositories {  
    mavenCentral()
}

dependencies {  
    testImplementation 'org.junit.jupiter:junit-jupiter-engine:5.9.3'
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
    implementation 'com.google.guava:guava:32.1.1-jre'
}

application {   
    mainClass = 'com.example.Main'
}

tasks.named('test', Test) { 
    useJUnitPlatform()
}
```

Gradle 脚本基于 Groovy 的动态闭包实现, Gradle 会将方法/属性调用动态委托给目标对象. 
=> [[🌪️(Groovy) 闭包]]














