# Dubbo源码篇之服务的启动与初始化（Bean加载 #
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


![Dubbo源码结构示意图](http://i.imgur.com/RfPHvgV.png) 
 
#####（1）xml ------> BeanDefinition
*由于大部分项目都会使用Spring，Dubbo也提供了通过Spring来进行配置——Dubbo加载Spring的集成是在dubbo-config下面的dubbo-config-spring子模块（通过Spring可扩展Schema实现）。
在该模块的META-INF文件夹下有两个文件：spring.handlers和spring.schemas*
 
![Spring Schema File](http://i.imgur.com/tdx5ZqH.png)

*其中，spring.schemas指定了Dubbo namespace的XSD文件的位置，spring.handlers指定了Dubbo的namespace由DubboNamespaceHandler来处理解析。*
    
 
    public class DubboNamespaceHandler extends NamespaceHandlerSupport {
    	static {
    		Version.checkDuplicate(DubboNamespaceHandler.class);
    	}
    	public void init() {
    		//配置<dubbo:application>标签解析器
    		registerBeanDefinitionParser("application", new DubboBeanDefinitionParser(ApplicationConfig.class, true));
    		//配置<dubbo:module>标签解析器
    		registerBeanDefinitionParser("module", new DubboBeanDefinitionParser(ModuleConfig.class, true));
    		//配置<dubbo:registry>标签解析器
    		registerBeanDefinitionParser("registry", new DubboBeanDefinitionParser(RegistryConfig.class, true));
    		//配置<dubbo:monitor>标签解析器
    		registerBeanDefinitionParser("monitor", new DubboBeanDefinitionParser(MonitorConfig.class, true));
    		//配置<dubbo:provider>标签解析器
    		registerBeanDefinitionParser("provider", new DubboBeanDefinitionParser(ProviderConfig.class, true));
    		//配置<dubbo:consumer>标签解析器
    		registerBeanDefinitionParser("consumer", new DubboBeanDefinitionParser(ConsumerConfig.class, true));
    		//配置<dubbo:protocol>标签解析器
    		registerBeanDefinitionParser("protocol", new DubboBeanDefinitionParser(ProtocolConfig.class, true));
    		//配置<dubbo:service>标签解析器
    		registerBeanDefinitionParser("service", new DubboBeanDefinitionParser(ServiceBean.class, true));
    		//配置<dubbo:refenrence>标签解析器
    		registerBeanDefinitionParser("reference", new DubboBeanDefinitionParser(ReferenceBean.class, false));
    		//配置<dubbo:annotation>标签解析器
    		registerBeanDefinitionParser("annotation", new DubboBeanDefinitionParser(AnnotationBean.class, true));
    	}
    }
    



*从这里也可以看到，Dubbo对应支持的标签其实不多，所有的Parser都封装到了DubboBeanDefinitionParser中。下面看看DubboBeanDefinitionParser里面做了什么事情:*


    public class DubboBeanDefinitionParser implements BeanDefinitionParser {
    	private static final Logger logger = LoggerFactory.getLogger(DubboBeanDefinitionParser.class);
    
    	private final Class<?> beanClass;
    
    	private final boolean required;
    
    	public DubboBeanDefinitionParser(Class<?> beanClass, boolean required) {
    		this.beanClass = beanClass;
    		this.required = required;
    	}
    
    	public BeanDefinition parse(Element element, ParserContext parserContext) {
    		return parse(element, parserContext, beanClass, required);
    	}
    
    	@SuppressWarnings("unchecked")
    	private static BeanDefinition parse(Element element, ParserContext parserContext, Class<?> beanClass, boolean required) {
    		//初始化BeanDefiniion
    		RootBeanDefinition beanDefinition = new RootBeanDefinition();
    		beanDefinition.setBeanClass(beanClass);
    		beanDefinition.setLazyInit(false);
    		String id = element.getAttribute("id");
    		if ((id == null || id.length() == 0) && required) {
    			String generatedBeanName = element.getAttribute("name");
    			if (generatedBeanName == null || generatedBeanName.length() == 0) {
    				//如果当前解析的类型是ProtocolConfig，则设置默认id为dubbo
    				if (ProtocolConfig.class.equals(beanClass)) {   
    					generatedBeanName = "dubbo";
    				} else {
     					//其他情况，默认id为接口类型
    					generatedBeanName = element.getAttribute("interface");  
    				}
    			}
    			if (generatedBeanName == null || generatedBeanName.length() == 0) {
    				//如果该节点没有interface属性（包含：registry,monitor,provider,consumer），则使用该节点的类型为id值
    				generatedBeanName = beanClass.getName();   
    			}
    			id = generatedBeanName;
    			int counter = 2;
    			while(parserContext.getRegistry().containsBeanDefinition(id)) { 
    				//生成不重复的id
    				id = generatedBeanName + (counter ++);
    			}
    		}
    	if (id != null && id.length() > 0) {//目前这个判断不知道啥意义，目测必定会返回true
    		if (parserContext.getRegistry().containsBeanDefinition(id))  {
     			//这个判断应该用于防止并发
    			throw new IllegalStateException("Duplicate spring bean id " + id);
    		}
    		//注册beanDefinition，BeanDefinitionRegistry相当于一张注册表
    		parserContext.getRegistry().registerBeanDefinition(id, beanDefinition);
    		beanDefinition.getPropertyValues().addPropertyValue("id", id);
    	}
    	//下面这几个if-else分别针对不同类型做特殊处理
    	if (ProtocolConfig.class.equals(beanClass)) {
    		......
     	}else if(ServiceBean.class.equals(beanClass)) {
    		......
      	}else if(ProviderConfig.class.equals(beanClass)) {
    		......
      	}else if(ConsumerConfig.class.equals(beanClass)) {
    		......
      	}
		//标签属性解析（略）
    	......	
    	return beanDefinition;
    }

######小结：
*xml ------> <dubbo:service>等自定义标签 ------> [dubbo-config-spring] DubboNamespaceHandler.java
 ------> new DubboBeanDefinitionParser(ServiceBean.class, true) ------> BeanDefinition*

*现在，我们的dubbo就已经把配置文件中定义的bean全部解析成对应的beanDefinition，为spring的getBean做好准备工作。*


#####（2）BeanDefinition ------> Bean

*从BeanDefinition转换成Bean的过程，Dubbo并没有做额外的处理，依赖的还是Spring初始化Bean机制，相信读过Spirng源码对这块都不会太陌生(不熟悉可参考[https://www.ibm.com/developerworks/cn/java/j-lo-spring-principle/](https://www.ibm.com/developerworks/cn/java/j-lo-spring-principle/ "Spring 框架的设计理念与设计模式分析"))，下面引用网上的一幅图来辅助我们了解spring内部是如何初始化bean的：*

![](http://www.ibm.com/developerworks/cn/java/j-lo-spring-principle/origin_image012.gif)

*到了这里，相信我们应该对Dubbo整个服务启动以及Bean初始化相关内容有了个大概的了解，如果想要进一步了解具体实现细节，还是推荐阅读相关源码，我始终坚信源码是最好的开发文档，小编接下来将在下几个博文当中具体介绍Dubbo服务暴露、注册、订阅等相关机制，欢迎垂阅并不吝赐教。*























































