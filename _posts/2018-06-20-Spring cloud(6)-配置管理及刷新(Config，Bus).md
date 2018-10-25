---
layout:     post
title:      Spring cloud(6)-配置管理及刷新(Config，Bus)
subtitle:   spring cloud学习
date:       2018-06-20
author:     ohcomeyes
header-img: img/post-bg-digital-native.jpg
catalog: true
tags:
    - Java
    - Spring cloud
    - 微服务
    - Config
    - Bus
---
### Spring Cloud Config（配置管理）
分布式系统中，由于服务数量巨多，为了方便服务配置文件统一管理，实时更新（有某些配置信息变化有一定频率和规律，并且希望能够做到尽量实时（营销类，或活动类应用系统）），所以需要分布式配置中心组件。
**分布式配置统一解决方案主要目标**
1. **`部署极其简单`：** 同一个上线包，无须改动配置，即可在多个环境中(RD/QA/PRODUCTION) 上线
2. **`部署动态化`：** 更改配置，无需重新打包或重启，即可实时生效
3. **`统一管理`：** 提供web平台，统一管理 多个环境(RD/QA/PRODUCTION)、多个产品 的所有配置
4. **`支持微服务架构`**

目前流行的配置管理平台有`360的QConf`，`百度的disconf` ，`淘宝的diamond`等等，相比较同类产品： 
![服务配置比较](http://upload-images.jianshu.io/upload_images/14603910-740078f07b738faa?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`Spring Cloud Config`最大的优势是Spring无缝集成，      Spring应用程序的迁移成本非常低，在配置获取的接口上是完全一致，  结合SpringBoot可使你的项目有更加统一的标准（包括依赖版本和约束规范），避免了应为集成不同开软件源造成的依赖版本冲突。  
支持任何语言的任何应用。它也能为你支持对应用开发环境、测试环境、生产环境的配置、切换、迁移。默认的配置实现通过git实现，同时非常容易的支持其他的扩展
![消息总线](http://upload-images.jianshu.io/upload_images/14603910-2bb814cf7e7486c1?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`Spring Cloud Config`原本放在`本地`文件的配置抽取出来放在`中心服务器`，从而能够提供更好的管理、发布能力。
* **`服务端`：** 负责将git svn中存储的配置文件发布成REST接口
* **`客户端`：** 可以从服务端REST接口获取配置，客户端并不能主动感知到配置的变化，从而主动去获取新的配置,由消息总线来通知各个Spring Cloud Config的客户端去服务端更新配置。

####服务端配置
**添加依赖**
```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-config-server</artifactId>
</dependency>
<!-- 可选，安全认证 -->
<dependency> 
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```
**开启服务注册(`@EnableConfigServer `开启)**
```java
@EnableConfigServer
@SpringBootApplication
public class ConfigServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```
**增加配置**
```yml
spring:
  application:
    name: config-server
  cloud:
    config:
      label: master
      server:
        git:
          uri: https://github.com/ohcomeyes/spring-cloud.git #注意use https not use ssh
          force-pull: true
          search-paths: config
      # username: ********
      # password: ********
server:
  port: 8888 # eureka中默认读取的端口是8888
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
 
# 需要添加上面所说的安全认证依赖 ConfigSever 默认实现的基本的 HttpBasic 的认证。
security:
  user: # user 就是用户名
    password: 123
```
#### eureka中需要管理配置的服务配置
**添加依赖**
```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-config-client</artifactId>
</dependency>
```
**开启服务**
```java
@EnableEurekaClient
@RestController
@SpringBootApplication
public class EurekaProviderApplication {

    @Value("${server.port}")
    String port;
    // 通过 @Value 获取服务端的 content 值的内容
    @Value("${version}")
    String version;


    @GetMapping("/")
    public String home() {
        return "Hello home,port:"+port+"--version:"+version;
    }

    @GetMapping("/dc")
    public String home1() {
        return "hello home1 ,port:"+port+"--version:"+version;
    }

    public static void main(String[] args) {
        SpringApplication.run(EurekaProviderApplication.class,args);
    }
}
```
**添加配置**
```yml
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
spring:
  application:
    name: eureka-provider
  cloud:
    config:
      label: master #远程仓库的分支
      profile: dev # 配置文件
      discovery:
        enabled: true # 从配置中心读取文件。
        service-id: config-server # 通过服务名称去 Eureka注册中心找服务
server:
  port: 8081
```
**`注意`**: 仓库的配置文件按照`{application}-{profile}.yml`或者`{application}-{profile}.properties`格式命名，  
`config-client`的配置文件名和配置中心的`{application}`名相匹配。  
http请求地址和资源文件映射如下:
* `/{application}/{profile}[/{label}]`
*` /{application}-{profile}.yml`
* `/{label}/{application}-{profile}.yml`
* `/{application}-{profile}.properties`
* `/{label}/{application}-{profile}.properties`

启动所有服务后，就能发现所有服务能够从配置中心获取到配置信息了

#### 配置刷新（单个刷新/局部刷新）
上面讲的是服务统一从配置中心获取配置信息，下面讲讲怎么来刷新
**Spring Cloud Config 中使用 `Refresh`**
要在微服务运行期间动态刷新配置，可以通过调用/refresh实现，但这样只能针对单个服务，而且要手动操作
**添加依赖**
```xml
<!-- actuator 健康监控 -->
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```
**添加配置**
```yml
# 说明：
#   1.要通过actuator暴露端点，必须同时是启用（enabled）和暴露（exposed）的
#   2.所有除了/health和/info的端点，默认都是不暴露的
#   3.所有除了/shutdown的端点，默认都是启用的
# PS.生产环境由于安全性的问题，注意不要暴露敏感端点
# springboot 1.5.X 以上默认开通了安全认证,下面配置关闭
management: 
  security:  
    enabled: false // 指定访问信息不进行用户验证  
```
**开启服务**
```java
//通过 @RefreshScope 开启 SpringCloudConfig 客户端的 refresh 刷新范围，
//@RefreshScope要加在声明@Controller声明的类上，否则refresh之后Conroller拿不到最新的值，会默认调用缓存。
@RefreshScope 
@EnableEurekaClient
@RestController
@SpringBootApplication
public class EurekaProviderApplication {

    @Value("${server.port}")
    String port;
    // 通过 @Value 获取服务端的 content 值的内容
    @Value("${version}")
    String version;


    @GetMapping("/")
    public String home() {
        return "Hello home,port:"+port+"--version:"+version;
    }

    @GetMapping("/dc")
    public String home1() {
        return "hello home1 ,port:"+port+"--version:"+version;
    }

    public static void main(String[] args) {
        SpringApplication.run(EurekaProviderApplication.class,args);
    }
}
```
#### Spring Cloud Bus（消息总线）自动刷新
上面讲到的是手动刷新，还只能针对单个服务，下面讲讲怎么通过消息总线来广播式自动刷新，通过`RabbitMQ`或`kafka` 加 `Git`的`Webhooks`來触发配置的更新
**Spring Cloud Bus**  
事件处理机制和消息中间件消息的发送和接收整合起来，主要由发送端、接收端和事件组成。针对不同的业务需求，可以设置不同的事件，发送端发送事件，接收端接受相应的事件，并进行相应的处理，用于实现微服务之间的通信。本质是利用了MQ的广播机制在分布式的系统中传播消息  。  

默认常用`rabbitmq`和`kafka`两个binder，说到rabbitmq,先了解下消息队列
**kafka和redis中的消息队列区别：**
1. redis是内存数据库，只是它的list数据类型刚好可以用作消息队列而已
2. kafka还提供了消息ACK和队列容量、消费速率等消息相关的功能，更加完善
3. redis 发布订阅除了表示不同的 topic 外，并不支持分组(之间的最大区别)，**`比如kafka中发布一个东西，多个订阅者可以分组，（同一个组里只有一个订阅者）会收到该消息，这样可以用作负载均衡。`**
4. 处理数据大小的级别不同

**消息队列(MQ)**
消息队列（Message Queue）是一种应用间的通信方式，消息发送后可以立即返回，由消息系统来确保消息的可靠传递。消息发布者只管把消息发布到 MQ 中而不用管谁来取，消息使用者只管从 MQ 中取消息而不管是谁发布的。这样发布者和使用者都不用知道对方的存在。  
其它常见场景包括最终一致性、应用解耦(订单-库存)、广播、错峰流控（秒杀等）等等

**MQ和RPC的区别**
* MQ 是生产者消费者模式，是面向数据的。
* RPC 是请求响应模式，是面向动作的。

**RabbitMQ**
RabbitMQ是实现了高级消息队列协议（AMQP）的开源消息代理软件，也称为面向消息的中间件，是一个在AMQP基础上完整的，可复用的企业消息系统。它可以用于大型软件系统各个模块之间的高效通信，支持高并发，支持可扩展。
* **`AMQP`**：Advanced Message Queue，高级消息队列协议。它是应用层协议的一个开放标准，为面向消息的中间件设计，基于此协议的客户端与消息中间件可传递消息，并不受产品、开发语言等条件的限制。  
主要特征是面向消息、队列、路由（包括点对点和发布/订阅）、可靠性、安全。  

RabbitMQ 最初起源于金融系统，用于在分布式系统中存储转发消息，在易用性、扩展性、高可用性等方面表现不俗。具体特点包括：
* 可靠性：RabbitMQ 使用一些机制来保证可靠性，如持久化、传输确认、发布确认。
* `灵活的路由`
* `消息集群`
* `高可用`
* `多种协议`
* `多语言客户端`
* `管理界面`
* `跟踪机制`
* `插件机制`

**webhooks**
* **`webhook`**：是个在特定情况下触发的一种api. 越来越多在web上的操作被描述为事件
* 事件是webhook的核心,当仓库发生特定action会触发webhook,

向服务实例请求Spring Cloud Bus的`/bus/refresh`接口，从而触发总线上其他服务实例的`/refresh`，      `/bus/refresh`接口还提供了`destination`参数，用来定位具体要刷新的应用程序
![spring cloud bus](http://upload-images.jianshu.io/upload_images/14603910-4cdf29958002c2bd?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
1. 提交代码触发post请求给bus/refresh
2. server端接收到请求并发送给Spring Cloud Bus
3. Spring Cloud bus接到消息并通知给其它客户端
4. 其它客户端接收到通知，请求Server端获取最新配置
5. 全部客户端均获取到最新的配置

### 结语
在`github`上有关于`Spring Cloud`完整的部署。  
最后，[给个 star 吧](https://github.com/ohcomeyes/spring-cloud)~  
[个人博客](https://ohcomeyes.github.io)~  
[简书](https://www.jianshu.com/u/299dd40d2451)~  
