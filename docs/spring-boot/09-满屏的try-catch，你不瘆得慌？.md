

## 前言
软件开发过程中难免遇到各种的BUG，各种的异常，一直就是在解决异常的路上永不停歇，如果你的代码中再出现`try(){...}catch(){...}finally{...}`代码块，你还有心情看下去吗？自己不觉得恶心吗？

冗余的代码往往回丧失写代码的动力，每天搬砖似的写代码，真的很难受。今天这篇文章教你如何去掉满屏的`try(){...}catch(){...}finally{...}`，解放你的双手。

## Spring Boot 版本
本文基于的Spring Boot的版本是`2.3.4.RELEASE`。

## 全局统一异常处理的前世今生
早在`Spring 3.x`就已经提出了`@ControllerAdvice`，可以与`@ExceptionHandler`、`@InitBinder`、`@ModelAttribute` 等注解注解配套使用，这几个此处就不再详细解释了。

这几个注解小眼一瞟只有`@ExceptionHandler`与异常有关啊，翻译过来就是`异常处理器`。**其实异常的处理可以分为两类，分别是`局部异常处理`和`全局异常处理`**。

**`局部异常处理`**：`@ExceptionHandler`和`@Controller`注解搭配使用，只有指定的controller层出现了异常才会被`@ExceptionHandler`捕获到，实际生产中怕是有成百上千个controller了吧，显然这种方式不合适。

**`全局异常处理`**：既然局部异常处理不合适了，自然有人站出来解决问题了，于是就有了`@ControllerAdvice`这个注解的横空出世了，`@ControllerAdvice`搭配`@ExceptionHandler`彻底解决了全局统一异常处理。当然后面还出现了`@RestControllerAdvice`这个注解，其实就是`@ControllerAdvice`和`@ResponseBody`结晶。

## Spring Boot的异常如何分类？

Java中的异常就很多，更别说Spring Boot中的异常了，这里不再根据传统意义上Java的异常进行分类了，而是按照`controller`进行分类，分为`进入controller前的异常`和`业务层的异常`，如下图：

