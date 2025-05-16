### 目标
在 Spring 中有一个 Event 事件功能，它可以提供事件的定义、发布以及监听事件来完成一些自定义的动作。比如你可以定义一个新用户注册的事件，当有用户执行注册完成后，在事件监听中给用户发送一些优惠券和短信提醒，这样的操作就可以把属于基本功能的注册和对应的策略服务分开，降低系统的耦合。以后在扩展注册服务，比如需要添加风控策略、添加实名认证、判断用户属性等都不会影响到依赖注册成功后执行的动作。

那么在本章节我们需要以观察者模式的方式，设计和实现 Spring Event 的容器事件和事件监听器功能，最终可以让我们在现有实现的 Spring 框架中可以定义、监听和发布自己的事件信息。
### 方案
其实事件的设计本身就是一种观察者模式的实现，它所要解决的就是一个对象状态改变给其他对象通知的问题，而且要考虑到易用和低耦合，保证高度的协作；在功能实现上我们需要定义出事件类、事件监听、事件发布，而这些类的功能需要结合到 Spring 的 `AbstractApplicationContext#refresh()`，以便于处理事件初始化和注册事件监听器的操作。整体设计结构如下图：
![[【small-spring】容器事件和事件监听器 架构图.png]]
- 在整个功能实现过程中，仍然需要在面向用户的应用上下文 `AbstractApplicationContext` 中添加相关事件内容，包括：初始化事件发布者、注册事件监听器、发布容器刷新完成事件。
- 使用观察者模式定义事件类、监听类、发布类，同时还需要完成一个广播器的功能，接收到事件推送时进行分析处理符合监听事件接受者感兴趣的事件，也就是使用 isAssignableFrom 进行判断。
- isAssignableFrom 和 instanceof 相似，不过 isAssignableFrom 是用来判断子类和父类的关系的，或者接口的实现类和接口的关系的，默认所有的类的终极父类都是Object。如果A.isAssignableFrom(B)结果是true，证明B可以转换成为A,也就是A可以由B转换而来。
### 实现
![[【small-spring】容器事件和事件监听器 核心类图.png|center|900]]
- 以上整个类关系图以围绕实现 event 事件定义、发布、监听功能实现和把事件的相关内容使用 AbstractApplicationContext#refresh 进行注册和处理操作。
- 在实现的过程中主要以扩展 spring context 包为主，事件的实现也是在这个包下进行扩展的，当然也可以看出来目前所有的实现内容，仍然是以 IOC 为主。
- `ApplicationContext` 容器继承事件发布功能接口 `ApplicationEventPublisher`，并在实现类中提供事件监听功能。
- `ApplicationEventMulticaster` 接口是注册监听器和发布事件的广播器，提供添加、移除和发布事件方法。
- 最后是发布容器关闭事件，这个仍然需要扩展到 `AbstractApplicationContext#close` 方法中，由注册到虚拟机的钩子实现。
#### 定义和实现事件
```java
public abstract class ApplicationEvent extends EventObject {

    /**
     * Constructs a prototypical Event.
     *
     * @param source The object on which the Event initially occurred.
     * @throws IllegalArgumentException if source is null.
     */
    public ApplicationEvent(Object source) {
        super(source);
    }

}
```
- 以继承 java.util.EventObject 定义出具备事件功能的抽象类 ApplicationEvent，后续所有事件的类都需要继承这个类。

```java
public class ApplicationContextEvent extends ApplicationEvent {

    /**
     * Constructs a prototypical Event.
     *
     * @param source The object on which the Event initially occurred.
     * @throws IllegalArgumentException if source is null.
     */
    public ApplicationContextEvent(Object source) {
        super(source);
    }

    /**
     * Get the <code>ApplicationContext</code> that the event was raised for.
     */
    public final ApplicationContext getApplicationContext() {
        return (ApplicationContext) getSource();
    }

}
```

```java
public class ContextClosedEvent extends ApplicationContextEvent{

    /**
     * Constructs a prototypical Event.
     *
     * @param source The object on which the Event initially occurred.
     * @throws IllegalArgumentException if source is null.
     */
    public ContextClosedEvent(Object source) {
        super(source);
    }

}
```

