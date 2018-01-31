### Spring源码篇之Spring IOC 核心实现 ###
***

### 一、Spring Bean的定义、解析与注册 ###

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

经过前面的分析，我们终于结束了对XML配置文件的解析，接下来将会面临更大的挑战，那就是对Bean实例创建的探索。从Demo跟踪我们发现Bean实例的创建是在BeanFactory中发生的

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














































































































