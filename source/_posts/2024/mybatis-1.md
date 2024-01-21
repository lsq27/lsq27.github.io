---
title: 读 Mybatis 源码（一）
date: 2024-01-06 10:10:05
tags:
  - 数据库
  - ORM
categories:
  - 读源码
---

作为中国最流行的`ORM`框架，`Mybatis`的设计很经典。我打算分几个模块阅读其源代码。

`session` 模块位于`org.apache.ibatis.session`包下，此模块的提供了用户使用 Mybatis 的主要接口和配置类。主要的类和接口有

- SqlSessionFactoryBuilder
- SqlSessionFactory
- SqlSession
- Configuration

<!-- more -->

## SqlSessionFactoryBuilder

### 作用

创建 `SqlSessionFactory`，支持 `XML`配置或代码配置。创建`SqlSessionFactory`时可以控制的参数有：环境名称，属性。环境名称决定加载哪种环境，包括数据源和事务管理器。属性会被 `MyBatis` 加载，并在配置中使用，可以用 ${属性名} 形式引用配置值。

简单的示例如下：

- 使用`XML`进行配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN" "https://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC" />
            <dataSource type="POOLED">
                <property name="driver" value="${driver}" />
                <property name="url" value="${url}" />
                <property name="username" value="${username}" />
                <property name="password" value="${password}" />
            </dataSource>
        </environment>
        <environment id="production">
            <transactionManager type="MANAGED">
            <!-- ... -->
            <dataSource type="JNDI">
            <!-- ... -->
        </environment>
    </environments>
    <mappers>
        <mapper resource="org/mybatis/example/BlogMapper.xml" />
    </mappers>
</configuration>
```

```java
String resource = "org/mybatis/example/mybatis-config.xml";
InputStream inputStream = Resources.getResourceAsStream(resource);
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream, "development", System.getProperties());
```

- 使用代码进行配置

```java
DataSource dataSource = new PooledDataSource(driver, url, System.getProperties());
TransactionFactory transactionFactory = new JdbcTransactionFactory();
Environment environment = new Environment("development", transactionFactory, dataSource);
Configuration configuration = new Configuration(environment);
configuration.addMapper(BlogMapper.class);
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(configuration);
```

### 方法

`SqlSessionFactoryBuilder` 的方法有如下三类

- 从字符流创建`SqlSessionFactory`

| 方法                                                            |
| --------------------------------------------------------------- |
| build(Reader reader)                                            |
| build(Reader reader, String environment)                        |
| build(Reader reader, Properties properties)                     |
| build(Reader reader, String environment, Properties properties) |

最终通过`build(Reader reader, String environment, Properties properties)`解析`XML`，调用`build(Configuration config)`。

- 从字节流创建`SqlSessionFactory`

| 方法                                                                      |
| ------------------------------------------------------------------------- |
| build(InputStream inputStream)                                            |
| build(InputStream inputStream, String environment)                        |
| build(InputStream inputStream, Properties properties)                     |
| build(InputStream inputStream, String environment, Properties properties) |

最终通过`build(InputStream inputStream, String environment, Properties properties)`解析`XML`，调用`build(Configuration config)`。

- 通过 `Configuration` 创建 `SqlSessionFactory`

| 方法                        |
| --------------------------- |
| build(Configuration config) |

### 源码

```java
/**
 * 根据配置，创建 SqlSessionFactory
 */
public class SqlSessionFactoryBuilder {
    /**
     * 从字符流创建 SqlSessionFactory
     *
     * @param reader      字符流
     * @param environment 环境名称
     * @param properties  属性
     * @return DefaultSqlSessionFactory
     */
    public SqlSessionFactory build(Reader reader) {
        return build(reader, null, null);
    }

    public SqlSessionFactory build(Reader reader, String environment) {
        return build(reader, environment, null);
    }

    public SqlSessionFactory build(Reader reader, Properties properties) {
        return build(reader, null, properties);
    }

    public SqlSessionFactory build(Reader reader, String environment, Properties properties) {
        try {
            // 解析 XML 文件，选择合适的环境，替换属性
            XMLConfigBuilder parser = new XMLConfigBuilder(reader, environment, properties);
            return build(parser.parse());
        } catch (Exception e) {
            throw ExceptionFactory.wrapException("Error building SqlSession.", e);
        } finally {
            ErrorContext.instance().reset();
            try {
                if (reader != null) {
                    reader.close();
                }
            } catch (IOException e) {
            }
        }
    }

