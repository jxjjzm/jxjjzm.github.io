# Welcome to jxjjzm GitHub Pages


You can use the [editor on GitHub](https://github.com/jxjjzm/jxjjzm.github.io/edit/master/README.md) to maintain and preview the content for your website in Markdown files.

Whenever you commit to this repository, GitHub Pages will run [Jekyll](https://jekyllrb.com/) to rebuild the pages in your site, from the content in your Markdown files.

***
#目录

* [java系列](#jxjjzm-java)
	* [专题之Java集合](#jxjjzm-java-collection)
	* [专题之Design Pattern](#jxjjzm-java-design-pattern)
	* [专题之Java8](#jxjjzm-java-java8)
	* [专题之Java并发](#jxjjzm-java-juc)
	* [专题之JVM](#jxjjzm-java-jvm)
* [Distributed System 系列](#jxjjzm-distributed-system)
	* [分布式技术杂谈](#jxjjzm-distributed-system-talk)
	* [Zookeeper](#jxjjzm-distributed-system-zookeeper)
	* [Redis](#jxjjzm-distributed-system-redis)
	* [](#jxjjzm-distributed-system-redis)
	* [](#jxjjzm-distributed-system-talk)
* [Dubbo系列](#jxjjzm-dubbo)
	* [Dubbo源码篇](#jxjjzm-dubbo-code)
	* [Dubbo实战篇](#jxjjzm-dubbo-inAction)
* [MQ系列](#mq)
* [Database系列](#database)
* [Netty系列](#netty)
* [Spring Cloud系列](#spring-cloud)



<h2 id="jxjjzm-java">Java系列</h2>
<h3 id="jxjjzm-java-collection">专题之Java集合</h3>

- [Java HashMap源码解读](https://github.com/jxjjzm/jxjjzm.github.io/blob/master/Java%E7%B3%BB%E5%88%97/Collection/Java%20HashMap%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB.md)
- [Java LinkedHashMap源码解读](https://github.com/jxjjzm/jxjjzm.github.io/blob/master/Java%E7%B3%BB%E5%88%97/Collection/Java%20LinkedHashMap%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB.md)
- [Java List源码解读](https://github.com/jxjjzm/jxjjzm.github.io/blob/master/Java%E7%B3%BB%E5%88%97/Collection/Java%20List%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB.md)
- [Java Set源码解读](https://github.com/jxjjzm/jxjjzm.github.io/blob/master/Java%E7%B3%BB%E5%88%97/Collection/Java%20Set%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB.md)

<h3 id="jxjjzm-java-juc">专题之Java并发</h3>

- [浅析Java内存模型（JMM)](https://github.com/jxjjzm/jxjjzm.github.io/blob/master/Java%E7%B3%BB%E5%88%97/Concurrent/%E6%B5%85%E6%9E%90Java%E5%86%85%E5%AD%98%E6%A8%A1%E5%9E%8B.md)
- [浅析Unsafe & CAS & AQS](https://github.com/jxjjzm/jxjjzm.github.io/blob/master/Java%E7%B3%BB%E5%88%97/Concurrent/%E6%B5%85%E6%9E%90Unsafe%20%26%20CAS%20%26%20AQS.md)
- [浅析synchronized & volatile & 锁优化](https://github.com/jxjjzm/jxjjzm.github.io/blob/master/Java%E7%B3%BB%E5%88%97/Concurrent/%E6%B5%85%E6%9E%90synchronized%20%26%20volatile%20%26%20%E9%94%81%E4%BC%98%E5%8C%96.md)
- [浅析Threadlocal](https://github.com/jxjjzm/jxjjzm.github.io/blob/master/Java%E7%B3%BB%E5%88%97/Concurrent/%E6%B5%85%E6%9E%90Threadlocal.md)
- [浅析ReentrantLock & Condition](https://github.com/jxjjzm/jxjjzm.github.io/blob/master/Java%E7%B3%BB%E5%88%97/Concurrent/%E6%B5%85%E6%9E%90ReentrantLock%20%26%20Condition.md)
- [浅析Semaphore](https://github.com/jxjjzm/jxjjzm.github.io/blob/master/Java%E7%B3%BB%E5%88%97/Concurrent/%E6%B5%85%E6%9E%90Semaphore.md)
- [浅析CountDownlatch & CyclicBarrier](https://github.com/jxjjzm/jxjjzm.github.io/blob/master/Java%E7%B3%BB%E5%88%97/Concurrent/%E6%B5%85%E6%9E%90CountDownlatch%20%26%20CyclicBarrier.md)
- [JUC锁框架概述](https://github.com/jxjjzm/jxjjzm.github.io/blob/master/Java%E7%B3%BB%E5%88%97/Concurrent/JUC%E9%94%81%E6%A1%86%E6%9E%B6%E6%A6%82%E8%BF%B0.md)
- [Executor框架与线程池](https://github.com/jxjjzm/jxjjzm.github.io/blob/master/Java%E7%B3%BB%E5%88%97/Concurrent/Executor%E6%A1%86%E6%9E%B6%E4%B8%8E%E7%BA%BF%E7%A8%8B%E6%B1%A0.md)

<h3 id="jxjjzm-java-jvm">专题之JVM</h3>


- [Jvm之自动内存管理机制(内存分配、垃圾回收)](https://github.com/jxjjzm/jxjjzm.github.io/blob/master/Java%E7%B3%BB%E5%88%97/Jvm/Jvm%E4%B9%8B%E8%87%AA%E5%8A%A8%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%9C%BA%E5%88%B6(%E5%86%85%E5%AD%98%E5%88%86%E9%85%8D%E3%80%81%E5%9E%83%E5%9C%BE%E5%9B%9E%E6%94%B6).md "Jvm之自动内存管理机制(内存分配、垃圾回收)")
- [Jvm之虚拟机类加载机制](https://github.com/jxjjzm/jxjjzm.github.io/blob/master/Java%E7%B3%BB%E5%88%97/Jvm/Jvm%E4%B9%8B%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%B1%BB%E5%8A%A0%E8%BD%BD%E6%9C%BA%E5%88%B6.md "Jvm之虚拟机类加载机制")
- [Jvm之虚拟机性能监控与故障处理工具](https://github.com/jxjjzm/jxjjzm.github.io/blob/master/Java%E7%B3%BB%E5%88%97/Jvm/Jvm%E4%B9%8B%E8%99%9A%E6%8B%9F%E6%9C%BA%E6%80%A7%E8%83%BD%E7%9B%91%E6%8E%A7%E5%91%BD%E4%BB%A4%E4%B8%8E%E6%95%85%E9%9A%9C%E5%A4%84%E7%90%86%E5%B7%A5%E5%85%B7.md)



<h3 id="jxjjzm-java-design-pattern">专题之Design Pattern</h3>

* [设计模式之单例模式](https://github.com/jxjjzm/jxjjzm.github.io/blob/master/Dubbo%E7%B3%BB%E5%88%97/Dubbo%E6%BA%90%E7%A0%81%E7%AF%87/Dubbo%E6%A0%B8%E5%BF%83%E6%9C%BA%E5%88%B6%E5%88%86%E6%9E%90%E4%B9%8B%E6%9C%8D%E5%8A%A1%E7%9A%84%E5%90%AF%E5%8A%A8%E4%B8%8E%E5%88%9D%E5%A7%8B%E5%8C%96%EF%BC%88Bean%E5%8A%A0%E8%BD%BD%EF%BC%89.md)


<h3 id="jxjjzm-java-java8">专题之Java8</h3>

* [Java8新特性之流（Stream）](https://github.com/jxjjzm/jxjjzm.github.io/blob/master/Java%E7%B3%BB%E5%88%97/Java8/Java8%E6%96%B0%E7%89%B9%E6%80%A7%E4%B9%8B%E6%B5%81%EF%BC%88Stream%EF%BC%89.md)





<h2 id="jxjjzm-distributed-system">Distributed System 系列</h2>

<h3 id="jxjjzm-distributed-system-talk">分布式技术杂谈</h3>

- [浅析分布式高可用之限流特技](https://github.com/jxjjzm/jxjjzm.github.io/blob/master/Distributed%20System/%E6%B5%85%E6%9E%90%E5%88%86%E5%B8%83%E5%BC%8F%E9%AB%98%E5%8F%AF%E7%94%A8%E4%B9%8B%E9%99%90%E6%B5%81%E7%89%B9%E6%8A%80.md "浅析分布式高可用之限流特技")
- [浅析分布式高可用之降级特技](https://github.com/jxjjzm/jxjjzm.github.io/blob/master/Distributed%20System/%E6%B5%85%E6%9E%90%E5%88%86%E5%B8%83%E5%BC%8F%E9%AB%98%E5%8F%AF%E7%94%A8%E4%B9%8B%E9%99%8D%E7%BA%A7%E7%89%B9%E6%8A%80.md)
- [浅析分布式网络通信](https://github.com/jxjjzm/jxjjzm.github.io/blob/master/Distributed%20System/%E6%B5%85%E6%9E%90%E5%88%86%E5%B8%83%E5%BC%8F%E7%BD%91%E7%BB%9C%E9%80%9A%E4%BF%A1.md "分布式网络通信")
- [浅析分布式锁技术](https://github.com/jxjjzm/jxjjzm.github.io/blob/master/Distributed%20System/%E6%B5%85%E6%9E%90%E5%88%86%E5%B8%83%E5%BC%8F%E9%94%81%E6%8A%80%E6%9C%AF.md "浅析分布式锁技术")
- [浅析Redis分布式缓存](https://github.com/jxjjzm/jxjjzm.github.io/blob/master/Distributed%20System/%E6%B5%85%E6%9E%90Redis%E5%88%86%E5%B8%83%E5%BC%8F%E7%BC%93%E5%AD%98.md "浅析Redis分布式缓存")
- [浅析分布式事务技术](https://github.com/jxjjzm/jxjjzm.github.io/blob/master/Distributed%20System/%E6%B5%85%E6%9E%90%E5%88%86%E5%B8%83%E5%BC%8F%E4%BA%8B%E5%8A%A1%E6%8A%80%E6%9C%AF.md "浅析分布式事务技术")


<h3 id="jxjjzm-distributed-system-zookeeper">Zookeeper</h3>

- [Zookeeper概述篇](https://github.com/jxjjzm/jxjjzm.github.io/blob/master/Zookeeper%E7%B3%BB%E5%88%97/Zookeeper%E6%A6%82%E8%BF%B0%E7%AF%87.md)
- [Zookeeper使用篇](https://github.com/jxjjzm/jxjjzm.github.io/blob/master/Zookeeper%E7%B3%BB%E5%88%97/Zookeeper%E4%BD%BF%E7%94%A8%E7%AF%87.md)
- [Zookeepe原理篇](https://github.com/jxjjzm/jxjjzm.github.io/blob/master/Zookeeper%E7%B3%BB%E5%88%97/Zookeepe%E5%8E%9F%E7%90%86%E7%AF%87.md)
- [Zookeepe运维篇](https://github.com/jxjjzm/jxjjzm.github.io/blob/master/Zookeeper%E7%B3%BB%E5%88%97/Zookeepe%E8%BF%90%E7%BB%B4%E7%AF%87.md)


<h3 id="jxjjzm-distributed-system-redis">Redis</h3>


* [Redis概述篇](https://github.com/jxjjzm/jxjjzm.github.io/blob/master/Redis%E7%B3%BB%E5%88%97/Redis%E6%A6%82%E8%BF%B0%E7%AF%87.md)
* [Redis使用篇——Redis安装与配置](https://github.com/jxjjzm/jxjjzm.github.io/blob/master/Redis%E7%B3%BB%E5%88%97/Redis%E4%BD%BF%E7%94%A8%E7%AF%87%E2%80%94%E2%80%94Redis%E5%AE%89%E8%A3%85%E4%B8%8E%E9%85%8D%E7%BD%AE.md)：





<h2 id="jxjjzm-dubbo">Dubbo系列</h2>
<h3 id="jxjjzm-dubbo-code">Dubbo源码篇</h3>


* [Dubbo核心机制分析之服务的启动与初始化（Bean加载）](https://github.com/jxjjzm/jxjjzm.github.io/blob/master/Dubbo%E7%B3%BB%E5%88%97/Dubbo%E6%BA%90%E7%A0%81%E7%AF%87/Dubbo%E6%A0%B8%E5%BF%83%E6%9C%BA%E5%88%B6%E5%88%86%E6%9E%90%E4%B9%8BBean%E5%8A%A0%E8%BD%BD.md)：*本篇博文主要剖析了Dubbo服务是如何启动的以及Dubbo是如何加载解析自定义配置文件并初始化Bean的。*
* [Dubbo核心机制分析之Extension](https://www.baidu.com/)：
* [Dubbo核心机制分析之Invoker](https://www.baidu.com/)：
* [Dubbo核心机制分析之Exchange](https://www.baidu.com/) ：
* [Dubbo核心机制分析之Filter](https://www.baidu.com/)：
* [Dubbo核心机制分析之Listener](https://www.baidu.com/)：
* [Dubbo核心机制分析之Proxy](https://www.baidu.com/)：
* [Dubbo核心机制分析之RPC](https://www.baidu.com/)：
* [Dubbo过程分析之Export](https://www.baidu.com/)：
* [Dubbo过程分析之Refer](https://www.baidu.com/)：
* [Dubbo过程分析之Registry](https://www.baidu.com/) ：


<h3 id="jxjjzm-dubbo-inAction">Dubbo实战篇</h3>

* [Dubbo快速入门](https://github.com/jxjjzm/jxjjzm.github.io/blob/master/Dubbo%E7%B3%BB%E5%88%97/Dubbo%E5%AE%9E%E6%88%98%E7%AF%87/Dubbo%E5%85%A5%E9%97%A8.md)：
* [Dubbo高级配置](https://www.baidu.com/)：
* [Dubbo问题集锦](https://www.baidu.com/)：



<h2 id="jxjjzm-spring">Spring系列</h2>
<h3 id="jxjjzm-spring-mvc">Spring+Spring MVC</h3>
TODO
<h3 id="jxjjzm-spring-boot">Spring Boot</h3>
TODO
<h3 id="jxjjzm-spring-mvc">Spring Cloud</h3>
TODO
<h3 id="jxjjzm-spring-mvc">Spring Data</h3>
TODO
<h3 id="jxjjzm-spring-mvc">Spring Batch</h3>
TODO
<h3 id="jxjjzm-spring-mvc">Spring Security</h3>
TODO






* * *
### Jekyll Themes

Your Pages site will use the layout and styles from the Jekyll theme you have selected in your [repository settings](https://github.com/jxjjzm/jxjjzm.github.io/settings). The name of this theme is saved in the Jekyll `_config.yml` configuration file.

### Support or Contact

Having trouble with Pages? Check out our [documentation](https://help.github.com/categories/github-pages-basics/) or [contact support](https://github.com/contact) and we’ll help you sort it out.
