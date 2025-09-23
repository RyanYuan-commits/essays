---
type: Java
sub-type: 工具
finished: "false"
---
## 1 核心概念

**项目**: 是一个可被构建的软件单元, 由一个根项目和若干个子项目构成.

**构建脚本**: 指定 Gradle 构建的步骤, 每个项目可以包含一个或多个构建脚本.

**依赖与依赖管理**: 依赖管理是一种自动化技术, 用于声明和解析项目所需的外部资源. 每个项目通常包含若干依赖项, Gradle 会在构建过程中解析这些依赖.

**任务**: 工作的基本单元, 如执行代码, 编译等, 在构建脚本和插件中定义.

**插件**: 用于拓展 Gradle 的功能, 可以按需引入插件和执行插件提供的任务.
## 2 项目结构

```text
project
├── gradle                          
│   ├── libs.versions.toml              
│   └── wrapper
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── gradlew                       
├── gradlew.bat                         
├── settings.gradle(.kts) 用于定义项目名称和子项目        
├── subproject-a
│   ├── build.gradle(.kts)              
│   └── src                         
└── subproject-b
    ├── build.gradle(.kts) 构建脚本     
    └── src           
```

- gradlew 和 gradlew.bat: 均为 Gradle Wrapper 的脚本文件, 让开发者不必在电脑上安装 Gradle 就可以直接运行 Gradle 命令, 脚本会自动下载和配置项目所需要的 Gradle 版本, 确保所有的开发者都使用相同的构建环境;
- setting.gradle(.kts): 项目的配置入口文件, 用于定义项目名称和声明子项目;
- gradle/wrapper
	- gradle-wrapper.jar: 包含 Gradle Wrapper 运行时代码, 当 gradlew 脚本被执行的时候, 会使用这个 JAR 文件来下载和运行正确的 Gradle 版本;
	- gradle-wrapper.properties: 配置文件, 用于定义 Gradle Wrapper 的行为, 比如指定要使用的 Gradle 版本和下载地址.
- build.gradle(.kts): 定义了构建逻辑, 具体包含依赖项, 任务, 插件等.




