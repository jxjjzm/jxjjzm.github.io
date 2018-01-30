### Spring源码篇之Spring IOC 核心实现 ###
***

### 一、整体脉络 ###

我和很多同学朋友都曾一起探讨过我们应该如何去更好地阅读相关框架的源码，我始终认为源码阅读是基于熟练使用并掌握其原理的基础上带着自己的问题去阅读，就拿Spring源码来说吧，Spring本身的代码800多M，笔者并不建议事无巨细地去阅读源码，因为这样很有可能在阅读源码的过程中会掉到各种实现细节中出不来，久而久之会失去耐性继续阅读下去。因此，个人建议阅读这种源码我们首先只需要从整体上去了解某个模块功能的实现原理，然后再在具体项目实战中碰到问题或者自己理解上遇到问题时带着具体问题去具体看，当然个人认为最好的方式还是模拟问题Demo一步一步Debug跟踪进去，这样可能更好理解。当然这些只是笔者自己的个人看法，你完全可以忽略它。


闲话不多说，我们进入正题，相信大家对于下面Spring的两种实例化方式已经熟悉得不能再熟悉了吧。

	public class SpringSourceCodeTest {
	    private static String splitLabel = ",";
	    public static void main(String[] args){
	
	        System.out.println("========================BeanFactory方式实例化Bean（过时，不推荐）==================");
	        BeanFactory bf = new XmlBeanFactory(new ClassPathResource("applicationContext.xml"));
	        People bfPeople = (People) bf.getBean("people");
	        System.out.println("id:"+bfPeople.getId()+splitLabel+"name:"+bfPeople.getName()+splitLabel+"memo:"+bfPeople.getMemo());
	        System.out.println("=========================ApplicationContext方式实例化Bean========================");
	        ApplicationContext ctx = new ClassPathXmlApplicationContext("applicationContext.xml");
	        People ctxPeople = (People) ctx.getBean("people");
	        System.out.println("id:"+ctxPeople.getId()+splitLabel+"name:"+ctxPeople.getName()+splitLabel+"memo:"+ctxPeople.getMemo());
	
	    }
	}


那么不知道你是否有过这样的思考，BeanFactory实例化与ApplicationContext实例化两种实例化方式之间究竟有何区别与联系？细心的你通过源码不难发现如下图所示其实ApplicationContext是间接的继承自BeanFactory，这也就是前面提到的说ApplicationContext是构建于BeanFactory之上的IoC容器，是相对比较高级的容器实现，除了拥有BeanFactory的所有支持，ApplicationContext还提供了其他高级特性，比如事件发布、国际化信息支持等。接下来让我们来具体来分析下这两种方式下Spring IOC容器是如何工作的？


