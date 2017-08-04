##Dubbo核心机制分析之Bean加载
//TODO:概述
***
###一、Spring可扩展Schema
在很多情况下，我们需要为系统提供可配置化支持，简单的做法可以直接基于Spring的标准Bean来配置，但配置较为复杂或者需要更多丰富控制的时候，会显得非常笨拙。
一般的做法会用原生态的方式去解析定义好的xml文件，然后转化为配置对象，这种方式当然可以解决所有问题，但实现起来比较繁琐，特别是是在配置非常复杂的时候，
解析工作是一个不得不考虑的负担。Spinrg2.5以后，spring支持自定义schema扩展xml配置，这是一个不错的折中方案，完成一个自定义配置一般需要以下步骤：

	* 设计配置属性和JavaBean 
	* 编写XSD文件 
	* 编写NamespaceHandler和BeanDefinitionParser完成解析工作 
	* 编写spring.handlers和spring.schemas串联起所有部件 
	* 在Bean文件中应用 
基于Spirng可扩展Schema提供自定义配置相关机制相信熟悉Spring的开发者都有所了解，这里不再解释具体细节。如果你对此机制不甚熟悉，可先去了解下相关内容，可参考：[http://www.cnblogs.com/jifeng/archive/2011/09/14/2176599.html](http://www.cnblogs.com/jifeng/archive/2011/09/14/2176599.html "基于Spring可扩展Schema提供自定义配置支持") (关于xsd:schema的各个属性具体含义可以参详W3C官方教程：[http://www.w3school.com.cn/schema/index.asp](http://www.w3school.com.cn/schema/index.asp "Schema教程"))。


###二、Dubbo Bean加载

![Dubbo源码结构示意图](http://i.imgur.com/RfPHvgV.png)