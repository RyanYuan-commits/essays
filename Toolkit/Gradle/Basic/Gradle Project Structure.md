---
type: Java
sub-type: 工具
finished: "false"
created: 2025-09-27 22:34:06
updated: 2025-09-27 22:34:06
---
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

- `gradlew` 和 `gradlew.bat`: 均为 Gradle Wrapper 的脚本文件, 让开发者不必在电脑上安装 Gradle 就可以直接运行 Gradle 命令, 脚本会自动下载和配置项目所需要的 Gradle 版本, 确保所有的开发者都使用相同的构建环境;
	
- `setting.gradle(.kts)`: 项目的配置入口文件, 用于定义项目名称和声明子项目;
	
- `gradle/wrapper`
	
	- `gradle-wrapper.jar`: 包含 Gradle Wrapper 运行时代码, 当 gradlew 脚本被执行的时候, 会使用这个 JAR 文件来下载和运行正确的 Gradle 版本;
		
	- `gradle-wrapper.properties`: 配置文件, 用于定义 Gradle Wrapper 的行为, 比如指定要使用的 Gradle 版本和下载地址.
	
- `build.gradle(.kts)`: 定义了构建逻辑, 具体包含依赖项, 任务, 插件等.