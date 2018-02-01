### Spring源码篇之Spring IOC 核心实现 ###
***

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

### 一、Spring Bean的定义、解析与注册 ###


#### 1.ApplicationContext实例化方式 ####

![](https://www.ibm.com/developerworks/cn/java/j-lo-spring-principle/origin_image006.png)



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
		        // 准备刷新的上下文环境 
		        prepareRefresh(); 
		        // Tell the subclass to refresh the internal bean factory. 
				//初始化BeanFactory，并进行XML文件的读取
		        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory(); 
		        // Prepare the bean factory for use in this context. 
				//对BeanFactory进行各种功能填充
		        prepareBeanFactory(beanFactory); 
		        try { 
		            // Allows post-processing of the bean factory in context subclasses. 
					//子类覆盖方法做额外的处理
		            postProcessBeanFactory(beanFactory); 
		            // Invoke factory processors registered as beans in the context. 
					//激活各种BeanFactory处理器
		            invokeBeanFactoryPostProcessors(beanFactory); 
		            // Register bean processors that intercept bean creation.
					//注册拦截Bean创建的Bean处理器，这里只是注册，真正的调用是在getBean时候 
		            registerBeanPostProcessors(beanFactory); 
		            // Initialize message source for this context. 
					//为上下文初始化Message源，即不同语言的消息体，国际化处理
		            initMessageSource(); 
		            // Initialize event multicaster for this context. 
					//初始化应用消息广播器，并放入“applicationEventMulticaster” bean中 
		            initApplicationEventMulticaster(); 
		            // Initialize other special beans in specific context subclasses. 
					//留给子类来初始化其它的Bean
		            onRefresh(); 
		            // Check for listener beans and register them. 
					//在所有注册的bean中查找Listener bean，注册到消息广播器中
		            registerListeners(); 
		            // Instantiate all remaining (non-lazy-init) singletons. 
					//初始化剩下的单实例（非惰性的）
		            finishBeanFactoryInitialization(beanFactory); 
		            // Last step: publish corresponding event. 
					//完成刷新过程，通知生命周期处理器LifecycleProcessor刷新过程，同时发出ContextRefreshEvent通知别人
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


- （1）初始化前的准备工作，例如对系统属性或者环境变量进行准备及验证。
- （2）初始化BeanFactory，并进行XML文件读取。之前有提到ClassPathXmlApplicationContext包含着BeanFactory所提供的一切特征，那么这一步骤将会复用BeanFactory中的配置文件读取解析及其他功能，这一步之后，ClasspathXmlApplicatinoContext实际上就已经包含了BeanFactory所提供的功能，也就是可以进行bean的提取等基础操作了。
- （3）对BeanFactory进行各种功能填充。@Qualifier与@Autowired这两个注解正是在这一步骤中增加的支持。
- （4）子类覆盖方法做额外的处理。Spring之所以强大为世人所推崇，除了它功能上为大家提供了便利外，还有一方面是它的完美架构，开放式的架构让使用它的程序员很容易根据业务需要扩展已经存在的功能，这种开放式的设计在Spring中随处可见。
- （5）激活各种BeanFactory处理器
- （6）注册拦截bean创建的bean处理器，这里只是注册，真正的调用是在getBean时候。
- （7）为上下文初始化Message源，即对不同语言的消息体进行国际化处理。
- （8）初始化应用消息广播器，并放入"applicationEventMulticaster" bean中。
- （9）留给子类来初始化其他的bean 。
- （10）在所有注册的bean中查找Listener bean,注册到消息广播器中。
- （11）初始化剩下的单实例（非惰性的）。
- （12）完成刷新过程，通知生命周期处理器lifecycleProcessor刷新过程，同时发出ContextRefreshEvent通知别人。




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
		// Load bean definitions into the given bean factory, typically through delegating to one or more bean definition readers.
		protected abstract void loadBeanDefinitions(DefaultListableBeanFactory beanFactory)
					throws BeansException, IOException;

这段代码清楚的说明了 BeanFactory 的创建过程，当 BeanFactory 已存在时就更新，如果没有就新创建。这段代码有两个核心之处值得我们关注，其一是DefaultListableBeanFactory这个类，这个类是构造Bean的核心类；其二是loadBeanDefinitions（beanFactory）这个方法，这个方法将开始加载、解析Bean的定义，也就是把用户定义的数据结构转化为 Ioc 容器中的特定数据结构。AbstractRefreshableApplicationContext中的loadBeanDefinitions（beanFactory）是一个抽象方法，其具体实现由其子类AbstractXmlApplicationContext来完成。


**(1)DefaultListableBeanFactory**

![](https://www.ibm.com/developerworks/cn/java/j-lo-spring-principle/origin_image009.png)

Spring Bean 的创建是典型的工厂模式，他的顶级接口是 BeanFactory 。BeanFactory 有三个子类：ListableBeanFactory、HierarchicalBeanFactory 和 AutowireCapableBeanFactory。但是通过分析源码我们可以发现最终的默认实现类是DefaultListableBeanFactory，他实现了所有的接口，是整个bean加载的核心部分。那为何要定义这么多层次的接口呢？查阅这些接口的源码和说明发现，每个接口都有他使用的场合，它主要是为了区分在 Spring 内部在操作过程中对象的传递和转化过程中，对对象的数据访问所做的限制。

- AliasRegistry:定义对alias的简单增删改等操作。
- SimpleAliasRegistry：主要使用map作为alias的缓存，并对接口AliasRegistry进行实现。
- SingletonBeanRegistry：定义对单例的注册及获取。
- BeanFactory：定义获取bean及bean的各种属性。
- DefaultSingletonBeanRegistry：对接口SingletonBeanRegistry各函数的实现。
- HierarchicalBeanFactory：继承BeanFactory，也就是在BeanFactory定义的功能的基础上增加了对parentFactory的支持（HierarchicalBeanFactory 表示的是这些 Bean 是有继承关系的，也就是每个 Bean 有可能有父 Bean）。
- BeanDefinitionRegistry：定义对BeanDefinition的各种增删改操作。
- FactoryBeanRegistrySupport：在DefaultSingletonBeanRegistry基础上增加了对FactoryBean的特殊处理功能。
- ConfigurableBeanFactory：提供配置Factory的各种方法。
- ListableBeanFactory：继承BeanFactory，ListableBeanFactory 接口表示这些 Bean 是可列表的。（提供各种条件获取bean的配置清单）
- AbstractBeanFactory：综合FactoryBeanRegistrySupport和ConfigurableBeanFactory的功能。
- AutowireCapableBeanFactory：继承BeanFactory，AutowireCapableBeanFactory 接口定义 Bean 的自动装配规则。（提供创建bean、自动注入、初始化以及应用bean的后处理器）。
- AbstractAutowireCapableBeanFactory：综合AbstractBeanFactory并对接口AutowireCapableBeanFactory进行实现。
- ConfigurableListableBeanFactory：BeanFactory配置清单，指定忽略类型及接口等。
- DefaultListableBeanFactory：综合上面所有功能，主要用于从XML文档中读取BeanDefiniton，对于注册及获取Bean都是使用从父类继承的方法去实现，而唯独与父类不同的个性化实现就是增加了XmlBeanDefinitionReader类型的reader属性。在XmlBeanFactory中主要使用reader属性对资源文件进行读取和注册。


**（2）loadBeanDefinitions（beanFactory）**

AbstractXmlApplicationContext类中的loadBeanDefinitions(beanFactory)方法如下：

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

		//Load the bean definitions with the given XmlBeanDefinitionReader.
		protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
				Resource[] configResources = getConfigResources();
				if (configResources != null) {
					reader.loadBeanDefinitions(configResources);
				}
				String[] configLocations = getConfigLocations();
				if (configLocations != null) {
					reader.loadBeanDefinitions(configLocations);
				}
			}


接下来继续跟踪至AbstractBeanDefinitionReader类中的loadBeanDefinitions(String... locations)：

		public int loadBeanDefinitions(String... locations) throws BeanDefinitionStoreException {
				Assert.notNull(locations, "Location array must not be null");
				int counter = 0;
				for (String location : locations) {
					counter += loadBeanDefinitions(location);
				}
				return counter;
			}

		public int loadBeanDefinitions(String location) throws BeanDefinitionStoreException {
				return loadBeanDefinitions(location, null);
			}


		public int loadBeanDefinitions(String location, Set<Resource> actualResources) throws BeanDefinitionStoreException {
				ResourceLoader resourceLoader = getResourceLoader();
				if (resourceLoader == null) {
					throw new BeanDefinitionStoreException(
							"Cannot import bean definitions from location [" + location + "]: no ResourceLoader available");
				}
		
				if (resourceLoader instanceof ResourcePatternResolver) {
					// Resource pattern matching available.
					try {
						Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location);
						int loadCount = loadBeanDefinitions(resources);
						if (actualResources != null) {
							for (Resource resource : resources) {
								actualResources.add(resource);
							}
						}
						if (logger.isDebugEnabled()) {
							logger.debug("Loaded " + loadCount + " bean definitions from location pattern [" + location + "]");
						}
						return loadCount;
					}
					catch (IOException ex) {
						throw new BeanDefinitionStoreException(
								"Could not resolve bean definition resource pattern [" + location + "]", ex);
					}
				}
				else {
					// Can only load single resources by absolute URL.
					Resource resource = resourceLoader.getResource(location);
					int loadCount = loadBeanDefinitions(resource);
					if (actualResources != null) {
						actualResources.add(resource);
					}
					if (logger.isDebugEnabled()) {
						logger.debug("Loaded " + loadCount + " bean definitions from location [" + location + "]");
					}
					return loadCount;
				}
			}


		public int loadBeanDefinitions(Resource... resources) throws BeanDefinitionStoreException {
				Assert.notNull(resources, "Resource array must not be null");
				int counter = 0;
				for (Resource resource : resources) {
					counter += loadBeanDefinitions(resource);
				}
				return counter;
			}


不要气馁，看源码最重要的就是耐心，继续看BeanDefinitionReader接口中的loadBeanDefinitions：

		//Load bean definitions from the specified resource locations.
		int loadBeanDefinitions(String... locations) throws BeanDefinitionStoreException;


我们终于知道什么叫山路十八弯，似乎绕了这么半天才真正地切入到正题，最终殊途同归 ———— XmlBeanDefinitionReadere类中的loadBeanDefinitions(Resource resource) 。

![](https://www.ibm.com/developerworks/cn/java/j-lo-spring-principle/image005.png)


		public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
				return loadBeanDefinitions(new EncodedResource(resource));
			}
		
		
		public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
				Assert.notNull(encodedResource, "EncodedResource must not be null");
				if (logger.isInfoEnabled()) {
					logger.info("Loading XML bean definitions from " + encodedResource.getResource());
				}
		
				Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
				if (currentResources == null) {
					currentResources = new HashSet<EncodedResource>(4);
					this.resourcesCurrentlyBeingLoaded.set(currentResources);
				}
				if (!currentResources.add(encodedResource)) {
					throw new BeanDefinitionStoreException(
							"Detected cyclic loading of " + encodedResource + " - check your import definitions!");
				}
				try {
					InputStream inputStream = encodedResource.getResource().getInputStream();
					try {
						InputSource inputSource = new InputSource(inputStream);
						if (encodedResource.getEncoding() != null) {
							inputSource.setEncoding(encodedResource.getEncoding());
						}
						return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
					}
					finally {
						inputStream.close();
					}
				}
				catch (IOException ex) {
					throw new BeanDefinitionStoreException(
							"IOException parsing XML document from " + encodedResource.getResource(), ex);
				}
				finally {
					currentResources.remove(encodedResource);
					if (currentResources.isEmpty()) {
						this.resourcesCurrentlyBeingLoaded.remove();
					}
				}
			}

