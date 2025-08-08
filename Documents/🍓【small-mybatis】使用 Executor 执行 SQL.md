# 目标
在上一章节我们实现了有/无连接池的数据源，可以在调用执行SQL的时候，通过我们实现池化技术完成数据库的操作。
那么关于池化数据源的调用、执行和结果封装，目前我们还都只是在 `DefaultSqlSession` 中进行发起的。那么这样的把代码流程写死的方式肯定不合适于我们扩展使用，也不利于 `SqlSession` 中每一个新增定义的方法对池化数据源的调用。
![[【small-mybatis】使用 Executor 执行 SQL 目标.png]]
- 解耦 `DefaultSqlSession#selectOne` 方法中关于对数据源的调用、执行和结果封装，提供新的功能模块替代这部分硬编码的逻辑处理。
- 只有提供了单独的执行方法入口，我们才能更好的扩展和应对这部分内容里的需求变化，包括了各类入参、结果封装、执行器类型、批处理等，来满足不同样式的用户需求，也就是配置到 Mapper.xml 中的具体信息。
# 设计
从我们对 `ORM` 框架渐进式的开发过程上，可以分出的执行动作包括，解析配置、代理对象、映射方法等，直至我们前面章节对数据源的包装和使用，只不过我们把数据源的操作硬捆绑到了 `DefaultSqlSession` 的执行方法上了。
那么现在为了解耦这块的处理，则需要单独提出一块执行器的服务功能，之后将执行器的功能随着 `DefaultSqlSession` 创建时传入执行器功能，之后具体的方法调用就可以调用执行器来处理了，从而解耦这部分功能模块。
![[【small-mybatis】使用 Executor 执行 SQL 核心架构.png|800]]
- 首先我们要提取出执行器的接口，定义出执行方法、事务获取和相应提交、回滚、关闭的定义，同时由于执行器是一种标准的执行过程，所以可以由抽象类进行实现，对过程内容进行模板模式的过程包装。在包装过程中定义抽象类，由具体的子类来实现。这一部分在下文的代码中会体现到 `SimpleExecutor` 简单执行器实现中。
- 之后是对 SQL 的处理，我们都知道在使用 JDBC 执行 SQL 的时候，分为了简单处理和预处理，预处理中包括准备语句、参数化传递、执行查询，以及最后的结果封装和返回。所以我们这里也需要把 JDBC 这部分的步骤，分为结构化的类过程来实现，便于功能的拓展。具体代码主要体现在语句处理器 `StatementHandler` 的接口实现中。
# 实现
## 工程结构
![[【small-mybatis】使用 Executor 执行 SQL 核心类图.png|900]]
- 以 `Executor` 接口定义为执行器入口，确定出事务和操作和 SQL 执行的统一标准接口。并以执行器接口定义实现抽象类，也就是用抽象类处理统一共用的事务和执行SQL的标准流程，也就是这里定义的执行 SQL 的抽象接口由子类实现。
- 在具体的简单 SQL 执行器实现类中，处理 `doQuery` 方法的具体操作过程。这个过程中则会引入进来 SQL 语句处理器的创建，创建过程仍有 `configuration` 配置项提供。
- 当执行器开发完成以后，接下来就是交给 `DefaultSqlSessionFactory` 开启 `openSession` 的时候随着构造函数参数传递给 `DefaultSqlSession` 中，这样在执行 `DefaultSqlSession#selectOne` 的时候就可以调用执行器进行处理了。也就由此完成解耦操作了。
## Executor 的定义和实现
### 执行器接口
```java
public interface Executor {

    ResultHandler NO_RESULT_HANDLER = null;

    <E> List<E> query(MappedStatement ms, Object parameter, ResultHandler resultHandler, BoundSql boundSql);

    Transaction getTransaction();

    void commit(boolean required) throws SQLException;

    void rollback(boolean required) throws SQLException;

    void close(boolean forceRollback);

}
```
- 在执行器中定义的接口包括事务相关的处理方法和执行SQL查询的操作，随着后续功能的迭代还会继续补充其他的方法。
### 抽象基类
```java
public abstract class BaseExecutor implements Executor {

    protected Configuration configuration;
    protected Transaction transaction;
    protected Executor wrapper;

    private boolean closed;

    protected BaseExecutor(Configuration configuration, Transaction transaction) {
        this.configuration = configuration;
        this.transaction = transaction;
        this.wrapper = this;
    }

    @Override
    public <E> List<E> query(MappedStatement ms, Object parameter, ResultHandler resultHandler, BoundSql boundSql) {
        if (closed) {
            throw new RuntimeException("Executor was closed.");
        }
        return doQuery(ms, parameter, resultHandler, boundSql);
    }

    protected abstract <E> List<E> doQuery(MappedStatement ms, Object parameter, ResultHandler resultHandler, BoundSql boundSql);

    @Override
    public void commit(boolean required) throws SQLException {
        if (closed) {
            throw new RuntimeException("Cannot commit, transaction is already closed");
        }
        if (required) {
            transaction.commit();
        }
    }

}
```
- 在抽象基类中封装了执行器的全部接口，这样具体的子类继承抽象类后，就不用在处理这些共性的方法。与此同时在 query 查询方法中，封装一些必要的流程处理，如果检测关闭等，在 Mybatis 源码中还有一些缓存的操作，这里暂时剔除掉，以核心流程为主。读者伙伴在学习的过程中可以与源码进行对照学习。
### SimpleExecutor 的实现
```java
public class SimpleExecutor extends BaseExecutor {

    public SimpleExecutor(Configuration configuration, Transaction transaction) {
        super(configuration, transaction);
    }

    @Override
    protected <E> List<E> doQuery(MappedStatement ms, Object parameter, ResultHandler resultHandler, BoundSql boundSql) {
        try {
            Configuration configuration = ms.getConfiguration();
            StatementHandler handler = configuration.newStatementHandler(this, ms, parameter, resultHandler, boundSql);
            Connection connection = transaction.getConnection();
            Statement stmt = handler.prepare(connection);
            handler.parameterize(stmt);
            return handler.query(stmt, resultHandler);
        } catch (SQLException e) {
            e.printStackTrace();
            return null;
        }
    }

}
```
- 简单执行器 SimpleExecutor 继承抽象基类，实现抽象方法 doQuery，在这个方法中包装数据源的获取、语句处理器的创建，以及对 Statement 的实例化和相关参数设置。最后执行 SQL 的处理和结果的返回操作。
- 关于 StatementHandler 语句处理器的实现，接下来介绍。
## 语句处理器
### 接口定义
```java
public interface StatementHandler {

    /** 准备语句 */
    Statement prepare(Connection connection) throws SQLException;

    /** 参数化 */
    void parameterize(Statement statement) throws SQLException;

    /** 执行查询 */
    <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException;

}
```
- 语句处理器的核心包括了；准备语句、参数化传递参数、执行查询的操作，这里对应的 Mybatis 源码中还包括了 update、批处理、获取参数处理器等。
### 抽象基类
```java
public abstract class BaseStatementHandler implements StatementHandler {

    protected final Configuration configuration;
    protected final Executor executor;
    protected final MappedStatement mappedStatement;

    protected final Object parameterObject;
    protected final ResultSetHandler resultSetHandler;

    protected BoundSql boundSql;

    public BaseStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, ResultHandler resultHandler, BoundSql boundSql) {
        this.configuration = mappedStatement.getConfiguration();
        this.executor = executor;
        this.mappedStatement = mappedStatement;
        this.boundSql = boundSql;		
		// 参数和结果集
        this.parameterObject = parameterObject;
        this.resultSetHandler = configuration.newResultSetHandler(executor, mappedStatement, boundSql);
    }

    @Override
    public Statement prepare(Connection connection) throws SQLException {
        Statement statement = null;
        try {
            // 实例化 Statement
            statement = instantiateStatement(connection);
            // 参数设置，可以被抽取，提供配置
            statement.setQueryTimeout(350);
            statement.setFetchSize(10000);
            return statement;
        } catch (Exception e) {
            throw new RuntimeException("Error preparing statement.  Cause: " + e, e);
        }
    }

    protected abstract Statement instantiateStatement(Connection connection) throws SQLException;

}
```
- 在语句处理器基类中，将参数信息、结果信息进行封装处理。不过暂时这里我们还不会做过多的参数处理，包括JDBC字段类型转换等。这部分内容随着我们整个执行器的结构建设完毕后，再进行迭代开发。
- 之后是对 BaseStatementHandler#prepare 方法的处理，包括定义实例化抽象方法，这个方法交由各个具体的实现子类进行处理。包括；SimpleStatementHandler 简单语句处理器和 PreparedStatementHandler 预处理语句处理器。
    - 简单语句处理器只是对 SQL 的最基本执行，没有参数的设置。
    - 预处理语句处理器则是我们在 JDBC 中使用的最多的操作方式，PreparedStatement 设置 SQL，传递参数的设置过程。
