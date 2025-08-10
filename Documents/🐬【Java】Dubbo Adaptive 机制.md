---
origin_url: https://time.geekbang.org/column/article/620941
finished: "false"
type: Java 框架
---

## 1 什么是  Adaptive 机制?

**Adaptive 自适应机制**: 根据运行时传入的 URL 参数, 动态地为 Dubbo 扩展点选择和调用最合适的实现, 从而实现配置与代码的解耦和高度灵活性.

举个例子, 比如 `Cluster` 集群容错接口, 传统方式, 问题在于无法在运行时动态的修改:

```xml
<dubbo:reference interface="com.example.UserService" id="userService" cluster="failover" />
```

使用`@Adaptive` 注解后, 默认的策略是 `failover`, 但是可以通过 URL 中的配置来动态变更.

```java
// 默认策略是 failover
@SPI(Cluster.DEFAULT)
public interface Cluster {

    String DEFAULT = "failover";

    @Adaptive
    <T> Invoker<T> join(Directory<T> directory, boolean buildFilterChain) throws RpcException;

}
```

## 2 自适应拓展点

```java
Cluster cluster = ApplicationModel.defaultModel()  
        .getExtensionLoader(Cluster.class)  
        .getAdaptiveExtension();
```

使用上面的代码, 来测试一下 Adaptive 机制.

```java
// 获取 AdaptiveExtension, 内置缓存机制
org.apache.dubbo.common.extension.ExtensionLoader#getAdaptiveExtension
=>
// 创建 AdaptiveExtension
org.apache.dubbo.common.extension.ExtensionLoader#createAdaptiveExtension
```

`org.apache.dubbo.common.extension.ExtensionLoader#createAdaptiveExtension`

```java
private T createAdaptiveExtension() {  
    try {  
	    // 核心创建代码
        T instance = (T) getAdaptiveExtensionClass().newInstance(); 

		// 类似 Spring 的 Bean 后置处理机制...
		  
        return instance;  
    } catch (Exception e) {  
        // ......
    }  
}
```

`org.apache.dubbo.common.extension.ExtensionLoader#getAdaptiveExtensionClass`

```java
private Class<?> getAdaptiveExtensionClass() {  
	// 获取 ExtensionLoader 所有关于当前拓展点的拓展对象
    getExtensionClasses();  
    if (cachedAdaptiveClass != null) {  
        return cachedAdaptiveClass;  
    }
	// 创建自适应拓展点对象
    return cachedAdaptiveClass = createAdaptiveExtensionClass();  
}
```

上面的方法中也用到了缓存机制, 每个拓展点只会有一个自适应拓展点对象.=

`org.apache.dubbo.common.extension.ExtensionLoader#createAdaptiveExtensionClass`

```java
private Class<?> createAdaptiveExtensionClass() {
    ClassLoader classLoader = type.getClassLoader();
    try {
        if (NativeUtils.isNative()) {
            return classLoader.loadClass(type.getName() + "$Adaptive");
        }
    } catch (Throwable ignore) {}

    // 调用 generate 方法生成一段 Java 编写的源代码
    String code = new AdaptiveClassCodeGenerator(type, cachedDefaultName).generate();
    org.apache.dubbo.common.compiler.Compiler compiler = extensionDirector.getExtensionLoader(
        org.apache.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
    // 通过源代码生成一个类对象
    return compiler.compile(code, classLoader);
}
```

其中, code 变量导出的代码为:

```java
package org.apache.dubbo.rpc.cluster;
import org.apache.dubbo.rpc.model.ScopeModel;
import org.apache.dubbo.rpc.model.ScopeModelUtil;
// 类名比较特别，是【接口的简单名称】+【$Adaptive】构成的
// 这就是自适应动态扩展点对象的类名
public class Cluster$Adaptive implements org.apache.dubbo.rpc.cluster.Cluster {

    public org.apache.dubbo.rpc.Invoker join(org.apache.dubbo.rpc.cluster.Directory arg0, boolean arg1) throws org.apache.dubbo.rpc.RpcException {
        if (arg0 == null) throw new IllegalArgumentException("org.apache.dubbo.rpc.cluster.Directory argument == null");
        if (arg0.getUrl() == null)
            throw new IllegalArgumentException("org.apache.dubbo.rpc.cluster.Directory argument getUrl() == null");

        org.apache.dubbo.common.URL url = arg0.getUrl();

        // 如果 cluster 为空, 走默认逻辑 failover
        String extName = url.getParameter("cluster", "failover");
        if (extName == null)
            throw new IllegalStateException("Failed to get extension (org.apache.dubbo.rpc.cluster.Cluster) name from" +
                    " url (" + url.toString() + ") use keys([cluster])");

        ScopeModel scopeModel = ScopeModelUtil.getOrDefault(url.getScopeModel(),
                org.apache.dubbo.rpc.cluster.Cluster.class);
        
        // 反正得到了一个 extName 扩展点名称，则继续获取指定的扩展点
        org.apache.dubbo.rpc.cluster.Cluster extension =
                (org.apache.dubbo.rpc.cluster.Cluster) scopeModel.getExtensionLoader(org.apache.dubbo.rpc.cluster.Cluster.class)
                .getExtension(extName);

        // 拿着指定的扩展点继续调用其对应的方法
        return extension.join(arg0, arg1);
    }

    // 这里默认抛异常，说明不是自适应扩展点需要处理的业务逻辑
    public org.apache.dubbo.rpc.cluster.Cluster getCluster(org.apache.dubbo.rpc.model.ScopeModel arg0,
                                                           java.lang.String arg1) {
        throw new UnsupportedOperationException("The method public static org.apache.dubbo.rpc.cluster.Cluster org" +
                ".apache.dubbo.rpc.cluster.Cluster.getCluster(org.apache.dubbo.rpc.model.ScopeModel,java.lang.String)" +
                " of interface org.apache.dubbo.rpc.cluster.Cluster is not adaptive method!");
    }

    // 这里默认也抛异常，说明也不是自适应扩展点需要处理的业务逻辑
    public org.apache.dubbo.rpc.cluster.Cluster getCluster(org.apache.dubbo.rpc.model.ScopeModel arg0,
                                                           java.lang.String arg1, boolean arg2) {
        throw new UnsupportedOperationException("The method public static org.apache.dubbo.rpc.cluster.Cluster org" +
                ".apache.dubbo.rpc.cluster.Cluster.getCluster(org.apache.dubbo.rpc.model.ScopeModel,java.lang.String," +
                "boolean) of interface org.apache.dubbo.rpc.cluster.Cluster is not adaptive method!");
    }

}
```

自适应拓展点对象的类名是 接口名 + $Adaptive, 拓展方法的基本流程是从 URL 中提取该接口关心的参数(如果没有, 走默认), 然后通过 `ExtensionLoader` 来获取实现类并触发对应方法调用.