我们尝试梳理下这段代码整个处理过程：

- 封装资源文件。当进入XmlBeanDefinitionReader后首先对参数Resource使用EncodeResource类进行封装,主要目的是考虑到Resource可能存在编码要求的情况，用于对资源文件的编码进行处理。
- 获取输入流。从Resource中获取对应的InputStream并构造InputSource。
- 通过构造的InputSource实例和Resource实例继续调用函数doLoadBeanDefinitions(inputSource, encodedResource.getResource());

doLoadBeanDefinitions方法如下：


		//Actually load bean definitions from the specified XML file.
		protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
					throws BeanDefinitionStoreException {
				try {
					Document doc = doLoadDocument(inputSource, resource);
					return registerBeanDefinitions(doc, resource);
				}
				catch (BeanDefinitionStoreException ex) {
					throw ex;
				}
				catch (SAXParseException ex) {
					throw new XmlBeanDefinitionStoreException(resource.getDescription(),
							"Line " + ex.getLineNumber() + " in XML document from " + resource + " is invalid", ex);
				}
				catch (SAXException ex) {
					throw new XmlBeanDefinitionStoreException(resource.getDescription(),
							"XML document from " + resource + " is invalid", ex);
				}
				catch (ParserConfigurationException ex) {
					throw new BeanDefinitionStoreException(resource.getDescription(),
							"Parser configuration exception parsing XML from " + resource, ex);
				}
				catch (IOException ex) {
					throw new BeanDefinitionStoreException(resource.getDescription(),
							"IOException parsing XML document from " + resource, ex);
				}
				catch (Throwable ex) {
					throw new BeanDefinitionStoreException(resource.getDescription(),
							"Unexpected exception parsing XML document from " + resource, ex);
				}
			}
		
		
		//Actually load the specified document using the configured DocumentLoader.
		protected Document doLoadDocument(InputSource inputSource, Resource resource) throws Exception {
				return this.documentLoader.loadDocument(inputSource, getEntityResolver(), this.errorHandler,
						getValidationModeForResource(resource), isNamespaceAware());
			}


