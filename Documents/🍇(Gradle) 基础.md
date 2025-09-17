---
type: Java
sub-type: 工具
finished: "false"
---
```embed
title: "Gradle"
image: "images/gradle-head.png"
url: "https://gradle.org/"
favicon: ""
```

## 1 认识 Gradle

### 1.1 什么是 Gradle

一个**自动化构建工具**;
### 1.2 核心概念

**项目**: 是一个可被构建的软件单元, 由一个根项目和若干个子项目构成.

**构建脚本**: 指定 Gradle 构建的步骤, 每个项目可以包含一个或多个构建脚本.

**依赖与依赖管理**: 依赖管理是一种自动化技术, 用于声明和解析项目所需的外部资源. 每个项目通常包含若干依赖项, Gradle 会在构建过程中解析这些依赖.

**任务**: 工作的基本单元, 如执行代码, 编译等, 在构建脚本和插件中定义.

**插件**: 用于拓展 Gradle 的功能, 可以按需引入插件和执行插件提供的任务.
### 1.3 项目结构

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

