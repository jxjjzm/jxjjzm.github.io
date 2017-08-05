##Dubbo核心机制分析之服务的启动与初始化（Bean加载）
*如果你对Dubbo基础知识不甚了解,建议先TODO*
***
###一、Dubbo启动

![Dubbo启动](http://images2015.cnblogs.com/blog/120296/201603/120296-20160325170344386-827572634.png)



####1.Dubbo启动脚本

![启动脚本start.sh](http://i.imgur.com/MTnIhDh.png)

*从Dubbo启动脚本中不难发现，容器启动入口在com.alibaba.dubbo.container.Main类：*

####2.Dubbo启动入口

![Dubbo启动入口类](http://i.imgur.com/UUnCVrb.png)

*在启动类Main中，加载参数或者配置文件中的容器（container）,启动容器（默认启动log4j,spring）,其中Spring如下：*

####3.Dubbo容器启动（以SpringContainer为例）

![SpringContainer](http://i.imgur.com/csrF6pP.png)

*在SpringContainer中，容器在启动时会根据配置文件初始化Spring上下文，接下来就是基于Spirng可扩展Schema提供的自定义配置机制解析具体的Bean组件，具体的解析工作是在dubbo-config-spring 完成的。*

####小结
主线一：
*com.alibaba.dubbo.container.Main.main(args) -> dubbo.properties -> dubbo.container -> container.start() -> spring, log4j, jetty...*

主线二：
*[dubbo-container-spring] SpringContainer.java -> classpath*:META-INF/spring/*.xml -> dubbo.xsd*


###二、初始化与Bean 加载
####1、Spring可扩展Schema
*在很多情况下，我们需要为系统提供可配置化支持，简单的做法可以直接基于Spring的标准Bean来配置，但配置较为复杂或者需要更多丰富控制的时候，会显得非常笨拙。
一般的做法会用原生态的方式去解析定义好的xml文件，然后转化为配置对象，这种方式当然可以解决所有问题，但实现起来比较繁琐，特别是是在配置非常复杂的时候，
解析工作是一个不得不考虑的负担。Spinrg2.5以后，spring支持自定义schema扩展xml配置，这是一个不错的折中方案，完成一个自定义配置一般需要以下步骤：*

	* 设计配置属性和JavaBean 
	* 编写XSD文件 
	* 编写NamespaceHandler和BeanDefinitionParser完成解析工作 
	* 编写spring.handlers和spring.schemas串联起所有部件 
	* 在Bean文件中应用 
*基于Spirng可扩展Schema提供自定义配置相关机制相信熟悉Spring的开发者都有所了解，这里不再解释具体细节。如果你对此机制不甚熟悉，可先去了解下相关内容，可参考：[http://www.cnblogs.com/jifeng/archive/2011/09/14/2176599.html](http://www.cnblogs.com/jifeng/archive/2011/09/14/2176599.html "基于Spring可扩展Schema提供自定义配置支持") (关于xsd:schema的各个属性具体含义可以参详W3C官方教程：[http://www.w3school.com.cn/schema/index.asp](http://www.w3school.com.cn/schema/index.asp "Schema教程"))。*

####2、 Dubbo Bean初始化（加载）


*先来看一下dubbo-config-spring 的源码目录结构：*

![Dubbo源码结构示意图](http://i.imgur.com/RfPHvgV.png) 
 
*由于大部分项目都会使用Spring，Dubbo也提供了通过Spring来进行配置——Dubbo加载Spring的集成是在dubbo-config下面的dubbo-config-spring子模块（通过Spring可扩展Schema实现）。
在该模块的META-INF文件夹下有两个文件：spring.handlers和spring.schemas*
 
![Spring Schema File](http://i.imgur.com/tdx5ZqH.png)

*其中，spring.schemas指定了Dubbo namespace的XSD文件的位置，spring.handlers指定了Dubbo的namespace由DubboNamespaceHandler来处理解析。*

![DubboNameSpaceHandle](http://i.imgur.com/H1wi6wG.png)

*从这里也可以看到，Dubbo对应支持的标签其实不多，所有的Parser都封装到了DubboBeanDefinitionParser中。下面看看DubboBeanDefinitionParser里面做了什么事情:*

































