    /**
     * 从字节流创建 SqlSessionFactory
     *
     * @param inputStream 字节流
     * @param environment 环境名称
     * @param properties  属性
     * @return DefaultSqlSessionFactory
     */
    public SqlSessionFactory build(InputStream inputStream) {
        return build(inputStream, null, null);
    }

    public SqlSessionFactory build(InputStream inputStream, String environment) {
        return build(inputStream, environment, null);
    }

    public SqlSessionFactory build(InputStream inputStream, Properties properties) {
        return build(inputStream, null, properties);
    }

    public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
        try {
            // 解析 XML 文件，选择合适的环境，替换属性
            XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
            return build(parser.parse());
        } catch (Exception e) {
            throw ExceptionFactory.wrapException("Error building SqlSession.", e);
        } finally {
            ErrorContext.instance().reset();
            try {
                if (inputStream != null) {
                    inputStream.close();
                }
            } catch (IOException e) {
            }
        }
    }

    /**
     * 根据配置创建 DefaultSqlSessionFactory
     *
     * @param config Configuration
     * @return DefaultSqlSessionFactory
     */
    public SqlSessionFactory build(Configuration config) {
        return new DefaultSqlSessionFactory(config);
    }
}
```

## SqlSessionFactory

### 作用

根据配置创建 `SqlSession`，需要 `DataSource` 或`Connection`。创建`SqlSession`时可以控制的参数有：事务等级、自动提交、执行器类型。
默认的 openSession() 方法会创建具备如下特性的 SqlSession：

- 事务作用域将会开启（不自动提交）。
- 从当前环境的 `DataSource` 实例中获取 `Connection` 对象。
- 事务隔离级别将会使用驱动或数据源的默认设置。
- 预处理语句不会被复用，也不会批量处理更新。

### 方法

`SqlSessionFactory` 的方法有如下三类

- 从 `DataSource` 打开会话

| 方法                                                                |
| ------------------------------------------------------------------- |
| openSession()                                                       |
| openSession(boolean autoCommit)                                     |
| openSession(TransactionIsolationLevel level)                        |
| openSession(ExecutorType execType)                                  |
| openSession(ExecutorType execType, boolean autoCommit)              |
| openSession(ExecutorType execType, TransactionIsolationLevel level) |

<mark>事务等级和自动提交的设置是互斥的，打开自动提交等同于关闭事务支持，默认关闭自动提交</mark>

- 从 `Connection` 打开会话

| 方法                                                      |
| --------------------------------------------------------- |
| openSession(Connection connection)                        |
| openSession(ExecutorType execType, Connection connection) |

<mark>Connection 已包含事务等级、自动提交信息，所以额外的参数只有执行器类型</mark>

- 获取配置

| 方法               |
| ------------------ |
| getConfiguration() |

### 源码

```java
/**
 * SqlSession 工厂接口，可以从 Connection 或 DataSource 创建 SqlSession
 */
public interface SqlSessionFactory {
    /**
     * 从 DataSource 打开会话，事务等级和自动提交的设置是互斥的
     *
     * @param execType   执行器类型，默认为 SIMPLE
     * @param level      事务等级
     * @param autoCommit 自动提交
     * @return SqlSession
     */
    SqlSession openSession();

    SqlSession openSession(boolean autoCommit);

    SqlSession openSession(TransactionIsolationLevel level);

    SqlSession openSession(ExecutorType execType);

    SqlSession openSession(ExecutorType execType, boolean autoCommit);

    SqlSession openSession(ExecutorType execType, TransactionIsolationLevel level);

    /**
     * 从 Connection 打开会话，Connection 已包含事务等级、自动提交信息，只需执行器类型
     *
     * @param execType   执行器类型，默认为 SIMPLE
     * @param connection Connection
     * @return DefaultSqlSession
     */
    SqlSession openSession(Connection connection);

    SqlSession openSession(ExecutorType execType, Connection connection);

    /**
     * 获取创建 SqlSessionFactory 的配置
     *
     * @return Configuration
     */
    @Override
    Configuration getConfiguration();
}
```

### 实现类

#### DefaultSqlSessionFactory

默认实现

```java
/**
 * 默认的 SqlSessionFactory 实现，可以从 Connection 或 DataSource 创建 DefaultSqlSession
 */
