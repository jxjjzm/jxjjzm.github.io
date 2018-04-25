# Dubbo源码篇之整体架构 #
***

### 一、Dubbo回顾 ###

Dubbo是一个分布式服务框架，致力于提供高性能和透明化的**RPC远程服务调用**方案，以及**SOA服务治理**方案。

（1）DUBBO的核心部分包含：


- 远程通讯: 提供对多种基于长连接的NIO框架抽象封装，包括多种线程模型，序列化，以及“请求-响应”模式的信息交换方式。


- 集群容错: 提供基于接口方法的透明远程过程调用，包括多协议支持，以及软负载均衡，失败容错，地址路由，动态配置等集群支持。


- 自动发现: 基于注册中心目录服务，使服务消费方能动态的查找服务提供方，使地址透明，使服务提供方可以平滑增加或减少机器。


（2）DUBBO的基本原理：

![](https://i.imgur.com/N8Wcbg3.png)

![](https://i.imgur.com/9H6eyMZ.png)


Dubbo源码系列博文重点针对的是对Dubbo比较熟悉的读者，如果你对Dubbo还处于不甚了解的状态建议先去熟悉下Dubbo，这里提供几个常见的链接地址仅供参考：




- [Dubbo官网](http://dubbo.io/ "Dubbo官网")
- [Dubbo GitHub](https://github.com/dubbo)
- [Dubbo用户手册](http://dubbo.apache.org/books/dubbo-user-book/ "Dubbo用户手册")
- [Dubbo管理员手册](http://dubbo.apache.org/books/dubbo-admin-book/ "Dubbo管理员手册")
- [Dubbo开发手册](http://dubbo.apache.org/books/dubbo-dev-book/ "Dubbo开发手册")





### 二、Dubbo整体设计 ###

![](https://i.imgur.com/6uEcvOg.png)

Dubbo是Alibaba开源的分布式服务框架，它最大的特点是按照分层的方式来架构，使用这种方式可以使各个层之间解耦合（或者最大限度地松耦合）。Dubbo框架设计一共划分了10个层，而最上面的Service层是留给实际想要使用Dubbo开发分布式服务的开发者实现业务逻辑的接口层。
（图中左边淡蓝背景的为服务消费方使用的接口，右边淡绿色背景的为服务提供方使用的接口， 位于中轴线上的为双方都用到的接口。 ）下面，结合Dubbo官方文档，我们分别理解一下框架分层架构中，各个层次的设计要点：



- 服务接口层（Service）：该层是与实际业务逻辑相关的，根据服务提供方和服务消费方的业务设计对应的接口和实现。
- 配置层（Config）：对外配置接口，以ServiceConfig和ReferenceConfig为中心，可以直接new配置类，也可以通过spring解析配置生成配置类。
- 服务代理层（Proxy）：服务接口透明代理，生成服务的客户端Stub和服务器端Skeleton，以ServiceProxy为中心，扩展接口为ProxyFactory。
- 服务注册层（Registry）：封装服务地址的注册与发现，以服务URL为中心，扩展接口为RegistryFactory、Registry和RegistryService。可能没有服务注册中心，此时服务提供方直接暴露服务。
- 集群层（Cluster）：封装多个提供者的路由及负载均衡，并桥接注册中心，以Invoker为中心，扩展接口为Cluster、Directory、Router和LoadBalance。将多个服务提供方组合为一个服务提供方，实现对服务消费方来透明，只需要与一个服务提供方进行交互。
- 监控层（Monitor）：RPC调用次数和调用时间监控，以Statistics为中心，扩展接口为MonitorFactory、Monitor和MonitorService。
- 远程调用层（Protocol）：封将RPC调用，以Invocation和Result为中心，扩展接口为Protocol、Invoker和Exporter。Protocol是服务域，它是Invoker暴露和引用的主功能入口，它负责Invoker的生命周期管理。Invoker是实体域，它是Dubbo的核心模型，其它模型都向它靠扰，或转换成它，它代表一个可执行体，可向它发起invoke调用，它有可能是一个本地的实现，也可能是一个远程的实现，也可能一个集群实现。
- 信息交换层（Exchange）：封装请求响应模式，同步转异步，以Request和Response为中心，扩展接口为Exchanger、ExchangeChannel、ExchangeClient和ExchangeServer。
- 网络传输层（Transport）：抽象mina和netty为统一接口，以Message为中心，扩展接口为Channel、Transporter、Client、Server和Codec。
- 数据序列化层（Serialize）：可复用的一些工具，扩展接口为Serialization、 ObjectInput、ObjectOutput和ThreadPool。





