![](https://www.ibm.com/developerworks/cn/java/j-lo-spring-principle/origin_image006.png)

#### 1.ApplicationContext实例化方式 ####


从以上Demo入手，我们很快定位到了ClassPathXmlApplicationContex类，看下ClassPathXmlApplicationContext的构造函数：

		public ClassPathXmlApplicationContext(String configLocation) throws BeansException {
				this(new String[] {configLocation}, true, null);
			}
		
		
		public ClassPathXmlApplicationContext(String[] configLocations, boolean refresh, ApplicationContext parent)
					throws BeansException {
		
				super(parent);
				setConfigLocations(configLocations);
				if (refresh) {
					refresh();
				}
			}
		

- super(parent) : 没什么太大看点，就是设置一下父级ApplicationContext，这里是null.
- setConfigLocations(configLocations) ： 这个方法就完成了两件事情 —— 1）将指定的Spring配置文件的路径存储到本地；2）解析Spring配置文件路径中的${PlaceHolder}占位符，替换为系统变量中PlaceHolder对应的Value值。
- refresh() ： 这个方法就是构建整个 Ioc 容器核心过程的完整的代码，它是ClassPathXmlApplicationContext的父类AbstractApplicationContext中的一个方法，顾名思义，用于刷新整个Spring上下文信息，定义了整个Spring上下文加载的流程。


正如前面所言，Spring Ioc 容器实际上就是 Context 组件结合其他两个组件（Bean和Core）共同构建的一个 Bean 关系网。那么我们脑袋中可能就会想Spring IOC是如何构建这个关系网？现在仿佛好象看到了曙光，构建的入口就在AbstractApplicationContext 类的 refresh 方法中。这个方法的代码如下：

		public void refresh() throws BeansException, IllegalStateException { 
		    synchronized (this.startupShutdownMonitor) { 
		        // Prepare this context for refreshing. 
		        prepareRefresh(); 
		        // Tell the subclass to refresh the internal bean factory. 
		        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory(); 
		        // Prepare the bean factory for use in this context. 
		        prepareBeanFactory(beanFactory); 
		        try { 
		            // Allows post-processing of the bean factory in context subclasses. 
		            postProcessBeanFactory(beanFactory); 
		            // Invoke factory processors registered as beans in the context. 
		            invokeBeanFactoryPostProcessors(beanFactory); 
		            // Register bean processors that intercept bean creation. 
		            registerBeanPostProcessors(beanFactory); 
		            // Initialize message source for this context. 
		            initMessageSource(); 
		            // Initialize event multicaster for this context. 
		            initApplicationEventMulticaster(); 
		            // Initialize other special beans in specific context subclasses. 
		            onRefresh(); 
		            // Check for listener beans and register them. 
		            registerListeners(); 
		            // Instantiate all remaining (non-lazy-init) singletons. 
		            finishBeanFactoryInitialization(beanFactory); 
		            // Last step: publish corresponding event. 
		            finishRefresh(); 
		        } 
		        catch (BeansException ex) { 
		            // Destroy already created singletons to avoid dangling resources. 
		            destroyBeans(); 
		            // Reset 'active' flag. 
		            cancelRefresh(ex); 
		            // Propagate exception to caller. 
		            throw ex; 
		        } 
		    } 
		}

AbstractApplicationContext 类的 refresh 方法是构建整个 Ioc 容器过程的完整的代码，了解了里面的每一行代码基本上就了解大部分 Spring 的原理和功能了。这段代码主要包含这样几个步骤：


- 构建 BeanFactory，以便于产生所需的“演员”
- 注册可能感兴趣的事件
- 创建 Bean 实例对象
- 触发被监听的事件

这里不再一一分析每个方法，我们重点来看下obtainFreshBeanFactory()方法，其作用是获取刷新Spring上下文的Bean工厂，代码如下：

		protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
				refreshBeanFactory();
				ConfigurableListableBeanFactory beanFactory = getBeanFactory();
				if (logger.isDebugEnabled()) {
					logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
				}
				return beanFactory;
			}

这段代码的核心在refreshBeanFactory()，这是一个抽象方法，有AbstractRefreshableApplicationContext和GenericApplicationContext这两个子类实现了这个方法，通过源码Debug或者根据ClassPathXmlApplicationContext继承关系图即知，调用的应当是AbstractRefreshableApplicationContext中实现的refreshBeanFactory，其源码为：

		protected final void refreshBeanFactory() throws BeansException {
				if (hasBeanFactory()) {
					destroyBeans();
					closeBeanFactory();
				}
				try {
					DefaultListableBeanFactory beanFactory = createBeanFactory();
					beanFactory.setSerializationId(getId());
					customizeBeanFactory(beanFactory);
					loadBeanDefinitions(beanFactory);
					synchronized (this.beanFactoryMonitor) {
						this.beanFactory = beanFactory;
					}
				}
				catch (IOException ex) {
					throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
				}
			}

这段代码清楚的说明了 BeanFactory 的创建过程，当 BeanFactory 已存在时就更新，如果没有就新创建。这段代码有两个核心之处值得我们关注，其一是DefaultListableBeanFactory这个类，这个类是构造Bean的核心类；其二是loadBeanDefinitions（beanFactory）这个方法，这个方法将开始加载、解析Bean的定义。


**(1)DefaultListableBeanFactory**