public class DefaultSqlSessionFactory implements SqlSessionFactory {

    private final Configuration configuration;

    public DefaultSqlSessionFactory(Configuration configuration) {
        this.configuration = configuration;
    }

    /**
     * 从 DataSource 打开会话，事务等级和自动提交的设置是互斥的
     *
     * @param execType   执行器类型，默认为 SIMPLE
     * @param level      事务等级
     * @param autoCommit 自动提交
     * @return DefaultSqlSession
     */
    @Override
    public SqlSession openSession() {
        return openSessionFromDataSource(configuration.getDefaultExecutorType(), null, false);
    }

    @Override
    public SqlSession openSession(boolean autoCommit) {
        return openSessionFromDataSource(configuration.getDefaultExecutorType(), null, autoCommit);
    }

    @Override
    public SqlSession openSession(ExecutorType execType) {
        return openSessionFromDataSource(execType, null, false);
    }

    @Override
    public SqlSession openSession(TransactionIsolationLevel level) {
        return openSessionFromDataSource(configuration.getDefaultExecutorType(), level, false);
    }

    @Override
    public SqlSession openSession(ExecutorType execType, TransactionIsolationLevel level) {
        return openSessionFromDataSource(execType, level, false);
    }

    @Override
    public SqlSession openSession(ExecutorType execType, boolean autoCommit) {
        return openSessionFromDataSource(execType, null, autoCommit);
    }

    /**
     * 从 Connection 打开会话，Connection 已包含事务等级、自动提交信息，只需执行器类型
     *
     * @param execType   执行器类型，默认为 SIMPLE
     * @param connection Connection
     * @return DefaultSqlSession
     */
    @Override
    public SqlSession openSession(Connection connection) {
        return openSessionFromConnection(configuration.getDefaultExecutorType(), connection);
    }

    @Override
    public SqlSession openSession(ExecutorType execType, Connection connection) {
        return openSessionFromConnection(execType, connection);
    }

    /**
     * 获取配置
     *
     * @return Configuration
     */
    @Override
    public Configuration getConfiguration() {
        return configuration;
    }

