## 1 继承关系

RequestMappingHandlerMapping 继承关系图

![[RequestMappingHandlerMapping 继承关系图.png]]

## 2 注册流程

父类 `AbstractHandlerMethodMapping` 实现了 `InitializingBean` 的 `afterPropertiesSet` 方法，在这个方法中去执行 `Handler Method` 的初始化：

```java
/**  
 * Detects handler methods at initialization. 
 * 在初始化的时候探查 Handler Method
 */
@Override  
public void afterPropertiesSet() {  
    initHandlerMethods();  
}

protected void initHandlerMethods() {  
    for (String beanName : getCandidateBeanNames()) {  
       if (!beanName.startsWith(SCOPED_TARGET_NAME_PREFIX)) {  
          processCandidateBean(beanName);  
       }  
    }  
    handlerMethodsInitialized(getHandlerMethods());  
}
```

父类 `ApplicationObjectSupport` 实现了 `ApplicationContextAware`，可以感知到应用上下文，通过 `getCandidateBeanNames` 方法来获取所有的 Bean 名称。

遍历这些 Bean，寻找可以注册成 Handler 的类。

---

```java
protected void processCandidateBean(String beanName) {  
    Class<?> beanType = null;  
    try {  
       beanType = obtainApplicationContext().getType(beanName);  
    }  
    catch (Throwable ex) {  
       // ......
    }
    // 判断是否是 Handler
    if (beanType != null && isHandler(beanType)) {  
       detectHandlerMethods(beanName);  
    }
}

@Override  
protected boolean isHandler(Class<?> beanType) {  
    return ((AnnotationUtils.findAnnotation(beanType, Controller.class) != null) ||  
          (AnnotationUtils.findAnnotation(beanType, RequestMapping.class) != null));  
}
```

根据类上是否含有 `@Controller` 或者 `@RequestMapping` 注解来判断是否是一个 Handler Bean，对 Handler Bean 执行 `detectHandlerMethods` 来扫描其中的方法。

---

```java
protected void detectHandlerMethods(final Object handler) { 
	// 兼容 BeanName 入参
    Class<?> handlerType = (
		  handler instanceof String ?  
          getApplicationContext().getType((String) handler) : 
          handler.getClass()
	);  

	// 屏蔽代理类的影响
    final Class<?> userType = ClassUtils.getUserClass(handlerType);  

	// 注册 Handler Method
    Map<Method, T> methods = MethodIntrospector.selectMethods(userType,  
          new MethodIntrospector.MetadataLookup<T>() {  
             @Override  
             public T inspect(Method method) {  
                return getMappingForMethod(method, userType);  
            }  
	});

	// ......

    for (Map.Entry<Method, T> entry : methods.entrySet()) {  
       registerHandlerMethod(handler, entry.getKey(), entry.getValue());  
    }
}
```

这个方法过滤出这个类中所有的 Handler Method，然后将其注册到 `mappingRegistry` 中；

扫描是通过 `MethodIntrospector.selectMethods`进行的，这个方法接收两个参数

- Class；也就是我们要寻找 `Handler` 方法的类
- MetadataLookup 接口实现类；这个函数式接口接口定义了 `inspect` 方法，通过方法返回值是否为 `null` 来告知调用方是否应该关注这个方法（关注返回非 `null` 元数据）。

过滤出来的 Handler Method 会被存放在 `methods` 中，这个 `Map` 的 `key` 是具体的方法对象，`value` 就是 `getMappingForMethod` 方法的返回值。

---

```java
@Override  
protected RequestMappingInfo getMappingForMethod(Method method, Class<?> handlerType) {  
	// 判断传入的方法上是否有 @RequestMapping
    RequestMappingInfo info = createRequestMappingInfo(method);  
    
    if (info != null) {  
	   // 获取传入的类上的 @RequestMapping
       RequestMappingInfo typeInfo = createRequestMappingInfo(handlerType);  
       if (typeInfo != null) {  
	      // 融合
          info = typeInfo.combine(info);  
       }  
    }  
    return info;  
}
```

通过方法上是否有 `@RequestMapping` 注解来判断处理器方法。

---

与注册有关的关键类和属性有这几个：
- `RequestMappingInfo`：封装了将一个 HTTP 请求映射到特定处理器方法所需的所有**元数据和条件**，比如路径条件、请求方法条件等，请求到达时会通过其中的条件来找到对应的处理方法；
- `HandlerMethod`：封装了一个特定的Java 方法（以及其所属的 Bean 实例），这个方法被标记为能够处理特定类型的 HTTP 请求；
- `MappingRegistry.pathLookup`：请求路径与 `RequestMappingInfo` 之间的映射关系；

```java
protected void registerHandlerMethod(Object handler, Method method, T mapping) {
    this.mappingRegistry.register(mapping, handler, method);  
}

public void register(T mapping, Object handler, Method method) {  
    this.readWriteLock.writeLock().lock();  
    try {
	   // 构建 handler method 
       HandlerMethod handlerMethod = createHandlerMethod(handler, method);  
       validateMethodMapping(handlerMethod, mapping);  
  
       Set<String> directPaths = AbstractHandlerMethodMapping.this.getDirectPaths(mapping);
       
       // 添加路径匹配
       for (String path : directPaths) {  
          this.pathLookup.add(path, mapping);  
       }  

	   // 添加名称匹配	
       String name = null;  
       if (getNamingStrategy() != null) {  
          name = getNamingStrategy().getName(handlerMethod, mapping);  
          addMappingName(name, handlerMethod);  
       }  

	   // 跨域处理配置
       CorsConfiguration corsConfig = initCorsConfiguration(handler, method, mapping);  
       if (corsConfig != null) {  
          corsConfig.validateAllowCredentials();  
          this.corsLookup.put(handlerMethod, corsConfig);  
       }  
  
       this.registry.put(mapping,  
             new MappingRegistration<>(mapping, handlerMethod, directPaths, name, corsConfig != null));  
    }  
    finally {  
       this.readWriteLock.writeLock().unlock();  
    }  
}
```


## 3 总结

通过扫描所有的 Bean，根据 Bean 上是否有 `@Controller` 或者 `@RequestMapping` 来判断是否为 Handler；

然后扫描 Handler 中的 Methods，将被 `@RequestMapping` 注释的方法注册为 Handler Method。