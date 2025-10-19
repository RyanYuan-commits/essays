---
type: Gradle
---
## 1	Graph Resolution

### 1.1	Fundamental Concepts

**Modules**: 模块是最小的发布单元, 而依赖项是对项目编译或者运行时需要的模块的引用, 比如下面的代码中, 模块是 `com.fasterxml.jackson.core:jackson-databind`;

```kotlin
dependencies {
    implementation("com.fasterxml.jackson.core:jackson-databind:2.17.2")
}
```

**Components**: 模块的每个版本被称为组件, `2.17.2` 是一个版本.;

**Metadata**: 元数据是 Component  的详细信息, 通常以 `ivy`, `pom` 或 `GMM` 的形式存在于组件的托管仓库中;

**Variants**: 代表应用所有可配置选项的交叉组合, 每个变体都对应一个独特, 可发布的应用版本;

**Attributes**: Gradle 使用属性来描述组件和变体的能力. 它们是键值对, 用于更精细地指定一个模块的特性或一个消费者所期望的特性. 通过匹配生产者和消费者之间的属性, Gradle 能够智能地选择最合适的变体, 从而解决依赖冲突并优化构建性能. 例如, 它可以区分不同操作系统的特定依赖项, 或者选择针对特定 JVM 版本编译的库.

## 2	Artifact Resolution