    /**
     * 从 DataSource 打开会话
     *
     * @param execType   执行器类型，默认为 SIMPLE
     * @param level      事务等级
     * @param autoCommit 自动提交
     * @return DefaultSqlSession
     */
    private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level,
            boolean autoCommit) {
        Transaction tx = null;
        try {
            // 获取环境，获取事务工厂，新建事务，新建执行器，最后实例化 DefaultSqlSession
            final Environment environment = configuration.getEnvironment();
            final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
            tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
            final Executor executor = configuration.newExecutor(tx, execType);
            return new DefaultSqlSession(configuration, executor, autoCommit);
        } catch (Exception e) {
            closeTransaction(tx); // may have fetched a connection so lets call close()
            throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
        } finally {
            ErrorContext.instance().reset();
        }
    }

    /**
     * 从 Connection 打开会话
     *
     * @param execType   执行器类型，默认为 SIMPLE
     * @param connection Connection
     * @return DefaultSqlSession
     */
    private SqlSession openSessionFromConnection(ExecutorType execType, Connection connection) {
        try {
            boolean autoCommit;
            try {
                autoCommit = connection.getAutoCommit();
            } catch (SQLException e) {
                // Failover to true, as most poor drivers
                // or databases won't support transactions
                autoCommit = true;
            }
            // 获取环境，获取事务工厂，新建事务，新建执行器，最后实例化 DefaultSqlSession
            final Environment environment = configuration.getEnvironment();
            final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
            final Transaction tx = transactionFactory.newTransaction(connection);
            final Executor executor = configuration.newExecutor(tx, execType);
            return new DefaultSqlSession(configuration, executor, autoCommit);
        } catch (Exception e) {
            throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
        } finally {
            ErrorContext.instance().reset();
        }
    }

    /**
     * 从环境获取事务工厂，没有就实例化一个 ManagedTransactionFactory
     *
     * @param environment
     * @return
     */
    private TransactionFactory getTransactionFactoryFromEnvironment(Environment environment) {
        if (environment == null || environment.getTransactionFactory() == null) {
            return new ManagedTransactionFactory();
        }
        return environment.getTransactionFactory();
    }

    /**
     * 关闭事务，只用于从 DataSource 打开会话
     *
     * @param tx
     */
    private void closeTransaction(Transaction tx) {
        if (tx != null) {
            try {
                tx.close();
            } catch (SQLException ignore) {
            }
        }
    }
}
```

#### SqlSessionManager

该类同时实现了`SqlSession`接口且主要作为`SqlSession`使用，见`SqlSession`章节。

## SqlSession

### 作用

在使用 `Mybatis`时，最重要的类莫过于 `SqlSession`。通过该接口中的操作，开发者可以执行查增改删的 `Statement` 并获取多种类型的结果、执行批量、管理事务、管理会话、获取 Mapper。

### 方法

SqlSession 的方法有如下几类

- 增删查改

| 方法名       | 功能               |
| ------------ | ------------------ |
| selectOne    | 获取单行数据       |
| selectList   | 获取列表数据       |
| selectMap    | 获取 Map 数据      |
| selectCursor | 获取游标数据       |
| select       | 获取并处理每行数据 |
| insert       | 插入数据           |
| update       | 更新数据           |
| delete       | 删除数据           |

- 事务和批量操作

| 方法命          | 功能                                                                                                     |
| --------------- | -------------------------------------------------------------------------------------------------------- |
| commit          | 执行缓存在 JDBC 驱动类中的批量更新语句并提交事务                                                         |
| rollback        | 清除缓存在 JDBC 驱动类中的批量更新语句并回滚事务                                                         |
| flushStatements | 将  `ExecutorType`  设置为  `ExecutorType.BATCH`  时，使用这个方法执行缓存在 JDBC 驱动类中的批量更新语句 |

- 关闭会话

| 方法命 | 功能     |
| ------ | -------- |
| close  | 关闭会话 |

任何打开的 `session`，都要保证被妥善关闭。最佳代码模式如下：

```java
try (SqlSession session = sqlSessionFactory.openSession()) {
    // 业务逻辑
    session.insert(...);
    session.update(...);
    session.delete(...);
    session.commit();
}
```

- 关闭会话

| 方法命           | 功能               |
| ---------------- | ------------------ |
| close            | 关闭会话           |
| clearCache       | 清理本地会话缓存   |
| getConfiguration | 获取当前配置       |
| getConnection    | 获取内部数据库连接 |

- 获取 Mapper

| 方法命    | 功能        |
| --------- | ----------- |
| getMapper | 获取 Mapper |

### 源码

```java
/**
 * Mybatis 的主要接口，通过该接口执行命令，获取 mapper，管理事务
 */
public interface SqlSession extends Closeable {

    /**
     * 获取单行数据映射的对象
     *
     * @param <T>       返回对象类型
     * @param statement 唯一定位需要执行的 statement
     * @return 映射的对象
     */
    <T> T selectOne(String statement);

    /**
     * 获取单行数据映射的对象
     *
     * @param parameter statement 的参数
     * @return 映射的对象
     */
    <T> T selectOne(String statement, Object parameter);

    /**
     * 获取多行数据映射的对象列表
     *
     * @param <E> 列表元素类型
     * @return 映射的对象列表
     */
    <E> List<E> selectList(String statement);

    /**
     * 获取多行数据映射的对象列表
     *
     * @return 映射的对象列表
     */
    <E> List<E> selectList(String statement, Object parameter);

