---
layout:     post
title:      Spring cloud(2)-服务发现(Eureka，Consul)
subtitle:   spring cloud学习
date:       2018-06-16
author:     ohcomeyes
header-img: img/post-bg-YesOrNo.jpg
catalog: true
tags:
    - Java
    - Spring cloud
    - 微服务
    - Eureka
    - Consul
---
在分布式系统领域有个著名的**`CAP定理`**（**`C-数据一致性`**；**`A-服务可用性`**；**`P-服务对网络分区故障的容错性`**，这三个特性在任何分布式系统中`不能同时满足，最多同时满足两个`）；
`eureka是AP`,`zookeeper是CP`。对于`服务发现`而言，`可用性`比`数据一致性`更加重要——`AP胜过CP`

|  | Consul | zookeeper | euerka | etcd | 
| ------------- | :-----------: | :---------: | :---------: | :---------: |   
|服务健康检查|服务状态，内存，硬盘等 |(弱)长连接，keepalive|可配支持| 连接心跳 |
| 多数据中心 | 支持 | — | — | — | 
| kv存储服务 | 支持 | 支持 | — | 支持 |
| 一致性 | raft | paxos | — | raft |
| CAP | **ca** | **cp** | **ap** | **cp** |
| 使用接口(多语言能力) | 支持http和dns | 客户端 | http（sidecar） | http/grpc |
| watch支持(客户端观察到服务提供者变化) |全量/支持long polling|支持| 支持long polling/大部分增量 |支持long polling|
| 自身监控 | metrics | — | metrics | metrics |
| 安全 | acl /https | acl |  — | https支持（弱） |
| spring cloud集成 | 已支持 | 已支持 | 已支持 | 已支持 |