我们再次整理一下数据准备阶段的逻辑，首先对传入的resource参数做封装，主要目的是考虑到Resource可能存在编码要求的情况，用于对资源文件的编码进行处理。其次，通过SAX读取XML文件的方式来准备InputSource对象，最后将准备的数据通过参数传入真正的核心处理部分doLoadBeanDefinitions,doLoadBeanDefinitions上面冗长的代码中如不考虑异常类的代码，其实只做了三件事，这三件事的每一件都必不可少。

- 获取对XML文件的验证模式。（DTD和XSD）
- 加载XML文件，并得到对应的Document。（XmlReaderFactoryReader类对于文档读取并没有亲力亲为，而是委托给了DocumentLoader去执行，这里的DocumentReader是个接口，而真正调用的是DefaultDocumentLoader，对于这部分并没有太多可以描述的，因为通过SAX解析XML文档的套路大致都差不多，Spring在这里并没有什么特殊的地方，同样首先创建DocumentBuiderFactory，再通过DocumentBuiderFactory创建DocumentBuider，进而解析inputSource来返回Document对象。）
- 根据返回的Document注册Bean信息。

当把文件转换为Document后，接下来的提取及注册bean就是我们的重头戏。

		//Register the bean definitions contained in the given DOM document.
		public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
				BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
				documentReader.setEnvironment(getEnvironment());
				int countBefore = getRegistry().getBeanDefinitionCount();
				documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
				return getRegistry().getBeanDefinitionCount() - countBefore;
			}

BeanDefinitionDocumentReader是一个接口，而实例化的工作是在createBeanDefinitionDocumentReader（）方法中完成的，而通过此方法，BeanDefinitionDocumentReader真正的类型其实已经是DefaultBeanDefinitionDocumentReader了。

		//This implementation parses bean definitions according to the "spring-beans" XSD
		public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
				this.readerContext = readerContext;
				logger.debug("Loading bean definitions");
				Element root = doc.getDocumentElement();
				doRegisterBeanDefinitions(root);
			}


		/**
		 * Register each bean definition within the given root {@code <beans/>} element.
		 */
		protected void doRegisterBeanDefinitions(Element root) {
			// Any nested <beans> elements will cause recursion in this method. In
			// order to propagate and preserve <beans> default-* attributes correctly,
			// keep track of the current (parent) delegate, which may be null. Create
			// the new (child) delegate with a reference to the parent for fallback purposes,
			// then ultimately reset this.delegate back to its original (parent) reference.
			// this behavior emulates a stack of delegates without actually necessitating one.
			BeanDefinitionParserDelegate parent = this.delegate;
			this.delegate = createDelegate(getReaderContext(), root, parent);
	
			if (this.delegate.isDefaultNamespace(root)) {
				String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
				if (StringUtils.hasText(profileSpec)) {
					String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
							profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
					if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
						return;
					}
				}
			}
	
			preProcessXml(root);
			parseBeanDefinitions(root, this.delegate);
			postProcessXml(root);
	
			this.delegate = parent;
		}


进入DefaultBeanDefinitionDocumentReader后，发现这个发放的重要目的之一就是提取Root，以便于再次将root作为参数继续BeanDefinition的注册。经过艰难险阻，磕磕绊绊，我们终于到了核心逻辑的底部 doRegisterBeanDefinitions(root),至少我们在这个方法中看到了希望，如果说以前一直是XML加载解析的准备，那么doRegisterBeanDefinitions(root)算是真正地开始进行解析了。别急，在进行解析之前请允许笔者插播几条说明：

- 通过上面的代码我们看到了处理流程，首先是对profile的处理，然后进行解析，可是当我们跟进preProcessXml(root)或者postProcessXml(root)发现代码是空的，既然是空的写着还有什么用呢？就像面向对象设计方法学中常说的一句话，一个类要么是面向继承而设计的，要么就用final修饰。在DefaultBeanDefinitionDocumentReader中并没有用final修饰，所以它是面向继承而设计的。这两个方法正是为子类而设计的以便于扩展，如果读者有了解过设计模式，可以很快速地反映出这是模板方法模式，如果继承自DefaultBeanDefinitionDocumentReader的子类需要在Bean解析前后做一些处理的话，那么只需要重写这两个方法就可以了。
- Profile属性的使用可以让我们同时在配置文件中部署两套配置来适用于生产环境和开发环境，这样可以方便的进行切换开发、部署环境，最常用的就是更换不同的数据库。


处理了profile后就可以进行XML的读取了，跟踪代码进入parseBeanDefinitions（root,this.delegate）。

		/**
			 * Parse the elements at the root level in the document:
			 * "import", "alias", "bean".
			 * @param root the DOM root element of the document
			 */
			protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
				if (delegate.isDefaultNamespace(root)) {
					NodeList nl = root.getChildNodes();
					for (int i = 0; i < nl.getLength(); i++) {
						Node node = nl.item(i);
						if (node instanceof Element) {
							Element ele = (Element) node;
							if (delegate.isDefaultNamespace(ele)) {
								parseDefaultElement(ele, delegate);
							}
							else {
								delegate.parseCustomElement(ele);
							}
						}
					}
				}
				else {
					delegate.parseCustomElement(root);
				}
			}
		
			private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
				if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
					importBeanDefinitionResource(ele);
				}
				else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
					processAliasRegistration(ele);
				}
				else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
					processBeanDefinition(ele, delegate);
				}
				else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
					// recurse
					doRegisterBeanDefinitions(ele);
				}
			}


我们知道Spring中的标签包括默认标签和自定义标签两种，而这两种标签的用法以及解析方式存在着很大的不同。（判断是否默认命名空间还是自定义命名空间的办法其实是使用node.getNameSpaceURI（）获取命名空间）

- 默认标签的解析

默认标签的解析是在parseDefaultElement函数中进行的，函数中的功能逻辑一目了然，分别对4种不同标签（import、alias、bean、和beans）做了不同的处理。


- 自定义标签的解析

