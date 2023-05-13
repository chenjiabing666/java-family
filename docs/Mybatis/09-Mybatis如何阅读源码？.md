
## 前言
- 前一篇文章简单的介绍了Mybatis的六个重要组件，这六剑客占据了Mybatis的半壁江山，和六剑客搞了基友，那么Mybatis就是囊中之物了。对六剑客感兴趣的朋友，可以看看这篇文章：[Mybatis源码解析篇之六剑客](https://mp.weixin.qq.com/s/lnJx0h_4Kk6fKuhptN1cdg)
- 有些初入门的朋友可能很害怕阅读源码，不知道如何阅读源码，与其我一篇文章按照自己的思路写完Mybatis的源码，但是你们又能理解多少呢？不如教会你们思路，让你们能够自己知道如何阅读源码。

## 环境配置
- 本篇文章讲的一切内容都是基于`Mybatis3.5`和`SpringBoot-2.3.3.RELEASE`。


## 从哪入手？
- 还是要说一说六剑客的故事，既然是Mybatis的重要组件，当然要从六剑客下手了，沿用上篇文章的一张图，此图记录了六剑客先后执行的顺序，如下：
  ![六剑客执行流程图](https://img.java-family.cn/Mybatis-%E5%85%AD%E5%89%91%E5%AE%A2/3.png)

- 阅读源码最重要的一点不能忘了，就是开启`DEBUG`模式，重要方法打上断点，重要语句打上断点，先把握整体，再研究细节，基本就不难了。

- 下面就以Myabtis的查询语句`selectList()`来具体分析下如何阅读。



## 总体把握六剑客
- 从六剑客开整，既然是重要组件，源码执行流程肯定都是围绕着六剑客，下面来对六剑客一一分析，如何打断点。

- 下面只是简单的教你如何打断点，对于六剑客是什么不再介绍，请看上篇文章。

### SqlSession
- 既然是接口，肯定不能在接口方法上打断点，上文介绍有两个实现类，分别是`DefaultSqlSession`、`SqlSessionTemplate`。那么SpringBoot在初始化的时候到底注入的是哪一个呢？这个就要看Mybatis的启动器的自动配置类了，其中有一段这样的代码，如下：
```java
  //如果容器中没有SqlSessionTemplate这个Bean，则注入
  @Bean
  @ConditionalOnMissingBean
  public SqlSessionTemplate sqlSessionTemplate(SqlSessionFactory sqlSessionFactory) {
    ExecutorType executorType = this.properties.getExecutorType();
    if (executorType != null) {
      return new SqlSessionTemplate(sqlSessionFactory, executorType);
    } else {
      return new SqlSessionTemplate(sqlSessionFactory);
    }
  }
```

- 从上面的代码可以知道，SpringBoot启动时注入了`SqlSessionTemplate`，此时就肯定从`SqlSessionTemplate`入手了。它的一些方法如下图：
  ![SqlSessionTemplate方法](https://img.java-family.cn/Mybatis-%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB%E6%95%99%E5%AD%A6/1.png)

- 从上图的标记可以知道，首当其冲的就是`构造方法`了;既然是分析`selectList()`的查询流程，当然全部的`selectList()`方法要打上断点了;上篇文章也讲了Mapper的接口最终是走的动态代理生成的实例，因此此处的`getMapper()`也打上断点。

- 对于初入门的来说，上面三处打上断点已经足够了，但是如果你仔细看一眼`selectList()`方法，如下：
```java
  @Override
  public <E> List<E> selectList(String statement) {
    //此处的sqlSessionProxy是什么，也是SqlSession类型的，此处断点运行到这里可以知道，就是DefaultSqlSession实例
    return this.sqlSessionProxy.selectList(statement);
  }
```

- `sqlSessionProxy`是什么，没关系，这个不能靠猜，那么此时断点走一波，走到`selectList()`方法内部，如下图：
  ![](https://img.java-family.cn/Mybatis-%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB%E6%95%99%E5%AD%A6/2.png)
- 从上图可以很清楚的看到了，其实就是`DefaultSqlSession`。哦，明白了，原来SqlSessionTemplate把过甩给了`DefaultSqlSession`了，太狡诈了。
- `DefaultSqlSession`如何打断点就不用说了吧，自己搞搞吧。

### Executor
- 上面文章讲过执行器是什么作用，也讲过Mybatis内部是根据什么创建执行器的。此处不再赘述了。
- SpringBoot整合各种框架有个特点，万变不离自动配置类，框架的一些初始化动作基本全是在自动配置类中完成，于是我们在配置类找一找在哪里注入了`Executor`的Bean，于是找到了如下的一段代码：
  ![](https://img.java-family.cn/Mybatis-%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB%E6%95%99%E5%AD%A6/3.png)
- 从上面的代码可以知道默认创建了`CachingExecutor`，二级缓存的执行器，别管那么多，看看它重写了`Executor`的哪些接口，与`selectList()`相关的方法打上断点，如下图：
  ![](https://img.java-family.cn/Mybatis-%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB%E6%95%99%E5%AD%A6/4.png)
- 从上图也知道哪些方法和`selectList()`相关了，显然的`query`是查询的意思，别管那么多，先打上断点。

- 此时再仔细瞅一眼`query()`的方法怎么执行的，哦？发现了什么，如下：
```java
@Override
  public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
      throws SQLException {
      //先尝试从缓存中获取
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
    //没有缓存，直接调用delegate的query方法
    return delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }
```

- 从上面的代码知道，有缓存了，直接返回了，没有缓存，调用了`delegate`中的`query`方法，那么这个`delegate`是哪个类的对象呢？参照sqlSession的分析的方法，调试走起，可以知道是`SimpleExecutor`的实例，如下图：
  ![](https://img.java-family.cn/Mybatis-%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB%E6%95%99%E5%AD%A6/5.png)
- 后面的`SimpleExecutor`如何打断点就不再说了，自己尝试找找。


### StatementHandler
- 很熟悉的一个接口，在学JDBC的时候就接触过类似的，执行语句和设置参数的作用。
- 这个接口很简单，大佬写的代码，看到方法名就知道这个方法是干什么的，如下图：
  ![](https://img.java-family.cn/Mybatis-%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB%E6%95%99%E5%AD%A6/6.png)
- 最重要的实现类是什么？当然是`PreparedStatementHandler`，因此在对应的方法上打上断点即可。


### ParameterHandler
- 这个接口很简单，也别选择了，总共两个方法，一个设置，一个获取，在实现类`DefaultParameterHandler`中对应的方法上打上断点即可。
  ![](https://img.java-family.cn/Mybatis-%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB%E6%95%99%E5%AD%A6/7.png)


### TypeHandler
- 类型处理器，也是一个简单的接口，总共'两个'方法，一个设置参数的转换，一个对结果的转换，啥也别说了，自己找到对应参数类型的处理器，在其中的方法打上断点。


### ResultSetHandler
- 结果处理器，负责对结果的处理，总共三个方法，一个实现类`DefaultResultSetHandler`，全部安排断点。



## 总结
- 授人以鱼不授人以渔，与其都分析了给你看，不如教会你阅读源码的方式，先自己去研究，不仅仅是阅读Mybatis的源码是这样，阅读任何框架的源码都是如此，比如Spring的源码，只要找到其中重要的组件，比如前置处理器，后置处理器，事件触发器等等，一切都迎刃而解。


