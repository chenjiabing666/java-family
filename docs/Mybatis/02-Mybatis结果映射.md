

## 前言
- 上一篇文章介绍了Mybatis基础的CRUD操作、常用的标签、属性等内容，如果对部分不熟悉的朋友可以看[Mybatis入门之基本操作](https://mp.weixin.qq.com/s/KdrEvlShnVoYA8nr0qLSNw)。
- 本篇文章继续讲解Mybatis的结果映射的内容，想要在企业开发中灵活的使用Mybatis，这部分的内容是必须要精通的。


## 什么是结果映射？
- 简单的来说就是一条`SQL查询语句`返回的字段如何与`Java实体类`中的属性相对应。
- 如下一条SQL语句，查询患者的用户id，科室id，主治医生id：
```xml
  <select id='selectPatientInfos' resultType='com.xxx.domain.PatientInfo'>
    select user_id,dept_id,doc_id from patient_info;
  </select>
```
- Java实体类`PatientInfo`如下：
```java
@Data
public class PatientInfo{
  private String userId;
  private String deptId;
  private String docId;
}
```

- 程序员写这条SQL的目的就是想查询出来的`user_id`,`dept_id`,`doc_id`分别赋值给实体类中的`userId`,`deptId`,`docId`。这就是简单的结果映射。


## 如何映射？
- Myabtis中的结果映射有很多种方式，下面会逐一介绍。

### 别名映射
- 这个简单，保持查询的SQL返回的字段和Java实体类一样即可，比如上面例子的SQL可以写成：
```xml
<select id='selectPatientInfos' resultType='com.xxx.domain.PatientInfo'>
   select user_id as userId,
   dept_id as deptId,
   doc_id as docId
   from patient_info; 
</select>
```
- 这样就能和实体类中的属性映射成功了。

### 驼峰映射
- Mybatis提供了驼峰命名映射的方式，比如数据库中的`user_id`这个字段，能够自动映射到`userId`属性。那么此时的查询的SQL变成如下即可：
```xml
<select id='selectPatientInfos' resultType='com.xxx.domain.PatientInfo'>
    select user_id,dept_id,doc_id from patient_info;
  </select>
```
- 如何开启呢？与SpringBoot整合后开启其实很简单，有两种方式，一个是配置文件中开启，一个是配置类开启。

#### 配置文件开启驼峰映射
- 只需要在`application.properties`文件中添加如下一行代码即可：
```properties
mybatis.configuration.map-underscore-to-camel-case=true
```
#### 配置类中开启驼峰映射【简单了解，后续源码章节着重介绍】
- 这种方式需要你对源码有一定的了解，上一篇入门教程中有提到，Mybatis与Springboot整合后适配了一个starter，那么肯定会有自动配置类，Mybatis的自动配置类是`MybatisAutoConfiguration`，其中有这么一段代码，如下：
  ![](https://img.java-family.cn/Mybatis-%E7%BB%93%E6%9E%9C%E6%98%A0%E5%B0%84/1.png)
- `@ConditionalOnMissingBean`这个注解的意思就是当IOC容器中没有`SqlSessionFactory`这个Bean对象这个配置才会生效;`applyConfiguration(factory)`这行代码就是创建一个`org.apache.ibatis.session.Configuration`赋值给`SqlSessionFactoryBean`。源码分析到这，应该很清楚了，无非就是自己在容器中创建一个`SqlSessionFactory`，然后设置属性即可，如下代码：

```java
    @Bean("sqlSessionFactory")
    public SqlSessionFactory sqlSessionFactory(DataSource dataSource) throws Exception {
        SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
        //设置数据源
        sqlSessionFactoryBean.setDataSource(dataSource);
        //设置xml文件的位置
        sqlSessionFactoryBean.setMapperLocations(new PathMatchingResourcePatternResolver().getResources(MAPPER_LOCATOIN));
        //创建Configuration
        org.apache.ibatis.session.Configuration configuration = new org.apache.ibatis.session.Configuration();
        // 开启驼峰命名映射
        configuration.setMapUnderscoreToCamelCase(true);
        configuration.setDefaultFetchSize(100);
        configuration.setDefaultStatementTimeout(30);
        sqlSessionFactoryBean.setConfiguration(configuration);
        //将typehandler注册到mybatis
        sqlSessionFactoryBean.setTypeHandlers(typeHandlers());
        return sqlSessionFactoryBean.getObject();
    }
```
- **注意**：如果对`SqlSessionFactory`没有特殊定制，不介意重写，因为这会自动覆盖自动配置类中的配置。


### resultMap映射

- 什么是`resultMap`？简单的说就是一个类似Map的结构，将数据库中的字段和JavaBean中的属性字段对应起来，这样就能做到一一映射了。
- 上述的例子使用resultMap又会怎么写呢？如下：
```xml

<!--创建一个resultMap映射-->
<resultMap id="patResultMap" type="com.xxx.domain.PatientInfo">
  <id property="userId" column="user_id" />
  <result property="docId" column="doc_id"/>
  <result property="deptId" column="dept_id"/>
</resultMap>

<!--使用resultMap映射结果到com.xxx.domain.PatientInfo这个Bean中-->
<select id='selectPatientInfos' resultMap='patResultMap'>
    select user_id,dept_id,doc_id from patient_info;
  </select>
```

- 其实很简单，就是创建一个`<resultMap>`，然后`<select>`标签指定这个resultMap即可。

- `<resultMap>`的属性如下：
  - `id`：唯一标识这个resultMap，同一个Mapper.xml中不能重复
  - `type`：指定JavaBean的类型，可以是全类名，也可以是别名
- 子标签`<result>`的属性如下：
  - `column`：SQL返回的字段名称
  - `property`：JavaBean中属性的名称
  - `javaType`：一个 Java 类的全限定名，或一个类型别名（关于内置的类型别名，可以参考上面的表格）。 如果你映射到一个 JavaBean，MyBatis 通常可以推断类型。然而，如果你映射到的是 HashMap，那么你应该明确地指定 javaType 来保证行为与期望的相一致。
  - `jdbcType`：JDBC 类型，所支持的 JDBC 类型参见这个表格之后的“支持的 JDBC 类型”。 只需要在可能执行插入、更新和删除的且允许空值的列上指定 JDBC 类型。这是 JDBC 的要求而非 MyBatis 的要求。如果你直接面向 JDBC 编程，你需要对可以为空值的列指定这个类型。
  - `typeHandler`： 这个属性值是一个类型处理器实现类的全限定名，或者是类型别名。
  - `resultMap`：结果映射的 ID，可以将此关联的嵌套结果集映射到一个合适的对象树中。 它可以作为使用额外 select 语句的替代方案。

### 总结
- 以上列举了三种映射的方式，分别是**别名映射**，**驼峰映射**、**`resultMap`映射**。
- 你以为这就结束了？要是世界这么简单多好，做梦吧，哈哈！！！
  ![](https://img.java-family.cn/Mybatis-%E7%BB%93%E6%9E%9C%E6%98%A0%E5%B0%84/2.jpg)


## 高级结果映射
- MyBatis 创建时的一个思想是：数据库不可能永远是你所想或所需的那个样子。 我们希望每个数据库都具备良好的第三范式或 BCNF 范式，可惜它们并不都是那样。 如果能有一种数据库映射模式，完美适配所有的应用程序，那就太好了，但可惜也没有。 而 ResultMap 就是 MyBatis 对这个问题的答案。
- 我们知道在数据库的关系中一对一，多对一，一对多，多对多的关系，那么这种关系如何在Mybatis中体现并映射成功呢？

### 关联(association)
- 关联（association）元素处理**有一个**类型的关系。 比如，在我们的示例中，一个员工属于一个部门。关联结果映射和其它类型的映射工作方式差不多。 你需要指定目标属性名以及属性的`javaType`（很多时候 MyBatis 可以自己推断出来），在必要的情况下你还可以设置 `JDBC` 类型，如果你想覆盖获取结果值的过程，还可以设置类型处理器。
- 关联的不同之处是，你需要告诉 MyBatis 如何加载关联。MyBatis 有两种不同的方式加载关联：
  - `嵌套 Select 查询`：通过执行另外一个 SQL 映射语句来加载期望的复杂类型。
  - `嵌套结果映射`：使用嵌套的结果映射来处理连接结果的重复子集。
- 首先，先让我们来看看这个元素的属性。你将会发现，和普通的结果映射相比，它只在 `select` 和 `resultMap` 属性上有所不同。
  - `property`：	映射到列结果的字段或属性。如果用来匹配的 JavaBean 存在给定名字的属性，那么它将会被使用。
  - `javaType`：一个 Java 类的完全限定名，或一个类型别名（关于内置的类型别名，可以参考上面的表格）
     `jdbcType`：	JDBC 类型， 只需要在可能执行插入、更新和删除的且允许空值的列上指定 JDBC 类型
  - `typeHandler`：使用这个属性，你可以覆盖默认的类型处理器。 这个属性值是一个类型处理器实现类的完全限定名，或者是类型别名。
     `column`：	数据库中的列名，或者是列的别名。一般情况下，这和传递给 `resultSet.getString(columnName)` 方法的参数一样。 注意：在使用复合主键的时候，你可以使用 `column="{prop1=col1,prop2=col2}"` 这样的语法来指定多个传递给嵌套 Select 查询语句的列名。这会使得` prop1 `和 `prop2` 作为参数对象，被设置为对应嵌套 Select 语句的参数。
  - `select`：用于加载复杂类型属性的映射语句的 ID，它会从 column 属性指定的列中检索数据，作为参数传递给目标 select 语句。 具体请参考下面的例子。注意：在使用复合主键的时候，你可以使用`column="{prop1=col1,prop2=col2}"` 这样的语法来指定多个传递给嵌套 Select 查询语句的列名。这会使得 prop1 和 prop2 作为参数对象，被设置为对应嵌套 Select 语句的参数。
  - `fetchType`：可选的。有效值为 `lazy` 和 `eager`。 指定属性后，将在映射中忽略全局配置参数 `lazyLoadingEnabled`，使用属性的值。


#### 例子
- 一对一的关系比如：一个员工属于一个部门，那么数据库表就会在员工表中加一个部门的id作为逻辑外键。
- 创建员工JavaBean
```java
@Data
public class User {
	private Integer id;
	private String username;
	private String password;
	private Integer age;
  private Integer deptId;
  //部门
	private Department department;   
}
```
- 部门JavaBean
```java
@Data
public class Department {
	private Integer id;
	private String name;
}
```

- 那么我们想要查询所有的用户信息和其所在的部门信息，此时的sql语句为:`select * from user u left join department d on u.department_id=d.id`;。但是我们在mybaits中如果使用这条语句查询，那么返回的结果类型是什么呢？如果是User类型的，那么查询结果返回的还有`Department`类型的数据，那么肯定会对应不上的。此时`<resultMap>`来了，它来了!!!


#### 关联的嵌套 Select 查询【可以忽略】
- 查询员工和所在的部门在Mybatis如何写呢？代码如下：

```xml
<resultMap id="userResult" type="com.xxx.domain.User">
	<id column="id" property="id"/>
	<result column="password" property="password"/>
	<result column="age" property="age"/>
	<result column="username" property="username"/>
  <result column="dept_id" property="deptId"/>
  <!--关联查询，select嵌套查询-->
  <association property="department" column="dept_id" javaType="com.xxx.domain.Department" select="selectDept"/>
</resultMap>

<!--查询员工-->
<select id="selectUser" resultMap="userResult">
  SELECT * FROM user WHERE id = #{id}
</select>

<!--查询部门-->
<select id="selectDept" resultType="com.xxx.domain.Department ">
  SELECT * FROM department WHERE ID = #{id}
</select>
```

- 就是这么简单，两个select语句，一个用来加载员工，一个用来加载部门。
- 这种方式虽然很简单，但在大型数据集或大型数据表上表现不佳。这个问题被称为`N+1` 查询问题。 概括地讲，N+1 查询问题是这样子的：
  - 你执行了一个单独的 SQL 语句来获取结果的一个列表（就是`+1`）。
  - 对列表返回的每条记录，你执行一个 `select` 查询语句来为每条记录加载详细信息（就是`N`）。
- 这个问题会导致成百上千的 SQL 语句被执行。有时候，我们不希望产生这样的后果。

#### 关联的嵌套结果映射【重点】
- `<association >`标签中还可以直接嵌套结果映射，此时的Mybatis的查询如下：
```xml
<!-- 定义resultMap -->
<resultMap id="UserDepartment" type="com.xxx.domain.User" >
	<id column="user_id" property="id"/>
	<result column="password" property="password"/>
	<result column="age" property="age"/>
	<result column="username" property="username"/>
  <result column="dept_id" property="deptId"/>
	
	<!--
		property: 指定User中对应的部门属性名称
		javaType: 指定类型，可以是全类名或者别名
	 -->
	<association property="department" javaType="com.xx.domain.Department">
    <!--指定Department中的属性映射，这里也可以使用单独拎出来，然后使用association中的resultMap属性指定-->
		<id column="id" property="id"/>
		<result column="dept_name" property="name"/>
	</association>
</resultMap>

<!-- 
	resultMap: 指定上面resultMap的id的值
 -->
 <select id="findUserAndDepartment" resultMap="UserDepartment">
 	select 
   u.id as user_id,
   u.dept_id,
   u.name,
   u.password,
   u.age,
   d.id,
   d.name as dept_name
   from user u left join department d on u.department_id=d.id
 </select>
```

#### 总结

- 至此`有一个`类型的关联已经完成了，学会一个`<association>`使用即能完成。
- **注意**： 关联的嵌套 Select 查询不建议使用，`N+1`是个重大问题，虽说Mybatis提供了延迟加载的功能，但是仍然不建议使用，企业开发中也是不常用的。


### 集合collection 

- 集合，顾名思义，就是处理`有很多个`类型的关联。
- 其中的属性和`association`中的属性类似，不再重复了。
- 比如这样一个例子：查询一个部门中的全部员工，查询SQL如何写呢？如下：
```sql
select * from department d left join user u on u.department_id=d.id;
```
- 此时的`User`实体类如下：
```java
@Data
public class User {
 private Integer id;
 private String username;
 private String password;
 private Integer age;
 private Integer deptId; 
}
```
- 此时的`Department`实体类如下：
```java
@Data
public class Department {
 private Integer id;
 private String name;
 private List<User> users;
}
```

- 和`association`类似，同样有两种方式，我们可以使用嵌套 Select 查询，或基于连接的嵌套结果映射集合。

### 集合的嵌套 Select 查询【可以忽略】
- 不太重要，查询如下：
```xml
<resultMap id="deptResult" type="com.xxx.domain.Department">
  <!--指定Department中的属性映射，这里也可以使用单独拎出来，然后使用association中的resultMap属性指定-->
		<id column="id" property="id"/>
		<result column="name" property="name"/>
  <!--
  ofType：指定实际的JavaBean的全类型或者别名
  select：指定嵌套的select查询
  javaType：集合的类型，可以不写，Mybatis可以推测出来
-->
  <collection property="users" javaType="java.util.ArrayList" column="id" ofType="com.xxx.doamin.User" select="selectByDeptId"/>
</resultMap>

<select id="selectDept" resultMap="deptResult">
  SELECT * FROM department  WHERE ID = #{id}
</select>

<select id="selectByDeptId" resultType="com.xxx.domain.User">
  SELECT * FROM user WHERE dept_id = #{id}
</select>
```

- **注意**：这里出现了一个不同于`association`的属性`ofType`，这个属性非常重要，它用来将 JavaBean（或字段）属性的类型和集合存储的类型区分开来。

### 集合的嵌套结果映射【重点】
- 现在你可能已经猜到了集合的嵌套结果映射是怎样工作的——除了新增的 `ofType` 属性，它和关联的完全相同。
- 此时的Mybatis查询如下：
```xml

<!--部门的resultMap-->
<resultMap id="deptResult" type="com.xxx.domain.Department">
  <!--指定Department中的属性映射，这里也可以使用单独拎出来，然后使用association中的resultMap属性指定-->
		<id column="dept_id" property="id"/>
		<result column="dept_name" property="name"/>
  <!--
  ofType：指定实际的JavaBean的全类型或者别名
  resultMap：指定员工的resultMap
-->
  <collection property="users" ofType="com.xxx.doamin.User" resultMap='userResult'/>
</resultMap>

<!--员工的resultMap-->
<resultMap id="userResult" type="com.xxx.domain.User">
    <id column="user_id" property="id"/>
   <result column="password" property="password"/>
   <result column="age" property="age"/>
   <result column="username" property="username"/>
</resultMap>

<select id="selectDeptById" resultType="com.xxx.domain.Department">
  select 
  d.id as dept_id,
  d.name as dept_name,
  u.id as user_id,
  u.password,
  u.name
  from department d left join user u on u.department_id=d.id
  where d.id=#{id}
</select>
```

## 总结
- 至此Mybatis第二弹之结果映射已经写完了，如果觉得作者写的不错，给个在看关注一波，后续还有更多精彩内容推出。






