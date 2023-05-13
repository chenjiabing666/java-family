
## 前言
- Mybatis的专题文章写到这里已经是第四篇了，前三篇讲了Mybatis的基本使用，相信只要认真看了的朋友，在实际开发中正常使用应该不是问题。没有看过的朋友，作者建议去看一看，三篇文章分别是[Mybatis入门之基本操作](https://mp.weixin.qq.com/s/KdrEvlShnVoYA8nr0qLSNw)、[Mybatis结果映射，你射准了吗？](https://mp.weixin.qq.com/s/czICR6jv1yz6adi6G3xFgQ)、[Mybatis动态SQL，你真的会了吗？](https://mp.weixin.qq.com/s/yuYAEXY_OGRsr0Eb3xZkog)。
- 当然，任何一个技术都不能浅藏辄止，今天作者就带大家深入底层源码看一看Mybatis的基础架构。此篇文章只是源码的入门篇，讲一些Mybatis中重要的组件，作者称之为`六剑客`。

## 环境版本
- 本篇文章讲的一切内容都是基于`Mybatis3.5`和`SpringBoot-2.3.3.RELEASE`。

## Myabtis的六剑客
- 其实Mybatis的底层源码和Spring比起来还是非常容易读懂的，作者将其中六个重要的接口抽离出来称之为`Mybatis的六剑客`，分别是`SqlSession`、`Executor`、`StatementHandler`、`ParameterHandler`、`ResultSetHandler`、`TypeHandler`。
- 六剑客在Mybatis中分别承担着什么角色？下面将会逐一介绍。
- 介绍六剑客之前，先来一张六剑客执行的流程图，如下：
![](https://img.java-family.cn/Mybatis-%E5%85%AD%E5%89%91%E5%AE%A2/3.png)


## SqlSession
- SqlSession是Myabtis中的核心API，主要用来执行命令，获取映射，管理事务。它包含了所有执行语句、提交或回滚事务以及获取映射器实例的方法。

### 有何方法
- 其中定义了将近20个方法，其中涉及的到语句执行，事务提交回滚等方法。下面对于这些方法进行分类总结。

#### 语句执行方法
- 这些方法被用来执行定义在 SQL 映射 XML 文件中的 SELECT、INSERT、UPDATE 和 DELETE 语句。你可以通过名字快速了解它们的作用，每一方法都接受语句的 ID 以及参数对象，参数可以是原始类型（支持自动装箱或包装类）、JavaBean、POJO 或 Map。
```java
<T> T selectOne(String statement, Object parameter)
<E> List<E> selectList(String statement, Object parameter)
<T> Cursor<T> selectCursor(String statement, Object parameter)
<K,V> Map<K,V> selectMap(String statement, Object parameter, String mapKey)
int insert(String statement, Object parameter)
int update(String statement, Object parameter)
int delete(String statement, Object parameter)
```
- 其中的最容易误解的就是`selectOne`和`selectList`，从方法名称就很容易知道区别，一个是查询单个，一个是查询多个。如果你对自己的SQL无法确定返回一个还是多个结果的时候，建议使用`selectList`。
- `insert`，`update`，`delete`方法返值是受影响的行数。
- select还有几个重用的方法，用于限制返回行数，在Mysql中对应的就是`limit`，如下：
```java
<E> List<E> selectList (String statement, Object parameter, RowBounds rowBounds)
<T> Cursor<T> selectCursor(String statement, Object parameter, RowBounds rowBounds)
<K,V> Map<K,V> selectMap(String statement, Object parameter, String mapKey, RowBounds rowbounds)
void select (String statement, Object parameter, ResultHandler<T> handler)
void select (String statement, Object parameter, RowBounds rowBounds, ResultHandler<T> handler)
```
- 其中的`RowBounds`参数中保存了限制的行数，起始行数。

#### 立即批量更新方法
- 当你将 ExecutorType 设置为 ExecutorType.BATCH 时，可以使用这个方法清除（执行）缓存在 JDBC 驱动类中的批量更新语句。
```java
List<BatchResult> flushStatements()
```

#### 事务控制方法
- 有四个方法用来控制事务作用域。当然，如果你已经设置了自动提交或你使用了外部事务管理器，这些方法就没什么作用了。然而，如果你正在使用由 Connection 实例控制的 JDBC 事务管理器，那么这四个方法就会派上用场：
```java
void commit()
void commit(boolean force)
void rollback()
void rollback(boolean force)
```
- 默认情况下 MyBatis 不会自动提交事务，除非它侦测到调用了插入、更新或删除方法改变了数据库。如果你没有使用这些方法提交修改，那么你可以在`commit` 和 `rollback` 方法参数中传入 true 值，来保证事务被正常提交（注意，在自动提交模式或者使用了外部事务管理器的情况下，设置 `force` 值对 `session` 无效）。大部分情况下你无需调用 `rollback()`，因为 MyBatis 会在你没有调用 `commit` 时替你完成回滚操作。不过，当你要在一个可能多次提交或回滚的 session 中详细控制事务，回滚操作就派上用场了。


#### 本地缓存方法
- Mybatis 使用到了两种缓存：本地缓存（local cache）和二级缓存（second level cache）。
- 默认情况下，本地缓存数据的生命周期等同于整个 session 的周期。由于缓存会被用来解决循环引用问题和加快重复嵌套查询的速度，所以无法将其完全禁用。但是你可以通过设置 `localCacheScope=STATEMENT` 来只在语句执行时使用缓存。
- 可以调用以下方法清除本地缓存。
```java
void clearCache()
```

#### 获取映射器
- 在SqlSession中你也可以获取自己的映射器，直接使用下面的方法，如下：
```java
<T> T getMapper(Class<T> type)
```
- 比如你需要获取一个`UserMapper`，如下：
```java
UserMapper mapper = sqlSessionTemplate.getMapper(UserMapper.class);
```

### 有何实现类
- 在Mybatis中有三个实现类，分别是`DefaultSqlSession`，`SqlSessionManager`、`SqlSessionTemplate`，其中重要的就是`DefaultSqlSession`，这个后面讲到Mybatis执行源码的时候会一一分析。
- 在与SpringBoot整合时，Mybatis的启动器配置类会默认注入一个`SqlSessionTemplate`，源码如下：
```java
@Bean
  @ConditionalOnMissingBean
  public SqlSessionTemplate sqlSessionTemplate(SqlSessionFactory sqlSessionFactory) 
    //根据执行器的类型创建不同的执行器，默认CachingExecutor
    ExecutorType executorType = this.properties.getExecutorType();
    if (executorType != null) {
      return new SqlSessionTemplate(sqlSessionFactory, executorType);
    } else {
      return new SqlSessionTemplate(sqlSessionFactory);
    }
  }
```


## Executor
- Mybatis的执行器，是Mybatis的调度核心，负责SQL语句的生成和缓存的维护，SqlSession中的crud方法实际上都是调用执行器中的对应方法执行。
- 继承结构如下图：
![继承结构](https://img.java-family.cn/Mybatis-%E5%85%AD%E5%89%91%E5%AE%A2/1.png)

### 实现类
- 下面我们来看看都有哪些实现类，分别有什么作用。

#### BaseExecutor
- 这是一个抽象类，采用模板方法的模式，有意思的是这个老弟模仿Spring的方式，真正的执行的方法都是`doxxx()`。
- 其中有一个方法值得注意，查询的时候走的`一级缓存`，因此这里注意下，既然这是个模板类，那么Mybatis执行select的时候默认都会走一级缓存。代码如下：
```java
private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    List<E> list;
    //此处的localCache即是一级缓存，是一个Map的结构
    localCache.putObject(key, EXECUTION_PLACEHOLDER);
    try {
      //执行真正的查询
      list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
    } finally {
      localCache.removeObject(key);
    }
    localCache.putObject(key, list);
    if (ms.getStatementType() == StatementType.CALLABLE) {
      localOutputParameterCache.putObject(key, parameter);
    }
    return list;
  }
```
#### CachingExecutor
- 这个比较有名了，二级缓存的维护类，与SpringBoot整合默认创建的就是这个家伙。下面来看一下如何走的二级缓存，源码如下：
```java
@Override
  public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
      throws SQLException {
    //查看当前Sql是否使用了二级缓存
    Cache cache = ms.getCache();
    //使用缓存了，直接从缓存中取
    if (cache != null) {
      flushCacheIfRequired(ms);
      if (ms.isUseCache() && resultHandler == null) {
        ensureNoOutParams(ms, boundSql);
        @SuppressWarnings("unchecked")
        //从缓存中取数据
        List<E> list = (List<E>) tcm.getObject(cache, key);
        if (list == null) {
          //没取到数据，则执行SQL从数据库查询
          list = delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
          //查到了，放入缓存中
          tcm.putObject(cache, key, list); // issue #578 and #116
        }
        //直接返回
        return list;
      }
    }
    //没使用二级缓存，直接执行SQL从数据库查询
    return delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }
```
- 这玩意就是走个二级缓存，其他没什么。

#### SimpleExecutor
- 这个类像个直男，最简单的一个执行器，就是根据对应的SQL执行，不会做一些额外的操作。

#### BatchExecutor
- 通过批量操作来优化性能。通常需要注意的是`批量更新`操作，由于内部有缓存的实现，使用完成后记得调用`flushStatements`来清除缓存。

#### ReuseExecutor 
- 　可重用的执行器，重用的对象是Statement，也就是说该执行器会缓存同一个sql的`Statement`，省去Statement的重新创建，优化性能。
- 内部的实现是通过一个`HashMap`来维护Statement对象的。由于当前Map只在该session中有效，所以使用完成后记得调用`flushStatements`来清除Map。

### SpringBoot中如何创建
- 在SpringBoot到底创建的是哪个执行器呢？其实只要阅读一下源码可以很清楚的知道，答案就在`org.apache.ibatis.session.Configuration`类中，其中创建执行器的源码如下：
```java
public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
    //没有指定执行器的类型，创建默认的，即是SimpleExecutor
    executorType = executorType == null ? defaultExecutorType : executorType;
    executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
    Executor executor;
    //类型是BATCH，创建BatchExecutor
    if (ExecutorType.BATCH == executorType) {
      executor = new BatchExecutor(this, transaction);
      //类型为REUSE，创建ReuseExecutor
    } else if (ExecutorType.REUSE == executorType) {
      executor = new ReuseExecutor(this, transaction);
    } else {
      //除了上面两种，创建的都是SimpleExecutor
      executor = new SimpleExecutor(this, transaction);
    }
    //如果全局配置了二级缓存，则创建CachingExecutor，SpringBoot中这个参数默认是true，可以自己设置为false
    if (cacheEnabled) {
    //创建CachingExecutor
      executor = new CachingExecutor(executor);
    }
    executor = (Executor) interceptorChain.pluginAll(executor);
    return executor;
  }
```

- 显而易见，SpringBoot中默认创建的是`CachingExecutor`，因为默认的`cacheEnabled`的值为`true`。

## StatementHandler
- 熟悉JDBC的朋友应该都能猜到这个接口是干嘛的，很显然，这个是对SQL语句进行处理和参数赋值的。

### 实现类
- 该接口也是有很多的实现类，如下图：
![](https://img.java-family.cn/Mybatis-%E5%85%AD%E5%89%91%E5%AE%A2/2.png)

#### SimpleStatementHandler
- 这个很简单了，就是对应我们JDBC中常用的Statement接口，用于简单SQL的处理

#### PreparedStatementHandler
- 这个对应JDBC中的PreparedStatement，预编译SQL的接口。

#### CallableStatementHandler
- 这个对应JDBC中CallableStatement，用于执行存储过程相关的接口。

#### RoutingStatementHandler
- 这个接口是以上三个接口的路由，没有实际操作，只是负责上面三个StatementHandler的创建及调用。


## ParameterHandler
- `ParameterHandler`在Mybatis中负责将sql中的占位符替换为真正的参数，它是一个接口，有且只有一个实现类`DefaultParameterHandler`。
- `setParameters`是处理参数最核心的方法。这里不再详细的讲，后面会讲到。

## TypeHandler
- 这位大神应该都听说过，也都自定义过吧，简单的说就是在预编译设置参数和取出结果的时候将Java类型和JDBC的类型进行相应的转换。当然，Mybatis内置了很多默认的类型处理器，基本够用，除非有特殊的定制，我们才会去自定义，比如需要将Java对象以`JSON`字符串的形式存入数据库，此时就可以自定义一个类型处理器。
- 很简单的东西，此处就不再详细的讲了，后面会单独出一篇如何自定义类型处理器的文章。


## ResultSetHandler
- 结果处理器，负责将JDBC返回的ResultSet结果集对象转换成List类型的集合或者`Cursor`。
- 具体实现类就是`DefaultResultSetHandler`，其实现的步骤就是将Statement执行后的结果集，按照Mapper文件中配置的ResultType或ResultMap来封装成对应的对象，最后将封装的对象返回。
- 源码及其复杂，尤其是其中对嵌套查询的解析，这里只做个了解，后续会专门写一篇文章介绍。


## 总结
- 至此，Mybatis源码第一篇就已经讲完了，本篇文章对Mybatis中的重要组件做了初步的了解，为后面更深入的源码阅读做了铺垫，如果觉得作者写的不错，在看分享一波，谢谢支持。










