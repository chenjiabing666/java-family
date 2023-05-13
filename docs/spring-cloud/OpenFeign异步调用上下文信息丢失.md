
# OpenFeign 异步调用丢失上下文信息

最近有读者问了这样一个问题：陈哥，**openFeign异步调用总是失败触发Sentinel的降级，同步调用就没问题？**

他还给我晒了一下代码，大致如下：

```java
CompletableFuture<T> future1 = CompletableFuture.supplyAsync(() -> {
	//openfeign的调用
	return feign.remoteCall();
},executor);

CompletableFuture<T> future2 = CompletableFuture.supplyAsync(() -> {
	//openfeign的调用
	return feign.remoteCall();
},executor);

CompletableFuture.allOf(future1,future2).join();

.....
```



**这种情况你遇到过吗？**



## 如何解决？

这个算是常见问题了：feign的调用导致上下文丢失；前面有一篇文章说过这种问题：[实战！openFeign如何实现全链路JWT令牌信息不丢失？](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247504759&idx=1&sn=e50d5b44eb64debf43c6d644f55c68b5&chksm=fcf70cbacb8085aca5cd88688973ed45cd8bd9a4642ae97727f3684b431f80316e5c073d6946&scene=178&cur_album_id=2042874937312346114#rd)

在集成OAuth2的时候如果做任何设置会丢失令牌信息，当时我们的解决方案是新建一个拦截器，如下：

```java
@Component
@Slf4j
public class FeignRequestInterceptor implements RequestInterceptor {
    @Override
    public void apply(RequestTemplate template) {
        HttpServletRequest httpServletRequest = RequestContextUtils.getRequest();
        Map<String, String> headers = getHeaders(httpServletRequest);
        for (Map.Entry<String, String> entry : headers.entrySet()) {
            template.header(entry.getKey(), entry.getValue());
        }
    }

    /**
     * 获取原请求头
     */
    private Map<String, String> getHeaders(HttpServletRequest request) {
        Map<String, String> map = new LinkedHashMap<>();
        Enumeration<String> enumeration = request.getHeaderNames();
        if (enumeration != null) {
            while (enumeration.hasMoreElements()) {
                String key = enumeration.nextElement();
                String value = request.getHeader(key);
                if (StrUtil.equals(OAuthConstant.TOKEN_NAME,key)){
                    map.put(key, value);
                    break;
                }
            }
        }
        return map;
    }
}
```



上述代码根本逻辑就是将请求头中Token信息放入**RequestTemplate**的头中。

> 注意：这里取出请求头中信息用的是**RequestContextHolder**。

看到这里是不是明白了，**RequestContextHolder**中是将请求信息放入**ThreadLocal**中的，只能取到同一个线程的数据。

因此要解决异步调用的问题，只需要在发起远程调用之前给异步线程添加上主线程的上下文信息，此时最上方调用失效的代码变成如下：

```java
//获取主线程的请求信息
RequestAttributes attributes = RequestContextHolder.getRequestAttributes();

CompletableFuture<T> future1 = CompletableFuture.supplyAsync(() -> {
   //将主线程的请求信息设置到异步线程中，否则会丢失请求上下文，导致调用失败
	RequestContextHolder.setRequestAttributes(attributes);
    
	//openfeign的调用
	return feign.remoteCall();
},executor);

CompletableFuture<T> future2 = CompletableFuture.supplyAsync(() -> {
   //将主线程的请求信息设置到异步线程中，否则会丢失请求上下文，导致调用失败
	RequestContextHolder.setRequestAttributes(attributes);
	
    //openfeign的调用
	return feign.remoteCall();
},executor);

CompletableFuture.allOf(future1,future2).join();

.....
```

## 最后说一句（别白嫖，求关注）

陈某每一篇文章都是精心输出，已经写了**3个专栏**，整理成**PDF**，获取方式如下：

1. [《Spring Cloud 进阶》](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU3MDAzNDg1MA==&action=getalbum&album_id=2042874937312346114#wechat_redirect)PDF：关注公众号：【**码猿技术专栏**】回复关键词 **Spring Cloud 进阶** 获取！
2. [《Spring Boot 进阶》](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU3MDAzNDg1MA==&action=getalbum&album_id=1532834475389288449#wechat_redirect)PDF：关注公众号：【**码猿技术专栏**】回复关键词 **Spring Boot进阶** 获取！
3. [《Mybatis 进阶》](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU3MDAzNDg1MA==&action=getalbum&album_id=1500819225232343046#wechat_redirect)PDF：关注公众号：【**码猿技术专栏**】回复关键词 **Mybatis 进阶** 获取！

如果这篇文章对你有所帮助，或者有所启发的话，帮忙**点赞**、**在看**、**转发**、**收藏**，你的支持就是我坚持下去的最大动力！

关注公众号：**【码猿技术专栏】**，公众号内有超赞的粉丝福利，回复：**加群**，可以加入技术讨论群，和大家一起讨论技术，吹牛逼！













