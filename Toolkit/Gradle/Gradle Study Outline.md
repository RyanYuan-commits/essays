 ## 01-核心基础

### Gradle 与构建工具演进

- Java 构建工具的演进历史
    
- Apache Ant 的特点与局限
    
- Apache Maven 的约定优于配置
    
- Gradle 的灵活性与性能优势
    
- Ant-Maven-Gradle 核心区别对比
    

### Gradle 项目核心构件

- 项目 (Project) 与子项目 (Subprojects)
    
- 任务 (Task)
    
- 构建脚本 (Build Script)
    
- 设置脚本 (Settings Script)
    
- 插件 (Plugin)
    
- 依赖 (Dependency)
    
- Gradle 包装器 (Gradle Wrapper)
    

### Gradle 构建生命周期

- 初始化阶段 (Initialization Phase)
    
- 配置阶段 (Configuration Phase)
    
- 执行阶段 (Execution Phase)
    
- 配置与执行分离的核心思想
    

### 第一个 Gradle Java 项目

- 使用 SDKMAN 安装 Gradle
    
- 通过 gradle init 初始化项目
    
- 标准的 Gradle 项目结构解析
    
- 运行核心任务 (build, run, test, clean)
    

## 02-中级应用与实践

### 构建脚本 DSL 对比

- Groovy DSL 的动态性与优缺点
    
- Kotlin DSL 的静态类型与优缺点
    
- 现代项目 DSL 的选择建议
    

### 高效依赖管理

- implementation 与 api 配置的根本区别
    
- implementation 实现封装与加速构建
    
- api 暴露公共 API 依赖
    
- 依赖配置最佳实践
    

### 其他重要依赖配置

- compileOnly 依赖
    
- runtimeOnly 依赖
    
- 测试范围的依赖配置
    

### Java 核心插件详解

- java 插件的用途
    
- java-library 插件的用途
    
- 如何根据模块角色选择插件
    

### 核心任务与生命周期

- 插件提供的核心任务 (compileJava, jar, test)
    
- 核心任务与生命周期任务的关联 (assemble, check, build)
    

### 多模块项目结构化

- 多模块项目的标准目录结构
    
- settings.gradle.kts 定义项目组成
    
- 使用 project() 声明项目间依赖
    
- 模块化对增量构建与并行执行的价值
    

## 03-高级定制与优化

### 编写自定义任务

- 创建继承 DefaultTask 的任务类
    
- 使用注解声明任务的输入与输出
    
- 编写任务动作 (@TaskAction)
    
- 在构建脚本中注册和配置任务
    

### 封装与复用构建逻辑

- 使用 buildSrc 在项目内共享逻辑
    
- 创建独立项目实现跨项目插件共享
    
- 编写可发布的自定义插件
    

### 可扩展的依赖管理

- 使用版本目录 (Version Catalogs) 集中管理依赖
    
- 创建和使用 libs.versions.toml
    
- 使用物料清单 (BOM) 统一依赖版本
    
- 结合使用版本目录和 BOM
    

### 核心性能优化技术

- 增量构建 (Incremental Build)
    
- 构建缓存 (Build Cache) 本地与远程
    
- 配置缓存 (Configuration Cache)
    
- 并行执行 (Parallel Execution)
    
- Gradle Daemon
    
- 使用构建扫描 (Build Scan) 分析性能
    

## 04-专家级实践模式

### 现代化项目最佳实践

- 始终使用 Gradle Wrapper
    
- 优先采用 Kotlin DSL
    
- 使用版本目录管理依赖
    
- 在 properties 文件中统一配置
    
- 避免滥用 clean 命令
    
- 使用 plugins DSL 块应用插件
    
- 保持 Gradle 与插件更新
    
- 始终包含 settings 文件
    

### 约定插件 (Convention Plugins)

- 约定插件的定义与作用
    
- 将构建逻辑从配置变为声明
    
- 实现构建逻辑的集中管理与一致性
    
- 提升构建逻辑的可测试性

翻译成英文


---


