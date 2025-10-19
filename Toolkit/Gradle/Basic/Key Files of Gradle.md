---
type: Gradle
sub-type: Basic Knowledge
---
一个标准的 Gradle 项目由这几个核心文件定义:

- [[Gradle Wrapper]]: 这是执行 Gradle 构建的唯一推荐方式, 是一个随项目源码一起提交到版本控制系统的 Script, 当你运行它时, 它会自动下载项目配置的 Gradle 版本, 保证了项目构建环境的封闭性.
	
- [[Gradle Setting File]]: 文件名为 `setting.gradle`, 位于项目根目录, 是 Gradle 构建的**入口点**. 核心职责是在多项目构建中定义项目结构, 通过 `include()` 方法声明哪些 Subjectprojects 需要被包含到构建中; 此外, 它也是配置项目仓库源的中心位置.
	
- [[Gradle Build Script]]: 文件名为 `build.gradle`, 根项目和子项目都有自己的构建脚本, 这个文件定义了项目应该如何被构建, 你将在这里应用插件, 声明依赖, 配置任务以及编写自己的构建逻辑.
