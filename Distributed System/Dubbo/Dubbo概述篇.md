### Dubbo概述篇 ###
***

### 一、Dubbo调用流程 ###

![](http://dubbo.apache.org/docs/zh-cn/user/sources/images/dubbo-architecture.jpg)





- 0.服务容器负责启动，加载，运行服务提供者。
- 1.服务提供者在启动时，向注册中心注册自己提供的服务。
- 2.服务消费者在启动时，向注册中心订阅自己所需的服务。
- 3.注册中心返回服务提供者地址列表给消费者，如果有变更，注册中心将基于长连接推送变更数据给消费者。
- 4.服务消费者，从提供者地址列表中，基于软负载均衡算法，选一台提供者进行调用，如果调用失败，再选另一台调用。
- 5.服务消费者和提供者，在内存中累计调用次数和调用时间，定时每分钟发送一次统计数据到监控中心。


这么设计的意义：


- Consumer 与Provider 解偶，双方都可以横向增减节点数。
- 注册中心对本身可做对等集群，可动态增减节点，并且任意一台宕掉后，将自动切换到另一台
- 去中心化，双方不直接依懒注册中心，即使注册中心全部宕机短时间内也不会影响服务的调用 （服务提供者和服务消费者仍能通过本地缓存通讯）
- 服务提供者无状态，任意一台宕掉后，不影响使用



### 二、Dubbo核心原理 ###


![](https://cdn.nlark.com/yuque/0/2018/png/151326/1544674351821-3880af9c-c216-4489-a9f0-dadd9fdd13ed.png)











































































