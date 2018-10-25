---
layout:     post
title:      Spring cloud(4)-熔断(Hystrix)
subtitle:   spring cloud学习
date:       2018-06-18
author:     ohcomeyes
header-img: img/post-bg-android.jpg
catalog: true
tags:
    - Java
    - Spring cloud
    - 微服务
    - Hystrix
---

### Spring Cloud Hystrix（熔断）
由于网络原因或者自身的原因，`服务并不能保证100%可用`，如果`单个服务出现问题`，调用这个服务就会`出现线程阻塞`，此时若有大量的请求涌入，Servlet容器的线程资源会被消耗完毕，导致服务瘫痪。`服务与服务之间的依赖性`，故障会传播，会对整个微服务系统造成灾难性的严重后果，这就是服务故障的`“雪崩”效应`。  
**雪崩应对策略：**
* **流量控制**：控制的方式有很多种，类似队列,令牌,漏桶等等。
* **网关限流**: 因为`Nginx`的高性能, 目前一线互联网公司大量采用`Nginx+Lua`的网关进行流量控制, 由此而来的`OpenResty`也越来越热门.
使用`OpenResty`，其是由`Nginx核心`加很多第三方模块组成，其最大的亮点是默认集成了`Lua开发环境`，使得`Nginx`可以作为一个`Web Server`使用。
借助于`Nginx`的`事件驱动模型`和`非阻塞IO`，可以实现高性能的Web应用程序。  
而且`OpenResty`提供了大量组件如`Mysql、Redis、Memcached`等等，使在`Nginx`上开发`Web应用`更方便更简单。目前在京东如实时价格、秒杀、动态服务、单品页、列表页等都在使用`Nginx+Lua`架构，其他公司如淘宝、去哪儿网等。
* **用户交互限流**：友好的提示，从源端限制流量流入。

**基于Netflix的开源框架 Hystrix实现的**
框架目标在于通过控制那些访问远程系统、服务和第三方库的节点，从而对延迟和故障提供更强大的容错能力。**`Hystrix具备了服务降级、服务熔断、线程隔离、请求缓存、请求合并以及服务监控等强大功能。`**
![image](http://upload-images.jianshu.io/upload_images/14603910-adeb652597111d5e?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
**添加依赖**
```xml
<!-- hystrix 断路器 -->
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-hystrix</artifactId>
</dependency>
```
**restTempate集成hystrix**
```java
@EnableHystrix
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
`Feign是自带断路器的`，如果在`Dalston版本`的，默认是没有打开的，通过配置打开
```sh
feign:
  hystrix:
    enabled: true
```
**`@HystrixCommand 表明该方法为hystrix包裹，可以对依赖服务进行隔离、降级、快速失败、快速重试等等`**
* `fallbackMethod `降级方法
* `commandProperties` 普通配置属性，可以配置HystrixCommand对应属性，例如采用线程池还是信号量隔离、熔断器熔断规则等等
* `ignoreExceptions` 忽略的异常，默认HystrixBadRequestException不计入失败
* `groupKey()` 组名称，默认使用类名称
* `commandKey` 命令名称，默认使用方法名

#### 消费者提供方法（defaultStores）
**restTemplate配置使用**
```java
@RestController
public class ConsumerController {

    @Autowired
    private RestTemplate restTemplate;

    @HystrixCommand(fallbackMethod = "defaultStores")
    @GetMapping(value = "/hello")
    public String hello() {
        return restTemplate.getForEntity("http://eureka-provider/", String.class).getBody();
    }

    public String defaultStores() {
        return "Ribbon + hystrix ,提供者服务挂了";
    }

}
```
**feign配置使用**
```java
@EnableHystrix
@EnableFeignClients
@EnableDiscoveryClient
@SpringBootApplication
public class FeignConsumerApplication {

	public static void main(String[] args) {
		SpringApplication.run(FeignConsumerApplication.class, args);
	}
}
```
```java
@FeignClient(value ="eureka-provider",fallbackFactory = HystrixClientFallbackFactory.class)
public interface  HomeClient {

    @GetMapping("/")
    String consumer();
}
```
```java
@Component
public class HystrixClientFallbackFactory implements FallbackFactory<HomeClient> {

    @Override
    public HomeClient create(Throwable throwable) {
        return () -> "feign + hystrix ,提供者服务挂了";
    }
}
```
**`Hystrix Dashboard`是作为断路器状态的一个组件，提供了数据监控和友好的图形化界面。**
```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-hystrix</artifactId>
</dependency>

<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-actuator</artifactId>
</dependency>

<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-hystrix-dashboard</artifactId>
</dependency>
```
**在程序的入口加上`@EnableHystrixDashboard`注解，开启`HystrixDashboard`**

### 结语
在`github`上有关于`Spring Cloud`完整的部署。  
最后，[给个 star 吧](https://github.com/ohcomeyes/spring-cloud)~  
[个人博客](https://ohcomeyes.github.io)~  
[简书](https://www.jianshu.com/u/299dd40d2451)~  
