---
layout:     post
title:      消息中间件部署及比较(rabbitMQ、activeMQ、zeroMQ、rocketMQ、Kafka、redis)
subtitle:   中间件学习
date:       2018-06-27
author:     ohcomeyes
header-img: img/post-bg-e2e-ux.jpg
catalog: true
tags:
    - Java
    - 消息中间件
    - 消息队列
---
一发一存一消费，没有最好的消息队列中间件(简称消息中间件)，只有最合适的消息中间件。   
**消息队列常用的使用场景：**
* `非实时性`：当不需要立即获得结果，但是并发量又需要进行控制的时候，差不多就是需要使用消息队列的时候。主要解决了应用耦合、异步处理、流量削锋等问题。
* `应用耦合`：多应用间通过消息队列对同一消息进行处理，避免调用接口失败导致整个过程失败；（如：订单->库存）
* `异步处理`：多应用对消息队列中同一消息进行处理，应用间并发处理消息，相比串行处理，减少处理时间；(点对多场景，广播场景(注册发短信，发邮件)等等)
* `限流削峰`：应用于秒杀或抢购活动中，避免流量过大导致应用系统挂掉的情况；(根据服务承受度设置队列大小，超过了就返回活动结束了，咱们经常各大商城秒杀，心里还没有点B数吗)减少压力,避免服务挂掉。
* `消息驱动的系统`：系统分为消息队列、消息生产者、消息消费者，生产者负责产生消息，消费者(可能有多个)负责对消息进行处理；(分工处理(各自对应相应的队列)，灵活应用(收到就处理/定时处理))

**消息队列是异步RPC的主要手段之一**
>两种模式：
1. `点对点`：每个消息只有一个消费者（Consumer），不可重复消费(一旦被消费，消息就不再在消息队列中)
2. `发布/订阅`：微信公众号(Topic)，大伙(订阅者)订阅关注之后，微信公众号运营平台(发布者)发布信息后，大伙微信就都收到信息了，这里其实还分pull/push的。一个是主动推送，一个是被动拉取  
**`基于发布/订阅模式做扩展就是横向扩展，多个队列及消费分组订阅(提高消费能力)`**
* `pull`：主动权在于消费方，优点是按需消费(吃自助餐，能吃多少拿多少)，而且服务端队列堆积的消息处理也相对简单(不用记录状态啊，状态都消费端)；缺点就是消息延迟(不知道啥时候去拉取更新)，这时候有小伙伴会问，那为啥不叫服务端通知一下呢(有句话叫不在其位不谋其政，服务端通知必然要记录通知状态和增加之间的通信带宽；当然也可以根据实际情况来选择和push组合起来用(男女搭配干活不累嘛)来提高消息的实时性)
* `push`：主动权就在服务方了，优点是实时性高，服务端可以统一管理来进行负载，不过也容易导致慢消费(就得考虑消费方受不受得了，毕竟你说你了解，但也只有对方才清楚你有多了解)；缺点就是发送消息的状态是集中式管理，压力大啊(要分发消息还要记录状态还要做备份，又当爹来又当妈，你说累不累)  
**`对于顺序消息，这种场景有限且成本太高的方式就得慎重考虑了，对那种全局有序但允许出现小误差的场景(日志推送)，pull模式就非常适合了(所以说kafka为啥常用于日志处理、大数据等方面)，要问为什么？自己去领悟`**
>实际开发中消息中间件选型基于几个方面： 
1. **`功能`**：这个就多了，优先级队列、延迟队列(划分不同的延迟队列来避免重新排序消耗性能，缺点嘛自己悟)、死信队列(放没有推送成功的)、消费模式(pull/push)、广播消费、消息回溯(可追溯嘛，不然被卖了都不知道是谁)、消息堆积+持久化、消息追踪(链路条，方便定位)、消息过滤(根据规则过滤啊，不同类别消息发送到不同topic)、多协议支持(通用性)、跨语言支持(流行程度)、流量控制(嘿嘿嘿，上面有)、消息顺序性(还要再说一遍？)、安全机制(身份认证，权限认证(读写))、消息幂等性(承诺知道不，答应人家的事就一定要做到)、事务性消息(不想说)等
2. **`性能`**：一般是指其吞吐量(统一大小的消息体和不同大小的消息体生产和消耗能力)，性能和功能很多时候是相悖的，鱼和熊掌不可兼得。
3. **`高可靠、高可用`**：先说可靠，主要在于消息的持久化这一块(消息只要写入就一定会被消费，不会因为故障导致数据丢失(这个就很好测试出来了吧))。如果是从系统的角度来看就得从整体的维度去衡量了(不能单单只靠消息中间件本身，要从生产端、服务端、消费端三个维度去保障)。  
再说可用，主要在于一个是对外部服务的依赖性(像kafka依赖zookeeper)，依赖也分强依赖和弱依赖，一个在于本身的备份机制所带来的保障性(像主从复制这种备份啊，增加多个slave来加强保障同时也会存在资源浪费，大部分时候Slave可能是空闲的)。
4. **`运维`**：通常有审核评估啊、监控啊、报警提醒啊、容灾啊、扩容啊、升级部署等等，一方面看中间件支撑的维度，一方面就看结合自动化运维的难易度
5. **`社区力度及生态发展`**：这个好理解吧，使用开源框架最开始基本上愉快的奔跑，但时不时的总会掉坑里，能不能爬出来一方面看自身的实力，一方面就看社区的力度了
6. **`成本`：** 尽量贴合团队自身的技术栈体系，让一个C栈的团队去深挖zeroMQ总比scala编写kafka要容易的多  

