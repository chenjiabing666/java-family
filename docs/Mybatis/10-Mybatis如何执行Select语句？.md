
## 前言
- 本篇文章是Myabtis源码分析的第三篇，前两篇分别介绍了Mybatis的重要组件和围绕着Mybatis中的重要组件教大家如何阅读源码的一些方法，有了前面两篇文章的基础，来看这篇文章的才不会觉得吃力，如果没有看过的朋友，陈某建议去看看，两篇文章分别是[Mybatis源码解析之六剑客](https://mp.weixin.qq.com/s/lnJx0h_4Kk6fKuhptN1cdg)和[Mybatis源码如何阅读，教你一招！！！](https://mp.weixin.qq.com/s/B9e-4y_jokLHtDnS0o6-7g)。
- 今天接上一篇，围绕Mybatis中的`selectList()`来看一看Mybatis底层到底做了什么，有什么高级的地方。


## 环境准备
- 本篇文章讲的一切内容都是基于`Mybatis3.5`和`SpringBoot-2.3.3.RELEASE`。
- 由于此篇文章是基于前两篇文章的基础之上，因此重复的内容不再详细赘述了。

## 撸起袖子就是干
- 二话不说，先来一张流程图，Mybatis六剑客，如下：
  ![六剑客执行流程图](https://img.java-family.cn/Mybatis-%E5%85%AD%E5%89%91%E5%AE%A2/3.png)
- 上图中的这六剑客在前面两篇文章中已经介绍的非常清楚了，此处略过。为什么源码解析的每一篇文章中都要放一张这个流程图呢？因为Mybatis底层就是围绕着这六剑客展开的，我们需要从全局掌握Mybatis的源码究竟如何执行的。

### 测试环境搭建
- 举个栗子：根据用户id查询用户信息，Mapper定义如下：
```java
List<UserInfo> selectList(@Param("userIds") List<String> userIds);
```
- 对应XML配置如下：
```java
<mapper namespace="cn.cb.demo.dao.UserMapper">
    <!--开启二级缓存-->
    <cache/>
    <select id="selectList" resultType="cn.cb.demo.domain.UserInfo">
        select * from user_info where status=1
        and user_id in
        <foreach collection="userIds" item="item" open="(" separator="," close=")" >
            #{item}
        </foreach>
    </select>
</mapper>
```
- 单元测试如下：
```java
    @Test
    void contextLoads() {
        List<UserInfo> userInfos = userMapper.selectList(Arrays.asList("192","198"));
        System.out.println(userInfos);
    }
```

### DEBUG走起
- 具体在哪里打上断点，上篇文章已经讲过了，不再赘述了。
- 由于SpringBoot与Mybatis整合之后，自动注入的是`SqlSessionTemplate`，因此代码执行到`org.mybatis.spring.SqlSessionTemplate#selectList(java.lang.String, java.lang.Object)`，如`图1`：
  ![](https://img.java-family.cn/Mybatis%E6%BA%90%E7%A0%81-3/1.png)
- 从源码可以看到，实际调用的还是`DefaultSqlSession`中的`selectList`方法。如下`图2`：
  ![](https://img.java-family.cn/Mybatis%E6%BA%90%E7%A0%81-3/2.png)
- **具体的逻辑如下**：
  1. 根据Mapper方法的`全类名`从Mybatis的配置中获取到这条SQL的详细信息，比如`paramterType`,`resultMap`等等。
  2. 既然开启了二级缓存，肯定先要判断这条SQL是否缓存过，因此实际调用的是`CachingExecutor`这个缓存执行器。

- `DefaultSqlSession`只是简单的获取SQL的详细配置，最终还是把任务交给了`Executor`（当然这里走的是二级缓存，因此交给了缓存执行器）。下面DEBUG走到`CachingExecutor#query(MappedStatement, java.lang.Object, RowBounds,ResultHandler)`，源码如下`图3`：
  ![](https://img.java-family.cn/Mybatis%E6%BA%90%E7%A0%81-3/3.png)
- 上图中的`query`方法实际做了两件事，实际执行的查询还是其中重载的方法`List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)`，如下`图4`：
  ![](https://img.java-family.cn/Mybatis%E6%BA%90%E7%A0%81-3/4.png)
- 根据上图源码的分析，其实CachingExecutor执行的逻辑并不是很难，反倒很容易理解，**具体的逻辑如下**：
  1. 如果开启了二级缓存，先根据`cacheKey`从二级缓存中查询，如果查询到了直接返回
  2. 如果未开启二级缓存，再执行`BaseExecutor`中的query方法从一级缓存中查询。
  3. 如果二级缓存中未查询到数据，再执行`BaseExecutor`中的query方法从一级缓存中查询。
  4. 将查询到的结果存入到二级缓存中。

- `BaseExecutor`中的`query`方法无非就是从一级缓存中取数据，没查到再从数据库中取数据，一级缓存实际就是一个Map结构，这里不再细说，真正执行SQL从数据库中取数据的是`SimpleExecutor`中的`public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql)`方法，源码如下`图5`：
  ![](https://img.java-family.cn/Mybatis%E6%BA%90%E7%A0%81-3/5.png)
- 从上面的源码也是可以知道，在真正执行SQL之前，是要调用`prepareStatement(handler, ms.getStatementLog())`方法做一些参数的预处理的，其中涉及到了六大剑客的另外两位，分别是`ParameterHandler`和`TypeHandler`，源码如`图6`：
  ![](https://img.java-family.cn/Mybatis%E6%BA%90%E7%A0%81-3/6.png)
- 从上图可以知道设置SQL参数的真正方法是`handler.parameterize(stmt)`，真正执行的是`DefaultParameterHandler`中的`setParameters`方法，由于篇幅较长，简单的说一下思路：
  1. 获取所有参数的映射
  2. 循环遍历，获取参数的值，使用对应的`TypeHandler`将其转换成相应类型的参数。
  3. 真正的设置参数的方法是`TypeHandler`中`setParameter`方法

- 继续`图6`的逻辑，参数已经设置完了，此时就该执行SQL了，真正执行SQL的是`PreparedStatementHandler`中的`<E> List<E> query(Statement statement, ResultHandler resultHandler)`方法，源码如下`图7`：
  ![](https://img.java-family.cn/Mybatis%E6%BA%90%E7%A0%81-3/7.png)
- 上图的逻辑其实很简单，一个是JDBC执行SQL语句，一个是调用六剑客之一的`ResultSetHandler`对结果进行处理。
- 真正对结果进行处理的是`DefaultResultSetHandler`中的`handleResultSets`方法，源码比较复杂，这里就不再展示了，具体的逻辑如下：
  1. 获取结果映射(`resultMap`)，如果没有指定，使用内置的结果映射
  2. 遍历结果集，对SQL返回的每个结果通过结果集和`TypeHandler`进行结果映射。
  3. 返回结果
- ResultSetHandler对结果处理结束之后就会返回。至此一条`selectList()`如何执行的大概心里已经有了把握，其他的更新，删除都是大同小异。



## 总结
- Mybatis的源码算是几种常用框架中比较简单的，都是围绕六大组件进行的，只要搞懂了每个组件是什么角色，有什么作用，一切都会很简单。
- 一条select语句简单执行的逻辑总结如下（前提：**默认配置**）：
  1. **SqlSesion**：`#SqlSessionTemplate.selectList()`实际调用`#DefaultSqlSession.selectList()`
  2. **Executor**：`#DefaultSqlSession.quer()`实际调用的是`#CachingExecutor().query()`，如果二级缓存中存在直接返回，不存在调用`#BaseExecutor.quer()`查询一级缓存，如果一级缓存中存在直接返回。不存在调用`#SimpleExecutor.doQuery()`方法查询数据库。
  3. **StatementHandler**：`#SimpleExecutor.doQuery()`生成`StatementHandler`实例，执行`#PreparedStatementHandler.parameterize()`方法设置参数，实际调用的是`#ParamterHandler.setParameters()`方法，该方法内部调用`TypeHandler.setParameter()`方法进行类型转换;参数设置成功后，调用`#PreparedStatementHandler.parameterize().query()`方法执行SQL，返回结果
  4. **ResultSetHandler**：`#DefaultResultSetHandler.handleResultSets()`对返回的结果进行处理，内部调用`#TypeHandler.getResult()`对结果进行类型转换。全部映射完成，返回结果。

- 以上就是六剑客在Select的执行流程，如果有错误之处欢迎指正，如果觉得陈某写得不错，有所收获，关注分享一波。

  

  