```java
public class ContextRefreshedEvent extends ApplicationContextEvent{
    /**
     * Constructs a prototypical Event.
     *
     * @param source The object on which the Event initially occurred.
     * @throws IllegalArgumentException if source is null.
     */
    public ContextRefreshedEvent(Object source) {
        super(source);
    }

}
```
- `ApplicationEvent` 是定义事件的抽象类，所有的事件包括 **关闭、刷新，以及用户自己实现的事件**，都需要继承这个类。
- `ContextClosedEvent`、`ContextRefreshedEvent`，分别是 Spring 框架自己实现的两个事件类，可以用于监听刷新和关闭动作。
#### 事件广播器
```java
public interface ApplicationEventMulticaster {

    /**
     * Add a listener to be notified of all events.
     * @param listener the listener to add
     */
    void addApplicationListener(ApplicationListener<?> listener);

    /**
     * Remove a listener from the notification list.
     * @param listener the listener to remove
     */
    void removeApplicationListener(ApplicationListener<?> listener);

    /**
     * Multicast the given application event to appropriate listeners.
     * @param event the event to multicast
     */
    void multicastEvent(ApplicationEvent event);

}
```
- 在事件广播器中定义了添加监听和删除监听的方法以及一个广播事件的方法 `multicastEvent` 最终推送时间消息也会经过这个接口方法来处理谁该接收事件。
```java
public abstract class AbstractApplicationEventMulticaster implements ApplicationEventMulticaster, BeanFactoryAware {

    public final Set<ApplicationListener<ApplicationEvent>> applicationListeners = new LinkedHashSet<>();

    private BeanFactory beanFactory;

    @Override
    public void addApplicationListener(ApplicationListener<?> listener) {
        applicationListeners.add((ApplicationListener<ApplicationEvent>) listener);
    }

    @Override
    public void removeApplicationListener(ApplicationListener<?> listener) {
        applicationListeners.remove(listener);
    }

    @Override
    public final void setBeanFactory(BeanFactory beanFactory) {
        this.beanFactory = beanFactory;
    }

    protected Collection<ApplicationListener> getApplicationListeners(ApplicationEvent event) {
        LinkedList<ApplicationListener> allListeners = new LinkedList<ApplicationListener>();
        for (ApplicationListener<ApplicationEvent> listener : applicationListeners) {
            if (supportsEvent(listener, event)) allListeners.add(listener);
        }
        return allListeners;
    }

    /**
     * 监听器是否对该事件感兴趣
     */
    protected boolean supportsEvent(ApplicationListener<ApplicationEvent> applicationListener, ApplicationEvent event) {
        Class<? extends ApplicationListener> listenerClass = applicationListener.getClass();

        // 按照 CglibSubclassingInstantiationStrategy、SimpleInstantiationStrategy 不同的实例化类型，需要判断后获取目标 class
        Class<?> targetClass = ClassUtils.isCglibProxyClass(listenerClass) ? listenerClass.getSuperclass() : listenerClass;
        Type genericInterface = targetClass.getGenericInterfaces()[0];

        Type actualTypeArgument = ((ParameterizedType) genericInterface).getActualTypeArguments()[0];
        String className = actualTypeArgument.getTypeName();
        Class<?> eventClassName;
        try {
            eventClassName = Class.forName(className);
        } catch (ClassNotFoundException e) {
            throw new BeansException("wrong event class name: " + className);
        }
        // 判定此 eventClassName 对象所表示的类或接口与指定的 event.getClass() 参数所表示的类或接口是否相同，或是否是其超类或超接口。
        // isAssignableFrom是用来判断子类和父类的关系的，或者接口的实现类和接口的关系的，默认所有的类的终极父类都是Object。如果A.isAssignableFrom(B)结果是true，证明B可以转换成为A,也就是A可以由B转换而来。
        return eventClassName.isAssignableFrom(event.getClass());
    }

}
```
- `AbstractApplicationEventMulticaster` 是对事件广播器的公用方法提取，在这个类中可以实现一些基本功能，避免所有直接实现接口放还需要处理细节。
- 除了像 `addApplicationListener`、`removeApplicationListener`，这样的通用方法，这里这个类中主要是对 `getApplicationListeners` 和 `supportsEvent` 的处理。
- `getApplicationListeners` 方法主要是摘取符合广播事件中的监听处理器，具体过滤动作在 `supportsEvent` 方法中。
- 在 `supportsEvent` 方法中，主要包括对 Cglib、Simple 不同实例化需要获取目标 Class，Cglib代理类需要获取父类的Class，普通实例化的不需要。接下来就是通过提取接口和对应的 `ParameterizedType` 和 `eventClassName`，方便最后确认是否为子类和父类的关系，以此证明此事件归这个符合的类处理。
- `Type actualTypeArgument = ((ParameterizedType) genericInterface).getActualTypeArguments()[0];` 用于获取泛型的具体参数
#### 事件发布者的定义和实现
```java
public interface ApplicationEventPublisher {

    /**
     * Notify all listeners registered with this application of an application
     * event. Events may be framework events (such as RequestHandledEvent)
     * or application-specific events.
     * @param event the event to publish
     */
    void publishEvent(ApplicationEvent event);

}
```
- ApplicationEventPublisher 是整个一个事件的发布接口，所有的事件都需要从这个接口发布出去。