Spring自定义标签的使用相信请参考：[基于Spring可扩展Schema提供自定义配置支持(spring配置文件中 配置标签支持)](http://www.cnblogs.com/jifeng/archive/2011/09/14/2176599.html "基于Spring可扩展Schema提供自定义配置支持(spring配置文件中 配置标签支持)")

基于Spring可扩展Schema提供自定义配置在Dubbo中得到广泛使用，了解了自定义标签的使用后，我们带着强烈的好奇心来探究一下自定义标签的解析过程。

		public BeanDefinition parseCustomElement(Element ele, BeanDefinition containingBd) {
				String namespaceUri = getNamespaceURI(ele);
				NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
				if (handler == null) {
					error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", ele);
					return null;
				}
				return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
			}

自定义标签的解析过程其实思路非常的简单，无非是根据对应的bean获取对应的命名空间，根据命名空间解析对应的处理器，然后根据用户自定义的处理器进行解析。

**阶段性总结：**

![](https://www.ibm.com/developerworks/cn/java/j-lo-spring-principle/origin_image010.png)

![](https://www.ibm.com/developerworks/cn/java/j-lo-spring-principle/origin_image011.png)


前面已经提到AbstractApplicationContext 类的 refresh 方法是构建整个 Ioc 容器过程的完整的代码，了解了里面的每一行代码基本上就了解大部分 Spring 的原理和功能了。我们回过头再来看下这段代码，可以发现不少容器的功能扩展，由于篇幅问题考虑择期另起炉灶，这里就此打住。


**2.BeanFactory实例化方式**

		BeanFactory bf = new XmlBeanFactory(new ClassPathResource("applicationContext.xml"));

同样，我们从前面的Demo入手，我们先来看下XmlBeanFactory，XmlBeanFactory继承自DefaultListableBeanFactory。


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

**（2）加载Bean**

当通过Resource相关类完成了对配置文件封装后配置文件的读取工作就全权交给了XmlBeanDefinitionReader来处理了。（XmlBeanDefinitionReader具体加载解析流程同上面一致，这里不再重复赘述。）

![](https://www.ibm.com/developerworks/cn/java/j-lo-spring-principle/image005.png)


### 二、Spring Bean的实例创建 ###

经过前面的分析，我们终于结束了对XML配置文件的解析，接下来将会面临更大的挑战，那就是对Bean实例创建的探索。

#### 1.BeanFactory方式 ####

		People bfPeople = (People) bf.getBean("people");

从上面Demo跟踪我们发现Bean实例的创建是在BeanFactory中发生的：

		public Object getBean(String name) throws BeansException {
				return doGetBean(name, null, null, false);
			}
		
		@SuppressWarnings("unchecked")
			protected <T> T doGetBean(
					final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
					throws BeansException {
		
				final String beanName = transformedBeanName(name);
				Object bean;
		
				// Eagerly check singleton cache for manually registered singletons.
				Object sharedInstance = getSingleton(beanName);
				if (sharedInstance != null && args == null) {
					if (logger.isDebugEnabled()) {
						if (isSingletonCurrentlyInCreation(beanName)) {
							logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
									"' that is not fully initialized yet - a consequence of a circular reference");
						}
						else {
							logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
						}
					}
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
				}
		
				else {
					// Fail if we're already creating this bean instance:
					// We're assumably within a circular reference.
					if (isPrototypeCurrentlyInCreation(beanName)) {
						throw new BeanCurrentlyInCreationException(beanName);
					}
		
					// Check if bean definition exists in this factory.
					BeanFactory parentBeanFactory = getParentBeanFactory();
					if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
						// Not found -> check parent.
						String nameToLookup = originalBeanName(name);
						if (args != null) {
							// Delegation to parent with explicit args.
							return (T) parentBeanFactory.getBean(nameToLookup, args);
						}
						else {
							// No args -> delegate to standard getBean method.
							return parentBeanFactory.getBean(nameToLookup, requiredType);
						}
					}
		
					if (!typeCheckOnly) {
						markBeanAsCreated(beanName);
					}
		
					try {
						final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
						checkMergedBeanDefinition(mbd, beanName, args);
		
						// Guarantee initialization of beans that the current bean depends on.
						String[] dependsOn = mbd.getDependsOn();
						if (dependsOn != null) {
							for (String dependsOnBean : dependsOn) {
								if (isDependent(beanName, dependsOnBean)) {
									throw new BeanCreationException(mbd.getResourceDescription(), beanName,
											"Circular depends-on relationship between '" + beanName + "' and '" + dependsOnBean + "'");
								}
								registerDependentBean(dependsOnBean, beanName);
								getBean(dependsOnBean);
							}
						}
		
						// Create bean instance.
						if (mbd.isSingleton()) {
							sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
								@Override
								public Object getObject() throws BeansException {
									try {
										return createBean(beanName, mbd, args);
									}
									catch (BeansException ex) {
										// Explicitly remove instance from singleton cache: It might have been put there
										// eagerly by the creation process, to allow for circular reference resolution.
										// Also remove any beans that received a temporary reference to the bean.
										destroySingleton(beanName);
										throw ex;
									}
								}
							});
							bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
						}
		
						else if (mbd.isPrototype()) {
							// It's a prototype -> create a new instance.
							Object prototypeInstance = null;
							try {
								beforePrototypeCreation(beanName);
								prototypeInstance = createBean(beanName, mbd, args);
							}
							finally {
								afterPrototypeCreation(beanName);
							}
							bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
						}
		
						else {
							String scopeName = mbd.getScope();
							final Scope scope = this.scopes.get(scopeName);
							if (scope == null) {
								throw new IllegalStateException("No Scope registered for scope '" + scopeName + "'");
							}
							try {
								Object scopedInstance = scope.get(beanName, new ObjectFactory<Object>() {
									@Override
									public Object getObject() throws BeansException {
										beforePrototypeCreation(beanName);
										try {
											return createBean(beanName, mbd, args);
										}
										finally {
											afterPrototypeCreation(beanName);
										}
									}
								});
								bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
							}
							catch (IllegalStateException ex) {
								throw new BeanCreationException(beanName,
										"Scope '" + scopeName + "' is not active for the current thread; " +
										"consider defining a scoped proxy for this bean if you intend to refer to it from a singleton",
										ex);
							}
						}
					}
					catch (BeansException ex) {
						cleanupAfterBeanCreationFailure(beanName);
						throw ex;
					}
				}
		
				// Check if required type matches the type of the actual bean instance.
				if (requiredType != null && bean != null && !requiredType.isAssignableFrom(bean.getClass())) {
					try {
						return getTypeConverter().convertIfNecessary(bean, requiredType);
					}
					catch (TypeMismatchException ex) {
						if (logger.isDebugEnabled()) {
							logger.debug("Failed to convert bean '" + name + "' to required type [" +
									ClassUtils.getQualifiedName(requiredType) + "]", ex);
						}
						throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
					}
				}
				return (T) bean;
			}


- （1）转化对应的beanName。

或许很多人不理解转换对应beanName是什么意思，传入的参数name不就是beanName吗？其实不是，这里传入的参数可能是别名，也可能是FactoryBean（去除factoryBean），所以需要进行一系列的解析.

		protected String transformedBeanName(String name) {
				return canonicalName(BeanFactoryUtils.transformedBeanName(name));
			}
		
		public static String transformedBeanName(String name) {
				Assert.notNull(name, "'name' must not be null");
				String beanName = name;
				while (beanName.startsWith(BeanFactory.FACTORY_BEAN_PREFIX)) {
					beanName = beanName.substring(BeanFactory.FACTORY_BEAN_PREFIX.length());
				}
				return beanName;
			}
		
		public String canonicalName(String name) {
		        String canonicalName = name;
		
		        String resolvedName;
		        do {
		            resolvedName = (String)this.aliasMap.get(canonicalName);
		            if (resolvedName != null) {
		                canonicalName = resolvedName;
		            }
		        } while(resolvedName != null);
		
		        return canonicalName;
		    }


- （2）尝试从缓存中获取单例bean

单例在Spring的用一个容器内只会被创建一次，后续再获取bean，就直接从单例缓存中获取了。当然这里也只是尝试加载，首先尝试从缓存中加载，如果加载不成功则再次尝试从singletonFactories中加载。因为在创建单例bean的时候会存在依赖注入的情况，而在创建依赖的时候为了避免循环依赖，在Spring中创建bean的原则是不等bean创建完成就会将创建bean的ObjectFactory提前曝光加入到缓存中，一旦下一个bean创建时候需要依赖上一个bean则直接使用ObjectFactory。

		public Object getSingleton(String beanName) {
				return getSingleton(beanName, true);
			}
		
			/**
			 * Return the (raw) singleton object registered under the given name.
			 * @return the registered singleton object, or {@code null} if none found
			 */
			protected Object getSingleton(String beanName, boolean allowEarlyReference) {
				Object singletonObject = this.singletonObjects.get(beanName);
				if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
					synchronized (this.singletonObjects) {
						singletonObject = this.earlySingletonObjects.get(beanName);
						if (singletonObject == null && allowEarlyReference) {
							ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
							if (singletonFactory != null) {
								singletonObject = singletonFactory.getObject();
								this.earlySingletonObjects.put(beanName, singletonObject);
								this.singletonFactories.remove(beanName);
							}
						}
					}
				}
				return (singletonObject != NULL_OBJECT ? singletonObject : null);
			}

