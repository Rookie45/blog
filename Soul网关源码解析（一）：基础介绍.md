## Soul网关源码解析（一）

[toc]

### 什么是网关

wiki上定义：

> 在计算机网络中，网关是转发其他服务器通信数据的服务器，接收从客户端发送来的请求时，它就像自己拥有资源的源服务器一样对请求进行处理。有时客户端可能都不会察觉，自己的通信目标是一个网关。 区别于路由器，经常在家庭中或者小型企业网络中使用，用于连接局域网和互联网。 网关也经常指把一种协议转成另一种协议的设备，比如语音网关。

笔者理解，网关类似古代城池的关口，任何想进入城池的人必须从关口入，城池内的人出去也必须从关口出。那么一个关口常见的功能有啥呢？

1. 要求见城池内某位大人，关口士兵拿着你的信物找到这位大人，并转述内容，这就是路由
2. 城墙很高，就别翻了，这就是防火墙
3. 没有通信证的人，休想进入城池，这就是鉴权
4. 城内人流拥挤，关口只开放一半，这就是限流
5. 城内拥塞不堪，已无能力容纳，关门吧，这就是熔断

### Soul网关

它一个异步的,高性能的,跨语言的,响应式的API网关，参考了Kong，Spring-Cloud-Gateway等优秀的网关后，站在巨人的肩膀上，Soul由此诞生！

#### 设计理念

Soul作者希望能够有一样东西像灵魂一样，保护您的微服务。

#### 支持的特性

- 支持各种语言(http协议)，支持 dubbo，springcloud协议。
- 插件化设计思想，插件热插拔,易扩展。
- 灵活的流量筛选，能满足各种流量控制。
- 内置丰富的插件支持，鉴权，限流，熔断，防火墙等等。
- 流量配置动态化，性能极高，网关消耗在 1~2ms。
- 支持集群部署，支持 A/B Test, 蓝绿发布。

#### 总体架构

![avatar](../pic/arch1.PNG)

### 环境准备

- JDK8+
- maven 3.2+
- Soul版本[2.2.1](https://github.com/dromara/soul/archive/2.2.1.zip)
- Mysql 5.x

> 笔者的maven版本是3.3.9，mysql版本是5.7，仅供参考

### 编译运行

1. 执行如下命令进行编译

```java
mvn clean install -Dmaven.test.skip=true -Dmaven.javadoc.skip=true -Drat.skip=true -Dcheckstyle.skip=true
```

2. 在mysql中执行路径`\soul-2.2.1\script`下soul.sql文件，执行完会看到下面的数据库表

   ![avatar](../pic/db1.PNG)

3. 修改路径为`soul-2.2.1\soul-admin\src\main\resources`下的配置文件application-local.yml，将其中的mysql配置修改为自己环境中搭建的mysql配置，如笔者搭建的mysql配置如下：

```
...
  datasource:
    url: jdbc:mysql://localhost:3306/soul?useUnicode=true&characterEncoding=utf-8
    username: root
    password: 123456
    driver-class-name: com.mysql.jdbc.Driver
...
```

4. 运行

   使用IDEA加载soul工程，加载完成后，可以看到IDEA识别出来SoulAdminBootstrap主程序，点击运行

   ![avatar](../pic/adminbootstrap.PNG)

5. 访问运行结果

   浏览器访问http://127.0.0.1:9095/，可以看到如下页面

   ![avatar](../pic/adminbootstrap.PNG)

6. 登录soul admin系统，默认用户名/密码为：admin/123456，可以看到如下页面

   ![avatar](../pic/adminbootstrap2.PNG)

   这里主体菜单有两类，一类是soul自带集成的插件列表，其中就包括springcloud和dubbo，另外waf防火墙，sign签名鉴权，monitor监控，rewrite转发，rate_limiter限流，divide分组以及hystrix熔断；另一类是系统管理，包括用户管理，插件的开启关闭，认证管理以及元数据的信息。
   
7. 找到'..\soul-2.2.1\soul-bootstrap\target'目录，再此文件夹打开命令提示符窗口，执行如下命令启动soul-bootstrap程序

```
java  -jar .\soul-bootstrap.jar  --soul.sync.websocket.url="ws://localhost:9095/websock"
...
2021-01-15 00:46:33.367  INFO 10460 --- [           main] b.s.s.d.w.WebsocketSyncDataConfiguration : you use websocket sync soul data.......
2021-01-15 00:46:33.583  INFO 10460 --- [           main] o.d.s.p.s.d.w.WebsocketSyncDataService   : websocket connection is successful.....
2021-01-15 00:46:33.720  INFO 10460 --- [           main] o.s.b.a.e.web.EndpointLinksResolver      : Exposing 2 endpoint(s) beneath base path '/actuator'
2021-01-15 00:46:34.876  INFO 10460 --- [           main] o.s.b.web.embedded.netty.NettyWebServer  : Netty started on port(s): 9195
2021-01-15 00:46:34.878  INFO 10460 --- [           main] o.d.s.b.SoulBootstrapApplication         : Started SoulBootstrapApplication in 4.212 seconds (JVM running for 4.648)

```

### 小结

本小结以网关的定义入手，以城池关口类比网关的功能，大致包括路由，防火墙，鉴权，限流以及熔断。接着引入soul网关，包括它的设计理念，现有的特性以及总体框架，接着列举了soul网关本地搭建的环境准备，最后一步一步说明，如何在本地run起soul网关，以及run起来之后，soul admin管理系统包含的菜单介绍。希望能帮到你，初识soul这样一个极致性能的网关项目。

### 参考

[Soul 官网](https://dromara.org/zh-cn/docs/soul/soul.html)