    /**
     * 获取多行数据映射的对象列表，分页
     *
     * @param rowBounds 分页参数
     * @return 映射的对象列表
     */
    <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds);

    /**
     * 在 selectList 的基础上，以 mapKey 属性为 key，将 List 转换为 Map
     *
     * @param <K>    返回的 Map 的 key 类型
     * @param <V>    返回的 Map 的 value 类型
     * @param mapKey 作为 key 的属性名
     * @return Map
     */
    <K, V> Map<K, V> selectMap(String statement, String mapKey);

    /**
     * 在 selectList 的基础上，以 mapKey 属性为 key，将 List 转换为 Map
     *
     * @return Map
     */
    <K, V> Map<K, V> selectMap(String statement, Object parameter, String mapKey);

    /**
     * 在 selectList 的基础上，以 mapKey 属性为 key，将 List 转换为 Map，分页
     *
     * @return Map
     */
    <K, V> Map<K, V> selectMap(String statement, Object parameter, String mapKey, RowBounds rowBounds);

    /**
     * 使用游标对 selectList 结果懒加载
     *
     * @return 游标
     */
    <T> Cursor<T> selectCursor(String statement);

    /**
     * 使用游标对 selectList 结果懒加载
     *
     * @return 游标
     */
    <T> Cursor<T> selectCursor(String statement, Object parameter);

    /**
     * 使用游标对 selectList 结果懒加载，分页
     *
     * @return 游标
     */
    <T> Cursor<T> selectCursor(String statement, Object parameter, RowBounds rowBounds);

    /**
     * 使用 ResultHandler 对每行映射的对象进行处理
     *
     * @param handler 处理每行对象
     */
    void select(String statement, ResultHandler handler);

    /**
     * 使用 ResultHandler 对每行映射的对象进行处理
     */
    void select(String statement, Object parameter, ResultHandler handler);

    /**
     * 使用 ResultHandler 对每行映射的对象进行处理，分页
     */
    void select(String statement, Object parameter, RowBounds rowBounds, ResultHandler handler);

    /**
     * 执行插入
     *
     * @return 插入的数量
     */
    int insert(String statement);

    /**
     * Execute an insert statement with the given parameter object. Any generated
     * autoincrement values or selectKey
     * entries will modify the given parameter object properties. Only the number of
     * rows affected will be returned.
     *
     * @return 插入的数量
     */
    int insert(String statement, Object parameter);

    /**
     * 执行更新
     *
     * @return 更新的数量
     */
    int update(String statement);

    /**
     * 执行更新
     *
     * @return 更新的数量
     */
    int update(String statement, Object parameter);

    /**
     * 执行删除
     *
     * @return 删除的数量
     */
    int delete(String statement);

    /**
     * 执行删除
     *
     * @return 删除的数量
     */
    int delete(String statement, Object parameter);

    /**
     * 刷新 batch statements 并提交数据库连接
     */
    void commit();

    /**
     * 刷新 batch statements 并提交数据库连接
     *
     * @param force 强制连接提交
     */
    void commit(boolean force);

    /**
     * 忽略等待的 batch statements，回滚连接。
     */
    void rollback();

    /**
     * 忽略等待的 batch statements，回滚连接。
     *
     * @param force 强制连接回滚
     */
    void rollback(boolean force);

    /**
     * 刷新批量statement
     *
     * @return 更新记录的 batchResult list
     *
     * @since 3.0.6
     */
    List<BatchResult> flushStatements();

    /**
     * 关闭会话
     */
    @Override
    void close();

    /**
     * 清除本地会话缓存
     */
    void clearCache();

    /**
     * 获取当前配置
     *
     * @return 配置
     */
    Configuration getConfiguration();

    /**
     * 获取 mapper
     *
     * @param <T>  mapper 类型
     * @param type mapper 接口类
     *
     * @return 绑定到当前 SqlSession 的 mapper
     */
    <T> T getMapper(Class<T> type);

    /**
     * 获取内部的数据库连接
     *
     * @return 连接
     */
    Connection getConnection();
}
```

### 实现类

#### DefaultSqlSession

默认实现，不是线程安全的。

```java
/**
 * SqlSession 默认实现，不是线程安全的
 */
public class DefaultSqlSession implements SqlSession {
    // 配置
    private final Configuration configuration;
    // 执行器
    private final Executor executor;
    // 自动提交
    private final boolean autoCommit;
    // 是否有脏数据，调用 insert、update 或 delete 会置为 true
    private boolean dirty;
    // 保存游标，此处何时回收？
    private List<Cursor<?>> cursorList;

    /**
     * 构造器
     *
     * @param configuration 配置
     * @param executor      执行器
     * @param autoCommit    自动提交
     */
    public DefaultSqlSession(Configuration configuration, Executor executor, boolean autoCommit) {
        this.configuration = configuration;
        this.executor = executor;
        this.dirty = false;
        this.autoCommit = autoCommit;
    }

    /**
     * 构造器，默认不自动提交
     *
     * @param configuration 配置
     * @param executor      执行器
     */
    public DefaultSqlSession(Configuration configuration, Executor executor) {
        this(configuration, executor, false);
    }

