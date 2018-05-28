### Elastic-Job入门篇 ###
***

### 一、作业（定时任务）系统 ###


#### 1.作业（定时任务）的必要性 ####

作业即定时任务。一般来说，系统可使用消息传递代替部分使用作业的场景（两者确实有相似之处）。但在某些场景下则不能互换：


- **时间驱动 OR 事件驱动**：内部系统一般可以通过事件来驱动，但涉及到外部系统，则只能使用时间驱动。如：抓取外部系统价格，每小时抓取，由于是外部系统，不能像内部系统一样发送事件触发事件。



- **批量处理 OR 逐条处理**：批量处理堆积的数据更加高效，在不需要实时性的情况下比消息中间件更有优势。而且有的业务逻辑只能批量处理，如：电商公司与快递公司结算，一个月结算一次，并且根据送货的数量有提成。比如，当月送货超过1000则额外给快递公司多1%的快递费。



- **非实时性 OR 实时性**：虽然消息中间件可以做到实时处理数据，但有的情况并不需要。如：VIP用户降级，如果超过1年无购买行为，则自动降级。这类需求没有强烈的时间要求，不需要按照时间精确的降级VIP用户。



- **系统内部 OR 系统解耦**：作业一般封装在系统内部，而消息中间件可用于系统间解耦。


#### 2.常见的作业系统 ####



- **Quartz**：Java事实上的定时任务标准。但Quartz关注点在于定时任务而非数据，并无一套根据数据处理而定制化的流程。虽然Quartz可以基于数据库实现作业的高可用，但缺少分布式并行调度的功能。



- **TBSchedule**：阿里早期开源的分布式任务调度系统。代码略陈旧，使用timer而非线程池执行任务调度。众所周知，timer在处理异常状况时是有缺陷的。而且TBSchedule作业类型较为单一，只能是获取/处理数据一种模式。还有就是文档缺失比较严重。



- **Crontab**：Linux系统级的定时任务执行器。缺乏分布式和集中管理功能。

综上所述，当前存在的作业系统缺少分布式、并行调度、弹性扩容缩容、集中管理、定制化流程型任务等功能，所以需要一个新的作业系统完善这些功能，Elastic-Job便应运而生。



### 二、Elastic-Job 介绍 ###


Elastic-Job是ddframe中dd-job的作业模块中分离出来的分布式弹性作业框架，去掉了dd-job中的监控和ddframe接入规范部分。该项目基于成熟的开源产品Quartz和Zookeeper及其客户端Curator进行二次开发。（项目开源地址：https://github.com/dangdangdotcom/elastic-job）


Elastic-job在2.x之后，出了两个产品线：Elastic-Job-Lite和Elastic-Job-Cloud。Elastic-Job-Lite定位为轻量级无中心化解决方案，使用jar包的形式提供分布式任务的协调服务；Elastic-Job-Cloud采用自研Mesos Framework的解决方案，额外提供资源治理、应用分发以及进程隔离等功能。（本文也是以Elastic-Job-Lite为主，1.x系列对应的就只有Elastic-Job-Lite，并且在2.x里修改了一些核心类名，差别虽大，原理类似，建议使用2.x系列。）

