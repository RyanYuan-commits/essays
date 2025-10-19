---
type: Gradle
sub-type: Dependency Management
---
Gradle 中的每一个依赖项都有自己适用的范围, 某些依赖项用于源代码编译, 某些依赖项只需在运行时使用:

```groovy
dependencies {
    implementation("com.google.guava:guava:30.0-jre")   // Needed to compile and run the app
    runtimeOnly("org.slf4j:slf4j-simple:2.0.13")        // Only needed at runtime
}
```

Dependency Configurations 用于定义项目中的依赖集, 依赖集用于描述依赖在构建中的各个阶段如何以及何时使用,

## 1	Dependency Configurations

Gradle 借助 [Configuration](https://docs.gradle.org.cn/current/dsl/org.gradle.api.artifacts.Configuration.html) 类型来表示依赖项的范围; Configuration 是 [FileCollection](https://docs.gradle.org/current/javadoc/org/gradle/api/file/FileCollection.html) 的实例, 是 Dependencies 的容器, 但是不包含 Artifacts, Configuration 有以下三个职责:

- 管理依赖的范围, 定义包含的依赖项在何时何地被使用;
	
- 管理依赖的集合, 作为一个容器, 收集所有声明给它的依赖项;
	
- 控制依赖的传递性.

许多 Gradle Plugins 会向项目中添加预定义的 Configuration, 例如 [[Java Library Plugin]] 中定义了如 `api`, `implementation`, `compileOnly` 等 Configuration Name.

| **配置名称**                 | **描述**                           | **用途**    |
| -------------------- | ---------------------------- | ----- |
| `api`                | 编译和运行时都需要的依赖项，并包含在发布的 API 中。 | 声明依赖项 |
| `implementation`     | 编译和运行时都需要的依赖项。               | 声明依赖项 |
| `compileOnly`        | 仅编译需要的依赖项，不包含在运行时或发布中。       | 声明依赖项 |
| `compileOnlyApi`     | 仅编译需要的依赖项，但包含在发布的 API 中。     | 声明依赖项 |
| `runtimeOnly`        | 仅运行时需要的依赖项，不包含在编译类路径中。       | 声明依赖项 |
| `testImplementation` | 编译和运行测试所需的依赖项。               | 声明依赖项 |
| `testCompileOnly`    | 仅测试编译所需的依赖项。                 | 声明依赖项 |
| `testRuntimeOnly`    | 仅运行测试所需的依赖项。                 | 声明依赖项 |
## 2	View Configurations

Gradle 的 dependencies Task 提供了对项目依赖项的概述, 可以通过 `--configuration` 参数来制定分析的 Configuration Name:

```sh
$ ./gradlew -q app:dependencies --configuration implementation

------------------------------------------------------------
Project ':app'
------------------------------------------------------------

implementation - Implementation only dependencies for source set 'main'.
\--- com.google.guava:guava:30.0-jre
```

## 3	Centralizing Dependencies

在 Gradle 中, 可以使用 Platforms 和 Version Catelogs 技术来中心化管理依赖.

### 3.1	Using Platforms

Platform 是一组依赖约束, 旨在管理 Library 或 Application 的传递依赖项, 当你在定义 Platform 时, 你实际上是在指定一组旨在**协同**使用的依赖项, 确保兼容性并简化依赖项管理.

```groovy
// platform/build.gradle
plugins {
    id("java-platform")
}

dependencies {
    constraints {
        api("org.apache.commons:commons-lang3:3.12.0")
        api("com.google.guava:guava:30.1.1-jre")
        api("org.slf4j:slf4j-api:1.7.30")
    }
}
```

然后, 你可以在项目中使用定义好的这个 Platform:

```groovy
plugins {
    id("java-library")
}

dependencies {
    implementation(platform(":platform"))
    implementation 'com.fasterxml.jackson:jackson-databind' 
    implementation 'org.apache.commons:commons-lang3'
}
```

Maven 的 BOM 在 Gradle 中作为一个 Platform 被支持, 如 SpringBoot 的 BOM 清单:

```groovy
dependencies {
    // import a BOM
    implementation platform('org.springframework.boot:spring-boot-dependencies:1.5.8.RELEASE')
    // define dependencies without versions
    implementation 'com.google.code.gson:gson'
    implementation 'dom4j:dom4j'
}
```

### 3.2	Using Version Catelogs

Version Catelogs 是依赖坐标的集中列表, 可以在多个项目中引用