    @Override
    public <T> T selectOne(String statement, Object parameter) {
        // 查询列表，如果列表很大此处会出现性能问题
        List<T> list = this.selectList(statement, parameter);
        // 只有一条就返回
        if (list.size() == 1) {
            return list.get(0);
        }
        // 如果超出一条会抛异常，零条返回 null
        if (list.size() > 1) {
            throw new TooManyResultsException(
                    "Expected one result (or null) to be returned by selectOne(), but found: " + list.size());
        } else {
            return null;
        }
    }

    @Override
    public <K, V> Map<K, V> selectMap(String statement, Object parameter, String mapKey, RowBounds rowBounds) {
        // 查询列表
        final List<? extends V> list = selectList(statement, parameter, rowBounds);
        final DefaultMapResultHandler<K, V> mapResultHandler = new DefaultMapResultHandler<>(mapKey,
                configuration.getObjectFactory(), configuration.getObjectWrapperFactory(),
                configuration.getReflectorFactory());
        final DefaultResultContext<V> context = new DefaultResultContext<>();
        for (V o : list) {
            context.nextResultObject(o);
            mapResultHandler.handleResult(context);
        }
        return mapResultHandler.getMappedResults();
    }

    @Override
    public <T> Cursor<T> selectCursor(String statement, Object parameter, RowBounds rowBounds) {
        try {
            MappedStatement ms = configuration.getMappedStatement(statement);
            dirty |= ms.isDirtySelect();
            Cursor<T> cursor = executor.queryCursor(ms, wrapCollection(parameter), rowBounds);
            registerCursor(cursor);
            return cursor;
        } catch (Exception e) {
            throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
        } finally {
            ErrorContext.instance().reset();
        }
    }

    private <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds, ResultHandler handler) {
        try {
            MappedStatement ms = configuration.getMappedStatement(statement);
            dirty |= ms.isDirtySelect();
            return executor.query(ms, wrapCollection(parameter), rowBounds, handler);
        } catch (Exception e) {
            throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
        } finally {
            ErrorContext.instance().reset();
        }
    }

    @Override
    public void select(String statement, Object parameter, RowBounds rowBounds, ResultHandler handler) {
        selectList(statement, parameter, rowBounds, handler);
    }

    @Override
    public int insert(String statement, Object parameter) {
        return update(statement, parameter);
    }

    @Override
    public int update(String statement, Object parameter) {
        try {
            dirty = true;
            MappedStatement ms = configuration.getMappedStatement(statement);
            return executor.update(ms, wrapCollection(parameter));
        } catch (Exception e) {
            throw ExceptionFactory.wrapException("Error updating database.  Cause: " + e, e);
        } finally {
            ErrorContext.instance().reset();
        }
    }

    @Override
    public int delete(String statement, Object parameter) {
        return update(statement, parameter);
    }

    // ...

    @Override
    public void commit(boolean force) {
        try {
            executor.commit(isCommitOrRollbackRequired(force));
            dirty = false;
        } catch (Exception e) {
            throw ExceptionFactory.wrapException("Error committing transaction.  Cause: " + e, e);
        } finally {
            ErrorContext.instance().reset();
        }
    }

    // ...

    /**
     * 关闭游标
     */
    private void closeCursors() {
        if (cursorList != null && !cursorList.isEmpty()) {
            for (Cursor<?> cursor : cursorList) {
                try {
                    cursor.close();
                } catch (IOException e) {
                    throw ExceptionFactory.wrapException("Error closing cursor.  Cause: " + e, e);
                }
            }
            cursorList.clear();
        }
    }

    /**
     * 注册游标
     *
     * @param <T>    返回对象类型
     * @param cursor 游标
     */
    private <T> void registerCursor(Cursor<T> cursor) {
        if (cursorList == null) {
            cursorList = new ArrayList<>();
        }
        cursorList.add(cursor);
    }

    /**
     * 是否需要提交或回滚，关闭自动提交且有脏数据时需要
     *
     * @param force 是否强制返回 true
     * @return 是否需要提交或回滚
     */
    private boolean isCommitOrRollbackRequired(boolean force) {
        return !autoCommit && dirty || force;
    }

    /**
     * 将 Collection 和数组包装成 ParamMap
     *
     * @param object
     * @return
     */
    private Object wrapCollection(final Object object) {
        return ParamNameResolver.wrapToMapIfCollection(object, null);
    }
}
```

#### SqlSessionManager

代理 SqlSession

```java
/**
 * SqlSession 管理器，内部有线程私有的 SqlSession，如果 SqlSession 为空，临时开打一个 SqlSession 执行命令
 */