### PreparedStatementHandler
```java
public class PreparedStatementHandler extends BaseStatementHandler{

    @Override
    protected Statement instantiateStatement(Connection connection) throws SQLException {
        String sql = boundSql.getSql();
        return connection.prepareStatement(sql);
    }

    @Override
    public void parameterize(Statement statement) throws SQLException {
        PreparedStatement ps = (PreparedStatement) statement;
        ps.setLong(1, Long.parseLong(((Object[]) parameterObject)[0].toString()));
    }

    @Override
    public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
        PreparedStatement ps = (PreparedStatement) statement;
        ps.execute();
        return resultSetHandler.<E> handleResultSets(ps);
    }

}
```
## 在 SqlSession 中使用 Executor
```java
public class DefaultSqlSessionFactory implements SqlSessionFactory {
    private final Configuration configuration;

    public DefaultSqlSessionFactory(Configuration configuration) {
        this.configuration = configuration;
    }

    @Override
    public SqlSession openSession() {
        Transaction tx = null;
        try {
            final Environment environment = configuration.getEnvironment();
            TransactionFactory transactionFactory = environment.getTransactionFactory();
            tx = transactionFactory.newTransaction(configuration.getEnvironment().getDataSource(), TransactionIsolationLevel.READ_COMMITTED, false);
            // 创建执行器
            final Executor executor = configuration.newExecutor(tx);
            // 创建DefaultSqlSession
            return new DefaultSqlSession(configuration, executor);
        } catch (Exception e) {
            try {
                assert tx != null;
                tx.close();
            } catch (SQLException ignore) {
            }
            throw new RuntimeException("Error opening session.  Cause: " + e);
        }
    }
}
```

```java
public class DefaultSqlSession implements SqlSession {

    private Configuration configuration;
    private Executor executor;

    public DefaultSqlSession(Configuration configuration, Executor executor) {
        this.configuration = configuration;
        this.executor = executor;
    }

    @Override
    public <T> T selectOne(String statement, Object parameter) {
        MappedStatement ms = configuration.getMappedStatement(statement);
        List<T> list = executor.query(ms, parameter, Executor.NO_RESULT_HANDLER, ms.getBoundSql());
        return list.get(0);
    }

}
```