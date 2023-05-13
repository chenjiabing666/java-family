
数据同步一直是一个令人头疼的问题。在业务量小，场景不多，数据量不大的情况下我们可能会选择在项目中直接写一些定时任务手动处理数据，例如从多个表将数据查出来，再汇总处理，再插入到相应的地方。

但是随着业务量增大，数据量变多以及各种复杂场景下的分库分表的实现，使数据同步变得越来越困难。

今天这篇文章使用**阿里**开源的中间件**Canal**解决数据增量同步的痛点。

文章目录如下：

![](https://img.java-family.cn/Spring%20Security/204.png)

## Canal是什么？

**canal**译意为水道/管道/沟渠，主要用途是基于 **MySQL** 数据库**增量日志**解析，提供增量数据订阅和消费。

从这句话理解到了什么？

基于MySQL，并且通过MySQL日志进行的增量解析，这也就意味着对原有的业务代码完全是无侵入性的。

> 工作原理：解析MySQL的binlog日志，提供增量数据。

基于日志增量订阅和消费的业务包括

- 数据库镜像
- 数据库实时备份
- 索引构建和实时维护(拆分异构索引、倒排索引等)
- 业务 cache 刷新
- 带业务逻辑的增量数据处理

当前的 canal 支持源端 MySQL 版本包括 5.1.x , 5.5.x , 5.6.x , 5.7.x , 8.0.x。

> 官方文档：https://github.com/alibaba/canal

## Canal数据如何传输？

先来一张官方图：

![](https://img.java-family.cn/Spring%20Security/196.png)

Canal分为服务端和客户端，这也是阿里常用的套路，比如前面讲到的注册中心**Nacos**：

- **服务端**：负责解析MySQL的binlog日志，传递增量数据给客户端或者消息中间件
- **客户端**：负责解析服务端传过来的数据，然后定制自己的业务处理。

目前为止支持的消息中间件很全面了，比如**Kafka**、**RocketMQ**，**RabbitMQ**。

## 数据同步还有其他中间件吗？

有，当然有，还有一些开源的中间件也是相当不错的，比如**Bifrost**。

常见的几款中间件的区别如下：

![](https://img.java-family.cn/Spring%20Security/197.png)

当然要我选择的话，首选阿里的中间件Canal。

## Canal服务端安装

服务端需要下载压缩包，下载地址：https://github.com/alibaba/canal/releases

目前最新的是**v1.1.5**，点击下载：

![](https://img.java-family.cn/Spring%20Security/198.png)

下载完成解压，目录如下：

![](https://img.java-family.cn/Spring%20Security/199.png)

本文使用**Canal+RabbitMQ**进行数据的同步，因此下面步骤完全按照这个base进行。

### **1、打开MySQL的binlog日志**

修改MySQL的日志文件，my.cnf 配置如下：

```properties
[mysqld]
log-bin=mysql-bin # 开启 binlog
binlog-format=ROW # 选择 ROW 模式
server_id=1 # 配置 MySQL replaction 需要定义，不要和 canal 的 slaveId 重复
```

### **2、设置MySQL的配置**

需要设置服务端配置文件中的MySQL配置，这样Canal才能知道需要监听哪个库、哪个表的日志文件。

一个 Server 可以配置多个实例监听 ，Canal 功能默认自带的有个 example 实例，本篇就用 example 实例 。如果增加实例，复制 example 文件夹内容到同级目录下，然后在 `canal.properties` 指定添加实例的名称。

**修改canal.deployer-1.1.5\conf\example\instance.properties配置文件**

```properties
# url
canal.instance.master.address=127.0.0.1:3306
# username/password
canal.instance.dbUsername=root
canal.instance.dbPassword=root
# 监听的数据库
canal.instance.defaultDatabaseName=test

# 监听的表，可以指定，多个用逗号分割，这里正则是监听所有
canal.instance.filter.regex=.*\\..*

```

### **3、设置RabbitMQ的配置**

服务端默认的传输方式是**tcp**，需要在配置文件中设置**MQ**的相关信息。

这里需要修改两处配置文件，如下；

**1、canal.deployer-1.1.5\conf\canal.properties**

这个配置文件主要是设置MQ相关的配置，比如URL，用户名、密码...

```properties
# 传输方式：tcp, kafka, rocketMQ, rabbitMQ
canal.serverMode = rabbitMQ
##################################################
######### 		    RabbitMQ	     #############
##################################################
rabbitmq.host = 127.0.0.1
rabbitmq.virtual.host =/
# exchange
rabbitmq.exchange =canal.exchange
# 用户名、密码
rabbitmq.username =guest
rabbitmq.password =guest
## 是否持久化
rabbitmq.deliveryMode = 2
```

2、**canal.deployer-1.1.5\conf\example\instance.properties**

这个文件设置MQ的路由KEY，这样才能路由到指定的队列中，如下：

```properties
canal.mq.topic=canal.routing.key
```

### **4、RabbitMQ新建exchange和Queue**

在RabbitMQ中需要新建一个**canal.exchange**（必须和配置中的相同）的exchange和一个名称为 **canal.queue**（名称随意）的队列。

其中绑定的路由KEY为：**canal.routing.key**（必须和配置中的相同），如下图：

![](https://img.java-family.cn/Spring%20Security/200.png)



### **5、启动服务端**

点击bin目录下的脚本，windows直接双击**startup.bat**，启动成功如下：

![](https://img.java-family.cn/Spring%20Security/201.png)



### **6、测试**

在本地数据库**test**中的**oauth_client_details**插入一条数据，如下：

```mysql
INSERT INTO `oauth_client_details` VALUES ('myjszl', 'res1', '$2a$10$F1tQdeb0SEMdtjlO8X/0wO6Gqybu6vPC/Xg8OmP9/TL1i4beXdK9W', 'all', 'password,refresh_token,authorization_code,client_credentials,implicit', 'http://www.baidu.com', NULL, 1000, 1000, NULL, 'false');
```

此时查看MQ中的**canal.queue**已经有了数据，如下：

![](https://img.java-family.cn/Spring%20Security/202.png)

其实就是一串JSON数据，这个JSON如下：

```json
{
	"data": [{
		"client_id": "myjszl",
		"resource_ids": "res1",
		"client_secret": "$2a$10$F1tQdeb0SEMdtjlO8X/0wO6Gqybu6vPC/Xg8OmP9/TL1i4beXdK9W",
		"scope": "all",
		"authorized_grant_types": "password,refresh_token,authorization_code,client_credentials,implicit",
		"web_server_redirect_uri": "http://www.baidu.com",
		"authorities": null,
		"access_token_validity": "1000",
		"refresh_token_validity": "1000",
		"additional_information": null,
		"autoapprove": "false"
	}],
	"database": "test",
	"es": 1640337532000,
	"id": 7,
	"isDdl": false,
	"mysqlType": {
		"client_id": "varchar(48)",
		"resource_ids": "varchar(256)",
		"client_secret": "varchar(256)",
		"scope": "varchar(256)",
		"authorized_grant_types": "varchar(256)",
		"web_server_redirect_uri": "varchar(256)",
		"authorities": "varchar(256)",
		"access_token_validity": "int(11)",
		"refresh_token_validity": "int(11)",
		"additional_information": "varchar(4096)",
		"autoapprove": "varchar(256)"
	},
	"old": null,
	"pkNames": ["client_id"],
	"sql": "",
	"sqlType": {
		"client_id": 12,
		"resource_ids": 12,
		"client_secret": 12,
		"scope": 12,
		"authorized_grant_types": 12,
		"web_server_redirect_uri": 12,
		"authorities": 12,
		"access_token_validity": 4,
		"refresh_token_validity": 4,
		"additional_information": 12,
		"autoapprove": 12
	},
	"table": "oauth_client_details",
	"ts": 1640337532520,
	"type": "INSERT"
}
```

每个字段的意思已经很清楚了，有表名称、方法、参数、参数类型、参数值.....

**客户端要做的就是监听MQ获取JSON数据，然后将其解析出来，处理自己的业务逻辑。**

## Canal客户端搭建

客户端很简单实现，要做的就是消费Canal服务端传递过来的消息，监听**canal.queue**这个队列。

### **1、创建消息实体类**

MQ传递过来的是JSON数据，当然要创建个实体类接收数据，如下：

```java
/**
 * @author 公众号 码猿技术专栏
 * Canal消息接收实体类
 */
@NoArgsConstructor
@Data
public class CanalMessage<T> {
    @JsonProperty("type")
    private String type;

    @JsonProperty("table")
    private String table;

    @JsonProperty("data")
    private List<T> data;

    @JsonProperty("database")
    private String database;

    @JsonProperty("es")
    private Long es;

    @JsonProperty("id")
    private Integer id;

    @JsonProperty("isDdl")
    private Boolean isDdl;

    @JsonProperty("old")
    private List<T> old;

    @JsonProperty("pkNames")
    private List<String> pkNames;

    @JsonProperty("sql")
    private String sql;

    @JsonProperty("ts")
    private Long ts;
}
```

### **2、MQ消息监听业务**

接下来就是监听队列，一旦有Canal服务端有数据推送能够及时的消费。

代码很简单，只是给出个接收的案例，具体的业务逻辑可以根据业务实现，如下：

```java
import cn.hutool.json.JSONUtil;
import cn.myjszl.middle.ware.canal.mq.rabbit.model.CanalMessage;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.rabbit.annotation.Exchange;
import org.springframework.amqp.rabbit.annotation.Queue;
import org.springframework.amqp.rabbit.annotation.QueueBinding;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;

/**
 * 监听MQ获取Canal增量的数据消息
 */
@Component
@Slf4j
@RequiredArgsConstructor
public class CanalRabbitMQListener {

    @RabbitListener(bindings = {
            @QueueBinding(
                    value = @Queue(value = "canal.queue", durable = "true"),
                    exchange = @Exchange(value = "canal.exchange"),
                    key = "canal.routing.key"
            )
    })
    public void handleDataChange(String message) {
        //将message转换为CanalMessage
        CanalMessage canalMessage = JSONUtil.toBean(message, CanalMessage.class);
        String tableName = canalMessage.getTable();
        log.info("Canal 监听 {} 发生变化；明细：{}", tableName, message);
        //TODO 业务逻辑自己完善...............
    }
}

```

### **3、测试**

下面向表中插入数据，看下接收的消息是什么样的，SQL如下：

```sql
INSERT INTO `oauth_client_details`
VALUES
	( 'myjszl', 'res1', '$2a$10$F1tQdeb0SEMdtjlO8X/0wO6Gqybu6vPC/Xg8OmP9/TL1i4beXdK9W', 'all', 'password,refresh_token,authorization_code,client_credentials,implicit', 'http://www.baidu.com', NULL, 1000, 1000, NULL, 'false' );
```

客户端转换后的消息如下图：

![](https://img.java-family.cn/Spring%20Security/203.png)

上图可以看出所有的数据都已经成功接收到，只需要根据数据完善自己的业务逻辑即可。

> 客户端案例源码已经上传GitHub，关注公众号：**码猿技术专栏**，回复关键词：**9530** 获取！

## 总结

数据增量同步的开源工具并不只有Canal一种，根据自己的业务需要选择合适的组件。



