---
type: Gradle
---
## 1	Consumer and Provider

![[Gradle Producer and Consumer.png|800]]

当构建一个库的时候, 我们扮演的是生产者的角色, creating artifacts that will be consumed by others, the consumers; 当依赖一个库的时候, 你扮演的是一个消费者的角色. Consumer 可以被宽泛的定义为: 依赖其他项目的项目以及声明对特定 artifacts 的依赖的配置.

## 2	Add a Dependency

在构建脚本 `build.gradle` 中使用 `dependencies{}` bolck 来添加依赖项, 允许你添加各种类型的依赖项, 如外部库, 本地 JAR 文件以及 multi-projects build 中的其他项目.

External Dependencies 通过 configuration name 声明 (eg. `implementation`, `compileOnly`), 后面跟着依赖的 notation, 例如:

```groovy
dependencies {
    // Configuration Name + Dependency Notation - GroupID : ArtifactID (Name) : Version
    configuration('<group>:<name>:<version>')
}
```

## 3	Types of Dependencies

### 3.1	Module Dependencies

依赖 Repository 中的某一个 Module:

```groovy
dependencies {
    implementation 'org.codehaus.groovy:groovy:3.0.5'
    implementation 'org.codehaus.groovy:groovy-json:3.0.5'
    implementation 'org.codehaus.groovy:groovy-nio:3.0.5'
}
```

### 3.2	Project Dependencies

项目依赖项允许你声明对同一 Build 中其他项目的依赖, 在多项目构建中很常见:

```groovy
dependencies {
    implementation project(':utils')
    implementation project(':api')
}
```

### 3.3	File Dependencies

用于告诉 Gradle 这个依赖项不在远程仓库或者本地缓存中, 而是在文件系统的某个特定路径, 常见的有: 不方便上传到私有仓库的 JAR, 本地开发的 JAR 等:

```groovy
dependencies {
    runtimeOnly files('libs/a.jar', 'libs/b.jar')
    runtimeOnly fileTree('libs') { include '*.jar' }
}
```

## 4	Listing Project Dependencies

Gradle 允许你通过  Command 来列出项目的依赖项, 这在进行依赖分析非常有帮助:

```sh
$ ./gradlew app:dependencies

> Task :app:dependencies

------------------------------------------------------------
Project ':app'
------------------------------------------------------------

implementation - Implementation dependencies for the 'main' feature. (n)
\--- com.google.guava:guava:30.0-jre (n)

runtimeClasspath - Runtime classpath of source set 'main'.
+--- com.google.guava:guava:30.0-jre
|    +--- com.google.guava:failureaccess:1.0.1
|    +--- com.google.guava:listenablefuture:9999.0-empty-to-avoid-conflict-with-guava
|    +--- com.google.code.findbugs:jsr305:3.0.2
|    +--- org.checkerframework:checker-qual:3.5.0
|    +--- com.google.errorprone:error_prone_annotations:2.3.4
|    \--- com.google.j2objc:j2objc-annotations:1.3
\--- org.apache.commons:commons-lang3:3.14.0

runtimeOnly - Runtime-only dependencies for the 'main' feature. (n)
\--- org.apache.commons:commons-lang3:3.14.0 (n)
```

