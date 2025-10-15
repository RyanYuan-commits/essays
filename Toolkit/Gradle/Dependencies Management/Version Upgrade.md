---
type: 工具
sub-type: Gradle
created: 2025-09-27 22:34:06
updated: 2025-09-27 22:34:06
---
## 1 版本配置

本地设置好 JDK 版本;

升级 Gradle 的版本到 7.3;

idea 中配置 Gradle 的运行 JDK 为 JDK17;

设置 Java 版本为 17;

## 2 VM 参数

本地配置 add-opens 参数:

```bash
--add-opens=java.base/jdk.internal.reflect=ALL-UNNAMED --add-opens=java.base/java.text=ALL-UNNAMED --add-opens=java.base/java.util=ALL-UNNAMED --add-opens=java.base/java.lang=ALL-UNNAMED --add-opens=java.base/jdk.internal.ref=ALL-UNNAMED --add-opens=java.base/java.net=ALL-UNNAMED --add-opens=java.base/java.nio=ALL-UNNAMED --add-opens=java.base/java.io=ALL-UNNAMED --add-opens=java.base/java.math=ALL-UNNAMED --add-opens=java.base/java.lang.reflect=ALL-UNNAMED --add-opens=java.base/jdk.internal.loader=ALL-UNNAMED --add-opens=java.base/sun.net.util=ALL-UNNAMED --add-opens=java.base/sun.net.www=ALL-UNNAMED --add-opens=java.security.jgss/sun.security.krb5=ALL-UNNAMED --add-opens=java.base/java.util.concurrent.atomic=ALL-UNNAMED --add-opens=java.base/java.time=ALL-UNNAMED --add-opens=java.base/java.util.concurrent=ALL-UNNAMED
```

## 3 build.gradle 配置

### 3.1 设置项目的编译版本

```groovy
java {
    sourceCompatibility = JavaVersion.VERSION_17
    targetCompatibility = JavaVersion.VERSION_17
}
```

### 3.2 使用新版依赖管理 API
| 旧方法         | 新方法                |
| ----------- | ------------------ |
| compile     | api                |
| testCompile | testImplementation |

### 3.3 使用新版的懒加载 API

将 `task.xx() {...}` 变为 `task.xx().configureEach {...}`

在 Gradle 5.0 之前, Gradle 的工作方式是一切皆急切的;

配置阶段中, Gradle 在读取你的 `build.gradle` 时, 它会立即, 无条件的创建并配置遇到的每个任务.

```groovy
tasks.withType(JavaCompile) {
    // 这段代码会在“配置阶段”为项目中每一个 JavaCompile 任务执行一遍
    println "Eagerly configuring task: ${name}" 
    options.encoding = 'UTF-8'
}
```

假设项目中有 `complieJava` 和 `compileTestJava` 两个任务, 在 **配置阶段**, Gradle 会立即打印:

```
Eagerly configuring task: compileJava
Eagerly configuring task: compileTestJava
```

然后进入 **执行阶段** 才真正的去执行这些任务.

---

但是问题在于, 当运行一个完全不相关的命令, 比如 `gradle tasks` 时, 为了展示所有的任务列表, Gradle 仍然需要经过配置阶段, 因此, 还是会执行配置动作, 即使 `JavaCompile` 类型的任务根本不会被执行;

为了解决这个问题, 从 Gradle5.0 开始, 引入了一整套懒加载机制, 也就是配置动作在任务真正被执行的时候才会调用.

使用新版 API 重写上面的案例:

```groovy
tasks.withType(JavaCompile).configureEach {
    // 这段代码只有在某个 JavaCompile 任务即将被执行时，才会运行
    println "Lazily configuring task: ${name}" 
    options.encoding = 'UTF-8'
}
```

`configureEach` 的意思是: 在 (需要时) 配置每一个, 将配置代码块变为了一个延迟执行的动作.