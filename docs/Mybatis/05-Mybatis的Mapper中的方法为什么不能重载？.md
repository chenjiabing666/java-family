
## 前言
- 在初入门`Mybatis`的时候可能都犯过一个错误，那就是在写`Mapper`接口的时候都重载过其中的方法，但是运行起来总是报错，那时候真的挺郁闷的，但是自己也查不出来原因，只能默默的改了方法名，哈哈，多么卑微的操作。
- 今天就写一篇文章从源码角度为大家解惑为什么`Mybatis`中的方法不能重载？

## 环境配置
- 本篇文章讲的一切内容都是基于`Mybatis3.5`和`SpringBoot-2.3.3.RELEASE`。

## 错误示范
- 举个栗子：假设现在有两个需求，一个是根据用户的id筛选用户，一个是根据用户的性别筛选，此时在Mapper中重载的方法如下：
```java
public interface UserMapper {
    List<UserInfo> selectList(@Param("userIds") List<String> userIds);

    List<UserInfo> selectList(Integer gender);
    }
```

- 这个并没有什么错误，但是启动项目，报出如下的错误：
```java
Caused by: org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'sqlSessionFactory' defined in class path resource [org/mybatis/spring/boot/autoconfigure/MybatisAutoConfiguration.class]: Bean instantiation via factory method failed; nested exception is org.springframework.beans.BeanInstantiationException: Failed to instantiate [org.apache.ibatis.session.SqlSessionFactory]: Factory method 'sqlSessionFactory' threw exception; nested exception is org.springframework.core.NestedIOException: Failed to parse mapping resource: 'file [H:\work_project\demo\target\classes\mapper\UserInfoMapper.xml]'; nested exception is org.apache.ibatis.builder.BuilderException: Error parsing Mapper XML. The XML location is 'file [H:\work_project\demo\target\classes\mapper\UserInfoMapper.xml]'. Cause: java.lang.IllegalArgumentException: Mapped Statements collection already contains value for cn.cb.demo.dao.UserMapper.selectList. please check file [H:\work_project\demo\target\classes\mapper\UserInfoMapper.xml] and file [H:\work_project\demo\target\classes\mapper\UserInfoMapper.xml]
	at org.springframework.beans.factory.support.ConstructorResolver.instantiate(ConstructorResolver.java:655)
	at org.springframework.beans.factory.support.ConstructorResolver.instantiateUsingFactoryMethod(ConstructorResolver.java:635)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.instantiateUsingFactoryMethod(AbstractAutowireCapableBeanFactory.java:1336)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBeanInstance(AbstractAutowireCapableBeanFactory.java:1176)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:556)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean(AbstractAutowireCapableBeanFactory.java:516)
	at org.springframework.beans.factory.support.AbstractBeanFactory.lambda$doGetBean$0(AbstractBeanFactory.java:324)
	at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton(DefaultSingletonBeanRegistry.java:226)
	at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:322)
	at org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:202)
	at org.springframework.beans.factory.config.DependencyDescriptor.resolveCandidate(DependencyDescriptor.java:276)
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.doResolveDependency(DefaultListableBeanFactory.java:1307)
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.resolveDependency(DefaultListableBeanFactory.java:1227)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.autowireByType(AbstractAutowireCapableBeanFactory.java:1509)
	... 81 more
```
- 这么一大串什么意思？懵逼了~
  ![](https://img.java-family.cn/Mybatis为什么方法不能重载/1.jpg)

- 大致的意思：`cn.cb.demo.dao.UserMapper.selectList`这个`id`已经存在了，导致创建`sqlSessionFactory`失败。


## 为什么不能重载？
- 通过上面的异常提示可以知道创建`sqlSessionFactory`失败了，这个想必已经不陌生吧，顾名思义，就是创建`SqlSession`的工厂。
- Springboot与Mybatis会有一个启动器的自动配置类`MybatisAutoConfiguration`，其中有一段代码就是创建`sqlSessionFactory`，如下图：
  ![](https://img.java-family.cn/Mybaits%E4%B8%AD%E7%9A%84%E6%96%B9%E6%B3%95%E4%B8%BA%E4%BB%80%E4%B9%88%E4%B8%8D%E8%83%BD%E9%87%8D%E8%BD%BD%EF%BC%9F/2.png)
- 既然是创建失败，那么肯定是这里出现异常了，这里的**大致思路**就是：
> 解析`XML`文件和`Mapper`接口，将Mapper中的方法与XML文件中`<select>`、`<insert>`等标签一一对应，那么Mapper中的方法如何与XML中`<select>`这些标签对应了，当然是唯一的`id`对应了，具体如何这个`id`的值是什么，如何对应？下面一一讲解。
- 如上图的`SqlSessionFactory`的创建过程中，前面的部分代码都是设置一些配置，并没有涉及到解析XML的内容，因此答案肯定是在最后一行`return factory.getObject();`，于是此处打上断点，一点点看。于是一直到了`org.mybatis.spring.SqlSessionFactoryBean#buildSqlSessionFactory`这个方法中，其中一段代码如下：
  ![](https://img.java-family.cn/Mybaits%E4%B8%AD%E7%9A%84%E6%96%B9%E6%B3%95%E4%B8%BA%E4%BB%80%E4%B9%88%E4%B8%8D%E8%83%BD%E9%87%8D%E8%BD%BD%EF%BC%9F/3.png)
- 这里的`xmlMapperBuilder.parse();`就是解析XML文件与Mapper接口，继续向下看。
- 略过不重要的代码，在`org.apache.ibatis.builder.xml.XMLMapperBuilder#configurationElement`这个方法中有一行重要的代码，如下图：
  ![](https://img.java-family.cn/Mybaits%E4%B8%AD%E7%9A%84%E6%96%B9%E6%B3%95%E4%B8%BA%E4%BB%80%E4%B9%88%E4%B8%8D%E8%83%BD%E9%87%8D%E8%BD%BD%EF%BC%9F/4.png)
- 此处就是根据XML文件中的`select|insert|update|delete`这些标签开始构建`MappedStatement`了。继续跟进去看。
- 略过不重要的代码，此时看到`org.apache.ibatis.builder.MapperBuilderAssistant#addMappedStatement`这个方法返回值就是`MappedStatement`，不用多说，肯定是这个方法了，仔细一看，很清楚的看到了构建`id`的代码，如下图：
  ![](https://img.java-family.cn/Mybaits%E4%B8%AD%E7%9A%84%E6%96%B9%E6%B3%95%E4%B8%BA%E4%BB%80%E4%B9%88%E4%B8%8D%E8%83%BD%E9%87%8D%E8%BD%BD%EF%BC%9F/5.png)
- 从上图可以知道，创建`id`的代码就是`id = applyCurrentNamespace(id, false);`，具体实现如下图：
  ![](https://img.java-family.cn/Mybaits%E4%B8%AD%E7%9A%84%E6%96%B9%E6%B3%95%E4%B8%BA%E4%BB%80%E4%B9%88%E4%B8%8D%E8%83%BD%E9%87%8D%E8%BD%BD%EF%BC%9F/6.png)
> 上图的代码已经很清楚了，`MappedStatement`中的`id=Mapper的全类名+'.'+方法名`。如果重载话，肯定会存在`id`相同的`MappedStatement`。

- 到了这其实并不能说明方法不能重载啊，重复就重复呗，并没有冲突啊。这里需要看一个结构，如下：
```java
protected final Map<String, MappedStatement> mappedStatements = new StrictMap<MappedStatement>("Mapped Statements collection")
      .conflictMessageProducer((savedValue, targetValue) ->
          ". please check " + savedValue.getResource() + " and " + targetValue.getResource());
```
- 构建好的`MappedStatement`都会存入`mappedStatements`中，如下代码：
```java
public void addMappedStatement(MappedStatement ms) {
    //key 是id 
    mappedStatements.put(ms.getId(), ms);
  }
```
- `StrictMap`的`put(k,v)`方法如下图：
  ![](https://img.java-family.cn/Mybaits%E4%B8%AD%E7%9A%84%E6%96%B9%E6%B3%95%E4%B8%BA%E4%BB%80%E4%B9%88%E4%B8%8D%E8%83%BD%E9%87%8D%E8%BD%BD%EF%BC%9F/7.png)

> 到了这里应该理解了吧，这下抛出的异常和上面的`异常信息`对应起来了吧。这个`StrictMap`不允许有重复的`key`，而存入的`key`就是`id`。因此Mapper中的方法不能重载。

## 如何找到XML中对应的SQL？
- 在使用Mybatis的时候只是简单的调用Mapper中的方法就可以执行SQL，如下代码：
```java
List<UserInfo> userInfos = userMapper.selectList(Arrays.asList("192","198"));
```
> 一行简单的调用到底如何找到对应的SQL呢？其实就是根据`id`从`Map<String, MappedStatement> mappedStatements`中查找对应的`MappedStatement`。

- 在`org.apache.ibatis.session.defaults.DefaultSqlSession#selectList`方法有这一行代码如下图：
  ![](https://img.java-family.cn/Mybaits%E4%B8%AD%E7%9A%84%E6%96%B9%E6%B3%95%E4%B8%BA%E4%BB%80%E4%B9%88%E4%B8%8D%E8%83%BD%E9%87%8D%E8%BD%BD%EF%BC%9F/8.png)
- `MappedStatement ms = configuration.getMappedStatement(statement);`这行代码就是根据`id`从`mappedStatements`获取对应的`MappedStatement`，源码如下:
```java
public MappedStatement getMappedStatement(String id) {
    return this.getMappedStatement(id, true);
  }
```

## 总结
- 文章写到这，想必已经很清楚Mapper中的方法为什么不能重载了，归根到底就是因为这个这个`id=Mapper的全类名+'.'+方法名`。