![](http://tech.lede.com/2017/06/23/rd/server/elasticJob/%E6%95%B4%E4%BD%93%E7%BB%93%E6%9E%84%E5%9B%BE.png)


#### 1.主要功能 ####



- 分布式：重写Quartz基于数据库的分布式功能，改用Zookeeper实现注册中心。


- 并行调度：采用任务分片方式实现。将一个任务拆分为n个独立的任务项，由分布式的服务器并行执行各自分配到的分片项。


- 弹性扩容缩容：将任务拆分为n个任务项后，各个服务器分别执行各自分配到的任务项。一旦有新的服务器加入集群，或现有服务器下线，elastic-job将在保留本次任务执行不变的情况下，下次任务开始前触发任务重分片。


- 集中管理：采用基于Zookeeper的注册中心，集中管理和协调分布式作业的状态，分配和监听。外部系统可直接根据Zookeeper的数据管理和- 监控elastic-job。


- 定制化流程型任务：作业可分为简单和数据流处理两种模式，数据流又分为高吞吐处理模式和顺序性处理模式，其中高吞吐处理模式可以开启足够多的线程快速的处理数据，而顺序性处理模式将每个分片项分配到一个独立线程，用于保证同一分片的顺序性。


- 失效转移：弹性扩容缩容在下次作业运行前重分片，但本次作业执行的过程中，下线的服务器所分配的作业将不会重新被分配。失效转移功能可以在本次作业运行中用空闲服务器抓取孤儿作业分片执行。同样失效转移功能也会牺牲部分性能。


- 运行时状态收集：监控作业运行时状态，统计最近一段时间处理的数据成功和失败数量，记录作业上次运行开始时间，结束时间和下次运行时间。


- 作业停止，恢复和禁用：用于操作作业启停，并可以禁止某作业运行（上线时常用）。


- Spring命名空间支持：elastic-job可以不依赖于spring直接运行，但是也提供了自定义的命名空间方便与spring集成。


- 运维平台：提供web控制台用于管理作业和注册中心。


- 稳定性：在服务器无波动的情况下，并不会重新分片；即使服务器有波动，下次分片的结果也会根据服务器IP和作业名称哈希值算出稳定的分片顺序，尽量不做大的变动。
高性能
同一服务器的批量数据处理采用自动切割并多线程并行处理。


- 灵活性：所有在功能和性能之间的权衡，都可通过配置开启/关闭。如：elastic-job会将作业运行状态的必要信息更新到注册中心。如果作业执行频度很高，会造成大量Zookeeper写操作，而分布式Zookeeper同步数据可能引起网络风暴。因此为了考虑性能问题，可以牺牲一些功能，而换取性能的提升。


- 幂等性：elastic-job可牺牲部分性能用以保证同一分片项不会同时在两个服务器上运行。


- 容错性：作业服务器与Zookeeper服务器通信失败则立即停止作业运行，防止作业注册中心将失效的分片分项配给其他作业服务器，而当前作业服务器仍在执行任务，导致重复执行。



#### 2.elastic-job的具体模块 ####



- 去中心化：去中心化指elastic-job并无调度中心这一概念，每个运行在集群中的作业服务器都是对等的，节点之间通过注册中心进行分布式协调。但elastic-job有主节点的概念，主节点用于处理一些集中式任务，如分片，清理运行时信息等，并无调度功能，定时调度都是由作业服务器自行触发。


- 注册中心：注册中心模块目前直接使用zookeeper，用于记录作业的配置，服务器信息以及作业运行状态。Zookeeper虽然很成熟，但原理复杂，使用较难，在海量数据支持的情况下也会有性能和网络问题。目前elastic-job已经抽象出注册中心的接口，下一步将会考虑支持多注册中心，如etcd，或由用户自行实现注册中心。无临时节点和监听机制的注册中心需要自行实现定时心跳监测等功能。


- 数据分片：数据分片是elastic-job中实现分布式的重要概念，将真实数据和逻辑分片对应，用于解耦作业框架和数据的关系。作业框架只负责将分片合理的分配给相关的作业服务器，而作业服务器需要根据所分配的分片匹配数据进行处理。服务器分片目前都存储在注册中心中，各个服务器根据自己的IP地址拉取分片。


- 分布式协调：分布式协调模块用于处理作业服务器的动态扩容缩容。一旦集群中有服务器发生变化，分布式协调将自动监测并将变化结果通知仍存活的作业服务器。协调时将会涉及主节点选举，重分片等操作。目前使用的Zookeeper的临时节点和监听器实现主动检查和通知功能。


- 定时任务处理：定时任务处理根据cron表达式定时触发任务，目前有防止任务同时触发，错过任务重出发等功能。主要还是使用Quartz本身的定时调度功能，为了便于控制，每个任务都使用独立的线程池。


- 定制化流程型任务：定制化流程型任务将定时任务分为多种流程，有不经任何修饰的简单任务；有用于处理数据的fetchData/processData的数据流任务；以后还将增加消息流任务，文件任务，工作流任务等。用户能以插件的形式扩展并贡献代码。


#### 3.流程图 ####

**作业启动**：

![](http://ovfotjrsi.bkt.clouddn.com/docs/img/principles/job_start.jpg)


**作业执行**：

![](http://ovfotjrsi.bkt.clouddn.com/docs/img/principles/job_exec.jpg)


### 三、Elastic-Job 快速使用 ###


#### 1.目录结构说明 ####


	elastic-job
	    ├──elastic-job-lite                                 lite父模块，不应直接使用
	    ├      ├──elastic-job-lite-core                     Java支持模块，可直接使用
	    ├      ├──elastic-job-lite-spring                   Spring命名空间支持模块，可直接使用
	    ├      ├──elastic-job-lite-lifecyle                 lite作业相关操作模块，不可直接使用
	    ├      ├──elastic-job-lite-console                  lite界面模块，可直接使用
	    ├──elastic-job-example                              使用示例
	    ├      ├──elastic-job-example-embed-zk              供示例使用的内嵌ZK模块
	    ├      ├──elastic-job-example-jobs                  作业示例
	    ├      ├──elastic-job-example-lite-java             基于Java的使用示例
	    ├      ├──elastic-job-example-lite-spring           基于Spring的使用示例
	    ├      ├──elastic-job-example-lite-springboot       基于SpringBoot的使用示例
	    ├──elastic-job-doc                                  markdown生成文档的项目，使用方无需关注
	    ├      ├──elastic-job-lite-doc                      lite相关文档



#### 2.quick start ####

**（1）引入maven依赖**

	<dependency>
	    <artifactId>elastic-job-lite-core</artifactId>
	    <groupId>com.dangdang</groupId>
	    <version>2.1.2</version>
	</dependency>
	<dependency>
	    <artifactId>elastic-job-common-core</artifactId>
	    <groupId>com.dangdang</groupId>
	    <version>2.1.2</version>
	    <exclusions>
	        <exclusion>
	            <artifactId>curator-framework</artifactId>
	            <groupId>org.apache.curator</groupId>
	        </exclusion>
	        <exclusion>
	            <artifactId>curator-recipes</artifactId>
	            <groupId>org.apache.curator</groupId>
	        </exclusion>
	    </exclusions>
	</dependency>
	<dependency>
	    <artifactId>curator-framework</artifactId>
	    <groupId>org.apache.curator</groupId>         
	    <version>2.10.0</version>
	</dependency>
	<dependency>
	    <artifactId>curator-recipes</artifactId>
	    <groupId>org.apache.curator</groupId>
	    <version>2.10.0</version>
	    <exclusions>
	        <exclusion>
	            <artifactId>curator-framework</artifactId>
	            <groupId>org.apache.curator</groupId>
	        </exclusion>
	    </exclusions>
	</dependency>


（注意：；引入elastic-job-common-core的时候，必须去掉curator-framework和curator-recipes这两个依赖，然后单独引入2.10.0版本的依赖。因为在elastic-job-common-core中，只有2.10.0版本的curator才会创建zk节点，通过elastic-job-common-core引入的curator是2.11.1版本。）


**（2）作业开发**

elastic-job提供了三种类型的job：


- SimpleJob ： 意为简单实现，未经任何封装的类型。需实现SimpleJob接口。该接口仅提供单一方法用于覆盖，此方法将定时执行。与Quartz原生接口相似，但提供了弹性扩缩容和分片等功能。

		/**
		 * 执行作业.
		 *
		 * @param shardingContext 分片上下文
		 */
		void execute(ShardingContext shardingContext);


- DataflowJob ： Dataflow类型用于处理数据流，需实现DataflowJob接口。该接口提供2个方法可供覆盖，分别用于抓取(fetchData)和处理(processData)数据。fetchData方法的返回值只有为null或长度为空时，作业会停止执行，不会执行processData。

		/**
		 * 获取待处理数据.
		 *
		 * @param shardingContext 分片上下文
		 * @return 待处理的数据集合
		 */
		List<T> fetchData(ShardingContext shardingContext);
		
		/**
		 * 处理数据.
		 *
		 * @param shardingContext 分片上下文
		 * @param data 待处理数据集合
		 */
		void processData(ShardingContext shardingContext, List<T> data);

流式处理：可通过DataflowJobConfiguration配置是否流式处理。




- ScriptJob ： Script类型作业意为脚本类型作业，支持shell，python，perl等所有类型脚本。只需通过控制台或代码配置scriptCommandLine即可，无需编码。执行脚本路径可包含参数，参数传递完毕后，作业框架会自动追加最后一个参数为作业运行时信息。

		#!/bin/bash
		echo sharding execution context is $*



**（3）作业启动配置**

作业启动配置有三种方式：


- java方式配置启动作业
- 使用Spring但不使用命名空间配置启动
- 基于Spring命名空间配置启动


（三种方式详细配置请参考官网配置手册：[http://elasticjob.io/docs/elastic-job-lite/02-guide/config-manual/](http://elasticjob.io/docs/elastic-job-lite/02-guide/config-manual/)）



#### 3.分片策略 ####

**（1）框架提供的分片策略**



- AverageAllocationJobShardingStrategy ： 基于平均分配算法的分片策略，也是默认的分片策略。如果分片不能整除，则不能整除的多余分片将依次追加到序号小的服务器。



- OdevitySortByNameJobShardingStrategy ： 根据作业名的哈希值奇偶数决定IP升降序算法的分片策略，作业名的哈希值为奇数则IP升序，作业名的哈希值为偶数则IP降序。用于不同的作业平均分配负载至不同的服务器。


（说明：AverageAllocationJobShardingStrategy的缺点是，一旦分片数小于作业服务器数，作业将永远分配至IP地址靠前的服务器，导致IP地址靠后的服务器空闲。而OdevitySortByNameJobShardingStrategy则可以根据作业名称重新分配服务器负载。）




- RotateServerByNameJobShardingStrategy ： 根据作业名的哈希值对服务器列表进行轮转的分片策略。


**（2）自定义分片策略**

实现JobShardingStrategy接口并实现sharding方法，接口方法参数为作业服务器IP列表和分片策略选项，分片策略选项包括作业名称，分片总数以及分片序列号和个性化参数对照表，可以根据需求定制化自己的分片策略。

	public interface JobShardingStrategy {
	
	    /**
	     * 作业分片.
	     * 
	     * @param jobInstances 所有参与分片的单元列表
	     * @param jobName 作业名称
	     * @param shardingTotalCount 分片总数
	     * @return 分片结果
	     */
	    Map<JobInstance, List<Integer>> sharding(List<JobInstance> jobInstances, String jobName, int shardingTotalCount);
	}


**（3）配置分片策略**

与配置通常的作业属性相同，在spring命名空间或者JobConfiguration中配置jobShardingStrategyClass属性，属性值是作业分片策略类的全路径。


......


#### 附录：参考链接 ####



- [http://tech.lede.com/2017/06/23/rd/server/elasticJob/](http://tech.lede.com/2017/06/23/rd/server/elasticJob/)
- [http://elasticjob.io/docs/elastic-job-lite/00-overview/](http://elasticjob.io/docs/elastic-job-lite/00-overview/)











































































































