public class SqlSessionManager implements SqlSessionFactory, SqlSession {
    private final SqlSessionFactory sqlSessionFactory;
    // SqlSession 代理实例
    private final SqlSession sqlSessionProxy;
    // 线程私有的 SqlSession
    private final ThreadLocal<SqlSession> localSqlSession = new ThreadLocal<>();

    /**
     * 私有构造函数
     */
    private SqlSessionManager(SqlSessionFactory sqlSessionFactory) {
        this.sqlSessionFactory = sqlSessionFactory;
        this.sqlSessionProxy = (SqlSession) Proxy.newProxyInstance(SqlSessionFactory.class.getClassLoader(),
                new Class[] { SqlSession.class }, new SqlSessionInterceptor());
    }

    /**
     * 新建 SqlSessionManager
     */
    public static SqlSessionManager newInstance(Reader reader) {
        return new SqlSessionManager(new SqlSessionFactoryBuilder().build(reader, null, null));
    }

    // ...

    // 启动内部 SqlSession
    public void startManagedSession() {
        this.localSqlSession.set(openSession());
    }

    // ...

    /**
     * 内部 SqlSession 是否启动
     */
    public boolean isManagedSessionStarted() {
        return this.localSqlSession.get() != null;
    }

    @Override
    public SqlSession openSession() {
        return sqlSessionFactory.openSession();
    }

    // ...

    @Override
    public Configuration getConfiguration() {
        return sqlSessionFactory.getConfiguration();
    }

    /**
     * 使用 sqlSessionProxy 执行命令
     */
    @Override
    public <T> T selectOne(String statement) {
        return sqlSessionProxy.selectOne(statement);
    }

    // ...

    /**
     * 使用 Configuration 获取 Mapper
     */
    @Override
    public <T> T getMapper(Class<T> type) {
        return getConfiguration().getMapper(type, this);
    }

    /**
     * 使用 localSqlSession 管理会话
     */
    @Override
    public Connection getConnection() {
        final SqlSession sqlSession = localSqlSession.get();
        if (sqlSession == null) {
            throw new SqlSessionException("Error:  Cannot get connection.  No managed session is started.");
        }
        return sqlSession.getConnection();
    }

    // ...

    /**
     * 拦截 sqlSessionProxy 方法调用，使用 localSqlSession 或临时生成一个 SqlSession
     */
    private class SqlSessionInterceptor implements InvocationHandler {
        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            // 使用 localSqlSession 执行方法
            final SqlSession sqlSession = SqlSessionManager.this.localSqlSession.get();
            if (sqlSession != null) {
                try {
                    return method.invoke(sqlSession, args);
                } catch (Throwable t) {
                    throw ExceptionUtil.unwrapThrowable(t);
                }
            }
            // 临时生成一个 SqlSession 执行方法
            try (SqlSession autoSqlSession = openSession()) {
                try {
                    final Object result = method.invoke(autoSqlSession, args);
                    autoSqlSession.commit();
                    return result;
                } catch (Throwable t) {
                    autoSqlSession.rollback();
                    throw ExceptionUtil.unwrapThrowable(t);
                }
            }
        }
    }
}
```

## Configuration

### 作用

构造`SqlSession`所需的配置均在该类中，`Configuration` 类包含所有配置开关，所有对`Mybatis`的配置最终均保存在类中。

```java
DataSource dataSource = new PooledDataSource(driver, url, System.getProperties());
TransactionFactory transactionFactory = new JdbcTransactionFactory();
Environment environment = new Environment("development", transactionFactory, dataSource);

Configuration configuration = new Configuration(environment);
configuration.setLazyLoadingEnabled(true);
configuration.setEnhancementEnabled(true);
configuration.getTypeAliasRegistry().registerAlias(Blog.class);
configuration.getTypeAliasRegistry().registerAlias(Post.class);
configuration.getTypeAliasRegistry().registerAlias(Author.class);
configuration.addMapper(BlogMapper.class);
configuration.addMapper(AuthorMapper.class);
```

### 方法

### 源码

该类相当大，只截取部分源码

```java
/**
 * 根据配置，创建 SqlSessionFactory
 */
