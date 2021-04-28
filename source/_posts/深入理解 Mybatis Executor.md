---
title: 深入理解 Mybatis Executor
tags:
  - Mybatis
categories:
  - - Tech
abbrlink: 1b8eb0fb
date: 2021-04-29 00:26:56
---

<img src="https://mybatis.org/images/mybatis-logo.png"/>
<!-- <img src="images/images.jfif"/> -->
<!-- ![how-to-install-mysql-on-centos](https://cdn.jsdelivr.net/gh/LiLittleCat/PicBed/images/blog/how-to-install-mysql-on-centos.png) -->
MyBatis 是一款优秀的持久层框架，它支持自定义 SQL、存储过程以及高级映射。MyBatis 免除了几乎所有的 JDBC 代码以及设置参数和获取结果集的工作。MyBatis 可以通过简单的 XML 或注解来配置和映射原始类型、接口和 Java POJO（Plain Old Java Objects，普通老式 Java 对象）为数据库中的记录。

本文介绍了 MyBatis 执行过程和 Executor 的执行体系相关内容。
<!-- more -->

## Mybatis 执行过程

抛开通过 Mapper 接口生成动态代理，MyBatis 的一次执行过程可大致分为以下几个部分：

1. 获取门面会话
2. 执行器逻辑
3. JDBC 处理器逻辑

如图所示：

![image-20210429001446963](https://cdn.jsdelivr.net/gh/LiLittleCat/PicBed/images/mybatis/image-20210429001446963.png)

SqlSession 中提供了基本 API：增、删、改、查

![image-20210428202606059](https://cdn.jsdelivr.net/gh/LiLittleCat/PicBed/images/mybatis/image-20210428202606059.png)

其中 `select()` 方法有很多，他们的区别主要在参数和返回结果上。

![image-20210428203039799](https://cdn.jsdelivr.net/gh/LiLittleCat/PicBed/images/mybatis/image-20210428203039799.png)

返回结果上，有 `void`，游标 `Cursor<T>`，`List<E>`，`Map<K, V>`，一条数据 `T`。参数上，`String` 就是 StatementId，Mybatis 中所有操作（增删改查）都有一个 Id，通过 Id 找到 SQL 映射，`Object` 就是 SQL 中的参数，`ResultHandler` 对结果集进行处理，`RowBound` 用于设置返回的范围，用于进行分页。多个 `select()` 方法重载的设计用意是为了方便调用，这种方便调用进行的重载就是`门面模式`。

除了基本 API 之外，还有辅助 API，即提交和关闭会话。会话中可执行多个 SQL，一个会话只能被一个线程调用，线程使用完 SqlSession 后应该立马把它关闭掉，但一个线程可调用多个会话。因此，SqlSession 不能跨线程使用，也即它不是线程安全的。

在 MyBatis 中，DefaultSqlSession 继承了 SqlSession 接口，是 SqlSession 的默认实现，在该实现类中，包含了两个重要对象 **Configuration** 和 **Executor**。

```java
public class DefaultSqlSession implements SqlSession {

  private final Configuration configuration;
  private final Executor executor;

  private final boolean autoCommit;
  private boolean dirty;
  private List<Cursor<?>> cursorList;

  public DefaultSqlSession(Configuration configuration, Executor executor, boolean autoCommit) {
    this.configuration = configuration;
    this.executor = executor;
    this.dirty = false;
    this.autoCommit = autoCommit;
  }

  public DefaultSqlSession(Configuration configuration, Executor executor) {
    this(configuration, executor, false);
  }
  
  ...
  
}
```

SqlSession 只提供 SQL 会话，具体执行交给 Executor。Executor 也提供了基本功能，其中包括改、查、缓存维护和事务管理。还提供了辅助 API，包括提交、关闭执行器和批处理刷新。

![image-20210428212838383](https://cdn.jsdelivr.net/gh/LiLittleCat/PicBed/images/mybatis/image-20210428212838383.png)

Executor 本身也不会去执行 SQL，执行 SQL 交给了 StatementHandler，StatementHandler 本质上使用了 JDBC 中的 Statement 进行参数处理、执行 SQL 和处理结果。

其中 1 个 SqlSession 对应 1 个 Executor ， 1 个 Executor 对应 n 个 StatementHandler，即执行多少条 SQL 就有多少个 StatementHandler。它们都不能够跨线程调用。

## Executor 执行器体系

Executor 的继承体系：

![image-20210428213919624](https://cdn.jsdelivr.net/gh/LiLittleCat/PicBed/images/mybatis/BaseExecutor.svg)



1. BaseExecutor：基础执行器，是一个抽象类，提供了关于 SimpleExecutor、ReuseExecutor、BatchExecutor 共性相关的功能实现，此外，BaseExecutor 还与 MyBatis 的一级缓存和事务管理相关
2. SimpleExecutor：简单执行器，每次都会创建一个新的 Statement（默认是 PreparedStatement）
3. ReuseExecutor：重用执行器，相同的 SQL 语句会使用同一个 Statement
4. BatchExecutor：批处理执行器，批处理提交修改，必须执行 flushStatement 才会生效
5. CachingExecutor：缓存执行器，与 MyBatis 二级缓存相关

### BaseExecutor

BaseExecutor 作为 SimpleExecutor、ReuseExecutor、BatchExecutor 的父类，实现了子类中的一些共性功能。

主要方法：

```java
/** 事务管理接口、定义了与JDBC执行相关的功能，如获取连接、提交、回滚、关闭连接等操作 */
protected Transaction transaction;
protected Executor wrapper;
/** 下方三个对象均是与一级缓存相关的属性, 作用域为 BaseExecutor 的生命周期范围 */
protected ConcurrentLinkedQueue<DeferredLoad> deferredLoads;
protected PerpetualCache localCache;
protected PerpetualCache localOutputParameterCache;
protected Configuration configuration;
/** 与动态sql中的嵌套子查询相关 */
protected int queryStack;
private boolean closed;

protected BaseExecutor(Configuration configuration, Transaction transaction) {
    this.transaction = transaction;
    this.deferredLoads = new ConcurrentLinkedQueue<>();
    this.localCache = new PerpetualCache("LocalCache");
    this.localOutputParameterCache = new PerpetualCache("LocalOutputParameterCache");
    this.closed = false;
    this.configuration = configuration;
    this.wrapper = this;
}

@Override
public Transaction getTransaction() {
    if (closed) {
        throw new ExecutorException("Executor was closed.");
    }
    return transaction;
}

/**
   * 关闭当前对象、关闭之前判断是否需要回滚
   */
@Override
public void close(boolean forceRollback) {
    try {
        try {
            rollback(forceRollback);
        } finally {
            if (transaction != null) {
                transaction.close();
            }
        }
    } catch (SQLException e) {
        // Ignore. There's nothing that can be done at this point.
        log.warn("Unexpected exception on closing transaction.  Cause: " + e);
    } finally {
        transaction = null;
        deferredLoads = null;
        localCache = null;
        localOutputParameterCache = null;
        closed = true;
    }
}

@Override
public boolean isClosed() {
    return closed;
}

@Override
public int update(MappedStatement ms, Object parameter) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing an update").object(ms.getId());
    if (closed) {
        throw new ExecutorException("Executor was closed.");
    }
    clearLocalCache();
    return doUpdate(ms, parameter);
}

@Override
public List<BatchResult> flushStatements() throws SQLException {
    return flushStatements(false);
}

public List<BatchResult> flushStatements(boolean isRollBack) throws SQLException {
    if (closed) {
        throw new ExecutorException("Executor was closed.");
    }
    return doFlushStatements(isRollBack);
}

@Override
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    BoundSql boundSql = ms.getBoundSql(parameter);
    CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
    return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
}

@SuppressWarnings("unchecked")
@Override
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
    if (closed) {
        throw new ExecutorException("Executor was closed.");
    }
    if (queryStack == 0 && ms.isFlushCacheRequired()) {
        // 如果不是嵌套查询，并且配置了flushCache为true，清空一级缓存
        clearLocalCache();
    }
    List<E> list;
    try {
        queryStack++;
        // 如果resultHandler为空，尝试从一级缓存中获取数据
        list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
        if (list != null) {
            // 与存储过程相关（CallableStatement）
            handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
        } else {
            // 一级缓存中没有对应数据，从数据库查询
            list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
        }
    } finally {
        queryStack--;
    }
    if (queryStack == 0) {
        // 延迟装载相关逻辑
        for (DeferredLoad deferredLoad : deferredLoads) {
            deferredLoad.load();
        }
        // issue #601
        deferredLoads.clear();
        if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
            // 如果全局配置的一级缓存作用域为Statement，清除本地缓存，因此一级缓存无法被关闭，只是减少范围至每一个Statement
            // issue #482
            clearLocalCache();
        }
    }
    return list;
}

@Override
public <E> Cursor<E> queryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds) throws SQLException {
    BoundSql boundSql = ms.getBoundSql(parameter);
    return doQueryCursor(ms, parameter, rowBounds, boundSql);
}

@Override
public void deferLoad(MappedStatement ms, MetaObject resultObject, String property, CacheKey key, Class<?> targetType) {
    if (closed) {
        throw new ExecutorException("Executor was closed.");
    }
    DeferredLoad deferredLoad = new DeferredLoad(resultObject, property, key, localCache, configuration, targetType);
    if (deferredLoad.canLoad()) {
        deferredLoad.load();
    } else {
        deferredLoads.add(new DeferredLoad(resultObject, property, key, localCache, configuration, targetType));
    }
}
/** 创建CacheKey逻辑，主要与mappedStatement的ID，参数，分页参数和sql有关 */
@Override
public CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql) {
    if (closed) {
        throw new ExecutorException("Executor was closed.");
    }
    CacheKey cacheKey = new CacheKey();
    cacheKey.update(ms.getId());
    cacheKey.update(rowBounds.getOffset());
    cacheKey.update(rowBounds.getLimit());
    cacheKey.update(boundSql.getSql());
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    TypeHandlerRegistry typeHandlerRegistry = ms.getConfiguration().getTypeHandlerRegistry();
    // mimic DefaultParameterHandler logic
    for (ParameterMapping parameterMapping : parameterMappings) {
        if (parameterMapping.getMode() != ParameterMode.OUT) {
            Object value;
            String propertyName = parameterMapping.getProperty();
            if (boundSql.hasAdditionalParameter(propertyName)) {
                value = boundSql.getAdditionalParameter(propertyName);
            } else if (parameterObject == null) {
                value = null;
            } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
                value = parameterObject;
            } else {
                MetaObject metaObject = configuration.newMetaObject(parameterObject);
                value = metaObject.getValue(propertyName);
            }
            cacheKey.update(value);
        }
    }
    if (configuration.getEnvironment() != null) {
        // issue #176
        cacheKey.update(configuration.getEnvironment().getId());
    }
    return cacheKey;
}

@Override
public boolean isCached(MappedStatement ms, CacheKey key) {
    return localCache.getObject(key) != null;
}

@Override
public void commit(boolean required) throws SQLException {
    if (closed) {
        throw new ExecutorException("Cannot commit, transaction is already closed");
    }
    clearLocalCache();
    flushStatements();
    if (required) {
        transaction.commit();
    }
}

@Override
public void rollback(boolean required) throws SQLException {
    if (!closed) {
        try {
            clearLocalCache();
            flushStatements(true);
        } finally {
            if (required) {
                transaction.rollback();
            }
        }
    }
}
/**
   * 清空本地缓存、在BaseExecutor中，update、query、commit、rollback方法中，均有调用回滚的逻辑
   */
@Override
public void clearLocalCache() {
    if (!closed) {
        localCache.clear();
        localOutputParameterCache.clear();
    }
}
/** 执行更新操作 */
protected abstract int doUpdate(MappedStatement ms, Object parameter) throws SQLException;
/** 执行Statement，填充结果、与批量执行相关 */
protected abstract List<BatchResult> doFlushStatements(boolean isRollback) throws SQLException;
/** 执行查询操作 */
protected abstract <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql)
    throws SQLException;

protected abstract <E> Cursor<E> doQueryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds, BoundSql boundSql)
    throws SQLException;
/** 关闭Statement对象 */
protected void closeStatement(Statement statement) {
    if (statement != null) {
        try {
            statement.close();
        } catch (SQLException e) {
            // ignore
        }
    }
}

/**
   * Apply a transaction timeout.
   *
   * @param statement
   *          a current statement
   * @throws SQLException
   *           if a database access error occurs, this method is called on a closed <code>Statement</code>
   * @since 3.4.0
   * @see StatementUtil#applyTransactionTimeout(Statement, Integer, Integer)
   */
protected void applyTransactionTimeout(Statement statement) throws SQLException {
    StatementUtil.applyTransactionTimeout(statement, statement.getQueryTimeout(), transaction.getTimeout());
}

private void handleLocallyCachedOutputParameters(MappedStatement ms, CacheKey key, Object parameter, BoundSql boundSql) {
    if (ms.getStatementType() == StatementType.CALLABLE) {
        final Object cachedParameter = localOutputParameterCache.getObject(key);
        if (cachedParameter != null && parameter != null) {
            final MetaObject metaCachedParameter = configuration.newMetaObject(cachedParameter);
            final MetaObject metaParameter = configuration.newMetaObject(parameter);
            for (ParameterMapping parameterMapping : boundSql.getParameterMappings()) {
                if (parameterMapping.getMode() != ParameterMode.IN) {
                    final String parameterName = parameterMapping.getProperty();
                    final Object cachedValue = metaCachedParameter.getValue(parameterName);
                    metaParameter.setValue(parameterName, cachedValue);
                }
            }
        }
    }
}
/** 查询数据库 */
private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    List<E> list;
    // 插入key和占位符，为一级缓存做准备
    localCache.putObject(key, EXECUTION_PLACEHOLDER);
    try {
        // 调用子类实现的方法查询
        list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
    } finally {
        localCache.removeObject(key);
    }
    // 查询结果保存到缓存中
    localCache.putObject(key, list);
    if (ms.getStatementType() == StatementType.CALLABLE) {
        localOutputParameterCache.putObject(key, parameter);
    }
    return list;
}

protected Connection getConnection(Log statementLog) throws SQLException {
    Connection connection = transaction.getConnection();
    if (statementLog.isDebugEnabled()) {
        return ConnectionLogger.newInstance(connection, statementLog, queryStack);
    } else {
        return connection;
    }
}

@Override
public void setExecutorWrapper(Executor wrapper) {
    this.wrapper = wrapper;
}

private static class DeferredLoad {

    private final MetaObject resultObject;
    private final String property;
    private final Class<?> targetType;
    private final CacheKey key;
    private final PerpetualCache localCache;
    private final ObjectFactory objectFactory;
    private final ResultExtractor resultExtractor;

    // issue #781
    public DeferredLoad(MetaObject resultObject,
                        String property,
                        CacheKey key,
                        PerpetualCache localCache,
                        Configuration configuration,
                        Class<?> targetType) {
        this.resultObject = resultObject;
        this.property = property;
        this.key = key;
        this.localCache = localCache;
        this.objectFactory = configuration.getObjectFactory();
        this.resultExtractor = new ResultExtractor(configuration, objectFactory);
        this.targetType = targetType;
    }

    public boolean canLoad() {
        return localCache.getObject(key) != null && localCache.getObject(key) != EXECUTION_PLACEHOLDER;
    }

    public void load() {
        @SuppressWarnings("unchecked")
        // we suppose we get back a List
        List<Object> list = (List<Object>) localCache.getObject(key);
        Object value = resultExtractor.extractObjectFromList(list, targetType);
        resultObject.setValue(property, value);
    }
}
```

BaseExecutor 主要为了子类提供了一些共性方法的实现，如事务相关的提交和回滚，一级缓存和获取数据库连接等。同时，定义了 doQuery、doUpdate、doFlushStatement 等抽象方法，由对应的子类去实现具体逻辑。以 query 方法为例，在 queryFormDatabase 中最终调用 doQuery 执行查询。

### SimpleExecutor

SimpleExecutor 是简单执行器，MyBatis 默认配置的执行器，如果没有配置指定执行器，使用的都是 SimpleExecutor。

```java
public class SimpleExecutor extends BaseExecutor {

    public SimpleExecutor(Configuration configuration, Transaction transaction) {
        super(configuration, transaction);
    }

    @Override
    public int doUpdate(MappedStatement ms, Object parameter) throws SQLException {
        Statement stmt = null;
        try {
            Configuration configuration = ms.getConfiguration();
            StatementHandler handler = configuration.newStatementHandler(this, ms, parameter, RowBounds.DEFAULT, null, null);
            stmt = prepareStatement(handler, ms.getStatementLog());
            return handler.update(stmt);
        } finally {
            closeStatement(stmt);
        }
    }

    @Override
    public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
        Statement stmt = null;
        try {
            Configuration configuration = ms.getConfiguration();
            StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
            stmt = prepareStatement(handler, ms.getStatementLog());
            return handler.query(stmt, resultHandler);
        } finally {
            closeStatement(stmt);
        }
    }

    @Override
    protected <E> Cursor<E> doQueryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds, BoundSql boundSql) throws SQLException {
        Configuration configuration = ms.getConfiguration();
        StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, null, boundSql);
        Statement stmt = prepareStatement(handler, ms.getStatementLog());
        Cursor<E> cursor = handler.queryCursor(stmt);
        stmt.closeOnCompletion();
        return cursor;
    }

    @Override
    public List<BatchResult> doFlushStatements(boolean isRollback) {
        // doFlushStatements 只是给 batch 用的，所以这里返回空
        return Collections.emptyList();
    }

    private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
        Statement stmt;
        Connection connection = getConnection(statementLog);
        stmt = handler.prepare(connection, transaction.getTimeout());
        handler.parameterize(stmt);
        return stmt;
    }

}
```

以查询为例，当调用 query 方法时，经过 BaseExecutor 的处理后，调用 queryFormDataBase 时，其内部又调用了doQuery 方法。所以实际上，真正去执行查询的方法为对应执行器的 doQuery 实现。但是在 SimpleExecutor 的 doQuery 方法中，我们可以看到，在 executor 层其实只做了三件事。

1. 通过当前绑定执行的 MappedStatement 去获取一个对应的 StatementHandler 对象
2. 创建 Connection 连接并生成 Statement 对象，并进行参数处理，每次使用 SimpleExecutor 执行查询或更新时，都会创建一个 Statement 对象
3. 交由对应的 StatementHandler 去执行 Statement

### ReuseExecutor

ReuseExecutor 名为重用执行器。重用执行器的作用是在一次会话中，重复使用同一个 Statement 对象以提升性能。以查询为例，每次执行查询时，都会去 Statement 缓存中查找，如果已经存在相同 SQL 的 Statement 对象，就会重复使用同一个 Statement 对象。

```java
public class ReuseExecutor extends BaseExecutor {
    /** 可重用的执行器内部用了一个map，用来缓存SQL语句对应的Statement，即key为SQL */
    private final Map<String, Statement> statementMap = new HashMap<>();

    public ReuseExecutor(Configuration configuration, Transaction transaction) {
        super(configuration, transaction);
    }

    @Override
    public int doUpdate(MappedStatement ms, Object parameter) throws SQLException {
        Configuration configuration = ms.getConfiguration();
        // 新建一个StatementHandler
        StatementHandler handler = configuration.newStatementHandler(this, ms, parameter, RowBounds.DEFAULT, null, null);
        // 准备语句
        Statement stmt = prepareStatement(handler, ms.getStatementLog());
        return handler.update(stmt);
    }

    @Override
    public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
        Configuration configuration = ms.getConfiguration();
        StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
        Statement stmt = prepareStatement(handler, ms.getStatementLog());
        return handler.query(stmt, resultHandler);
    }

    @Override
    protected <E> Cursor<E> doQueryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds, BoundSql boundSql) throws SQLException {
        Configuration configuration = ms.getConfiguration();
        StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, null, boundSql);
        Statement stmt = prepareStatement(handler, ms.getStatementLog());
        return handler.queryCursor(stmt);
    }

    @Override
    public List<BatchResult> doFlushStatements(boolean isRollback) {
        for (Statement stmt : statementMap.values()) {
            closeStatement(stmt);
        }
        // flush的时候清除缓存
        statementMap.clear();
        return Collections.emptyList();
    }

    private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
        Statement stmt;
        // 得到绑定的SQL语句
        BoundSql boundSql = handler.getBoundSql();
        String sql = boundSql.getSql();
        // 如果缓存中已经有了Statement，直接获取
        if (hasStatementFor(sql)) {
            stmt = getStatement(sql);
            applyTransactionTimeout(stmt);
        } else {
            // 如果没有，prepare一个放入缓存
            Connection connection = getConnection(statementLog);
            stmt = handler.prepare(connection, transaction.getTimeout());
            putStatement(sql, stmt);
        }
        handler.parameterize(stmt);
        return stmt;
    }

    private boolean hasStatementFor(String sql) {
        try {
            Statement statement = statementMap.get(sql);
            return statement != null && !statement.getConnection().isClosed();
        } catch (SQLException e) {
            return false;
        }
    }

    private Statement getStatement(String s) {
        return statementMap.get(s);
    }

    private void putStatement(String sql, Statement stmt) {
        statementMap.put(sql, stmt);
    }

}
```

ReuseExecutor 实现 Statement 对象的复用逻辑很简单，在类内部维护一个 Map，使用对应的 SQL 作为 key 来缓存同个会话中的 Statement 对象，每次需要执行时，都先去本地的缓存 Map 中查找，如果存在则直接使用之前的 Statement 对象，如果不存在再去创建 Statement 对象。

### BatchExecutor

BatchExecutor 名为批量处理器，用来批量执行 SQL，但是注意，批量处理只对更新（增删改）语句有效，对查询无效。在同一个会话内，增删改 SQL 会一次性处理好 SQL 和参数然后发送给数据库，从而减少与数据库的交互次数，减少 IO 消耗，提升性能。

```java
public class BatchExecutor extends BaseExecutor {

    public static final int BATCH_UPDATE_RETURN_VALUE = Integer.MIN_VALUE + 1002;
    /** 批量执行的Statement对象 **/
    private final List<Statement> statementList = new ArrayList<>();
    /** 批量执行的结果对象 **/
    private final List<BatchResult> batchResultList = new ArrayList<>();
    /** 上一次处理的SQL语句 **/
    private String currentSql;
    /** 上一次处理的MappedStatement对象 **/
    private MappedStatement currentStatement;

    public BatchExecutor(Configuration configuration, Transaction transaction) {
        super(configuration, transaction);
    }

    @Override
    public int doUpdate(MappedStatement ms, Object parameterObject) throws SQLException {
        final Configuration configuration = ms.getConfiguration();
        final StatementHandler handler = configuration.newStatementHandler(this, ms, parameterObject, RowBounds.DEFAULT, null, null);
        final BoundSql boundSql = handler.getBoundSql();
        // 获取SQL语句
        final String sql = boundSql.getSql();
        final Statement stmt;
        // 判断当前要处理的sql语句是否等于上一次执行的sql，MappedStatement也是同理
        // 只有这两个对象都满足时，才能复用上一次的Statement对象
        if (sql.equals(currentSql) && ms.equals(currentStatement)) {
            int last = statementList.size() - 1;
            // 即使满足上述的两个条件，也只能从statement缓存list中获取最后一个对象，相同的sql还必须满足连贯顺序才能复用上一次的Statement对象
            stmt = statementList.get(last);
            applyTransactionTimeout(stmt);
            handler.parameterize(stmt);// fix Issues 322
            BatchResult batchResult = batchResultList.get(last);
            batchResult.addParameterObject(parameterObject);
        } else {
            // 创建和保存新的Statement对象和BatchResult对象
            // 并将当前sql和MappedStatement设置为该次执行的对应对象
            Connection connection = getConnection(ms.getStatementLog());
            stmt = handler.prepare(connection, transaction.getTimeout());
            handler.parameterize(stmt);    // fix Issues 322
            currentSql = sql;
            currentStatement = ms;
            statementList.add(stmt);
            batchResultList.add(new BatchResult(ms, sql, parameterObject));
        }
        handler.batch(stmt);
        return BATCH_UPDATE_RETURN_VALUE;
    }

    @Override
    public <E> List<E> doQuery(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql)
        throws SQLException {
        Statement stmt = null;
        try {
            flushStatements();
            Configuration configuration = ms.getConfiguration();
            StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameterObject, rowBounds, resultHandler, boundSql);
            Connection connection = getConnection(ms.getStatementLog());
            stmt = handler.prepare(connection, transaction.getTimeout());
            handler.parameterize(stmt);
            return handler.query(stmt, resultHandler);
        } finally {
            closeStatement(stmt);
        }
    }

    @Override
    protected <E> Cursor<E> doQueryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds, BoundSql boundSql) throws SQLException {
        flushStatements();
        Configuration configuration = ms.getConfiguration();
        StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, null, boundSql);
        Connection connection = getConnection(ms.getStatementLog());
        Statement stmt = handler.prepare(connection, transaction.getTimeout());
        handler.parameterize(stmt);
        Cursor<E> cursor = handler.queryCursor(stmt);
        stmt.closeOnCompletion();
        return cursor;
    }

    @Override
    public List<BatchResult> doFlushStatements(boolean isRollback) throws SQLException {
        try {
            List<BatchResult> results = new ArrayList<>();
            if (isRollback) {
                return Collections.emptyList();
            }
            for (int i = 0, n = statementList.size(); i < n; i++) {
                Statement stmt = statementList.get(i);
                applyTransactionTimeout(stmt);
                BatchResult batchResult = batchResultList.get(i);
                try {
                    // 执行并记录批量执行的数量
                    batchResult.setUpdateCounts(stmt.executeBatch());
                    MappedStatement ms = batchResult.getMappedStatement();
                    List<Object> parameterObjects = batchResult.getParameterObjects();
                    KeyGenerator keyGenerator = ms.getKeyGenerator();
                    if (Jdbc3KeyGenerator.class.equals(keyGenerator.getClass())) {
                        Jdbc3KeyGenerator jdbc3KeyGenerator = (Jdbc3KeyGenerator) keyGenerator;
                        jdbc3KeyGenerator.processBatch(ms, stmt, parameterObjects);
                    } else if (!NoKeyGenerator.class.equals(keyGenerator.getClass())) { //issue #141
                        for (Object parameter : parameterObjects) {
                            keyGenerator.processAfter(this, ms, stmt, parameter);
                        }
                    }
                    // Close statement to close cursor #1109
                    closeStatement(stmt);
                } catch (BatchUpdateException e) {
                    StringBuilder message = new StringBuilder();
                    message.append(batchResult.getMappedStatement().getId())
                        .append(" (batch index #")
                        .append(i + 1)
                        .append(")")
                        .append(" failed.");
                    if (i > 0) {
                        message.append(" ")
                            .append(i)
                            .append(" prior sub executor(s) completed successfully, but will be rolled back.");
                    }
                    throw new BatchExecutorException(message.toString(), e, results, batchResult);
                }
                results.add(batchResult);
            }
            return results;
        } finally {
            for (Statement stmt : statementList) {
                closeStatement(stmt);
            }
            currentSql = null;
            statementList.clear();
            batchResultList.clear();
        }
    }

}
```

值得注意的时，在使用 BatchExecutor 的时候，如果是相同的 SQL 语句，会进行合并使用同一个 Statement 对象，但是有一个前提是，相同的 SQL 语句必须是相同的 MappedStatement，并且 SQL 必须满足连贯的先后顺序。

### CachingExecutor

CachingExecutor是缓存执行器，用于实现MyBatis的二级缓存，直接继承与Executor。

```java
public class CachingExecutor implements Executor {

    /** 装饰者模式内部持有一个Executor */
    private final Executor delegate;
    /** 事务缓存管理器 **/
    private final TransactionalCacheManager tcm = new TransactionalCacheManager();

    public CachingExecutor(Executor delegate) {
        this.delegate = delegate;
        delegate.setExecutorWrapper(this);
    }

    @Override
    public Transaction getTransaction() {
        return delegate.getTransaction();
    }

    @Override
    public void close(boolean forceRollback) {
        try {
            // issues #499, #524 and #573
            if (forceRollback) {
                tcm.rollback();
            } else {
                tcm.commit();
            }
        } finally {
            delegate.close(forceRollback);
        }
    }

    @Override
    public boolean isClosed() {
        return delegate.isClosed();
    }

    @Override
    public int update(MappedStatement ms, Object parameterObject) throws SQLException {
        flushCacheIfRequired(ms);
        return delegate.update(ms, parameterObject);
    }

    @Override
    public <E> Cursor<E> queryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds) throws SQLException {
        flushCacheIfRequired(ms);
        return delegate.queryCursor(ms, parameter, rowBounds);
    }

    @Override
    public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
        BoundSql boundSql = ms.getBoundSql(parameterObject);
        CacheKey key = createCacheKey(ms, parameterObject, rowBounds, boundSql);
        return query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
    }

    @Override
    public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
        throws SQLException {
        Cache cache = ms.getCache();
        if (cache != null) {
            flushCacheIfRequired(ms);
            if (ms.isUseCache() && resultHandler == null) {
                ensureNoOutParams(ms, boundSql);
                @SuppressWarnings("unchecked")
                List<E> list = (List<E>) tcm.getObject(cache, key);
                if (list == null) {
                    list = delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
                    tcm.putObject(cache, key, list); // issue #578 and #116
                }
                return list;
            }
        }
        return delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
    }

    @Override
    public List<BatchResult> flushStatements() throws SQLException {
        return delegate.flushStatements();
    }

    @Override
    public void commit(boolean required) throws SQLException {
        delegate.commit(required);
        tcm.commit();
    }

    @Override
    public void rollback(boolean required) throws SQLException {
        try {
            delegate.rollback(required);
        } finally {
            if (required) {
                tcm.rollback();
            }
        }
    }

    private void ensureNoOutParams(MappedStatement ms, BoundSql boundSql) {
        if (ms.getStatementType() == StatementType.CALLABLE) {
            for (ParameterMapping parameterMapping : boundSql.getParameterMappings()) {
                if (parameterMapping.getMode() != ParameterMode.IN) {
                    throw new ExecutorException("Caching stored procedures with OUT params is not supported.  Please configure useCache=false in " + ms.getId() + " statement.");
                }
            }
        }
    }

    @Override
    public CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql) {
        return delegate.createCacheKey(ms, parameterObject, rowBounds, boundSql);
    }

    @Override
    public boolean isCached(MappedStatement ms, CacheKey key) {
        return delegate.isCached(ms, key);
    }

    @Override
    public void deferLoad(MappedStatement ms, MetaObject resultObject, String property, CacheKey key, Class<?> targetType) {
        delegate.deferLoad(ms, resultObject, property, key, targetType);
    }

    @Override
    public void clearLocalCache() {
        delegate.clearLocalCache();
    }

    private void flushCacheIfRequired(MappedStatement ms) {
        Cache cache = ms.getCache();
        if (cache != null && ms.isFlushCacheRequired()) {
            tcm.clear(cache);
        }
    }

    @Override
    public void setExecutorWrapper(Executor executor) {
        throw new UnsupportedOperationException("This method should not be called");
    }

}
```

CachingExecutor 通过装饰者模式，在内部持有一个 Executor 对象，包装上缓存实现逻辑。同时，CachingExecutor 中还有一个事务缓存管理器来实现事务数据的隔离性。

## 总结

MyBatis 的 Executor 方法执行栈：

1. 当查询时，通过 SqlSession 的 selectList 方法进行开始执行（不管是执行 selectOne 方法还是 selectList 方法，最终调用的方法都是 selectList）。SqlSession 再调用内部持有的 Executor 的 query 方法执行。

   ```java
   @Override
   public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
     try {
       // 根据statement id找到对应的MappedStatement
       MappedStatement ms = configuration.getMappedStatement(statement);
       // 转而用执行器来查询结果,注意这里传入的ResultHandler是null
       return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
     } catch (Exception e) {
       throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
     } finally {
       ErrorContext.instance().reset();
     }
   }
   ```

2. 在开启二级缓存的时候，SqlSession 内部的 Executor 对象实际上是 CachingExecutor 对象，在执行完相应的缓存逻辑后，调用 CachingExecutor 内部 Executor 对象的 query 方法。

3. 由于在 query 方法的实现只在 BaseExecutor 中，也就是说，当 CachingExecutor 调用 query 方法时，实际上是在调用 BaseExecutor 的 query 方法。

4. 最终执行查询时，会进入SimpleExecutor的doQuery方法，去完成一次真正的数据库查询

最终 Executor 的执行体系如下图：

![image-20210429000500432](https://cdn.jsdelivr.net/gh/LiLittleCat/PicBed/images/mybatis/image-20210429000500432.png)
