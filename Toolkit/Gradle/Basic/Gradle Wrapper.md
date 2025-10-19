---
type: Rust
sub-type: Basic Knowledge
---
Gradle包装器, 是标准化的在项目中集成和执行 Gradle 构建的方式; 允许新开发者通过 `./gradlew` 命令来完成项目的初始化, Wrapper 会搞定 Gradle 的下载和配置.

一个包含 Gradle Wrapper 的项目通常有以下文件：

- `gradlew`: 用于 macOS/Linux 的 shell 脚本;
- `gradlew.bat`: 用于 Windows 的批处理脚本;
- `gradle/wrapper/gradle-wrapper.jar`: Wrapper 的核心逻辑，负责下载 Gradle;
- `gradle/wrapper/gradle-wrapper.properties`: 关键配置文件，定义了要下载的 Gradle 版本和下载地址.

---

```bash
# 使用 Gradle 为当前项目生成 Wrapper 文件
gradle wrapper

# 修改 Wrapper 的版本, 会更新 gradle-wrapper.properties 文件
./gradlew wrapper --gradle-version [VERSION]
```