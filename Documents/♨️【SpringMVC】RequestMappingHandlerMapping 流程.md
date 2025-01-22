这个类对添加了 @RequestingMapping 或者 @Controller 注解的类中对方法做 handler 对注册和映射等操作。
RequestMappingHandlerMapping 继承关系图
![[RequestMappingHandlerMapping 继承关系图.png]]
在上面的继承关系图中，其父类 AbstractHandlerMethodMapping 实现了 InitializingBean，在 Bean 属性填充之后去执行接口定义的 afterPropertiesSet，做一些 Bean 的初始化工作。
在 AbstractHandlerMethodMapping 类中，afterPropertiesSet 实现的 afterPropertiesSet 是这样的：
```java
/**  
 * Detects handler methods at initialization. 
 * 在初始化的时候探查 Handler Method
 */
@Override  
public void afterPropertiesSet() {  
    initHandlerMethods();  
}
```
从 AbstractHandlerMethodMapping 类中的 initHandlerMethods 方法开始看：
```java
protected void initHandlerMethods() {  
	// ...
	// 对 BeanFactory 中所有的 BeanName 进行遍历
    for (String beanName : beanNames) {  
       if (!beanName.startsWith(SCOPED_TARGET_NAME_PREFIX)) {  
          Class<?> beanType = null;  
          try {  
             beanType = getApplicationContext().getType(beanName);  
          }  
          catch (Throwable ex) {  
             // ......
          }  
          if (beanType != null && isHandler(beanType)) {  
             detectHandlerMethods(beanName);  
          }  
       }  
    }  
    handlerMethodsInitialized(getHandlerMethods());  
}
```
在上面的方法中，SpringMVC 会获取 BeanFactory 中加载的所有 BeanName，然后进行遍历，当发现这个类是一个 Handler 的时候，就去执行 detectHandlerMethods 方法。
在 RequestMappingHandlerMapping 类中，判断类是否是一个 Handler 是通过类上是否有 @Controller 注解或者 @RequestMapping 注解：
>[!info] RequestMappingHandlerMapping 类中判断 Bean 是否是一个 HandlerBean 的方式
```java
@Override  
protected boolean isHandler(Class<?> beanType) {  
    return ((AnnotationUtils.findAnnotation(beanType, Controller.class) != null) ||  
          (AnnotationUtils.findAnnotation(beanType, RequestMapping.class) != null));  
}
```
---
	当发现这个 Bean 是一个 Handler Bean 之后，调用 detectHandlerMethods 方法，这个方法会检测这个类中所有的 Handler 方法（判断逻辑是看方法上有没有 @RequestingMapping 注解），然后将其注册到 AbstractHandlerMethodMapping 的 mappingRegistry 中保存起来：
```java
protected void detectHandlerMethods(final Object handler) {  
    Class<?> handlerType = (handler instanceof String ?  
          getApplicationContext().getType((String) handler) : handler.getClass());  
    final Class<?> userType = ClassUtils.getUserClass(handlerType);  
  
    Map<Method, T> methods = MethodIntrospector.selectMethods(userType,  
          new MethodIntrospector.MetadataLookup<T>() {  
             @Override  
             public T inspect(Method method) {  
                return getMappingForMethod(method, userType);  
             }  
	  }); 
    for (Map.Entry<Method, T> entry : methods.entrySet()) {  
       registerHandlerMethod(handler, entry.getKey(), entry.getValue());  
    }  
}
```
其中，MethodIntrospector.selectMethods 接收两个参数
- 一个是用户类（屏蔽掉代理类带来的影响），也就是我们要寻找 Handler 方法的类
- 另一个是一个实现了 MetadataLookup 接口的匿名内部类，MetadataLookup 接口只定义了一个 inspect 方法，用户通过是否返回 null 来告诉 inspect 方法的调用者，自己**是否关注这个方法**。
selectMethods 的具体逻辑是这样的：遍历类中所有的方法，对其执行用户实现的 MetadataLookup 接口的 inspect 方法，如果用户关注这个方法（有返回值）将这些方法保存在 Map 中，这个 Map 的 key 是具体的方法对象，而 value 就是用户返回的那个值。
在上面的 detectHandlerMethods 方法中，这个 Map 的 value 就是 getMappingForMethod 的返回值，这个方法在 RequestMappingHandlerMapping 类中的实现是这样的：
```java
@Override  
protected RequestMappingInfo getMappingForMethod(Method method, Class<?> handlerType) {  
    RequestMappingInfo info = createRequestMappingInfo(method);  
    if (info != null) {  
       RequestMappingInfo typeInfo = createRequestMappingInfo(handlerType);  
       if (typeInfo != null) {  
          info = typeInfo.combine(info);  
       }  
    }  
    return info;  
}
```
其中 createRequestMappingInfo 方法会先判断传入的类或者方法上有无 @RequestMapping 注解，如果没有，返回 null，getMappingForMethod 方法也会返回 null 然后结束；
如果方法上有 @RequestMapping 注解的话，会根据注解的内容创建一个 RequestMappingInfo 类，然后再次调用 createRequestMappingInfo 方法获取类上的 @RequestMapping 注解中的数据；
最终，调用 combine 方法将两份数据合并，获取到最终的 RequestMappingInfo 实例，这个实例会保有 Handler 类和 Handler 方法中 @RequestMapping 注解的所有信息。