![](https://img.java-family.cn/Spring%20Boot%E7%AC%AC%E5%85%AB%E5%BC%B9%EF%BC%8C%E5%85%A8%E5%B1%80%E5%BC%82%E5%B8%B8/1.png)

进入controller之前异常一般是`javax.servlet.ServletException`类型的异常，因此在全局异常处理的时候需要统一处理。几个常见的异常如下：
1. `NoHandlerFoundException`：客户端的请求没有找到对应的controller，将会抛出`404`异常。
2. `HttpRequestMethodNotSupportedException`：若匹配到了（匹配结果是一个列表，不同的是http方法不同，如：Get、Post等），则尝试将请求的http方法与列表的控制器做匹配，若没有对应http方法的控制器，则抛该异常
3. `HttpMediaTypeNotSupportedException`：然后再对请求头与控制器支持的做比较，比如`content-type`请求头，若控制器的参数签名包含注解`@RequestBody`，但是请求的`content-type`请求头的值没有包含`application/json`，那么会抛该异常（当然，不止这种情况会抛这个异常）
4. `MissingPathVariableException`：未检测到路径参数。比如url为：/user/{userId}，参数签名包含`@PathVariable("userId")`，当请求的url为/user，在没有明确定义url为/user的情况下，会被判定为：缺少路径参数

## 如何统一异常处理？
在统一异常处理之前其实还有许多东西需要优化的，比如统一结果返回的形式。当然这里不再细说了，不属于本文范畴。

**统一异常处理很简单，这里以前后端分离的项目为例，步骤如下**：
1. 新建一个统一异常处理的一个类
2. 类上标注`@RestControllerAdvice`这一个注解，或者同时标注`@ControllerAdvice`和`@ResponseBody`这两个注解。
3. 在方法上标注`@ExceptionHandler`注解，并且指定需要捕获的异常，可以同时捕获多个。

下面是作者随便配置一个demo，如下：
```java
/**
 * 全局统一的异常处理，简单的配置下，根据自己的业务要求详细配置
 */
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {


    /**
     * 重复请求的异常
     * @param ex
     * @return
     */
    @ExceptionHandler(RepeatSubmitException.class)
    public ResultResponse onException(RepeatSubmitException ex){
        //打印日志
        log.error(ex.getMessage());
        //todo 日志入库等等操作

        //统一结果返回
        return new ResultResponse(ResultCodeEnum.CODE_NOT_REPEAT_SUBMIT);
    }


    /**
     * 自定义的业务上的异常
     */
    @ExceptionHandler(ServiceException.class)
    public ResultResponse onException(ServiceException ex){
        //打印日志
        log.error(ex.getMessage());
        //todo 日志入库等等操作

        //统一结果返回
        return new ResultResponse(ResultCodeEnum.CODE_SERVICE_FAIL);
    }


    /**
     * 捕获一些进入controller之前的异常，有些4xx的状态码统一设置为200
     * @param ex
     * @return
     */
    @ExceptionHandler({HttpRequestMethodNotSupportedException.class,
            HttpMediaTypeNotSupportedException.class, HttpMediaTypeNotAcceptableException.class,
            MissingPathVariableException.class, MissingServletRequestParameterException.class,
            ServletRequestBindingException.class, ConversionNotSupportedException.class,
            TypeMismatchException.class, HttpMessageNotReadableException.class,
            HttpMessageNotWritableException.class,
            MissingServletRequestPartException.class, BindException.class,
            NoHandlerFoundException.class, AsyncRequestTimeoutException.class})
    public ResultResponse onException(Exception ex){
        //打印日志
        log.error(ex.getMessage());
        //todo 日志入库等等操作

        //统一结果返回
        return new ResultResponse(ResultCodeEnum.CODE_FAIL);
    }
}
```

**注意**：**上面的只是一个例子，实际开发中还有许多的异常需要捕获，比如`TOKEN失效`、`过期`等等异常，如果整合了其他的框架，还要注意这些框架抛出的异常，比如`Shiro`，`Spring Security`等等框架。**

## 异常匹配的顺序是什么？
有些朋友可能疑惑了，如果我同时捕获了父类和子类，那么到底能够被那个异常处理器捕获呢？比如`Exception`和`ServiceException`。
![](https://img.java-family.cn/Spring%20Boot%E7%AC%AC%E5%85%AB%E5%BC%B9%EF%BC%8C%E5%A6%82%E4%BD%95%E6%89%A9%E5%B1%95%E5%85%A8%E9%9D%A2%E6%8E%A5%E5%8F%A3MVC/1.jpg)

此时可能就疑惑了，**这里先揭晓一下答案，当然是`ServiceException`的异常处理器捕获了，精确匹配，如果没有`ServiceException`的异常处理器才会轮到它的`父亲`，`父亲`没有才会到`祖父`。总之一句话，精准匹配，找那个关系最近的。**

为什么呢？这可不是凭空瞎说的，源码为证，出处`org.springframework.web.method.annotation.ExceptionHandlerMethodResolver#getMappedMethod`，如下：
```java
@Nullable
	private Method getMappedMethod(Class<? extends Throwable> exceptionType) {
		List<Class<? extends Throwable>> matches = new ArrayList<>();
    //遍历异常处理器中定义的异常类型
		for (Class<? extends Throwable> mappedException : this.mappedMethods.keySet()) {
      //是否是抛出异常的父类，如果是添加到集合中
			if (mappedException.isAssignableFrom(exceptionType)) {    
        //添加到集合中
				matches.add(mappedException);  
			}
		}
    //如果集合不为空，则按照规则进行排序
		if (!matches.isEmpty()) {
			matches.sort(new ExceptionDepthComparator(exceptionType));
      //取第一个
			return this.mappedMethods.get(matches.get(0));
		}
		else {
			return null;
		}
	}
```

**在初次异常处理的时候会执行上述的代码找到最匹配的那个异常处理器方法，后续都是直接从缓存中（一个`Map`结构，`key`是异常类型，`value`是异常处理器方法）。**

别着急，上面代码最精华的地方就是对`matches`进行排序的代码了，我们来看看`ExceptionDepthComparator`这个比较器的关键代码，如下：
```java
//递归调用，获取深度，depth值越小越精准匹配
private int getDepth(Class<?> declaredException, Class<?> exceptionToMatch, int depth) {
    //如果匹配了，返回
		if (exceptionToMatch.equals(declaredException)) {
			// Found it!
			return depth;
		}
		// 递归结束的条件，最大限度了
		if (exceptionToMatch == Throwable.class) {
			return Integer.MAX_VALUE;
		}
    //继续匹配父类
		return getDepth(declaredException, exceptionToMatch.getSuperclass(), depth + 1);
	}
```

**精髓全在这里了，一个递归搞定，计算深度，`depth`初始值为0。值越小，匹配度越高越精准。**


## 总结

全局异常的文章万万千，能够讲清楚的能有几篇呢？**只出最精的文章，做最野的程序员**，如果觉得不错的，关注分享走一波，谢谢支持！！！







