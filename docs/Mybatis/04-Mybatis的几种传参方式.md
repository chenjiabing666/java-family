
## 前言
- 前几天恰好面试一个应届生，问了一个很简单的问题：你了解过Mybatis中有几种传参方式吗？
- 没想到其他问题回答的很好，唯独这个问题一知半解，勉强回答了其中两种方式。
- 于是这篇文章就来说一说Mybatis传参的几种常见方式，给正在面试或者准备面试的朋友巩固一下。


## 单个参数
- 单个参数的传参比较简单，可以是任意形式的，比如`#{a}`、`#{b}`或者`#{param1}`，**但是为了开发规范，尽量使用和入参时一样**。
- Mapper如下：
```java
UserInfo selectByUserId(String userId);
```
- XML如下：
```xml
<select id="selectByUserId" resultType="cn.cb.demo.domain.UserInfo">
        select * from user_info where user_id=#{userId} and status=1
  </select>
```

## 多个参数
- 多个参数的情况下有很多种传参的方式，下面一一介绍。

### 使用索引【不推荐】
- 多个参数可以使用类似于索引的方式传值，比如`#{param1}`对应第一个参数，`#{param2}`对应第二个参数.......
- Mapper方法如下：
```java
UserInfo selectByUserIdAndStatus(String userId,Integer status);
```
- XML如下：
```xml
<select id="selectByUserIdAndStatus" resultType="cn.cb.demo.domain.UserInfo">
        select * from user_info where user_id=#{param1} and status=#{param2}
    </select>
```
- **注意**：由于开发规范，此种方式不推荐开发中使用。

### 使用@Param
- `@Param`这个注解用于指定key，一旦指定了key，在SQL中即可对应的key入参。
- Mapper方法如下：
```java
UserInfo selectByUserIdAndStatus(@Param("userId") String userId,@Param("status") Integer status);
```
- XML如下：
```xml
<select id="selectByUserIdAndStatus" resultType="cn.cb.demo.domain.UserInfo">
        select * from user_info where user_id=#{userId} and status=#{status}
    </select>
```

### 使用Map
- Mybatis底层就是将入参转换成`Map`，入参传Map当然也行，此时`#{key}`中的`key`就对应Map中的`key`。
- Mapper中的方法如下：
```java
UserInfo selectByUserIdAndStatusMap(Map<String,Object> map);
```
- XML如下：
```java
<select id="selectByUserIdAndStatusMap" resultType="cn.cb.demo.domain.UserInfo">
        select * from user_info where user_id=#{userId} and status=#{status}
    </select>
```
- 测试如下：
```java
@Test
    void contextLoads() {
        Map<String,Object> map=new HashMap<>();
        map.put("userId","1222");
        map.put("status",1);
        UserInfo userInfo = userMapper.selectByUserIdAndStatusMap(map);
        System.out.println(userInfo);
    }
```

### POJO【推荐】
- 多个参数可以使用实体类封装，此时对应的`key`就是属性名称，注意一定要有`get`方法。
- Mapper方法如下：
```java
UserInfo selectByEntity(UserInfoReq userInfoReq);
```
- XML如下：
```xml
<select id="selectByEntity" resultType="cn.cb.demo.domain.UserInfo">
        select * from user_info where user_id=#{userId} and status=#{status}
    </select>
```
- 实体类如下：
```java
@Data
public class UserInfoReq {
    private String userId;
    private Integer status;
}
```

## List传参
- List传参也是比较常见的，通常是SQL中的`in`。
- Mapper方法如下：
```java
List<UserInfo> selectList( List<String> userIds);
```
- XML如下：
```xml
<select id="selectList" resultMap="userResultMap">
        select * from user_info where status=1
        and user_id in
        <foreach collection="list" item="item" open="(" separator="," close=")" >
            #{item}
        </foreach>
    </select>
```

## 数组传参
- 这种方式类似List传参，依旧使用`foreach`语法。
- Mapper方法如下：
```java
List<UserInfo> selectList( String[] userIds);
```
- XML如下：
```xml
<select id="selectList" resultMap="userResultMap">
        select * from user_info where status=1
        and user_id in
        <foreach collection="array" item="item" open="(" separator="," close=")" >
            #{item}
        </foreach>
    </select>
```

## 总结
- 以上几种传参的方式在面试或者工作中都会用到，不了解的朋友可以收藏下。


