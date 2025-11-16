---
type: Toolkit
sub-type: Intellij Plugins
---
## 1	关于 Settings

IntelliJ 平台插件的设置用于持久化保存控制 IDE 行为及外观的配置信息, 插件可扩展应用级或项目级设置, 并在 IDE 设置对话框中提供自定义 UI. 

## 2	设置声明方式

通过声明如下扩展点(EP) 于 `plugin.xml` 文件中实现设置功能：

- 应用级：`com.intellij.applicationConfigurable`
	
- 项目级：`com.intellij.projectConfigurable`   

## 3	主要属性说明 (详见 plugin.xml)

- `instance`：实现类的全限定名(FQN) **必填之一**
    
- `provider`：Provider 实现类 (如使用 ConfigurableProvider, **必填之一**)
    
- `id`：设置唯一标识 (建议用插件 ID + 类名确保唯一)
    
- `displayName`：面板显示名称(支持直接字符串或通过 `key`+`bundle` 提供本地化)
    
- `parentId`：设置分组(决定设置出现在设置树中所属节点, 推荐使用, 如 appearance, editor, build, language, tools 等)
    
- `nonDefaultProject`：仅对项目级设置有效, `true` 表示不针对默认项目
    
- 其他可选：`groupWeight`(分组排序权重)、`dynamic`(动态子节点)、`childrenEPName`(指定动态子节点的扩展点)

## 4	分组 (parentId) 属性常用取值

- `appearance`：外观与行为
    
- `build`：构建、执行、部署
    
- `build.tools`：构建集成(如 Maven/Gradle)
    
- `editor`：编辑器相关设置
    
- `language`：语言和框架
    
- `tools`：第三方工具、SSH 配置等
    
- `root`：超级父节点, 不建议使用
    
- `default`/`other`：不建议使用, 自定义时务必指定 parentId 

## 5	实现基础接口说明

大多数设置实现类使用 `Configurable`(UI 及保存逻辑)；特殊场景可用 `ConfigurableProvider`(如需根据运行时条件动态展示)

应用级实现类需有无参构造函数, 项目级实现类需有 `Project` 类型参数的构造函数；禁止其他依赖注入. 

## 6	生命周期及 UI 注意事项

只有用户点击对应设置时才会实例化实现类, 不要在构造函数里处理 Swing 组件及复杂逻辑. 

初始化顺序为 `createComponent()` → `reset()`, 移除 UI 资源可通过 `disposeUIResources()`. 

若实现特殊标记接口, 如 NoScroll, NoMargin, Beta, 可影响 UI 布局和标签显示. 

## 7	推荐做法

plugin.xml 中声明所有必要属性, 提升 IDE 启动和设置菜单响应速度. 
    
参考官方实现(如 ConsoleConfigurable, AutoImportOptionsConfigurable)与说明文档提供的样例. 