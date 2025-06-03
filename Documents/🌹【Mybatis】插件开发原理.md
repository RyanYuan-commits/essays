### 1 Mybatis 插件原理
MyBatis 中的关键的四个对象都是代理对象，MyBatis 所允许拦截的⽅法如下：
- 执⾏器 `Executor` (update、query、commit、rollback等⽅法)；
- SQL 语法构建器 `StatementHandler` (prepare、parameterize、batch、updates query 等⽅法)；
- 参数处理器 `ParameterHandler`(getParameterObject、setParameters⽅法)；
- 结果集处理器 `ResultSetHandler`(handleResultSets、handleOutputParameters等⽅法)。
如果存在对应的插件，Mybatis 构建完上述的对象后是不会直接返回的，而是会生成一个代理对象，以 `ResultSetHandler` 为例：
```java
public ResultSetHandler newResultSetHandler(Executor executor, MappedStatement mappedStatement, 
				RowBounds rowBounds, ParameterHandler parameterHandler, ResultHandler resultHandler, BoundSql boundSql) {  
  ResultSetHandler resultSetHandler = new DefaultResultSetHandler(executor, mappedStatement, 
										  parameterHandler, resultHandler, boundSql, rowBounds);  
  resultSetHandler = (ResultSetHandler) interceptorChain.pluginAll(resultSetHandler);
  return resultSetHandler;
}
```
`interceptorChain` 保存了所有的拦截器，拦截器在 Mybatis 初始化的时从配置文件中读取并创建，调⽤拦截器链中的拦截器依次的对⽬标进⾏拦截或增强。`interceptor.plugin(target)` 中的 `target` 就可以理解为 `mybatis` 中的四⼤对象。返回的 `target` 是被重重代理后的对象 如果我们想要拦截 `Executor` 的 `query` ⽅法，那么可以这样定义插件：
```java
@Intercepts({
    @Signature(
    type = Executor.class,
    method = "query",
    args={MappedStatement.class,Object.class,RowBounds.class,ResultHandler.class}
    )
})
public class ExeunplePlugin implements Interceptor {
    //省略逻辑
}
```
除此之外，我们还需将插件配置到sqlMapConfig.xml中。
```java
<plugins>
  <plugin interceptor="com.zjq.plugin.ExamplePlugin"></plugin>
</plugins>
```
这样 MyBatis 在启动时可以加载插件，并保存插件实例到相关对象(InterceptorChain，拦截器链) 中。待准备⼯作做完后，MyBatis 处于就绪状态。我们在执⾏SQL时，需要先通过 DefaultSqlSessionFactory 创建 SqlSession。Executor 实例会在创建 SqlSession 的过程中被创建， Executor实例创建完毕后，MyBatis 会通过 JDK 动态代理为实例⽣成代理类。这样，插件逻辑即可在 Executor相关⽅法被调⽤前执⾏。 以上就是MyBatis插件机制的基本原理。
### 2 自定义插件