public class Configuration {
    // 全局性地开启或关闭所有映射器配置文件中已配置的任何缓存。
    protected boolean cacheEnabled = true;
    // 配置默认的执行器。SIMPLE 就是普通的执行器；REUSE 执行器会重用预处理语句（PreparedStatement）； BATCH
    // 执行器不仅重用语句还会执行批量更新。
    protected ExecutorType defaultExecutorType = ExecutorType.SIMPLE;
    // 是否开启驼峰命名自动映射，即从经典数据库列名 A_COLUMN 映射到经典 Java 属性名 aColumn。
    protected boolean mapUnderscoreToCamelCase;
    // MyBatis 利用本地缓存机制（Local Cache）防止循环引用和加速重复的嵌套查询。 默认值为 SESSION，会缓存一个会话中执行的所有查询。
    // 若设置值为 STATEMENT，本地缓存将仅用于执行语句，对相同 SqlSession 的不同查询将不会进行缓存。
    protected LocalCacheScope localCacheScope = LocalCacheScope.SESSION;
    protected final InterceptorChain interceptorChain = new InterceptorChain();
    protected final Map<String, MappedStatement> mappedStatements = new StrictMap<MappedStatement>(
            "Mapped Statements collection")
            .conflictMessageProducer((savedValue, targetValue) -> ". please check " + savedValue.getResource() + " and "
                    + targetValue.getResource());

    /**
     * 创建默认执行器
     *
     * @param transaction 事务
     * @return 执行器
     */
    public Executor newExecutor(Transaction transaction) {
        return newExecutor(transaction, defaultExecutorType);
    }

    /**
     * 创建指定类型执行器
     *
     * @param transaction
     * @param executorType
     * @return
     */
    public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
        executorType = executorType == null ? defaultExecutorType : executorType;
        Executor executor;
        if (ExecutorType.BATCH == executorType) {
            executor = new BatchExecutor(this, transaction);
        } else if (ExecutorType.REUSE == executorType) {
            executor = new ReuseExecutor(this, transaction);
        } else {
            executor = new SimpleExecutor(this, transaction);
        }
        if (cacheEnabled) {
            executor = new CachingExecutor(executor);
        }
        return (Executor) interceptorChain.pluginAll(executor);
    }

    public void addInterceptor(Interceptor interceptor) {
        interceptorChain.addInterceptor(interceptor);
    }

    public void addMappers(String packageName, Class<?> superType) {
        mapperRegistry.addMappers(packageName, superType);
    }

    public void addMappers(String packageName) {
        mapperRegistry.addMappers(packageName);
    }

    public <T> void addMapper(Class<T> type) {
        mapperRegistry.addMapper(type);
    }

    public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
        return mapperRegistry.getMapper(type, sqlSession);
    }

    public void addMappedStatement(MappedStatement ms) {
        mappedStatements.put(ms.getId(), ms);
    }

    public MappedStatement getMappedStatement(String id) {
        return this.getMappedStatement(id, true);
    }

    public MappedStatement getMappedStatement(String id, boolean validateIncompleteStatements) {
        if (validateIncompleteStatements) {
            buildAllStatements();
        }
        return mappedStatements.get(id);
    }

    /*
     * Parses all the unprocessed statement nodes in the cache. It is recommended to
     * call this method once all the mappers
     * are added as it provides fail-fast statement validation.
     */
    protected void buildAllStatements() {
        parsePendingResultMaps();
        if (!incompleteCacheRefs.isEmpty()) {
            synchronized (incompleteCacheRefs) {
                incompleteCacheRefs.removeIf(x -> x.resolveCacheRef() != null);
            }
        }
        if (!incompleteStatements.isEmpty()) {
            synchronized (incompleteStatements) {
                incompleteStatements.removeIf(x -> {
                    x.parseStatementNode();
                    return true;
                });
            }
        }
        if (!incompleteMethods.isEmpty()) {
            synchronized (incompleteMethods) {
                incompleteMethods.removeIf(x -> {
                    x.resolve();
                    return true;
                });
            }
        }
    }
}
```

## 参考

- [mybatis – MyBatis 3 | Java API](https://mybatis.org/mybatis-3/zh_CN/java-api.html)