## 01 - Core Fundamentals

### Gradle and the Evolution of Build Tools

- History of Java Build Tool Evolution
    
- Features and Limitations of Apache Ant
    
- Apache Maven's "Convention over Configuration"
    
- Gradle's Flexibility and Performance Advantages
    
- Core Differences Comparison: Ant vs. Maven vs. Gradle
    

### Core Components of a Gradle Project

- Project and Subprojects
    
- Task
    
- Build Script
    
- Settings Script
    
- Plugin
    
- Dependency
    
- Gradle Wrapper
    

### The Gradle Build Lifecycle

- Initialization Phase
    
- Configuration Phase
    
- Execution Phase
    
- Core Concept: Separation of Configuration and Execution
    

### Your First Gradle Java Project

- Installing Gradle using SDKMAN
    
- Initializing a project with `gradle init`
    
- Understanding the Standard Gradle Project Structure
    
- Running Core Tasks (build, run, test, clean)
    

## 02 - Intermediate Application and Practices

### Build Script DSL Comparison

- Groovy DSL: Dynamics, Pros, and Cons
    
- Kotlin DSL: Static Typing, Pros, and Cons
    
- Recommendations for Choosing a DSL in Modern Projects
    

### Effective Dependency Management

- The Fundamental Difference Between `implementation` and `api` Configurations
    
- `implementation`: Encapsulation and Faster Builds
    
- `api`: Exposing Public API Dependencies
    
- Dependency Configuration Best Practices
    

### Other Important Dependency Configurations

- `compileOnly` Dependencies
    
- `runtimeOnly` Dependencies
    
- Test Scope Dependency Configurations
    

### Core Java Plugins Explained

- Purpose of the `java` Plugin
    
- Purpose of the `java-library` Plugin
    
- How to Choose Plugins Based on Module Roles
    

### Core Tasks and the Lifecycle

- Core Tasks Provided by Plugins (compileJava, jar, test)
    
- Relationship Between Core Tasks and Lifecycle Tasks (assemble, check, build)
    

### Structuring Multi-Module Projects

- Standard Directory Structure for Multi-Module Projects
    
- Defining Project Composition with `settings.gradle.kts`
    
- Declaring Inter-Project Dependencies using `project()`
    
- The Value of Modularity for Incremental Builds and Parallel Execution
    

## 03 - Advanced Customization and Optimization

### Writing Custom Tasks

- Creating a Task Class Inheriting from `DefaultTask`
    
- Declaring Task Inputs and Outputs using Annotations
    
- Writing a Task Action (`@TaskAction`)
    
- Registering and Configuring Tasks in the Build Script
    

### Encapsulating and Reusing Build Logic

- Using `buildSrc` to Share Logic Within a Project
    
- Creating a Separate Project for Cross-Project Plugin Sharing
    
- Writing a Publishable Custom Plugin
    

### Scalable Dependency Management

- Centralizing Dependencies with Version Catalogs
    
- Creating and Using `libs.versions.toml`
    
- Unifying Dependency Versions with a Bill of Materials (BOM)
    
- Using Version Catalogs and BOMs Together
    

### Core Performance Optimization Techniques

- Incremental Build
    
- Build Cache (Local and Remote)
    
- Configuration Cache
    
- Parallel Execution
    
- Gradle Daemon
    
- Analyzing Performance with Build Scans
    

## 04 - Expert-Level Practices and Patterns

### Modern Project Best Practices

- Always Use the Gradle Wrapper
    
- Prefer the Kotlin DSL
    
- Use Version Catalogs for Dependency Management
    
- Centralize Configuration in `properties` Files
    
- Avoid Overusing the `clean` Task
    
- Apply Plugins Using the `plugins` DSL Block
    
- Keep Gradle and Plugins Updated
    
- Always Include a `settings` File
    

### Convention Plugins

- Definition and Purpose of Convention Plugins
    
- Transforming Build Logic from Configuration to Declaration
    
- Achieving Centralized Management and Consistency of Build Logic
    
- Improving the Testability of Build Logic