* **`raft`**：Raft强依赖 Leader 节点的可用性来确保集群数据的一致性。
![raft](http://upload-images.jianshu.io/upload_images/14603910-251e590fe9a45725?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* **`paxos`**: 第一次由提交者Leader向所有其他服务器发出prepare消息请求准备，所有服务器中大多数如果回复诺言承诺就表示准备好了，可以接受写入；第二次提交者向所有服务器发出正式建议propose，所有服务器中大多数如果回复已经接收就表示成功了。 
* **`long polling`**：长轮询，客户端向服务器发送Ajax请求，服务器接到请求后hold住连接，直到有新消息才返回响应信息并关闭连接，客户端处理完响应信息后再向服务器发送新的请求。 
* **`metrics`**：作为一款监控指标的度量类库，它提供了很多模块可以为第三方库或者应用提供辅助统计信息， 它还可以将度量数据发送给Ganglia和Graphite以提供图形化的监控

* **`Eureka的自我保护模式`**  
`Eureka Server`在运行期间，会统计心跳失败的比例在15分钟之内是否
低于85%，如果出现低于的情况（实际在
生产环境上通常是由于网
络不稳定导致），`Eureka Server`会将当前的实例注册信息保护起来，同时提
示警告。保护模式主要用于一组客户端和`Eureka Server`之间存在网络分
区场景下的保护。一旦进入保护模式，`Eureka Server`将会尝试保护其服务注
册表中的信息，不再删除服务注册表中的数据（也就是不会注销任何微服务）。  
**所以`Eureka`的哲学是，同时保留`”好数据“`与`”坏数据“`总比丢掉任何”好数据“要更好，所以这种模式在实践中非常有效。，**

#### 为啥不使用zookeeper做发现服务呢？
1. `ZooKeeper`是分布式协调服务，它的职责是保证数据（注：配置数据，状态数据）在其管辖下的所有服务之间保持同步、一致；（强一致性）
2. 发现服务的核心应该是需要强调服务的高可用
3. `ZooKeeper`使用单一主进程`Leader`用于处理客户端所有事务请求，采用`ZAB协议`将服务器数状态以事务形式广播到所有`Follower`上；如果三台服务挂了两台怎么选出`leader`；`1 不大于 (3/2)=1的`，
4. 正确的设置与维护`ZooKeeper服务`就非常的困难
5. 集群中出现了网络分割的故障（交换机故障导致交换机底下的子网间不能互访）`ZooKeeper`会将它们都从自己管理范围中剔除出去，外界就不能访问到这些节点了，本身这些节点是`“健康”`的，能提供服务的
6. 发现服务就算是返回了`包含不实的信息的结果也比什么都不返回要好`（因为暂时的网络故障而找不到可用的服务器）

因此， `Eureka`可以很好的应对因网络故障导致部分节点失去联系的情况，而不会像`zookeeper`那样使整个注册服务瘫痪。  

### Spring Cloud Eureka（服务注册）
* **添加依赖**
```xml
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-eureka-server</artifactId>
    </dependency>
```
* **开启服务注册**

通过 `@EnableEurekaServer `注解启动一个服务注册中心提供给其他应用进行对话,这个注解需要在`springboot工程`的启动`application类`上加

```java
    package io.ymq.example.eureka.server;
    import org.springframework.boot.SpringApplication;
    import org.springframework.boot.autoconfigure.SpringBootApplication;
    import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;
    @SpringBootApplication
    @EnableEurekaServer
    public class EurekaServerApplication {
        public static void main(String[] args) {
            SpringApplication.run(EurekaServerApplication.class, args);
        }
    }
```

### Spring Cloud Service Provider（服务提供者）
* **服务提供方**
* **将自身服务注册到 Eureka 注册中心，从而使服务消费方能够找到**
* **添加依赖**
```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-eureka-server</artifactId>
</dependency>
```

* **开启服务注册**  
在应用主类中通过加上 `@EnableEurekaClient`，但只有`Eureka` 可用，你也可以使用`@EnableDiscoveryClient`。需要配置才能找到`Eureka注册中心服务器`  
`discovery service`有许多种实现（`eureka、consul、zookeeper`等）  
`@EnableDiscoveryClient`基于`spring-cloud-commons`,  
`@EnableEurekaClient`基于`spring-cloud-netflix`。  
就是如果选用的注册中心是`eureka`，那么就推荐`@EnableEurekaClient`，  
如果是其他的注册中心，那么推荐使用`@EnableDiscoveryClient`。
```java
@SpringBootApplication
@EnableEurekaClient
@RestController
public class EurekaProviderApplication {

    @RequestMapping("/")
    public String home() {
        return "Hello world";
    }

	public static void main(String[] args) {
		SpringApplication.run(EurekaProviderApplication.class, args);
	}
}
```

* **添加配置 application.yml**  
添加配置找到Eureka服务器 
```sh
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
spring:
  application:
    name: eureka-provider
server:
  port: 8081
```

### Spring Cloud Service Consumer（服务消费者）
向`Eureka`注册服务，消费者使用`Ribbon`开启负载均衡

#### Ribbon是什么？
`Ribbon`是`Netflix`发布的开源项目，主要功能是提供客户端的软件负载均衡算法，将`Netflix`的中间层服务连接在一起。`Ribbon`客户端组件提供一系列完善的配置项如连接超时，重试等。简单的说，就是在配置文件中列出`Load Balancer（简称LB）`后面所有的机器，`Ribbon`会自动的帮助你基于某种规则`（如简单轮询，随即连接等）`去连接这些机器。我们也很容易使用`Ribbon`实现自定义的负载均衡算法。

#### Ribbon的核心组件
`Ribbon`在工作时首选会通过`ServerList`来获取所有可用的服务列表，然后通过`ServerListFilter`过虑掉一部分地址，最后在剩下的地址中通过`IRule`选择出一台服务器作为最终结果。
* **`ServerList`**：用于获取地址列表。它既可以是静态的(提供一组固定的地址)，也可以是动态的(从注册中心中定期查询地址列表)。
* **`ServerListFilter`**：仅当使用动态`ServerList`时使用，用于在原始的服务列表中使用一定策略过虑掉一部分地址。
* **`IRule`**：选择一个最终的服务地址作为`LB结果`。选择策略有`轮询、根据响应时间加权、断路器(当Hystrix可用时)`等。

#### Ribbon提供的主要负载均衡策略
* **`简单轮询负载均衡（RoundRobin）`：** 以轮询的方式依次将请求调度不同的服务器，即每次调度执行i = (i + 1) mod n，并选出第i台服务器。
* **`随机负载均衡 （Random）`：** 随机选择状态为UP的Server
* **`加权响应时间负载均衡 （WeightedResponseTime）`：** 根据相应时间分配一个weight，相应时间越长，weight越小，被选中的可能性越低。
* **`区域感知轮询负载均衡（ZoneAvoidanceRule）`** ：复合判断server所在区域的性能和server的可用性选择server

#### 服务提供者（提供服务）
* **开启服务注册**
注册三台
```java
@SpringBootApplication
@EnableEurekaClient
@RestController
public class EurekaProviderApplication {
    @Value("${server.port}")
    String port;
    @RequestMapping("/")
    public String home() {
        return "Hello world ,port:" + port;
    }
    public static void main(String[] args) {
        SpringApplication.run(EurekaProviderApplication.class, args);
    }
}
```

* **添加配置 application.yml**  
端口依次为8081,8082,8083
```sh
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/

spring:
  application:
    name: eureka-provider

server:
  port: 8081
```
#### 服务消费者（依赖于其它服务）
* **pom.xml添加依赖**
```xml
<!-- 客户端负载均衡 -->
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-ribbon</artifactId>
</dependency>

<!-- eureka客户端 -->
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>
```
* **开启服务负载均衡**  
通过@EnableDiscoveryClient向服务注册中心注册；并且向程序的ioc注入一个bean: restTemplate并通过@LoadBalanced注解表明这个restRemplate开启负载均衡的功能
```java
@EnableDiscoveryClient
@SpringBootApplication
public class RibbonConsumerApplication {
    @LoadBalanced
    @Bean
    RestTemplate restTemplate() {
        return new RestTemplate();
    }
	public static void main(String[] args) {
		SpringApplication.run(RibbonConsumerApplication.class, args);
	}
}
```
* **消费提供者方法**
```java
/**
 * 描述:调用提供者的 `home` 方法
 **/
@RestController
public class ConsumerController {

    @Autowired
    private RestTemplate restTemplate;

    @GetMapping(value = "/hello")
    public String hello() {
        return restTemplate.getForEntity("http://eureka-provider/", String.class).getBody();
    }
}
```
* **添加配置 application.yml**  
指定服务的注册中心地址，配置自己的服务端口，服务名称
```sh
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/

spring:
  application:
    name: ribbon-consumer

server:
  port: 9000
```
**依次启动服务`(Eureka服务，三台service provider和service Consumer)`，查看`ribbon`是否开启负载均衡**

### Spring Cloud  Consul（针对Consul的服务治理实现）
由于`Consul`自身提供了服务端，所以我们不需要像之前`实现Eureka`的时候创建服务注册中心，直接通过[下载consul的服务端程序](https://www.consul.io)就可以使用。
`Consul`内置了`服务注册`与`发现框架`（一站式）、具有以下性质(参考上面列表)：
* 分布一致性协议实现
* 健康检查
* Key/Value存储
* 多数据中心方案

#### Consul的优势
* 使用 Raft 算法来保证一致性, 比复杂的 Paxos 算法更直接
* 支持多数据中心，内外网的服务采用不同的端口进行监听
* 多数据中心集群可以避免单数据中心的单点故障,而其部署则需要考虑网络延迟, 分片等情况等
* 支持健康检查
* 支持 http 和 dns 协议接口
* 官方提供web管理界面

#### Consul的角色
* **`client`**：客户端, 无状态, 将 HTTP 和 DNS 接口请求转发给局域网内的服务端集群，所有注册到当前节点的服务会被转发到server，本身是不持久化这些信息
* **`server`**：服务端, 保存配置信息, 高可用集群, 在局域网内与本地客户端通讯,功能和client都一样，唯一不同的是，它会把所有的信息持久化的本地，这样遇到故障，信息是可以被保留的。 通过广域网与其他数据中心通讯. 每个数据中心的 server 数量推荐为 3 个或是 5 个.
* **`server-leader`**：表明这个server是它们的老大，它和其它server不一样的一点是，它需要负责同步注册的信息给其它的server，同时也要负责各个节点的健康监测。
* **`raft`**：server节点之间的数据一致性保证，一致性协议使用的是raft，而zookeeper用的paxos，etcd采用的也是taft。
* **`服务发现协议`**：consul采用http和dns协议，etcd只支持http
* **`服务注册`**：支持两种方式实现服务注册，consul官方建议使用第二种方式。  
1. 一种是通过consul的服务注册http API，由服务自己调用API实现注册，  
2. 另一种方式是通过json个是的配置文件实现注册，将需要注册的服务以json格式的配置文件给出。
* **`服务发现`**：支持两种方式实现服务发现，  
1. 一种是通过http API来查询有哪些服务，
2. 另外一种是通过consul agent 自带的DNS（8600端口），域名是以NAME.service.consul的形式给出，NAME即在定义的服务配置文件中，服务的名称。DNS方式可以通过check的方式检查服务。
* 服务间的通信协议：Consul使用gossip协议管理成员关系、广播消息到整个集群

### 结语
关于`consul`的环境搭建以及应用后续再补充吧~  
在`github`上有关于`Spring Cloud`完整的部署。 
最后，[给个 star 吧](https://github.com/ohcomeyes/spring-cloud)~  
[个人博客](https://ohcomeyes.github.io)~  
[简书](https://www.jianshu.com/u/299dd40d2451)~
