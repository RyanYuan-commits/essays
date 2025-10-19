---
type: Gradle
---
## 1	Graph Resolution

### 1.1	Fundamental Concepts

**Modules**: 模块是最小的发布单元, 而依赖项是对项目编译或者运行时需要的模块的引用, 比如下面的代码中, 模块是 `com.fasterxml.jackson.core:jackson-databind`

```kotlin
dependencies {
    implementation("com.fasterxml.jackson.core:jackson-databind:2.17.2")
}
```

**Components**: 模块的每个版本被称为组件, `2.17.2` 是一个版本.

**Metadata**: 元数据是 Component  的详细信息, 通常以 `ivy`, `pom` 或 `GMM` 的形式存在于组件的托管仓库中

## 2	Artifact Resolution

