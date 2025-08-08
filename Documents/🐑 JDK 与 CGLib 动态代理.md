## 1 简单介绍

动态代理允许在**运行时**创建代理类，无需手动编写。主要有两种实现方式：
1. ==JDK 动态代理==：基于接口的代理，由 `java.lang.reflect.Proxy` 类实现，是基于接口的代理机制，要求被代理的目标必须实现至少一个接口；
2. ==CGLIB 动态代理==：基于类的代理，依赖 `net.sf.cglib.proxy.Enhancer` 实现，通过继承目标类生成子类实现。

## 2 JDK 动态代理实现

### 2.1 使用过程

#### 目标类实现接口

```java
interface UserService {  
    void saveUser(String name);  
}  
  
class UserServiceImpl implements UserService {  
    @Override  
    public void saveUser(String name) {  
        System.out.println("保存用户：" + name);  
    }  
}
```
#### 实现 `InvocationHandler` 接口来处理方法调用

```java
class UserServiceProxy implements InvocationHandler {
    private Object target;
    public UserServiceProxy(Object target) {
        this.target = target;
    }
    public UserServiceProxy() {}
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {  
        System.out.println("Before method: " + method.getName());  
        Object result = null;  
        if (target != null) result = method.invoke(target, args);  
        System.out.println("After method: " + method.getName());  
        return result;  
    }
}
```
#### 通过 `Proxy.newProxyInstance()` 生成代理对象

```java
public static void main(String[] args) {  
    // 需要目标类  
    UserService target = new UserServiceImpl();  
    UserService proxy1 = (UserService) Proxy.newProxyInstance(  
            target.getClass().getClassLoader(),  
            target.getClass().getInterfaces(),  
            new UserServiceProxy(target)  
    );  
    proxy1.saveUser("test1");  
  
    // 直接代理接口  
    UserService proxy2 = (UserService) Proxy.newProxyInstance(  
            UserService.class.getClassLoader(),  
            new Class[]{UserService.class},  
            new UserServiceProxy()  
    );  
    proxy2.saveUser("test2");  
}
```
### 2.2 代理类代码

`Proxy.newProxyInstance()` 生成的代理类继承自 `Proxy` 并实现目标接口:

```java
public final class $Proxy0 extends Proxy implements UserService {
    private InvocationHandler h;
    public $Proxy0(InvocationHandler h) {
        super(h);
    }
    public void saveUser(String username) {
        h.invoke(this, method, new Object[]{username});
    }
}
```
## 3 CGLib 动态代理

### 3.1 使用过程

#### 添加 CGLIB 依赖

```xml
<dependency>  
    <groupId>cglib</groupId>  
    <artifactId>cglib</artifactId>  
    <version>${cglib.version}</version>  
</dependency>
```
#### 实现 `MethodInterceptor` 接口，处理方法调用

```java
class UserMapperInterceptor implements MethodInterceptor {  
    @Override  
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {  
        System.out.println("Before method: " + method.getName());  
        Object result = methodProxy.invokeSuper(o, objects);  
        System.out.println("After method: " + method.getName());  
        return result;  
    }  
}
```
#### 通过 `Enhancer` 生成代理对象

```java
public static void main(String[] args) {  
    Enhancer enhancer = new Enhancer();  
    enhancer.setSuperclass(UserMapper.class);  
    enhancer.setCallback(new UserMapperInterceptor());  
    UserMapper userMapper = (UserMapper) enhancer.create();  
    userMapper.saveUser("test");  
}
```
### 2.3 代理类代码

`Enhancer` 生成的代理类继承自目标类，并重写方法：

```java
public class UserService$$EnhancerByCGLIB$$... extends UserService {
	private MethodInterceptor interceptor;
	@Override
	public void saveUser(String username) {
		interceptor.intercept(
			this, 
			method, 
			new Object[]{username}, 
			MethodProxy.create(...));
	}
}
```