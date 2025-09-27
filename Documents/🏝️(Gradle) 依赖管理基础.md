---
type: 工具
sub-type: Gradle
finished: "false"
---
## 1 常见术语

### 1.1 Artifact

构建生成的文件或目录, 如 JAR 包, ZIP 分发包或者可执行文件;

攻坚通常设计为用户或者其他项目使用或部署到托管系统, 项目间依赖目录更加常见.

### 1.2 Capability

标准的 Gradle 依赖冲突解决, 是基于 GAV 坐标的, 只在 Group 和 Name 完全相同的情况下才能识别出冲突;

但是实际开发中, 可能会出现多个不同的库提供了相同的功能, 它们互相排斥, 不应该存在于相同的 ClassPath 中;

比如最典型的日志框架: `log4j:log4j`, `org.slf4j:slf4j-log4j12`, `ch.qos.logback:logback-classic`, 这三个库的 GAV 完全不同, 但是它们都试图提供一个 Log4j 的实现.

Gradle Capacity 允许为依赖项声明一个 "虚拟的, 可冲突的组件": 

```groovy
// 假设这是我们无法修改的两个库
dependencies {
    implementation 'com.example:lib-a:1.0' // 内部依赖了 log4j:log4j:1.2.17
    implementation 'com.example:lib-b:1.0' // 内部依赖了 org.slf4j:slf4j-log4j12:1.7.30
}

// --- 以下是我们的解决方案 ---

// 1. 声明哪个库具备何种“能力”
dependencies {
    // 我们可以通过组件元数据规则来为外部依赖添加能力声明
    components.withModule('log4j:log4j') {
        // 声明 log4j:log4j 具备 "log4j-impl" 这个能力，版本为 1.2.17
        capabilities {
            requireCapability('com.my-company.capabilities:log4j-impl:1.2.17')
        }
    }
    components.withModule('org.slf4j:slf4j-log4j12') {
        // 声明 slf4j-log4j12 具备同样的能力，但版本更高
        capabilities {
            requireCapability('com.my-company.capabilities:log4j-impl:1.7.30')
        }
    }
}

// 2. 解决冲突：告诉 Gradle 在所有提供 "log4j-impl" 能力的库中，我们信任哪个
configurations.all {
    resolutionStrategy {
        capabilities {
            // 当遇到 'log4j-impl' 能力的冲突时，
            // 选择 'org.slf4j:slf4j-log4j12' 作为获胜者
            select('com.my-company.capabilities:log4j-impl').select {
                // 'candidates' 是所有提供该能力的库列表
                // 我们选择版本号最高的那个
                def highestVersionCandidate = candidates.max { it.version }
                // 或者直接指定胜利者
                // def winner = candidates.find { it.id.module == 'slf4j-log4j12' }
                highestVersionCandidate
            }
        }
    }
} 
```

首先为两个不同的库赋予了虚拟能力, 然后 Gradle 在识别依赖的时候就可以识别出冲突;

然后我们可以定义一个规则, 来指定出现冲突的时候选择版本的方式.