这个方法首先尝试从singletonObjects里面获取实例，如果获取不到再从earlySingletonObjects里面获取，如果还获取不到再尝试从singletonFactories里面获取beanName对应的ObjectFactory，然后调用这个ObjectFactory的getObject来创建bean，并放到earlySingletonObjects里面去，并且从singletonFactories里面remove掉这个ObjectFactory，而对于后面的所有内存操作都只为了循环依赖检测时候使用，也就是在allowEarlyReference为true的情况下才会使用。



- （3）从bean的实例中获取对象

无论是从缓存中获取到的bean还是通过不同的scope策略加载的bean都只是最原始的bean状态，并不一定是我们最终想要的bean。举个例子，假如我们需要对工厂bean进行处理，那么这里得到的其实就是工厂bean的初始状态，但是我们真正需要的是工厂bean中定义的factory-method方法中返回的bean,而getObjectForBeanInstance方法就是完成这个工作的。

		/**
			 * Get the object for the given bean instance, either the bean
			 * instance itself or its created object in case of a FactoryBean.
			 * @return the object to expose for the bean
			 */
			protected Object getObjectForBeanInstance(
					Object beanInstance, String name, String beanName, RootBeanDefinition mbd) {
		
				// Don't let calling code try to dereference the factory if the bean isn't a factory.
				if (BeanFactoryUtils.isFactoryDereference(name) && !(beanInstance instanceof FactoryBean)) {
					throw new BeanIsNotAFactoryException(transformedBeanName(name), beanInstance.getClass());
				}
		
				// Now we have the bean instance, which may be a normal bean or a FactoryBean.
				// If it's a FactoryBean, we use it to create a bean instance, unless the
				// caller actually wants a reference to the factory.
				if (!(beanInstance instanceof FactoryBean) || BeanFactoryUtils.isFactoryDereference(name)) {
					return beanInstance;
				}
		
				Object object = null;
				if (mbd == null) {
					object = getCachedObjectForFactoryBean(beanName);
				}
				if (object == null) {
					// Return bean instance from factory.
					FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
					// Caches object obtained from FactoryBean if it is a singleton.
					if (mbd == null && containsBeanDefinition(beanName)) {
						mbd = getMergedLocalBeanDefinition(beanName);
					}
					boolean synthetic = (mbd != null && mbd.isSynthetic());
					object = getObjectFromFactoryBean(factory, beanName, !synthetic);
				}
				return object;
			}


在getBean方法中，getObjectForBeanInstance是个高频率使用的方法，无论是从缓存中获得bean还是根据不同的scope策略加载bean。总之，我们得到bean的实例后要做的第一步就是调用这个方法来检测一下正确性，其实就是用于检测当前bean是否是FactoryBean类型的bean,如果是，那么需要调用该bean对应的FactoryBean实例中的getObject()作为返回值。



- （4）原型模式的依赖检查

只有在单例模式下才会尝试解决循环依赖，如果窜在A中有B的属性，B中有A的属性，那么当依赖注入的时候就会产生当A还未创建完的时候因为对于B的创建再次返回创建A，造成循环依赖。也就是情况：isPrototypeCurrentInCreation(beanName)判断为true。

这里有必要普及下Spring循环依赖问题。Spring容器循环依赖包括构造器循环依赖和setter循环依赖那么Spring容器如何解决循环依赖呢？在Spring中将循环依赖的处理分成了3种情况。第一种，构造器循环依赖，次依赖是无法解决的，只能抛出BeanCurrentlyIncreationException异常表示循环依赖；第二种，setter循环依赖表示通过setter注入方式构成的循环依赖是通过Spring容器提前暴露刚完成构造器注入但未完成其他步骤（如setter注入）的bean来完成的，而且只能够解决单例作用域的bean循环依赖，为了避免这种循环依赖，在Spring中创建bean的原则是不等bean创建完成就会将创建bean的ObjectFactory提前曝光加入到缓存中，一旦下一个bean创建时候需要依赖上一个bean则直接使用ObjectFactory；第三种，protype范围的循环依赖，Spring容器无法完成依赖注入，因为Spring容器不进行缓存protype作用域的bean,因此无法提前暴露一个创建中的bean。