```java
public abstract class AbstractApplicationContext extends DefaultResourceLoader implements ConfigurableApplicationContext {

    public static final String APPLICATION_EVENT_MULTICASTER_BEAN_NAME = "applicationEventMulticaster";

    private ApplicationEventMulticaster applicationEventMulticaster;

    @Override
    public void refresh() throws BeansException {

        // 6. 初始化事件发布者
        initApplicationEventMulticaster();

        // 7. 注册事件监听器
        registerListeners();

        // 9. 发布容器刷新完成事件
        finishRefresh();
    }

    private void initApplicationEventMulticaster() {
        ConfigurableListableBeanFactory beanFactory = getBeanFactory();
        applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
        beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, applicationEventMulticaster);
    }

    private void registerListeners() {
        Collection<ApplicationListener> applicationListeners = getBeansOfType(ApplicationListener.class).values();
        for (ApplicationListener listener : applicationListeners) {
            applicationEventMulticaster.addApplicationListener(listener);
        }
    }

    private void finishRefresh() {
        publishEvent(new ContextRefreshedEvent(this));
    }

    @Override
    public void publishEvent(ApplicationEvent event) {
        applicationEventMulticaster.multicastEvent(event);
    }

    @Override
    public void close() {
        // 发布容器关闭事件
        publishEvent(new ContextClosedEvent(this));

        // 执行销毁单例bean的销毁方法
        getBeanFactory().destroySingletons();
    }

}

```
- 在抽象应用上下文 AbstractApplicationContext#refresh 中，主要新增了 `初始化事件发布者`、`注册事件监听器`、`发布容器刷新完成事件`，三个方法用于处理事件操作。
- 初始化事件发布者(initApplicationEventMulticaster)，主要用于实例化一个 SimpleApplicationEventMulticaster，这是一个事件广播器。
- 注册事件监听器(registerListeners)，通过 getBeansOfType 方法获取到所有从 spring.xml 中加载到的事件配置 Bean 对象。
- 发布容器刷新完成事件(finishRefresh)，发布了第一个服务器启动完成后的事件，这个事件通过 publishEvent 发布出去，其实也就是调用了 applicationEventMulticaster.multicastEvent(event); 方法。
- 最后是一个 close 方法中，新增加了发布一个容器关闭事件。`publishEvent(new ContextClosedEvent(this));`

`SimpleApplicationEventMulticaster`：事件广播的简单实现
```java
public class SimpleApplicationEventMulticaster extends AbstractApplicationEventMulticaster {  
  
    public SimpleApplicationEventMulticaster(BeanFactory beanFactory) {  
        setBeanFactory(beanFactory);  
    }  
  
    @SuppressWarnings("unchecked")  
    @Override  
    public void multicastEvent(final ApplicationEvent event) {  
        for (final ApplicationListener listener : getApplicationListeners(event)) {  
            listener.onApplicationEvent(event);  
        }  
    }  
  
}
```