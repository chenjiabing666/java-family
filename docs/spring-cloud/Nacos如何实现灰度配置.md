# Nacos实现灰度配置
实际生产环境中难免会涉及到配置的更新，而有些配置是否可行仅仅在本地、测试环境运行是很难保证生产环境不出错，此时就需要将配置变更到生产环境中进行测试。

如何变更？直接修改使其全部生效吗？

**答案是：不行**

原因很简单：如果这个配置有问题，那么将使得整个集群服务瘫痪

此时就要采用**灰度配置**：只针对某些服务做变更，一旦这些配置没问题，将作用于所有服务，这样能够使得服务平稳的运行，不至于整个集群瘫痪。

## Nacos中如何灰度配置

在Nacos1.1.0起配置已经支持灰度配置，在配置编辑中，勾选**Beta发布**，在文本框中勾选需要下发服务的**IP**地址，多个用**英文**逗号分隔。

比如在我的[《Spring Cloud Alibaba微服务实战》](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247506948&idx=1&sn=34e282405b10d075bb3e05cfb69663c5&chksm=fcf703c9cb808adf321e465da578dc7dcf97aa90bd97639532af56dd8d619e96c34aef7c4b97&token=708074424&lang=zh_CN#rd)专栏中的**blog-article-dev.yaml**配置文件中，加入灰度发布需要的版本信息，作用的服务ip地址为127.0.0.1，如下图：

![](https://img.java-family.cn/Nacos%E7%81%B0%E5%BA%A6%E9%85%8D%E7%BD%AE/1.png)

点击**发布Beta**则会创建一个灰度配置，如下：

![](https://img.java-family.cn/Nacos%E7%81%B0%E5%BA%A6%E9%85%8D%E7%BD%AE/2.png)

可以看到出现了两个版本的配置，如下：

- **正式版**：这个是针对除了Beta版中指定的IP服务生效
- **Beta版**：灰度配置，只对特定的IP生效

底部有两个按钮，功能如下：

- **停止Beta**：直接删除灰度配置
- **发布**：将灰度配置发布到正式版，将会覆盖掉正式的配置

如果经过线上的测试，证明你的灰度配置没问题，则直接点击发布，将会覆盖掉正式配置，一键生效将作用于整个集群。



## 总结

灰度配置在实际的生产环境中是非常重要的，使得你的服务能够平稳的运行。

和灰度配置同样重要的是**灰度发布**，如何能够实现你的整个**微服务全链路灰度发布**，这个将放在陈某的[实战专栏]((https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247506948&idx=1&sn=34e282405b10d075bb3e05cfb69663c5&chksm=fcf703c9cb808adf321e465da578dc7dcf97aa90bd97639532af56dd8d619e96c34aef7c4b97&token=708074424&lang=zh_CN#rd))中介绍。



<hr>

这几天一直忙着[《Spring Cloud Alibaba微服务实战》](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247506948&idx=1&sn=34e282405b10d075bb3e05cfb69663c5&chksm=fcf703c9cb808adf321e465da578dc7dcf97aa90bd97639532af56dd8d619e96c34aef7c4b97&token=708074424&lang=zh_CN#rd)专栏视频的录制，刚把Nacos、OpenFeign**项目实战**部分录完：

![](D:\BlogImage\Spring Cloud Alibaba微服务实战\78.png)

有需要订阅的可以加陈某微信：speical_coder