- （5）检测parentBeanFactory

从代码上看，如果缓存没有数据的话直接转到父类工厂上去加载了，这是为什么呢？可能读者忽略了一个很重要的判断条件：parentBeanFactory != null && !containsBeanDefinition(beanName)，parentBeanFactory如果为空，则其他一切都是浮云，这个没什么说的，但是!containsBeanDefinition(beanName)就比较重要了，它是在检测如果当前加载的XML配置文件中不包含beanName所对应的配置，就只能到parentBeanFactory去尝试下了，然后再去递归的调用getBean方法。


- （6）将存储XML配置文件的GernericBeanDefinition转换为RootBeanDefinition。

因为从XMl文件中读取到的Bean信息是存储在GernericBeanDefinition中的，但是所有的Bean后续处理都是针对RootBeanDefinition的，所以这里需要进行一个转换，转换的同时如果父类bean不为空的话，则会一并合并父类的属性。


- （7）寻找依赖

因为bean的初始化过程中很可能会用到某些属性，而某些属性很可能是动态配置的，并且配置成依赖于其他的bean,那么这个时候就有必要先加载依赖的bean,所以在Spring的加载顺序中，在初始化某个bean的时候首先会初始化这个bean所对应的依赖。


- （8）针对不同的scope进行bean的创建

我们都知道，在Spring中存在着不同的scope,其中默认的是singleton,但是还有些其他的诸如prototype、request之类的。在这个步骤中，spring会根据不同的配置进行不同的初始化策略。


- （9）类型转换

程序到这里返回bean后已经基本结束了，通常对该方法的调用参数requiredType是为空的，但是可能会存在这样的情况，返回bean其实是个String，但是requiredType却传入Integer类型，那么这时候本步骤就会起作用了，它的功能就是将返回的bean转换为requiredType所指定的类型。


到此我们似乎只是对getBean方法的过程步骤有了个整体了解，但是具体针对不同的scope是如何进行bean的创建工作还是不清楚，下面我们重点来看下这点：

不过在此之前我们还是先得来解决一个问题，之前我们讲解了从缓存中获取单例的过程，那么如果缓存中不存在已经加载的单例bean就需要从头开始bean的加载过程了，而Spring使用getSingleton的重载方法实现bean的加载过程。

		public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
				Assert.notNull(beanName, "'beanName' must not be null");
				//全局变量需要同步
				synchronized (this.singletonObjects) {
					//首先检查对应的bean是否已经加载过，因为singleton模式其实就是复用以创建的bean,所以这一步是必须的
					Object singletonObject = this.singletonObjects.get(beanName);
					//如果为空才可以进行sigleto的bean的初始化
					if (singletonObject == null) {
						if (this.singletonsCurrentlyInDestruction) {
							throw new BeanCreationNotAllowedException(beanName,
									"Singleton bean creation not allowed while the singletons of this factory are in destruction " +
									"(Do not request a bean from a BeanFactory in a destroy method implementation!)");
						}
						if (logger.isDebugEnabled()) {
							logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
						}
						beforeSingletonCreation(beanName);
						boolean newSingleton = false;
						boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
						if (recordSuppressedExceptions) {
							this.suppressedExceptions = new LinkedHashSet<Exception>();
						}
						try {
							//初始化bean
							singletonObject = singletonFactory.getObject();
							newSingleton = true;
						}
						catch (IllegalStateException ex) {
							// Has the singleton object implicitly appeared in the meantime ->
							// if yes, proceed with it since the exception indicates that state.
							singletonObject = this.singletonObjects.get(beanName);
							if (singletonObject == null) {
								throw ex;
							}
						}
						catch (BeanCreationException ex) {
							if (recordSuppressedExceptions) {
								for (Exception suppressedException : this.suppressedExceptions) {
									ex.addRelatedCause(suppressedException);
								}
							}
							throw ex;
						}
						finally {
							if (recordSuppressedExceptions) {
								this.suppressedExceptions = null;
							}
							afterSingletonCreation(beanName);
						}
						if (newSingleton) {
							//加入缓存
							addSingleton(beanName, singletonObject);
						}
					}
					return (singletonObject != NULL_OBJECT ? singletonObject : null);
				}
			}


上述代码其实是使用了回调方法，使得程序可以在单例创建的前后做一些准备及处理操作，而真正的获取单例bean的方法其实并不是在此方法中实现的，其实现逻辑是在ObjectFactory类型的实例singletonFactory中实现的。而这些准备及处理操作包括如下内容：

- （1）检查缓存是否已经加载过。
- （2）若没有加载，则记录beanName的正在加载状态。
- （3）加载单例前记录加载状态。
- （4）通过调用参数传入的ObjectFactory的个体Object方法实例化bean.
- （5）加载单例后的处理方法调用。
- （6）将结果记录至缓存并删除加载bean过程中所记录的各种辅助状态。
- （7）返回处理结果。


ObjectFactory的核心部分其实只是调用了createBean的方法（其实针对不同的scope进行bean的创建都是调用了），所以我们还需要到createBean方法中追寻真理。

		/**
			 * Central method of this class: creates a bean instance,
			 * populates the bean instance, applies post-processors, etc.
			 * @see #doCreateBean
			 */
		protected Object createBean(final String beanName, final RootBeanDefinition mbd, final Object[] args)
					throws BeanCreationException {
		
				if (logger.isDebugEnabled()) {
					logger.debug("Creating instance of bean '" + beanName + "'");
				}
				// Make sure bean class is actually resolved at this point.
				//锁定class,根据设置的class属性或者根据className来解析class
				resolveBeanClass(mbd, beanName);
		
				// Prepare method overrides.
				//验证及准备覆盖的方法
				try {
					mbd.prepareMethodOverrides();
				}
				catch (BeanDefinitionValidationException ex) {
					throw new BeanDefinitionStoreException(mbd.getResourceDescription(),
							beanName, "Validation of method overrides failed", ex);
				}
		
				try {
					// Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
					//给BeanPostProcessors一个机会来返回代理来替代真正的实例
					Object bean = resolveBeforeInstantiation(beanName, mbd);
					if (bean != null) {
						return bean;
					}
				}
				catch (Throwable ex) {
					throw new BeanCreationException(mbd.getResourceDescription(), beanName,
							"BeanPostProcessor before instantiation of bean failed", ex);
				}
		
				Object beanInstance = doCreateBean(beanName, mbd, args);
				if (logger.isDebugEnabled()) {
					logger.debug("Finished creating instance of bean '" + beanName + "'");
				}
				return beanInstance;
			}


从代码中我们可以总结出函数完成的具体步骤及功能。

