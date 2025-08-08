---
type: Java 框架
finished: "false"
---
## 1 扫描 DubboService

```java
private void scanServiceBeans(Set<String> packagesToScan, BeanDefinitionRegistry registry) {
    
    // ......

    DubboClassPathBeanDefinitionScanner scanner =
            new DubboClassPathBeanDefinitionScanner(registry, environment, resourceLoader);

    // ......

    for (String packageToScan : packagesToScan) {

        // ......

        // ComponentScan 是否包含都能扫描到
        scanner.scan(packageToScan);
        Set<BeanDefinitionHolder> beanDefinitionHolders =
                findServiceBeanDefinitionHolders(scanner, packageToScan, registry, beanNameGenerator);

        if (!CollectionUtils.isEmpty(beanDefinitionHolders)) {

            // ......

            // 处理扫描到的 BeanDefinition
            for (BeanDefinitionHolder beanDefinitionHolder : beanDefinitionHolders) {
                processScannedBeanDefinition(beanDefinitionHolder, registry, scanner);
                servicePackagesHolder.addScannedClass(beanDefinitionHolder.getBeanDefinition().getBeanClassName());
            }
        } else {
            // ......
        }

        servicePackagesHolder.addScannedPackage(packageToScan);
    }
}
```

代码来自 `ServiceAnnotationPostProcessor`, 利用扫描器将含有 `@DubboService` 注解的类注册成 `BeanDefinition`, 并调用 `processScannedBeanDefinition` 方法做后置处理.

```java
private void processScannedBeanDefinition(BeanDefinitionHolder beanDefinitionHolder, BeanDefinitionRegistry registry,
                                              DubboClassPathBeanDefinitionScanner scanner) {

	Class<?> beanClass = resolveClass(beanDefinitionHolder);

	Annotation service = findServiceAnnotation(beanClass);

	// 获取注解配置
	Map<String, Object> serviceAnnotationAttributes = AnnotationUtils.getAttributes(service, true);

	// 服务接口全类名, 如 com.ryan.service.UserService
	String serviceInterface = resolveInterfaceName(serviceAnnotationAttributes, beanClass);

	// Bean 名称, 如 userServiceImpl
	String annotatedServiceBeanName = beanDefinitionHolder.getBeanName();

	// 生成带有标识的 Bean Name, 如 ServiceBean:com.ryan.server.service.UserService
	String beanName = generateServiceBeanName(serviceAnnotationAttributes, serviceInterface);

	AbstractBeanDefinition serviceBeanDefinition =
			buildServiceBeanDefinition(serviceAnnotationAttributes, serviceInterface, annotatedServiceBeanName);

	registerServiceBeanDefinition(beanName, serviceBeanDefinition, serviceInterface);

}
    
private AbstractBeanDefinition buildServiceBeanDefinition(Map<String, Object> serviceAnnotationAttributes,
                                                              String serviceInterface,
                                                              String refServiceBeanName) {

	BeanDefinitionBuilder builder = rootBeanDefinition(ServiceBean.class);

	// Definition 构建

	return builder.getBeanDefinition();

}
```

简单来说, 是将 `@DubboService` 上的配置提取出来, 构建出类型为 `ServiceBean` 的 Bean 定义, 并将其添加到 `BeanDefinitionRegistry` 中.