![](https://www.ibm.com/developerworks/cn/java/j-lo-spring-principle/origin_image009.png)

Spring Bean 的创建是典型的工厂模式，他的顶级接口是 BeanFactory 。BeanFactory 有三个子类：ListableBeanFactory、HierarchicalBeanFactory 和 AutowireCapableBeanFactory。但是通过分析源码我们可以发现最终的默认实现类是DefaultListableBeanFactory，他实现了所有的接口，是整个bean加载的核心部分。那为何要定义这么多层次的接口呢？查阅这些接口的源码和说明发现，每个接口都有他使用的场合，它主要是为了区分在 Spring 内部在操作过程中对象的传递和转化过程中，对对象的数据访问所做的限制。

- AliasRegistry:定义对alias的简单增删改等操作。
- SimpleAliasRegistry：主要使用map作为alias的缓存，并对接口AliasRegistry进行实现。
- SingletonBeanRegistry：定义对单例的注册及获取。
- BeanFactory：定义获取bean及bean的各种属性。
- DefaultSingletonBeanRegistry：对接口SingletonBeanRegistry各函数的实现。
- HierarchicalBeanFactory：继承BeanFactory，也就是在BeanFactory定义的功能的基础上增加了对parentFactory的支持（HierarchicalBeanFactory 表示的是这些 Bean 是有继承关系的，也就是每个 Bean 有可能有父 Bean）。
- ListableBeanFactory：继承BeanFactory，ListableBeanFactory 接口表示这些 Bean 是可列表的。
- AutowireCapableBeanFactory：继承BeanFactory，AutowireCapableBeanFactory 接口定义 Bean 的自动装配规则。
- BeanDefinitionRegistry：定义对BeanDefinition的各种增删改操作。
- FactoryBeanRegistrySupport：在DefaultSingletonBeanRegistry基础上增加了对FactoryBean的特殊处理功能。

**（2）loadBeanDefinitions（beanFactory）**


		protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
				// Create a new XmlBeanDefinitionReader for the given BeanFactory.
				XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);
		
				// Configure the bean definition reader with this context's
				// resource loading environment.
				beanDefinitionReader.setEnvironment(this.getEnvironment());
				beanDefinitionReader.setResourceLoader(this);
				beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));
		
				// Allow a subclass to provide custom initialization of the reader,
				// then proceed with actually loading the bean definitions.
				initBeanDefinitionReader(beanDefinitionReader);
				loadBeanDefinitions(beanDefinitionReader);
			}


辗转至此，我们才真正地切入正题，来到Spring整个资源加载的切入点（至于XmlBeanDefinitionReader）

![](https://www.ibm.com/developerworks/cn/java/j-lo-spring-principle/origin_image010.png)









### 一、配置文件封装 ###

问：Spring 是如何实现将不同来源的资源抽象成统一资源的访问方式？

在JAVA中，将不同来源的资源抽象成URL，通过注册不同的handler（URLStreamHandler）来处理不同来源的资源的读取逻辑，一般handler的类型使用不同前缀（协议，protocol）来识别，如"file:"、"http::"、"jar:" 等，然而URL没有默认定义相对Classpath或ServletContext等资源的handler,虽然可以注册自己的URLStreamHandler来解析特定的URL前缀（协议），比如"classpath:",然而这需要了解URL的实现机制，而且URL也没有提供一些基本方法，如检查当前资源是否存在、检查当前资源是否可读等方法。因而Spring对其内部使用到的资源实现了自己的抽象结构：Resource接口来封装底层资源。

	public interface InputStreamSource {
	    InputStream getInputStream() throws IOException;
	}
	
	public interface Resource extends InputStreamSource {
	    boolean exists();
	
	    boolean isReadable();
	
	    boolean isOpen();
	
	    URL getURL() throws IOException;
	
	    URI getURI() throws IOException;
	
	    File getFile() throws IOException;
	
	    long contentLength() throws IOException;
	
	    long lastModified() throws IOException;
	
	    Resource createRelative(String var1) throws IOException;
	
	    String getFilename();
	
	    String getDescription();
	}

Resource 接口封装了各种可能的资源类型，也就是对使用者来说屏蔽了文件类型的不同。对资源的提供者来说，如何把资源包装起来交给其他人用这也是一个问题，我们看到 Resource 接口继承了 InputStreamSource 接口，这个接口中有个 getInputStream 方法，返回的是 InputStream 类。这样所有的资源都被可以通过 InputStream 这个类来获取，所以也屏蔽了资源的提供者。Spring 对不同来源的资源文件都有相应的Resource实现：文件（FileSystemResource）、Classpath 资源（ClassPathResource）、URL资源（URLResource）、inputStream资源（InputStreamResource）、Byte数组（ByteArrayResource）等。此外，Resource接口还定义了一些资源相关的基本方法。

![](https://www.ibm.com/developerworks/cn/java/j-lo-spring-principle/origin_image007.png)



另外还有一个问题就是加载资源的问题，也就是资源的加载者要统一，从上图中可以看出这个任务是由 ResourceLoader 接口完成，他屏蔽了所有的资源加载者的差异，只需要实现这个接口就可以加载所有的资源，他的默认实现是 DefaultResourceLoader。


### 二、加载Bean ###

当通过Resource相关类完成了对配置文件进行封装后配置文件的读取工作就全权交给XmlBeanDefinitionReader来处理了。

![](https://www.ibm.com/developerworks/cn/java/j-lo-spring-principle/image005.png)























































































































