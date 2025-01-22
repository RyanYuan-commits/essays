# 目标
其实本章节要解决的问题就是关于如何给代理对象中的属性填充相应的值，因为在之前把`AOP动态代理，融入到Bean的生命周期`时，创建代理对象是在整个创建 Bean 对象之前，也就是说这个代理对象的创建并不是在 Bean 生命周期中。

所以本章节中我们要把代理对象的创建融入到 Bean 的生命周期中，也就是需要把创建代理对象的逻辑迁移到 Bean 对象执行初始化方法之后，在执行代理对象的创建。
# 方案
按照创建代理对象的操作 `DefaultAdvisorAutoProxyCreator` 实现的 `InstantiationAwareBeanPostProcessor` 接口，那么原本在 Before 中的操作，则需要放到 After 中处理。整体设计如下：
![[【small-spring】给代理对象的属性设置值 架构图.png]]
- 在创建 Bean 对象 `createBean` 的生命周期中，有一个阶段是在 Bean 对象属性填充完成以后，执行 Bean 的初始化方法和 BeanPostProcessor 的前置和后置处理，例如：感知 Aware 对象、处理 init-method 方法等。那么在这个阶段的 `BeanPostProcessor After` 就可以用于创建代理对象操作。
- 在 DefaultAdvisorAutoProxyCreator 用于创建代理对象的操作中，需要把创建操作从 postProcessBeforeInstantiation 方法中迁移到 postProcessAfterInitialization，这样才能满足 Bean 属性填充后的创建操作。
# 实现
![[【small-spring】给代理对象的属性设置值 核心类图.png]]
- 虽然本章节要完成的是关于代理对象中属性的填充问题，但实际解决的思路是处理在 Bean 的生命周期中合适的位置（`初始化 initializeBean`）中处理代理类的创建。
- 所以以上的改动并不会涉及太多内容，主要包括：DefaultAdvisorAutoProxyCreator 类创建代理对象的操作放置在 postProcessAfterInitialization 方法中以及对应在 AbstractAutowireCapableBeanFactory 完成初始化方法的调用操作。
- 另外还有一点要注意，就是目前我们在 Spring 框架中，AbstractAutowireCapableBeanFactory 类里使用的是 CglibSubclassingInstantiationStrategy 创建对象，所以有需要判断对象获取接口的方法中，也都需要判断是否为 CGlib创建，否则是不能正确获取到接口的。如：`ClassUtils.isCglibProxyClass(clazz) ? clazz.getSuperclass() : clazz;`
## 判断 CgLib 对象
```java
public class TargetSource {

    private final Object target;

    /**
     * Return the type of targets returned by this {@link TargetSource}.
     * <p>Can return <code>null</code>, although certain usages of a
     * <code>TargetSource</code> might just work with a predetermined
     * target class.
     *
     * @return the type of targets returned by this {@link TargetSource}
     */
    public Class<?>[] getTargetClass() {
        Class<?> clazz = this.target.getClass();
        clazz = ClassUtils.isCglibProxyClass(clazz) ? clazz.getSuperclass() : clazz;
        return clazz.getInterfaces();
    }

}
```
- 在 `TargetSource#getTargetClass` 是用于获取 target 对象的接口信息的，那么这个 target 可能是 `JDK代理` 创建也可能是 `CGlib创建`，为了保证都能正确的获取到结果，这里需要增加判读 `ClassUtils.isCglibProxyClass(clazz)`
## 迁移创建 AOP 代理方法
```java
public class DefaultAdvisorAutoProxyCreator implements InstantiationAwareBeanPostProcessor, BeanFactoryAware {

    private DefaultListableBeanFactory beanFactory;

    @Override
    public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
        return null;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {

        if (isInfrastructureClass(bean.getClass())) return bean;

        Collection<AspectJExpressionPointcutAdvisor> advisors = beanFactory.getBeansOfType(AspectJExpressionPointcutAdvisor.class).values();

        for (AspectJExpressionPointcutAdvisor advisor : advisors) {
            ClassFilter classFilter = advisor.getPointcut().getClassFilter();
            // 过滤匹配类
            if (!classFilter.matches(bean.getClass())) continue;

            AdvisedSupport advisedSupport = new AdvisedSupport();

            TargetSource targetSource = new TargetSource(bean);
            advisedSupport.setTargetSource(targetSource);
            advisedSupport.setMethodInterceptor((MethodInterceptor) advisor.getAdvice());
            advisedSupport.setMethodMatcher(advisor.getPointcut().getMethodMatcher());
            advisedSupport.setProxyTargetClass(false);

            // 返回代理对象
            return new ProxyFactory(advisedSupport).getProxy();
        }

        return bean;
    }  

}
```
- 关于 DefaultAdvisorAutoProxyCreator 类的操作主要就是把创建 AOP 代理的操作从 postProcessBeforeInstantiation 移动到 postProcessAfterInitialization 中去。
- 通过设置一些 AOP 的必备参数后，返回代理对象 `new ProxyFactory(advisedSupport).getProxy()` 这个代理对象中就包括间接调用了 TargetSource 中对 getTargetClass() 的获取。
## 在 Bean 的生命周期中执行
```java
public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory implements AutowireCapableBeanFactory {

    private InstantiationStrategy instantiationStrategy = new CglibSubclassingInstantiationStrategy();

    @Override
    protected Object createBean(String beanName, BeanDefinition beanDefinition, Object[] args) throws BeansException {
        Object bean = null;
        try {
            // ...

            // 执行 Bean 的初始化方法和 BeanPostProcessor 的前置和后置处理方法
            bean = initializeBean(beanName, bean, beanDefinition);
        } catch (Exception e) {
            throw new BeansException("Instantiation of bean failed", e);
        }
        // ...
        return bean;
    }

    private Object initializeBean(String beanName, Object bean, BeanDefinition beanDefinition) {

        // ...

        wrappedBean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
        return wrappedBean;
    }

    @Override
    public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName) throws BeansException {
        Object result = existingBean;
        for (BeanPostProcessor processor : getBeanPostProcessors()) {
            Object current = processor.postProcessAfterInitialization(result, beanName);
            if (null == current) return result;
            result = current;
        }
        return result;
    }

}
```