**`先贴一个图(网上Q来的)，一些功能支不支持主要取决于它使用的模式，看完上面详细说明应该就比较清楚`**

![消息中间件](http://upload-images.jianshu.io/upload_images/14603910-56b41526e4948b79?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  

**`先从比较有代表性的两个MQ(rabbitMQ,kafka)，功能对比(图还是Q来的)`**

![中间件功能比较1](http://upload-images.jianshu.io/upload_images/14603910-3aa9041d55d7c4bd?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![中间件功能比较2](http://upload-images.jianshu.io/upload_images/14603910-150c3a5918b32e71?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![中间件功能比较3](http://upload-images.jianshu.io/upload_images/14603910-99b33e752cd91b19?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  

**应用方面：**  
* RabbitMQ,遵循AMQP协议，由内在高并发的erlanng语言开发，用在实时的对可靠性要求比较高的消息传递上。  
* kafka它主要用于处理活跃的流式数据,大数据量的数据处理上。  

**架构模型方面：**
* RabbitMQ遵循`AMQP协议`，RabbitMQ的broker由Exchange,Binding,queue组成，其中exchange和binding组成了消息的路由键；客户端Producer通过连接channel和server进行通信，Consumer从queue获取消息进行消费（长连接，queue有消息会推送到consumer端，consumer循环从输入流读取数据）。rabbitMQ以broker为中心；有消息的确认机制。
* kafka遵从一般的MQ结构，producer，broker，consumer，以consumer为中心，消息的消费信息保存的客户端consumer上，consumer根据消费的点，从broker上批量pull数据；无消息确认机制。

**吞吐量：**
* rabbitMQ在吞吐量方面稍逊于kafka，他们的出发点不一样，rabbitMQ支持对消息的可靠的传递，支持事务，不支持批量的操作；基于存储的可靠性的要求存储可以采用内存或者硬盘。
* kafka具有高的吞吐量，内部采用消息的批量处理，zero-copy机制，数据的存储和获取是本地磁盘顺序批量操作，具有O(1)的复杂度，消息处理的效率很高。

**可用性方面：**
* rabbitMQ支持miror(镜像)的queue，主queue失效，miror queue接管。
* kafka的broker支持主备模式。

**集群负载均衡方面：**
* rabbitMQ的负载均衡需要单独的loadbalancer进行支持。
* kafka采用zookeeper对集群中的broker、consumer进行管理，可以注册topic到zookeeper上；通过zookeeper的协调机制，producer保存对应topic的broker信息，可以随机或者轮询发送到broker上；并且producer可以基于语义指定分片，消息发送到broker的某分片上。

### rabbitMQ
**基于erlang开发**  
是采用Erlang语言实现的AMQP协议的消息中间件，最初起源于金融系统，用于在分布式系统中存储转发消息。RabbitMQ发展到今天，被越来越多的人认可，这和它在可靠性、可用性、扩展性、功能丰富等方面的卓越表现是分不开的。  
优点：
* 由于erlang语言的特性，mq性能较好，高并发；
* 健壮、稳定、易用、跨平台、支持多种语言、文档齐全；
* 有消息确认机制和持久化机制，可靠性高；
* 高度可定制的路由；
* 管理界面较丰富，在互联网公司也有较大规模的应用；
* 社区活跃度高； 

缺点：
* 尽管结合erlang语言本身的并发优势，性能较好，但是不利于做二次开发和维护；
* 实现了代理架构，意味着消息在发送到客户端之前可以在中央节点上排队。此特性使得RabbitMQ易于使用和部署，但是使得其运行速度较慢，因为中央节点增加了延迟，消息封装后也比较大；
* 需要学习比较复杂的接口和协议，学习和维护成本较高；
### activeMQ
**基于java开发**  
是Apache出品的、采用Java语言编写的完全基于JMS1.1规范的面向消息的中间件，为应用程序提供高效的、可扩展的、稳定的和安全的企业级消息通信。不过由于历史原因包袱太重，目前市场份额没有后面三种消息中间件多，其最新架构被命名为Apollo,(京东的消息中间件就是基于activeMQ开发的)  
优点：
* 跨平台(JAVA编写与平台无关有，ActiveMQ几乎可以运行在任何的JVM上)
* 可以用JDBC：可以将数据持久化到数据库
* 支持JMS ：支持JMS的统一接口;
* 支持自动重连；
* 有安全机制：支持基于shiro，jaas等多种安全配置机制，可以对Queue/Topic进行认证和授权
* 监控完善：拥有完善的监控，包括Web Console，JMX，Shell命令行，Jolokia的REST API；
* 界面友善：提供的Web Console可以满足大部分情况，还有很多第三方的组件可以使用，如hawtio；

缺点：
* 社区活跃度不及RabbitMQ高；
* 会出莫名其妙的问题，会丢失消息；
* 不适合用于上千个队列的应用场景；
### zeroMQ
**基于C开发**  
号称史上最快的消息队列，基于C语言开发。ZeroMQ是一个消息处理队列库，可在多线程、多内核和主机之间弹性伸缩，虽然大多数时候我们习惯将其归入消息队列家族之中，但是其和前面的几款有着本质的区别，ZeroMQ本身就不是一个消息队列服务器，更像是一组底层网络通讯库，对原有的Socket API上加上一层封装而已。  
优点：
* 号称最快的消息队列系统，尤其针对大吞吐量的需求场景
* 单独部署或集成到应用中使用，不需要安装和运行一个消息服务器或中间件，因为你的应用程序将扮演了这个服务角色
* 能够实现高级/复杂的队列，但是开发人员需要自己组合多种技术框架
* 跨平台，多语言支持
* 可作为Socket通信库使用

缺点：
* 仅提供非持久性的队列，也就是说如果down机，数据将会丢失

### rocketMQ
**基于java开发（阿里消息中间件）**  
是阿里开源的消息中间件，目前已经捐献个Apache基金会，它是由Java语言开发的，具备高吞吐量、高可用性、适合大规模分布式系统应用等特点，经历过双11的洗礼，实力不容小觑。  
优点：
* 单机支持 1 万以上持久化队列
* RocketMQ 的所有消息都是持久化的，先写入系统 pagecache(页高速缓冲存储器)，然后刷盘，可以保证内存与磁盘都有一份数据，访问时，直接从内存读取。
* 模型简单，接口易用（JMS 的接口很多场合并不太实用）
* 性能非常好，可以大量堆积消息在broker(集群中包含一个或多个服务器，这些服务器被称为broker)中；
* 支持多种消费，包括集群消费、广播消费等。
* 各个环节分布式扩展设计，主从HA(高可用性集群)；
* 开发度较活跃，版本更新很快。

缺点：
* 支持的客户端语言不多，目前是java及c++，其中c++不成熟；
* RocketMQ社区关注度及成熟度也不及前两者；
* 没有web管理界面，提供了一个CLI(命令行界面)管理工具带来查询、管理和诊断各种问题；
* 没有在 mq 核心中去实现JMS等接口；

### kafka
**基于Scala和Java开发**  
起初是由LinkedIn公司采用Scala语言开发的一个分布式、多分区、多副本且基于zookeeper协调的分布式消息系统，现已捐献给Apache基金会。它是一种高吞吐量的分布式发布订阅消息系统，以可水平扩展和高吞吐率而被广泛使用。目前越来越多的开源分布式处理系统如Cloudera、Apache Storm、Spark、Flink等都支持与Kafka集成。  
优点：
* 客户端语言丰富，支持java、.net、php、ruby、python、go等多种语言；
* 性能卓越，单机写入TPS约在百万条/秒，消息大小10个字节；
* 提供完全分布式架构, 并有replica机制, 拥有较高的可用性和可靠性, 理论上支持消息无限堆积；
* 支持批量操作；
* 消费者采用Pull方式获取消息, 消息有序, 通过控制能够保证所有消息被消费且仅被消费一次;
* 有优秀的第三方Kafka Web管理界面Kafka-Manager；
* 在日志领域比较成熟，被多家公司和多个开源项目使用；

缺点：
* Kafka单机超过64个队列/分区，Load会发生明显的飙高现象，队列越多，load越高，发送消息响应时间变长
* 使用短轮询方式，实时性取决于轮询间隔时间；
* 消费失败不支持重试；
* 支持消息顺序，但是一台代理宕机后，就会产生消息乱序；
* 社区更新较慢；

### redis
Redis的PUB/SUB机制，即发布-订阅模式。利用的Redis的列表(lists)数据结构。比较好的使用模式是，生产者lpush消息，消费者brpop消息，并设定超时时间，可以减少redis的压力。只有在Redis宕机且数据没有持久化的情况下丢失数据，可以根据业务通过AOF和缩短持久化间隔来保证很高的可靠性，而且也可以通过多个client来提高消费速度。但相对于专业的消息队列来说，该方案消息的状态过于简单(没有状态)，且没有ack机制，消息取出后消费失败依赖于client记录日志或者重新push到队列里面。
   
redis不支持分组(这点很重要，在做负载均衡的时候劣势就体现出来)，不过可以完全当做一个轻量级的队列使用，但redis他爹做了disque，可以去试一试。
#### 部署安装
#### rabbitMQ
几个重要概念：  
* Broker：简单来说就是消息队列服务器实体。
* Exchange：消息交换机，它指定消息按什么规则，路由到哪个队列。
* Queue：消息队列载体，每个消息都会被投入到一个或多个队列。
* Binding：绑定，它的作用就是把exchange和queue按照路由规则绑定起来。
* Routing Key：路由关键字，exchange根据这个关键字进行消息投递。
* vhost：虚拟主机，一个broker里可以开设多个vhost，用作不同用户的权限分离。
* producer：消息生产者，就是投递消息的程序。
* consumer：消息消费者，就是接受消息的程序。
* channel：消息通道，在客户端的每个连接里，可建立多个channel，每个channel代表一个会话任务。  

使用过程：
1. 客户端连接到消息队列服务器，打开一个channel。
2. 客户端声明一个exchange，并设置相关属性。
3. 客户端声明一个queue，并设置相关属性。
4. 客户端使用routing key，在exchange和queue之间建立好绑定关系。
5. 客户端投递消息到exchange。
6. exchange接收到消息后，就根据消息的key和已经设置的binding，进行消息路由，将消息投递到一个或多个队列里。

**单机**  
RabbitMQ 安装需要依赖 Erlang 环境  
所以先安装erlang，再安装rabbitMQ,中间肯定会遇到很多问题，不同的环境问题不一样，大多数是依赖包的问题，自行补齐就行了。

rabbitmq默认创建的用户guest，密码也是guest，这个用户默认只能是本机访问,从外部访问需要添加上面的配置。建议删除guest用户  

配置外部访问
```sh
# rabbitmq.config文件默认是没有的，这里是直接新建
vi rabbitmq.config
# 添加下面内容
[{rabbit, [{loopback_users, []}]}].
```
删除guest用户  
```sh
rabbitmqctl  delete_user guest
```
添加用户
```sh
rabbitmqctl add_user <username> <newpassword>
```
给新增用户赋予超级管理员权限
```sh 
rabbitmqctl set_user_tags <username> administrator
```
启动服务(建议后台模式运行)
```sh
service rabbitmq-server start &
```
开启管理UI(建议后台模式运行)
```sh
rabbitmq-plugins enable rabbitmq_management &
```
访问：ip:15672,用新添加的用户登陆
![rabbitmq](http://upload-images.jianshu.io/upload_images/14603910-24915bab5c29dda7?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### kafka
几个重要概念： 
* Broker：Kafka集群包含一个或多个服务器，这种服务器被称为broker
* Topic：每条发布到Kafka集群的消息都有一个类别，这个类别被称为Topic。
* Partition：Parition是物理上的概念，每个Topic包含一个或多个Partition.
* Producer：负责发布消息到Kafka broker
* Consumer：消息消费者，向Kafka broker读取消息的客户端。
* Consumer Group：每个Consumer属于一个特定的Consumer Group（可为每个Consumer指定group name，若不指定group name则属于默认的group）。

**单机**  
Kafka使用Zookeeper来维护集群信息,所以需要先安装zookeeper，而zookeeper是由java编写的，所以需要先安装jdk
* jdk：1.8版本及以上(后面需要用到Kafka监控工具KafkaOffsetMonitor目前所依赖的jdk版本较高)
* zookeeper：新版本的kafka自带有zookeeper

下载地址：[https://kafka.apache.org/downloads](https://kafka.apache.org/downloads)
```shell
# 下载解压
mkdir kafka && cd kafka
wget http://mirrors.shuosc.org/apache/kafka/1.0.0/kafka_2.11-1.0.0.tgz
tar -xzf kafka_2.11-1.0.0.tgz
cd kafka_2.11-1.0.1
# 启动zookeeper
bin/zookeeper-server-start.sh config/zookeeper.properties
# 启动kafka
bin/kafka-server-start.sh config/server.properties
# 创建Topic test
bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test
# 参数描述：
# create: 创建Topic
# zookeeper：zookeeper集群信息，多个用,分开
# replication-factor：复制因子
# partitions：分区信息
# topic：主题名

# 向topic发送消息
bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test
> 随便输入

# 向topic获取消息
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --from-beginning
# 就能看到前面输入的消息了
```
Kafka监控工具   
kafka没有自带的web ui,这里使用KafkaOffsetMonitor, 程序一个jar包的形式运行，部署较为方便。只有监控功能，使用起来也较为安全。  

KafkaOffsetMonitor托管在Github上，可以通过Github下载。
下载地址：[https://github.com/Morningstar/kafka-offset-monitor/releases](https://github.com/Morningstar/kafka-offset-monitor/releases)  
```shell
# 下载好之后cd到KafkaOffsetMonitor所在目录，可以直接启动，也可以编写shell脚本来启动
java -cp KafkaOffsetMonitor-assembly-0.4.1-SNAPSHOT.jar com.quantifind.kafka.offsetapp.OffsetGetterWeb
--port 8089
--zk 127.0.0.1:2181 
--refresh 5.minutes 
--retain 1.day
```
编写脚本启动
```shell
mkdir kafka-monitor.sh
chmod  +x kafka-monitor.sh
vi kafka-monitor.sh
# 把之前启动的命令复制进来，如：
#! /bin/bash
java -Xms512M -Xmx512M -Xss1024K \
     -cp KafkaOffsetMonitor-assembly-0.4.1-SNAPSHOT.jar \
     com.quantifind.kafka.offsetapp.OffsetGetterWeb \
     --zk localhost:2181 \
     --port 8089 \
     --refresh 10.seconds \
     --retain 5.days
# 保存，然后就可以启动脚本了
```
zk ：zookeeper主机地址，如果有多个，用逗号隔开  
port ：应用程序端口（没设置的话，日志里面会输出随机的端口号）  
refresh ：应用程序在数据库中刷新和存储点的频率  
retain ：在db中保留多长时间  
dbName ：保存的数据库文件名，默认为offsetapp  

**github上详细参数说明：**
![KafkaOffsetMonitor](http://upload-images.jianshu.io/upload_images/14603910-3ecd84780dbd24fc?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
访问：ip:端口

![ui](http://upload-images.jianshu.io/upload_images/14603910-436f00c4fbbb071d?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

        Topic：订阅的主题
        Partition:分区编号
        Offest：表示该parition已经消费了多少条message
        logSize:表示该partition已经写了多少条message
        Lag：表示有多少条message没有被消费。
        Owner：表示消费者
        Created：该partition创建时间
        Last Seen：消费状态刷新最新时间。
**注意**  
本机能访问，但其它机器不能访问应该就是防火墙的问题，开放对应端口，或者关闭防火墙  
如果能访问，但是没有信息显示出来，只显示了黑色的屏，是因为ui页面所依赖的ajax.googleapis.com等公共库被墙了  
[解决办法](https://github.com/justjavac/ReplaceGoogleCDN)

### 结语
未完待续......。  
[个人博客](https://ohcomeyes.github.io)~  
[简书](https://www.jianshu.com/u/299dd40d2451)~  
