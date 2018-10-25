---
layout:     post
title:      Spring cloud(5)-路由网关(Zuul)
subtitle:   spring cloud学习
date:       2018-06-19
author:     ohcomeyes
header-img: img/post-bg-desk.jpg
catalog: true
tags:
    - Java
    - Spring cloud
    - 微服务
    - Zuul
---

### Spring Cloud Zuul（路由网关）
基于`Netflix`的开源框架` zuul`实现的
各个微服务之间都不存在单点，并且都注册于 Eureka ，基于此进行服务的注册于发现，再通过 Ribbon 进行服务调用，并具有客户端负载功能。
**问题点？**
* 将我们具体的微服务地址加端口暴露出去？
* 如果系统庞大，服务拆分的足够多那又有谁来维护这些路由关系呢？
* 服务调用之间的一些鉴权、签名校验怎么做？
* 由于服务端地址较多，客户端请求维护难度？  

**针对上述问题：SpringCloud 全家桶自然也有对应的解决方案: `Zuul`。**
Spring Cloud Zuul 将自身注册为 Eureka 服务治理下的应用，从 Eureka 中获取服务实例信息，从而维护路由规则和服务实例。 

#### API服务网关(API Gateway)服务  
我们在所有的请求进来之前抽出一层网关应用，将服务提供的所有细节都进行了包装，这样所有的客户端都是和网关进行交互`(统一入口)`，简化了客户端开发
* 将细粒度的服务组合起来提供一个粗粒度的服务
* 所有请求都导入一个统一的入口，那么整个服务只需要暴露一个api
* 对外屏蔽了服务端的实现细节，也减少了客户端与服务器的网络调用次数
![zuul](http://upload-images.jianshu.io/upload_images/14603910-84a2139ebd78e68a?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
**通过Zuul我们可以完成以下功能**：
* `客户端负载`：Zuul 注册于 Eureka 并集成了 Ribbon 所以自然也是可以从注册中心获取到服务列表进行客户端负载。
* `动态路由`：解放运维。
* `监控与审查`
* `身份认证与安全`
* `压力测试`: 逐渐增加某一个服务集群的流量，以了解服务性能;
* `金丝雀测试`
* `服务迁移`
* `负载剪裁`: 为每一个负载类型分配对应的容量，对超过限定值的请求弃用;
* `静态应答处理`

**添加依赖**
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zuul</artifactId>
</dependency>
```
**`spring-cloud-starter-zuul本身已经集成了hystrix和ribbon`**
**注意点**：
* `（传统路由）`当使用path与url的映射关系来配置路由规则时，对于路由转发的请求则不会采用HystrixCommand来包装，所以这类路由请求就没有线程隔离和断路器保护功能，并且也不会有负载均衡的能力
* `（服务路由）`使用Zuul的时候尽量使用**`path和serviceId`**的组合进行配置，这样不仅可以保证API网关的健壮和稳定，也能用到Ribbon的客户端负载均衡功能。

**开启服务注册**
```java
@EnableZuulProxy
@SpringBootApplication
public class ZuulApplication {

    public static void main(String[] args) {
        SpringApplication.run(ZuulApplication.class, args);
    }
}
```

**路由配置顺序注意**：
* 按照配置的顺序进行路由规则控制，则需要使用`YAML(文件格式，类似propeties) ` 
* 如果是使用`propeties`文件，则会丢失顺序。
```properties
# 项目配置
spring.application.name=sbc-gateway-zuul
server.port=8383
# 简化路由配置[zuul.routes.微服务Id = 指定路径]
zuul.routes.user = /api/**(指定正则表达式来匹配路径)

# 忽略指定微服务
zuul.ignored-services=微服务Id1,微服务Id2...

# 指定多个微服务负载
zuul.routes.user.path: /user/**
zuul.routes.user.serviceId: user
ribbon.eureka.enabled=false
user.ribbon.listOfServers: 微服务1, 微服务2...

# forward跳转本地
zuul.routes.user.path=/user/**
zuul.routes.user.url=forward:/user

# 对指定路由开启自定义敏感头
zuul.routes.[route].customSensitiveHeaders=true 
zuul.routes.[route].sensitiveHeaders=[这里设置要过滤的敏感头]
# 全局
zuul.sensitiveHeaders=[这里设置要过滤的敏感头]

# --Zuul的Http客户端支持Apache Http、Ribbon的RestClient和OkHttpClient，默认使用Apache HTTP客户端。--
# 启用Ribbon的RestClient
ribbon.restclient.enabled=true

# 启用OkHttpClient
ribbon.okhttp.enabled=true

# eureka地址
eureka.client.service-url.defaultZone=http://localhost:8761/eureka/
```
#### Zuul容错与回退
* Zuul的Hystrix监控的`粒度`是`微服务`，而不是某个`API`
* Zuul提供了一个`ZuulFallbackProvider`接口（最新版本建议直接继承类`FallbackProvider`,`新版增加异常处理`），实现该接口就可以为Zuul实现回退功能
```java
/**
 * @program: spring-cloud
 * @description: zuul容错与回退
 * @author: Mr.Tang
 * @create: 2018-06-20 14:25
 **/
@Component
public class ProductServiceFallbackProvider implements FallbackProvider {
    protected final Logger logger = LoggerFactory.getLogger(ProductServiceFallbackProvider.class);
    @Override
    public String getRoute() {
        // 注意: 这里是route的名称，不是服务的名称，否则无法起到回退作用 
        return "ribbon-consumer";
    }

    @Override
    public ClientHttpResponse fallbackResponse() {
        return new ClientHttpResponse() {
            @Override
            public HttpHeaders getHeaders() {
                HttpHeaders headers = new HttpHeaders(); headers.setContentType(MediaType.APPLICATION_JSON_UTF8);
                return headers;
            }

            @Override
            public InputStream getBody() throws IOException {
                logger.info("ribbon-consumer: 服务不可以");
                return new ByteArrayInputStream("该服务暂不可用，请稍后重试!".getBytes());
            }

            @Override
            public HttpStatus getStatusCode() throws IOException {
                return HttpStatus.OK;
            }

            @Override
            public int getRawStatusCode() throws IOException {
                return 200;
            }

            @Override
            public String getStatusText() throws IOException {
                return "OK";
            }

            @Override
            public void close() {

            }
        };
    }
    @Override
    public ClientHttpResponse fallbackResponse(Throwable cause) {
        if (cause != null && cause.getCause() != null) {
            String reason = cause.getCause().getMessage();
            logger.info("Excption {}",reason);
        }
        return fallbackResponse();
    }
}
```
**路由重试(或者启动备用服务来分散压力)**
网络等原因导致服务不可用，需要进行重试，zuul结合`Spring Retry`
**添加Spring Retry依赖**
```xml
<dependency>
	<groupId>org.springframework.retry</groupId>
	<artifactId>spring-retry</artifactId>
</dependency>
```
**开启重试**
不使用重试，就必须考虑到是否能够接受单个服务实例关闭和eureka刷新服务列表之间带来的短时间的熔断(在未刷新之前还是会访问到熔断中的服务降级方法)
```properties
#是否开启重试功能
zuul.retryable=true
#对当前服务的重试次数
ribbon.MaxAutoRetries=2
#切换相同Server的次数
ribbon.MaxAutoRetriesNextServer=0
```
**注意**
断路器的其中一个作用就是防止故障或者压力扩散。当压力过大是，一个服务的宕机，路由将压力转向其它服务时有可能也会压垮其它服务。断路器的形式更像是提供一种友好的错误信息，提高服务之间的容错性。

#### Zuul过滤器
Zuul大部分功能都是通过过滤器来实现的。Zuul中定义了四种标准过滤器类型
* **`PRE`：** 路由之前。我们可利用这种过滤器实现身份验证、在集群中选择请求的微服务、记录调试信息等。
* **`ROUTING`：** 路由之时。这种过滤器用于构建发送给微服务的请求，并使用Apache HttpClient或Netfilx Ribbon请求微服务。
* **`POST`：** 路由之后。这种过滤器可用来为响应添加标准的HTTP Header、收集统计信息和指标、将响应从微服务发送给客户端等。
* **`ERROR`：** 在其他阶段发生错误时执行该过滤器。 

**Zuul中默认实现的Filter**
| 类型 |  顺序 | 过滤器 |  功能 |
| ------------- | :-----------: | :---------: | :---------: |  
| pre |-3 |ServletDetectionFilter|标记处理Servlet的类型|
| pre | -2 | Servlet30WrapperFilter | 包装HttpServletRequest请求 |
| pre | -1 | FormBodyWrapperFilter | 包装请求体 |
| route | 1 | DebugFilter | 标记调试标志 |
| route | 5 | PreDecorationFilter | 处理请求上下文供后续使用 |
| route | 10 | RibbonRoutingFilter | serviceId请求转发 |
| route |100| SimpleHostRoutingFilter | url请求转发 |
| route | 500 | SendForwardFilter | forward请求转发 |
| post | 0 | SendErrorFilter | 处理有错误的请求响应 |
| post | 1000 | SendResponseFilter | 处理正常的请求响应 |

除了默认的过滤器类型，Zuul还允许我们创建自定义的过滤器类型。`自定义(PRE)`如下：
**`ZuulFilter` 是Zuul中核心组件，通过继承该抽象类，覆写几个关键方法达到自定义调度请求的作用**
```java
/**
 * @program: spring-cloud
 * @description: token验证
 * @author: Mr.Tang
 * @create: 2018-06-21 10:08
 **/
public class TokenFilter extends ZuulFilter{
    protected final Logger logger = LoggerFactory.getLogger(TokenFilter.class);
    @Override
    public String filterType() {
        //pre:路由之前
        //routing:路由中
        //post:路由后
        //error:发生错误时
        return "pre";
    }

    @Override
    public int filterOrder() {
        // filter执行顺序，通过数字指定 ,优先级为0，数字越大，优先级越低
        return 0;
    }

    @Override
    public boolean shouldFilter() {
        // 是否执行该过滤器，此处为true，说明需要过滤
        return true;
    }

    @Override
    public Object run() {
        this.logger.info("This is pre-type zuul filter.");
        RequestContext requestContext = RequestContext.getCurrentContext();
        HttpServletRequest request = requestContext.getRequest();

        String token = request.getParameter("token");
        if(StringUtils.isNotBlank(token)){
            requestContext.setSendZuulResponse(true); //对请求进行路由
            requestContext.setResponseStatusCode(200);
            requestContext.set("isSuccess", true);
            return null;
        }else{
            requestContext.setSendZuulResponse(false); //不对其进行路由
            requestContext.setResponseStatusCode(400);
            requestContext.setResponseBody("token is empty");
            requestContext.set("isSuccess", false);
            return null;
        }
    }
}
```  
* `filterType()`方法是该过滤器的类型;
* `filterOrder()`方法返回的是执行顺序;
* `shouldFilter()`方法则是判断是否需要执行该过滤器;
* `run()`则是所要执行的具体过滤动作。

过滤器之间并不会直接进行通信，而是通过`RequestContext`来共享信息，`RequestContext`是线程安全的。
**开启过滤器**
在程序的启动类 添加 Bean
```java
@Bean
public TokenFilter tokenFilter() {
	return new TokenFilter();
}
```

**禁用过滤器**
格式为:`zuul.[filter-name].[filter-type].disable=true` 如：
```properties
zuul.FormBodyWrapperFilter.pre.disable=true
```
**@EnableZuulServer和@EnableZuulProxy**
* `@EnableZuulProxy`包含`@EnableZuulServer`的功能,不会自动加载任何代理过滤器。
* 当我们需要运行一个没有代理功能的Zuul服务，或者有选择的开关部分代理功能时，那么需要使用 `@EnableZuulServer `替代 `@EnableZuulProxy`。 这时候我们可以添加任何 `ZuulFilter`类型实体类都会被自动加载，
#### Zuul 高可用
Zuul 现在既然作为了对外的第一入口，那肯定不能是单节点，对于 Zuul 的高可用有以下两种方式实现。  
1. `Eureka 高可用`：部署多个 Zuul 节点，并且都注册于 Eureka(有一个严重的缺点：那就是客户端也得注册到 Eureka 上才能对 Zuul 的调用做到负载，这显然是不现实的。)
2. `基于 Nginx 高可用`：在调用 Zuul 之前使用 Nginx 之类的负载均衡工具进行负载，这样 Zuul 既能注册到 Eureka ，客户端也能实现对 Zuul 的负载
![nginx+zuul](http://upload-images.jianshu.io/upload_images/14603910-47daf2cb773d87cc?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
### 结语
在`github`上有关于`Spring Cloud`完整的部署。  
最后，[给个 star 吧](https://github.com/ohcomeyes/spring-cloud)~  
[个人博客](https://ohcomeyes.github.io)~  
[简书](https://www.jianshu.com/u/299dd40d2451)~  
