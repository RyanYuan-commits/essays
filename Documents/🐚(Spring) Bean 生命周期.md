---
type: Java
sub-type: Spring
finished: "false"
created: 2025-09-27 22:34:06
updated: 2025-09-27 22:34:06
---
## 1 实例化 instantiate

为 Bean 分配内存并创建实例, 此时, Bean 对象只是一个空壳, 属性值还未填充.

`AbstractBeanFactory#doGetBean`

实例化方法 `getObjectForBeanInstance` 在 `AbstractBeanFactory#doGetBean` 方法中被调用.

## 2 属性填充 populate

在实例创建后, Spring 容器会根据配置文件或注解, 为 Bean 的属性注入值.

属性填充方法 `populateBean` 在 `AbstractAutowireCapableBeanFactory#doCreateBean` 方法中被调用.

会注入 `@Autowired`, `@Resource`, `@Value` 中配置的内容.

## 3 BeanPostProcessor 前置处理

初始化阶段之前的重要扩展点, 允许你在 Bean 初始化之前对其进行定制和修改;

会调用 `BeanPostProcessor` 的 `postProcessBeforeInitialization()` 方法.

## 4 初始化 Initialize

是 Bean 对象可用前的最后一步, 执行自定义的初始化逻辑.

会按顺序执行这些方法:

- 被 `@PostConstruct` 注释的方法;
- 实现 `InitializingBean` 接口的 `afterPropertiesSet` 方法;
- 自定义的 init 方法, 配置文件中的 `init-method` 属性, 或者在 `@Bean` 注解的 `initMethod` 属性中指定.

## 5 BeanPostProcessor 后置处理

初始化阶段后的拓展点;

会调用 `BeanPostProcessor` 的 `postProcessAfterInitialization` 方法.

## 6 Bean 的使用

Bean 完全就绪, 可以应用程序使用了.

## 7 Bean 销毁 Destruct

当 Spring 容器关闭的时候, 会执行 Bean 的销毁操作, 以释放资源;

会按顺序执行这些方法:

- 被 `@PreDestory` 注释的方法;
- 实现 `DisposableBean` 的 `destroy` 方法;
- 自定义的方法, 通过配置文件的 `destroy-method` 属性或者 `@Bean` 注解的 `destroyMethod` 来指定.

