---
type: Gradle
sub-type: Basic
---
## 1	Running Commands

```bash
gradle [taskName...] [--option-name...]

# more task
gradle [taskName1 taskName2...] [--option-name...]
```

## 2	Executing tasks

在 Gradle 中, 任务属于特定的项目, 在多项目构建中, 为了清楚地指明要运行哪个任务, 要使用冒号作为项目分隔符. 在根项目级别执行名为 `test` 的任务:

```groovy
gradle :test
```

对于嵌套子项目, 需要书写完整路径:

```groovy
gradle :subproject:test
```

对于不带任何 `:` 的任务, Gradle 会在当前目录执行该任务.

## 3	[Task options](https://docs.gradle.org.cn/current/userguide/command_line_interface.html#command_line_interface_reference)