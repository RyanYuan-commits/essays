---
type: Gradle
sub-type: Basic Knowledge
---
## 1	Basic Skeleton

```groovy
// 设置文件基本结构
rootProject.name = 'my-project'  // 设置根项目名称

include 'subproject1', 'subproject2'  // 包含子项目

// 插件管理
pluginManagement {
    repositories {
        gradlePluginPortal()
        mavenCentral()
    }
}

// 依赖解析管理
dependencyResolutionManagement {
    repositories {
        mavenCentral()
    }
}
```

## 2	Core Configuration Sections

### 2.1	Project Inclusion & Configuration

```groovy
// ====== Basic Subprojects Inclusion ======
// Include single subproject
include 'core'
include 'webapp'

// Include multiple subprojects (comma-separated)
include 'api', 'impl', 'utils'

// Include nested subprojects
include 'project-a'
include 'project-b'
include 'project-c:subproject-c1'
include 'project-c:subproject-c2'

// ====== Project Path Mapping ======
// When subproject directory structure differs from default convention
include 'api'
include 'services:hotel-service'
include 'services:booking-service'

// Remap project paths to physical directories
project(':api').projectDir = new File(rootDir, 'core/api')
project(':services:hotel-service').projectDir = new File(rootDir, 'services/hotel')
project(':services:booking-service').projectDir = new File(rootDir, 'services/booking')
```

### 2.2	Plugin Management

```groovy
pluginManagement {
    // Plugin repository configuration
    repositories {
        gradlePluginPortal()        // Gradle official plugin portal
        mavenCentral()              // Maven Central
        maven {
            url 'https://maven.company.com/repo'
        }
        ivy {
            url 'https://ivy.company.com/repo'
        }
    }
    
    // Resolution strategy
    resolutionStrategy {
        eachPlugin {
            if (requested.id.id == 'com.example.myplugin') {
                useModule('com.example:myplugin:1.0')
            }
        }
    }
    
    // Plugin version management
    plugins {
        id 'org.springframework.boot' version '2.7.0'
        id 'io.spring.dependency-management' version '1.0.11.RELEASE'
        id 'com.github.johnrengelman.shadow' version '7.1.2'
    }
}
```

### 2.3	Dependency Resolution Management

Centralized management for dependencies repositories, since Gradle 6.8

```groovy
dependencyResolutionManagement {
    // Repository mode configuration
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    // Alternatives: RepositoriesMode.PREFER_PROJECT (default)
    //               RepositoriesMode.PREFER_SETTINGS
    
    // Repository declarations
    repositories {
        mavenCentral()  // Maven Central
        
        maven {
            url 'https://maven.company.com/releases'
            name 'CompanyReleases'
            credentials {
                username = 'user'
                password = 'password'
            }
            content {
                includeGroup 'com.company'
            }
        }
        
        ivy {
            url 'https://ivy.company.com/repo'
            patternLayout {
                artifact '[organisation]/[module]/[revision]/[artifact]-[revision].[ext]'
            }
        }
        
        // Repository exclusively for plugins
        exclusiveContent {
            forRepository {
                maven {
                    url 'https://repo.company.com/plugins'
                }
            }
            filter {
                includeGroupByRegex 'com\\.company\\..*'
            }
        }
    }
    
    // Version catalogs (Gradle 7.0+ feature)
    versionCatalogs {
        libs {
            version('springboot', '2.7.0')
            library('spring-boot-starter', 'org.springframework.boot', 'spring-boot-starter').versionRef('springboot')
            library('spring-boot-starter-web', 'org.springframework.boot', 'spring-boot-starter-web').versionRef('springboot')
            bundle('spring', ['spring-boot-starter', 'spring-boot-starter-web'])
        }
    }
}
```

### 2.4	Feature Enablement

```groovy
// Enable Gradle features
enableFeaturePreview('TYPESAFE_PROJECT_ACCESSORS')
enableFeaturePreview('VERSION_CATALOGS')

// Configure features
features {
    // Type-safe project accessors (Gradle 7.0+)
    typeSafeProjectAccessors.enable()
    
    // Version catalogs
    versionCatalogs.enable()
}
```

### 2.5	Build Lifecycle Hooks

```groovy
// Execute before project evaluation
gradle.beforeProject { project ->
    println "About to evaluate project: ${project.name}"
}

// Execute after project evaluation
gradle.afterProject { project ->
    if (project.state.failure) {
        println "Project ${project.name} evaluation failed"
    } else {
        println "Project ${project.name} evaluation succeeded"
    }
}

// Execute after task graph is ready
gradle.taskGraph.whenReady { graph ->
    println "Task graph ready, contains ${graph.allTasks.size()} tasks"
}

// Execute when build completes
gradle.buildFinished { result ->
    if (result.failure) {
        println "Build failed: ${result.failure.message}"
    } else {
        println "Build succeeded!"
    }
}
```