- （1）根据设置的class属性或者根据className来解析Class。
- （2）对override属性进行标记及验证。

		/**
			 * Validate and prepare the method overrides defined for this bean.
			 * Checks for existence of a method with the specified name.
			 * @throws BeanDefinitionValidationException in case of validation failure
			 */
			public void prepareMethodOverrides() throws BeanDefinitionValidationException {
				// Check that lookup methods exists.
				MethodOverrides methodOverrides = getMethodOverrides();
				if (!methodOverrides.isEmpty()) {
					for (MethodOverride mo : methodOverrides.getOverrides()) {
						prepareMethodOverride(mo);
					}
				}
			}
		
			/**
			 * Validate and prepare the given method override.
			 * Checks for existence of a method with the specified name,
			 * marking it as not overloaded if none found.
			 * @param mo the MethodOverride object to validate
			 * @throws BeanDefinitionValidationException in case of validation failure
			 */
			protected void prepareMethodOverride(MethodOverride mo) throws BeanDefinitionValidationException {
				int count = ClassUtils.getMethodCountForName(getBeanClass(), mo.getMethodName());
				if (count == 0) {
					throw new BeanDefinitionValidationException(
							"Invalid method override: no method with name '" + mo.getMethodName() +
							"' on class [" + getBeanClassName() + "]");
				}
				else if (count == 1) {
					// Mark override as not overloaded, to avoid the overhead of arg type checking.
					mo.setOverloaded(false);
				}
			}

很多读者可能会不知道这个方法的作用，因为在Spring的配置里面根本就没有诸如override-method之类的配置，那么这个方法到底是干什么用的呢？其实在Spring中确实没有override-method这样的配置，但是在Spring配置中存在lookup-method和replace-method两个配置功能，而这两个配置的加载其实就是将配置统一存放在BeanDefinition中的methodOverrides属性里，这两个功能实现原理其实是在bean实例化的时候如果检测到存在methodOverrides属性，会动态地为当前bean生成代理并使用对应的拦截器为bean做增强处理。

- （3）应用初始化前的后处理器，解析指定bean是否存在初始化前的短路操作。

		/**
			 * Apply before-instantiation post-processors, resolving whether there is a
			 * before-instantiation shortcut for the specified bean.
			 * @param beanName the name of the bean
			 * @param mbd the bean definition for the bean
			 * @return the shortcut-determined bean instance, or {@code null} if none
			 */
			protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
				Object bean = null;
				if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
					// Make sure bean class is actually resolved at this point.
					if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
						Class<?> targetType = determineTargetType(beanName, mbd);
						if (targetType != null) {
							bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
							if (bean != null) {
								bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
							}
						}
					}
					mbd.beforeInstantiationResolved = (bean != null);
				}
				return bean;
			}

在真正调用doCreate方法创建bean的实例前使用了resolveBeforeInstantiation这样一个方法对BeanDefinition中的属性做些前置处理。此方法中最吸引我们的无疑是两个方法applyBeanPostProcessorsBeforeInstantiation和applyBeanPostProcessorsAfterInitialization。这两个方法实现的非常简单，无非是对后处理器中的所有InstantiationAwareBeanPostProcessor类型的后处理器进行postProcessBeforeInstantiation方法和BeanPostProcessor的postProcessAfterInitialization方法的调用。

- （4）创建bean。

当经历过resolveBeforeInstantiation方法后程序有两个选择，如果创建了代理或者说重写了InstantiationAwareBeanPostProcessor的postProcessBeforeInstantiation方法并在postProcessBeforeInstantiation方法中改变了bean,则直接返回就可以了，否则需要进行常规bean的创建。而这常规bean的创建就是在doCreateBean中完成的。


			protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args) {
				// Instantiate the bean.
				BeanWrapper instanceWrapper = null;
				if (mbd.isSingleton()) {
					instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
				}
				if (instanceWrapper == null) {
					instanceWrapper = createBeanInstance(beanName, mbd, args);
				}
				final Object bean = (instanceWrapper != null ? instanceWrapper.getWrappedInstance() : null);
				Class<?> beanType = (instanceWrapper != null ? instanceWrapper.getWrappedClass() : null);
		
				// Allow post-processors to modify the merged bean definition.
				synchronized (mbd.postProcessingLock) {
					if (!mbd.postProcessed) {
						applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
						mbd.postProcessed = true;
					}
				}
		
				
				boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
						isSingletonCurrentlyInCreation(beanName));
				if (earlySingletonExposure) {
					if (logger.isDebugEnabled()) {
						logger.debug("Eagerly caching bean '" + beanName +
								"' to allow for resolving potential circular references");
					}
					addSingletonFactory(beanName, new ObjectFactory<Object>() {
						@Override
						public Object getObject() throws BeansException {
							return getEarlyBeanReference(beanName, mbd, bean);
						}
					});
				}
		
				// Initialize the bean instance.
				Object exposedObject = bean;
				try {
					populateBean(beanName, mbd, instanceWrapper);
					if (exposedObject != null) {
						exposedObject = initializeBean(beanName, exposedObject, mbd);
					}
				}
				catch (Throwable ex) {
					if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
						throw (BeanCreationException) ex;
					}
					else {
						throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
					}
				}
		
				if (earlySingletonExposure) {
					Object earlySingletonReference = getSingleton(beanName, false);
					if (earlySingletonReference != null) {
						if (exposedObject == bean) {
							exposedObject = earlySingletonReference;
						}
						else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
							String[] dependentBeans = getDependentBeans(beanName);
							Set<String> actualDependentBeans = new LinkedHashSet<String>(dependentBeans.length);
							for (String dependentBean : dependentBeans) {
								if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
									actualDependentBeans.add(dependentBean);
								}
							}
							if (!actualDependentBeans.isEmpty()) {
								throw new BeanCurrentlyInCreationException(beanName,
										"Bean with name '" + beanName + "' has been injected into other beans [" +
										StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
										"] in its raw version as part of a circular reference, but has eventually been " +
										"wrapped. This means that said other beans do not use the final version of the " +
										"bean. This is often the result of over-eager type matching - consider using " +
										"'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
							}
						}
					}
				}
		
				// Register bean as disposable.
				try {
					registerDisposableBeanIfNecessary(beanName, bean, mbd);
				}
				catch (BeanDefinitionValidationException ex) {
					throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
				}
		
				return exposedObject;
			}


我们来看看整个函数的概要思路：

