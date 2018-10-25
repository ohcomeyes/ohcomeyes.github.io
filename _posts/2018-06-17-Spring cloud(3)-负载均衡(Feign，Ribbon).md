---
layout:     post
title:      Spring cloud(3)-负载均衡(Feign，Ribbon)
subtitle:   spring cloud学习
date:       2018-06-17
author:     ohcomeyes
header-img: img/post-bg-alibaba.jpg
catalog: true
tags:
    - Java
    - Spring cloud
    - 微服务
    - Feign
    - Ribbon
---
### Spring Cloud Feigin（负载均衡）
上文提到的服务消费者采用的是`RestTemplate+ribbon`(实现负载均衡)
**目前，在`Spring cloud `中服务之间通过`restful方式`调用有两种方式**
* **restTemplate+ribbon**  
RestTemplate定义了36个与REST资源交互的方法，其中的大多数都对应于HTTP的方法。这里面只有11个独立的方法，有十个有三种重载形式，第十一个则重载了六次，一共形成了36个方法。  
1. `delete()` 在特定的URL上对资源执行HTTP DELETE操作
2. `exchange()`在URL上执行特定的HTTP方法，返回包含对象的ResponseEntity，这个对象是从响应体中映射得到的
3. `execute()` 在URL上执行特定的HTTP方法，返回一个从响应体映射得到的对象
4. `getForEntity()` 发送一个HTTPGET请求，返回的ResponseEntity包含了响应体所映射成的对象
5. `getForObject()` 发送一个HTTP GET请求，返回的请求体将映射为一个对象
6. `postForEntity()`POST数据到一个URL，返回包含一个对象的ResponseEntity，这个对象是从响应体中映射得到的
7. `postForObject()` POST 数据到一个URL，返回根据响应体匹配形成的对象
8. `headForHeaders()` 发送HTTP HEAD请求，返回包含特定资源URL的HTTP头
9. `optionsForAllow()` 发送HTTP OPTIONS请求，返回对特定URL的Allow头信息
10. `postForLocation()` POST 数据到一个URL，返回新创建资源的URL
11. `put()` PUT 资源到特定的URL
* **feign(默认集成了ribbon,hystrix(熔断-后面会讲到))**  
Feign是一个`声明式`的伪Http客户端，它使得写Http客户端变得更简单。
使用Feign，只需要创建一个接口并注解，它具有`可插拔的注解特性`，可使用Feign 注解和`JAX-RS注解`，Feign支持`可插拔`的编码器和解码器，`Feign默认集成了Ribbon`，并和`Eureka结合`，`默认实现了负载均衡`的效果。  
**关于JAX-WS与JAX-RS两者是不同风格的SOA架构。**  
1.`JAX-WS`：Java API for XML Web Services，规范是一组XML web services的JAVA API，允许开发者可以选择RPC-oriented或者message-oriented 来实现自己的web services。以动词为中心，指定的是每次执行函数(**`RPC`**)。  
2.`JAX-RS`：Java API for RESTful Web Services，以名词为中心，每次执行的时候指的是资源(**`REST`**)。

**Feign特性**
* 可插拔的注解支持(servlet3的新特性),包括Feign注解和JAX-RS注解
* 支持可插拔的HTTP编码器和解码器
* 支持Hystrix和它的Fallback(后面解释，其实就是服务挂了，新的服务请求的话会返回fallback(可自定义)，避免一直耗费servlet容器线程资源)
* 支持Ribbon的负载均衡
* 支持HTTP请求和响应的压缩

`Feign`能干`Ribbon`和`Hystrix`的事情，但是要用`Ribbon`和`Hystrix`自带的注解必须要引入相应的jar包才可以，  
Feign还提供了`HTTP请求`的模板，通过编写简单的接口和注解，就可以定义好HTTP请求的参数、格式、地址等信息

**添加依赖**
```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-feign</artifactId>
</dependency>
```
**开启Feign**
```java
@EnableFeignClients
@EnableDiscoveryClient
@SpringBootApplication
public class FeignConsumerApplication {

	public static void main(String[] args) {
		SpringApplication.run(FeignConsumerApplication.class, args);
	}
}
```
**创建接口(feign提供的http请求模板)**
通过`@FeignClient（"服务名"）`，来指定调用哪个服务
```java
@FeignClient("eureka-provider")
public interface  HomeClient {

    @GetMapping("/") //服务提供者”/“访问的方法
    String consumer();
}
```
**消费方法**
```java
//不解释了吧
@RestController
public class ConsumerController {

    @Autowired
    private HomeClient homeClient;

    @GetMapping(value = "/hello")
    public String hello() {
        return  homeClient.consumer();
    }
}
```
**消费配置**
```sh
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/

spring:
  application:
    name: feign-consumer

server:
  port: 9000
```
### 结语
在`github`上有关于`Spring Cloud`完整的部署。  
最后，[给个 star 吧](https://github.com/ohcomeyes/spring-cloud)~  
[个人博客](https://ohcomeyes.github.io)~  
[简书](https://www.jianshu.com/u/299dd40d2451)~  
