---
layout:     post
title:      Spring cloud(1)-简介以及选择
subtitle:   spring cloud学习
date:       2018-06-15
author:     ohcomeyes
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Java
    - Spring cloud
    - 微服务
---
### 简介
`Spring Cloud`是一个基于`Spring Boot`实现的云应用开发工具，它为基于`JVM`的云应用开发中涉及的配置管理、服务发现、断路器、智能路由、微代理、控制总线、全局锁、决策竞选、分布式会话和集群状态管理等操作提供了一种简单的开发方式。

服务化的核心就是将传统的一站式应用根据业务拆分成一个一个的服务，而微服务在这个基础上要更彻底地去耦合（不再共享`DB`、`KV`，去掉重量级`ESB`），并且强调`DevOps`和快速演化。
* `DevOps`:要求开发、测试、运维进行一体化的合作，进行更小、更频繁、更自动化的应用发布，以及围绕应用架构来构建基础设施的架构。这就要求应用充分的内聚，也方便运维和管理。

#### 为什么要使用，和之间的服务化解决方案相比的优势(举例：`Nginx`)
最初的服务化解决方案是给提供相同服务提供一个统一的域名，然后服务调用者向这个域名发送HTTP请求，由`Nginx`负责请求的分发和跳转。
![nginx](http://upload-images.jianshu.io/upload_images/14603910-292aa7225db053d0?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
* 配置文件中耦合了服务调用的逻辑，这削弱了微服务的完整性
* 服务的信息分散在各个系统，无法统一管理和维护,服务消费者并不知道有哪些实例在给他们提供服务。
* 无法直观的看到服务提供者和服务消费者当前的运行状况和通信频率。
* 消费者的失败重发，负载均衡等都没有统一策略，这加大了开发每个服务的难度，不利于快速演化。

#### spring cloud和dubbo比较

| 微服务功能 | Dubbo | Spring Cloud | 
| ------------- | :-----------: | :---------: | 
| 服务注册和发现 | Zookeeper | Eureka,Consul | 
| 服务调用方式 | RPC | RESTful API | 
| 断路器 | 有 | Hystrix |
| 负载均衡 | 有 |Ribbon,Feign(RESTful Web Service客户端，整合了Ribbon和Hystrix)|
| 服务路由和过滤 | 有 | 有 |
| 分布式配置 | 无 | 有 |
| 分布式锁 | 无 | 有 |
| 集群选主 | 无 | 有 |
| 分布式消息 | 无 | 有 |

`Spring Cloud`抛弃了`Dubbo`的`RPC`通信，采用的是基于`HTTP`的`REST`方式。严格来说，这两种方式各有优劣。虽然从一定程度上来说，后者牺牲了服务调用的性能，但也避免了上面提到的原生`RPC`带来的问题。而且`REST`相比`RPC`更为灵活，服务提供方和调用方的依赖只依靠一纸契约，不存在代码级别的强依赖，这在强调快速演化的微服务环境下，显得更加合适。
### 结语
在`github`上有关于`Spring Cloud`完整的部署。  
最后，[给个 star 吧](https://github.com/ohcomeyes/spring-cloud)~  
[个人博客](https://ohcomeyes.github.io)~  
[简书](https://www.jianshu.com/u/299dd40d2451)~  