![](https://www.ibm.com/developerworks/cn/java/j-lo-spring-principle/origin_image014.png)

- （1）如果是单例则需要首先清除缓存。
- （2）实例化bean,将BeanDefinition转换为BeanWrapper。
	- 如果存在工厂方法则使用工厂方法进行初始化。
	- 一个类有多个构造函数，每个构造函数都有不同的参数，所以需要根据参数锁定构造函数并进行初始化。
	- 如果既不存在工厂方法也不存在带有参数的构造函数，则使用默认的构造函数进行bean的实例化。
- （3）MergeBeanDefinitionPostProcessor的应用。（bean合并后的处理，Autowired注解正是通过此方法实现诸如类型的预解析）
- （4）依赖处理
- （5）属性填充。将所有的属性填充至bean的实例中。
- （6）循环依赖检查。
- （7）注册DisposableBean。（如果配置了destroy-method,这里需要注册以便于在销毁时候调用）。
- （8）完成创建并返回。




#### 2.ApplicationContext方式 ####

其实ApplicationContext和BeanFactory这两种方式针对bean实例的创建而言底层都是一样的，最终还是由 getBean(String name) 来实现，下面我们来具体分析下ApplicationContext方式是怎么调到getBean(String name) ?

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

前面提到AbstractApplicationContext 类的 refresh 方法是构建整个 Ioc 容器过程的完整的代码，了解了里面的每一行代码基本上就了解大部分 Spring 的原理和功能了。不错，对于bean实例的创建入口也在此处 —— finishBeanFactoryInitialization(beanFactory)。

 AbstractApplicationContext.finishBeanFactoryInitialization代码如下：

		protected void finishBeanFactoryInitialization(
		    ConfigurableListableBeanFactory beanFactory) { 
		 
			......
			
		    // Stop using the temporary ClassLoader for type matching. 
		    beanFactory.setTempClassLoader(null); 
		 
		    // Allow for caching all bean definition metadata, not expecting further changes.
		    beanFactory.freezeConfiguration(); 
		 
		    // Instantiate all remaining (non-lazy-init) singletons. 
		    beanFactory.preInstantiateSingletons(); 
		}

从上面代码中可以发现 Bean 的实例化是在 BeanFactory 中发生的。DefaultListableBeanFactory.preInstantiateSingletons 方法的代码如下：

		public void preInstantiateSingletons() throws BeansException { 
		    if (this.logger.isInfoEnabled()) { 
		        this.logger.info("Pre-instantiating singletons in " + this); 
		    } 
		    synchronized (this.beanDefinitionMap) { 
		        for (String beanName : this.beanDefinitionNames) { 
		            RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName); 
		            if (!bd.isAbstract() && bd.isSingleton() 
		                && !bd.isLazyInit()) {
		                if (isFactoryBean(beanName)) { 
		                    final FactoryBean factory = 
		                        (FactoryBean) getBean(FACTORY_BEAN_PREFIX+ beanName); 
		                    boolean isEagerInit; 
		                    if (System.getSecurityManager() != null 
		                        && factory instanceof SmartFactoryBean) { 
		                        isEagerInit = AccessController.doPrivileged(
		                            new PrivilegedAction<Boolean>() { 
		                            public Boolean run() { 
		                                return ((SmartFactoryBean) factory).isEagerInit(); 
		                            } 
		                        }, getAccessControlContext()); 
		                    } 
		                    else { 
		                        isEagerInit = factory instanceof SmartFactoryBean 
		                            && ((SmartFactoryBean) factory).isEagerInit(); 
		                    } 
		                    if (isEagerInit) { 
		                        getBean(beanName); 
		                    } 
		                } 
		                else { 
		                    getBean(beanName); 
		                } 
		            } 
		        } 
		    } 
		}


这里补充下我们频繁看到的FactoryBean，这是一个非常重要的 Bean 。可以说 Spring 一大半的扩展的功能都与这个 Bean 有关，这是个特殊的 Bean 他是个工厂 Bean，可以产生 Bean 的 Bean，这里的产生 Bean 是指 Bean 的实例，如果一个类继承 FactoryBean 用户可以自己定义产生实例对象的方法只要实现他的 getObject 方法。然而在 Spring 内部这个 Bean 的实例对象是 FactoryBean，通过调用这个对象的 getObject 方法就能获取用户自定义产生的对象，从而为 Spring 提供了很好的扩展性。Spring 获取 FactoryBean 本身的对象是在前面加上 & 来完成的。


至此，我们终于明白为什么上面说ApplicationContext方式针对bean的创建底层也是通过调用getBean方法来实现的吧。getbean方法上文已经详细讲到，这里不再赘述下去，ApplicationContext方式bean的创建过程函数调用栈如下：

![](https://cloud.githubusercontent.com/assets/1736354/7929379/cec01bcc-092f-11e5-81ad-88c285f33845.png)

#### 阶段性总结： ####

如何创建 Bean 的实例对象以及如何构建 Bean 实例对象之间的关联关系式 Spring 中的一个核心关键，下面是这个过程的流程图和时序图。

![](https://www.ibm.com/developerworks/cn/java/j-lo-spring-principle/origin_image012.gif)

![](https://www.ibm.com/developerworks/cn/java/j-lo-spring-principle/origin_image013.png)



### 三、Spring IOC容器的扩展功能 ###

前面提到的说ApplicationContext是构建于BeanFactory之上的IoC容器，是相对比较高级的容器实现，除了拥有BeanFactory的所有支持，ApplicationContext还提供了其他高级特性，比如事件发布、国际化信息支持等扩展功能。聪明如你其实早就发现AbstractApplicationContext 类的 refresh 方法中还有很多我们并未重点提到的方法，不错其他的那些基本上都是为Spring IOC容器的扩展而存在的。例如，BeanFactoryPostProcessor、BeanPostProcessor他们分别是在构建 BeanFactory 和构建 Bean 对象时调用，还有就是 InitializingBean 和 DisposableBean 他们分别是在 Bean 实例创建和销毁时被调用。用户可以实现这些接口中定义的方法，这样Spring 就会在适当的时候调用他们。

我们把 Ioc 容器比作一个箱子，这个箱子里有若干个球的模子，可以用这些模子来造很多种不同的球，还有一个造这些球模的机器，这个机器可以产生球模。那么他们的对应关系就是 BeanFactory 就是那个造球模的机器，球模就是 Bean，而球模造出来的球就是 Bean 的实例。那前面所说的几个扩展点又在什么地方呢？ BeanFactoryPostProcessor 对应到当造球模被造出来时，你将有机会可以对其做出设当的修正，也就是他可以帮你修改球模。而 InitializingBean 和 DisposableBean 是在球模造球的开始和结束阶段，你可以完成一些预备和扫尾工作。BeanPostProcessor 就可以让你对球模造出来的球做出适当的修正。最后还有一个 FactoryBean，它可是一个神奇的球模。这个球模不是预先就定型了，而是由你来给他确定它的形状，既然你可以确定这个球模型的形状，当然他造出来的球肯定就是你想要的球了，这样在这个箱子里你可以发现所有你想要的球









































































