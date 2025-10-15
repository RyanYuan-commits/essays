---
type: 工具
sub-type: Gradle
finished: "false"
created: 2025-09-27 22:34:06
updated: 2025-09-27 22:34:06
---
resolutionStrategy 配置块允许你对 Gradle 解析项目的过程进行精细化的控制和干预;

## 1 版本冲突控制

`force(...)`: 强制版本, 优先级最高的指令,强制一个或多个依赖使用指定的版本, 无视任何其他版本请求;

`failOnVersionConfilg`: Gradle 默认自动选择最高的版本来解决冲突, 使用此配置后, 一旦出现版本冲突, 构建将直接失败;

```groovy
configurations.all {
    resolutionStrategy {
        // 规则1：版本冲突时立即失败，确保依赖版本清晰
        failOnVersionConflict()

        // 规则2：强制 guava 库使用特定版本
        force 'com.google.guava:guava:31.1-jre'

        // 规则3：将 commons-logging 替换为 slf4j 桥接
        dependencySubstitution {
            substitute module('commons-logging:commons-logging') with module('org.slf4j:jcl-over-slf4j:2.0.7')
        }

        // 规则4：不使用任何 milestone 版本的 spring 库
        componentSelection {
            all { selection ->
                if (selection.candidate.group == 'org.springframework' && selection.candidate.version.contains("M")) {
                    selection.reject("不允许使用 Spring 的里程碑版本")
                }
            }
        }

        // 规则5：SNAPSHOT 依赖的缓存时间设置为 1 小时
        cacheChangingModulesFor(1, 'hours')
    }
}
```

