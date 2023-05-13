


## 前言

`Spring Boot`中的热部署相信大家用的最多的就是`devtools`，没办法，官推的。

`JRebel`相对于`devtools`，个人觉得无论是加载速度还是使用便捷，`JRebel`完胜。

作为**前辈级别**的开发利器，`JRebel`真的值得开一章节来好好介绍下。

## JRebel收费怎么破？

前面作者单独写过一篇激活`JRebel`的文章教程，没钱的可以去看看：[撸了个反向代理工具，搞一搞JRebel](https://mp.weixin.qq.com/s/VBGoGMz0y2Y-y6NcdMcaLg)。

> **特此声明**：作者支持原版，不差钱的建议装个原版的，毕竟这么好的工具值得。

## 什么是本地热部署？

传统的开发中，项目在启动过程中代码有所改动是不会重新编译运行的，而是要关闭项目重新启动后修改的代码才会生效。

> **本地热部署**则是能够在项目运行中感知到特定文件代码的修改而使项目不重新启动就能生效。


## 什么是远程热部署？

远程热部署的`远程`两字指的是**远程服务器**，平时开发中，只要本地代码改动了，必须要重新打包上传服务器重新启动之后才会生效，**你这样干过吗？.......**

![嗯？好像干过](https://img.java-family.cn/Spring%20Boot%E4%BD%BF%E7%94%A8JRebel%E7%83%AD%E9%83%A8%E7%BD%B2/2.jpg)

> **远程热部署**则是本地代码改变之后，不用重新打包上传服务器重启项目就能生效，本地改变之后能够自动改变服务器上的项目代码。

有些人听到这里懵逼了，这是什么鬼？还有这么神奇的东西...........

![别惊讶，就是这么神奇](https://img.java-family.cn/Spring%20Boot%E4%BD%BF%E7%94%A8JRebel%E7%83%AD%E9%83%A8%E7%BD%B2/3.jpg)


## JRebel和devtools的区别
前辈和后辈的比较其实没什么可比性，如果不是JRebel**收费**了，绝对是所有程序员的首选。但还是要说说他们之间的区别，如下：

1. `JRebel`加载的速度优于`devtools`
2. JRebel不仅仅局限于Spring Boot项目，可以用在任何的Java项目中。
3. `devtools` 方式的热部署在功能上有限制，方法内的修改可以实现热部署，但新增的方法或者修改方法参数之后热部署是不生效的。


## 如何安装JRebel？
本地热部署只需要在`IDEA`中装一个JRebel的插件，远程热部署需要在服务器上装一个JRebel，这两种方式在上一篇文章都介绍过，不会的可以去看看：[撸了个反向代理工具，搞一搞JRebel](https://mp.weixin.qq.com/s/VBGoGMz0y2Y-y6NcdMcaLg)。


## 如何本地热部署？
`JRebel`插件安装完成之后，将`IDEA`中的`自动编译`开启，然后找到`IDEA`中的`JRebel`的工具面板，将所需要热部署的项目或者模块勾选上即可，如下图：

![](https://img.java-family.cn/Spring%20Boot%E4%BD%BF%E7%94%A8JRebel%E7%83%AD%E9%83%A8%E7%BD%B2/4.png)

> 勾选成功之后将会在项目或者模块的`src/resource`下生成一个`rebel.xml`文件。

此时在`Spring Boot`的主启动类上右键，将会出现以`JRebel`启动的选项，如下图：

![](https://img.java-family.cn/Spring%20Boot%E4%BD%BF%E7%94%A8JRebel%E7%83%AD%E9%83%A8%E7%BD%B2/5.png)

当然在`IDEA`的右上角也存在启动的按钮，如下图：

![](https://img.java-family.cn/Spring%20Boot%E4%BD%BF%E7%94%A8JRebel%E7%83%AD%E9%83%A8%E7%BD%B2/6.png)

> `①`是本地启动和`DEBUG`模式启动，`②`是远程热部署的时候更新按钮。


此时就已经配置成功了，如果勾选的项目或者模块出现了改变，按`CRTL+SHIFT+F9`则会自动重新编译加载改变的部分，不用再重新启动项目了。

## 如何远程热部署？

远程热部署需要在服务器上安装并激活`JRebel`，参照上篇文章：[撸了个反向代理工具，搞一搞JRebel](https://mp.weixin.qq.com/s/VBGoGMz0y2Y-y6NcdMcaLg)。

激活成功后需要设置远程连接的密码，在`JRebel`的根目录下执行以下命令：
```shell
java -jar jrebel.jar -set-remote-password 123456789
```
> 此处设置的`123456789`则是远程的密码，在`IDEA`连接服务器的时候需要。

服务器配置成功后，在IDEA中JRebel的面板中设置远程热部署的模块，如下图：

![](https://img.java-family.cn/Spring%20Boot%E4%BD%BF%E7%94%A8JRebel%E7%83%AD%E9%83%A8%E7%BD%B2/4.png)

> 勾选成功后，将会在`src/resource`下生成一个`rebel-remote.xml`文件。

此时将`Spring Boot`项目打包成一个`Jar`，上传到服务器，执行以下命令启动项目：
```shell
nohup java -agentpath:/usr/local/jrebel/lib/libjrebel64.so  -Drebel.remoting_plugin=true -Xdebug -Xrunjdwp:transport=dt_socket,server=y,suspend=n,address=9083 -jar xxx.jar &
```

`libjrebel64.so`这个文件是`JRebel`的`lib`目录下的文件。

`-Xdebug`之后，`-jar`之前的命令是开启远程调试的，如果不需要的可以去掉，不知道远程调试的，可以看：[惊呆了！Spring Boot还能开启远程调试~](https://mp.weixin.qq.com/s/EoJ8OaJWoXZSrP3Lvexd9Q)。

> 项目启动成功后，服务器上的配置就完成了。

此时在IDEA中需要设置连接到刚才启动的项目，打开`File->setting->JRbel&XRebel->JRbel Remote Servers`，如下图：

![](https://img.java-family.cn/Spring%20Boot%E4%BD%BF%E7%94%A8JRebel%E7%83%AD%E9%83%A8%E7%BD%B2/7.png)

步骤如下：
1. 点击`+`号添加一个服务
2. 填写信息
  - `server name`随便起个服务的名字
  - `server URL`格式：`http://ip:port`，这里的`ip`是服务器的IP，`port`是项目端口号。
  - 远程密码则是上文设置的`JRebel`的密码`123456789`。
3. 点击`OK`，即可添加成功。

以上设置成功后，点击右上角的远程部署按钮，下图中的`②`号按钮，则会自动更新服务器上已启动项目的代码使之本地修改在服务端自动生效：

![](https://img.java-family.cn/Spring%20Boot%E4%BD%BF%E7%94%A8JRebel%E7%83%AD%E9%83%A8%E7%BD%B2/6.png)

在`JRebel Console`这个面板中将会打印出远程热部署更新的日志信息，如下图：

![](https://img.java-family.cn/Spring%20Boot%E4%BD%BF%E7%94%A8JRebel%E7%83%AD%E9%83%A8%E7%BD%B2/8.png)

> 只要本地有了更改，点击远程热部署按钮，则会自动上传代码到服务器端并实时更新，不用重新启动项目。


## 多模块开发的一个坑
如果是多模块开发，比如分为`api`（最终的`Jar`包），`core`（核心包），`service`（业务层的包），最终打包运行在服务器端的是`api`这个模块，其余两个模块都是属于依赖模块，虽然在`JRebel`远程热部署选项中都勾选了，但是它们的代码更改并不会在服务端生效。

这个如何解决呢？很简单，在`api`项目下的`rebel-remote.xml`文件中将其余两个模块添加进去，默认的如下：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<rebel-remote xmlns="http://www.zeroturnaround.com/rebel/remote">
    <id>xx.xx.xx.api</id>
</rebel-remote>
```

添加之后的代码如下：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<rebel-remote xmlns="http://www.zeroturnaround.com/rebel/remote">
    <id>xx.xxx.xx.api</id>
    <id>xx.xx.xx.service</id>
    <id>xx.xx.xx.core</id>
</rebel-remote>
```

> 以上的`<id>`标签中指定的是模块的包名（package）。


## 总结

作为热部署界的前辈，`JRebel`依然是敌得过后浪，果然是姜还是老的辣......

希望这篇文章介绍的`JRebel`能够提高读者们的开发效率，反正我是提高了，哈哈~































