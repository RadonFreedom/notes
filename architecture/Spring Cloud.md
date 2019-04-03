# 微服务中一些重要概念

#### RPC

Dubbo. 在Spring Cloud中是Feign.

**API 网关**

> It is a single entry point into the system, used to handle requests by routing them to the appropriate backend service or by invoking multiple backend services and [aggregating the results](http://techblog.netflix.com/2013/01/optimizing-netflix-api.html). Also, it can be used for authentication, insights, stress and canary testing, service migration, static response handling, active traffic management.

[API 网关性能比较：NGINX vs. ZUUL vs. Spring Cloud Gateway vs. Linkerd](https://infoq.cn/article/comparing-api-gateway-performances)

#### 服务自动注册和发现

Zookeeper, 在SpringCloud中是eureka

#### 本地负载均衡 (服务之间沟通)

在SpringCloud中是ribbon

#### 客户端负载均衡 (客户端与服务沟通)

Nginx, zuul

#### 智能容错，熔断机制

Netflix Hystrix



# DUBBO 和 Spring Cloud

## Dubbo和Spring Cloud有何不同？

首先做一个简单的功能对比：

| 微服务架构所需功能               | Dubbo         | Spring Cloud        |
| -------------------------------- | ------------- | ------------------- |
| 服务注册中心                     | Zookeeper     | Netflix Eureka      |
| 服务调用方式                     | RPC           | REST API            |
| 服务监控                         | Dubbo-monitor | Spring Boot Admin   |
| 断路器                           | 不完善        | Netflix Hystrix     |
| 服务网关（客户端与服务负载均衡） | 无            | Netflix Zuul        |
| 分布式配置                       | 无            | Spring Cloud Config |
| 消息总线                         | 无            | Spring Cloud Bus    |
| 服务负载均衡（服务之间负载均衡） | 自带          | Ribbon              |

**从上图可以看出其实Dubbo的功能只是Spring Cloud体系的一部分。**

这样对比是不够公平的，首先 Dubbo 是 SOA 时代的产物，它的关注点主要在于服务的调用，流量分发、流量监控和熔断。而 Spring Cloud 诞生于微服务架构时代，考虑的是微服务治理的方方面面，另外由于依托了 Spring、Spring Boot 的优势之上，两个框架在开始目标就不一致，Dubbo 定位服务治理、Spring Cloud 是一个生态。

**如果仅仅关注于服务治理的这个层面，Dubbo其实还优于Spring Cloud很多：**

- Dubbo 支持更多的协议，如：rmi、hessian、http、webservice、thrift、memcached、redis 等。
- Dubbo 使用 RPC 协议效率更高，在极端压力测试下，Dubbo 的效率会高于 Spring Cloud 效率一倍多。
- Dubbo 有更强大的后台管理，Dubbo 提供的后台管理 Dubbo Admin 功能强大，提供了路由规则、动态配置、访问控制、权重调节、均衡负载等诸多强大的功能。
- 可以限制某个 IP 流量的访问权限，设置不同服务器分发不同的流量权重，并且支持多种算法，利用这些功能我们可以在线上做灰度发布、故障转移等，Spring Cloud 到现在还不支持灰度发布、流量权重等功能。

**所以Dubbo专注于服务治理；Spring Cloud关注于微服务架构生态。**

## Dubbo重新发布对Spring Cloud有影响吗？

国内技术人喜欢拿 Dubbo 和 Spring Cloud 进行对比，是因为两者都是服务治理非常优秀的开源框架。但它们两者的出发点是不一样的，Dubbo 关注于服务治理这块并且以后也会继续往这个方向去发展。Spring Cloud 关注的是微服务治理的生态。因为微服务治理的方方面面都是它所关注的内容，服务治理也只是微服务生态的一部分而已。因此可以大胆的断定，Dubbo 未来会在服务治理方面更为出色，而 Spring Cloud 在微服务治理上面无人能敌。

同时根据 Dubbo 最新的更新技术来看，Dubbo 也会积极的拥抱开源，拥抱新技术。Dubbo 接下来的版本将会很快的支持 Spring Boot，方便我们享受高效开发的同时，也可以支持高效的服务调用。Dubbo 被广泛应用于中国各互联网公司，如今阿里又重新重视起来并且发布了新版本和一系列的计划，对于正在使用 Dubbo 的公司来说是一个喜讯，对于中国广大的开发者来说更是一件非常喜悦的事情。我们非常乐于看到中国有一款非常优秀的开源框架，让我们有更多的选择，有更好的支持。

**两者其实不一定有竞争关系，如果使用得当甚至可以互补；另外两个关注的领域也不一致，因此对 Spring Cloud 的影响甚微。**

## 搭建微服务架构使用哪个？

简而言之，Dubbo确实类似于Spring Cloud的一个子集，Dubbo功能和文档完善，在国内有很多的成熟用户，然而鉴于Dubbo的社区现状（曾经长期停止维护，2017年7月31日团队又宣布重点维护），使用起来还是有一定的门槛。

Dubbo具有调度、发现、监控、治理等功能，**支持相当丰富的服务治理能力**。Dubbo架构下，注册中心对等集群，并会缓存服务列表已被数据库失效时继续提供发现功能，本身的服务发现结构有很强的**可用性与健壮性**，足够支持高访问量的网站。

虽然Dubbo 支持短连接大数据量的服务提供模式，但绝大多数情况下都是使用长连接小数据量的模式提供服务使用的。所以，**对于类似于电商等同步调用场景多并且能支撑搭建Dubbo 这套比较复杂环境的成本的产品而言，Dubbo 确实是一个可以考虑的选择**。但如果产品业务中由于后台业务逻辑复杂、时间长而导致异步逻辑比较多的话，可能Dubbo 并不合适。同时，**对于人手不足的初创产品而言，这么重的架构维护起来也不是很方便。**

Spring Cloud由众多子项目组成，如Spring Cloud Config、Spring Cloud Netflix、Spring Cloud Consul 等，提供了搭建分布式系统及微服务常用的工具，如配置管理、服务发现、断路器、智能路由、微代理、控制总线、一次性token、全局锁、选主、分布式会话和集群状态等，**满足了构建微服务所需的所有解决方案**。比如使用Spring Cloud Config 可以实现统一配置中心，对配置进行统一管理；使用Spring Cloud Netflix 可以实现Netflix 组件的功能 - 服务发现（Eureka）、智能路由（Zuul）、客户端负载均衡（Ribbon）。

但它并没有重复造轮子，而是选用目前各家公司开发的比较成熟的、**经得住实践考验**的服务框架（我们需要特别感谢Netflix ，这家很早就成功实践微服务的公司，几年前把自家几乎整个微服务框架栈贡献给了社区，Spring Cloud主要是对Netflix开源组件的进一步封装），通过Spring Boot 进行封装集成并简化其使用方式。基于Spring Boot，意味着其使用方式如Spring Boot **简单易用**；能够与Spring Framework、Spring Boot、Spring Data 等其他Spring 项目完美融合，意味着**能从Spring获得巨大的便利**，意味着**能减少已有项目的迁移成本**。

其实，**从社区活跃度和功能完整度，再对照业务需求和团队状况，基本可以确定如何选型**。这里分享网易考拉海购实践以及团队选型的心声：

> 当前开源上可选用的微服务框架主要有Dubbo、Spring Cloud等，鉴于Dubbo完备的功能和文档且在国内被众多大型互联网公司选用，考拉自然也选择了Dubbo作为服务化的基础框架。其实相比于Dubbo，Spring Cloud可以说是一个更完备的微服务解决方案，它从功能性上是Dubbo的一个超集，个人认为从选型上对于一些中小型企业Spring Cloud可能是一个更好的选择。提起Spring Cloud，一些开发的第一印象是http+JSON的rest通信，性能上难堪重用，其实这也是一种误读。
> 微服务选型要评估以下几点：内部是否存在异构系统集成的问题；备选框架功能特性是否满足需求；http协议的通信对于应用的负载量会否真正成为瓶颈点（Spring Cloud也并不是和http+JSON强制绑定的，如有必要Thrift、protobuf等高效的RPC、序列化协议同样可以作为替代方案）；社区活跃度、团队技术储备等。作为已经没有团队持续维护的开源项目，选择Dubbo框架内部就必须要组建一个维护团队，先不论你要准备要集成多少功能做多少改造，作为一个支撑所有工程正常运转的基础组件，问题的及时响应与解答、重大缺陷的及时修复能力就已足够重要。

鉴于服务发现对服务化架构的重要性，再补充一点：Dubbo 实践通常以ZooKeeper 为注册中心（Dubbo 原生支持的Redis 方案需要服务器时间同步，且性能消耗过大）。针对分布式领域著名的CAP理论（C——数据一致性，A——服务可用性，P——服务对网络分区故障的容错性），**Zookeeper 保证的是CP ，但对于服务发现而言，可用性比数据一致性更加重要 ，而 Eureka 设计则遵循AP原则** 。

可能很多人正在犹豫，在服务治理的时候应该选择那个框架呢？如果公司对效率有极高的要求建议使用 Dubbo，相对比 RPC 的效率会比 HTTP 高很多；如果团队不想对技术架构做大的改造建议使用 Dubbo，Dubbo 仅仅需要少量的修改就可以融入到内部系统的架构中。但如果技术团队喜欢挑战新技术，建议选择 Spring Cloud，Spring Cloud 架构体系有有趣很酷的技术。如果公司选择微服务架构去重构整个技术体系，那么 Spring Cloud 是当仁不让之选，它可以说是目前最好的微服务框架没有之一。

最后，技术选型是一个综合的问题，需要考虑团队的情况、业务的发展以及公司的产品特征。最炫最酷的技术并不一定是最好的，选择适合自己团队、符合公司业务的框架才是最佳方案。



![img](images/Spring Cloud/435188-20180412214045427-176081083.png)



# 注册中心 EUREKA



## F&Q

### bootstrap.yml application.yml 的区别？

**bootstrap.yml is loaded before application.yml.**

It is typically used for the following:

- when using Spring Cloud Config Server, you should specify `spring.application.name` and `spring.cloud.config.server.git.uri` inside `bootstrap.yml`
- some `encryption/decryption` information

Technically, `bootstrap.yml` is loaded by a parent Spring `ApplicationContext`. That parent `ApplicationContext` is loaded before the one that uses `application.yml`.











