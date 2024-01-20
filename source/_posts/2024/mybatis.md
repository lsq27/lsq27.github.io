---
title: 读 Mybatis 源码
date: 2024-01-20 16:25:47
tags:
  - 数据库
categories:
  - 读源码
---

通过编程方式使用 Mybatis（不使用 Spring）时，最重要的类莫过于 SqlSession。通过该接口中的操作，开发者可以实现执行查增改删的 Statement 并获取多种类型的结果、执行批量、管理事务、管理会话、获取 Mapper。

## 方法

SqlSession 的方法有如下四类

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

| 方法命          | 功能                                                               |
| --------------- | ------------------------------------------------------------------ |
| commit          | flush batch statements and commit                                  |
| rollback        | discard pending batch statements and roll database connection back |
| flushStatements | flush batch statements                                             |

- 管理会话

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

## 定义

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

## 实现类

### DefaultSqlSession

默认实现，不是线程安全的。

#### 源码

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

### SqlSessionManager

代理 SqlSession

#### 源码

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
