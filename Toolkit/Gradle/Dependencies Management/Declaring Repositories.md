---
type: Gradle
sub-type: Dependency Management
---
Gradle 需要知道在哪里可以下载项目中使用的依赖项, 可以通过构建文件的 `repositories` 块来添加任意数量的仓库, 如果一个依赖在列出的多个仓库中均可用, Gradle 会选择第一个.

## 1	Declaring a Public Repositories

构建软件的组织可能希望利用公共二进制仓库来下载和使用开源依赖项, 流行的公共仓库有 [Maven Central](https://repo.maven.apache.org/maven2/) 和 [Google Android](https://maven.google.com/)  等, Gradle 为这些广泛使用的仓库提供了内置的简写符号

```groovy
repositories {
    mavenCentral()
    google()
    gradlePluginPortal()
}
```

在底层, Gradle 通过简写符号解析公共仓库的相应 URL 来解析依赖项. 所有简写符号都通过 [RepositoryHandler](https://docs.gradle.org.cn/current/dsl/org.gradle.api.artifacts.dsl.RepositoryHandler.html) API 提供.

## 2	Declaring a Private or Custom Repositories

```groovy
repositories {
    maven {
        url = uri("https://maven-central.storage.apis.com")
    }
    ivy {
        url = uri("https://github.com/ivy-rep/")
    }
}
```

## 3	Declaring a Local Repositories

```groovy
repositories {
    mavenCentral()
    google()
    gradlePluginPortal()
}
```