### Spring源码篇之Spring MVC ###
***

在正式开撸前先我觉得有必要借用网上一副简图来让我们回顾下传统Web容器的初始化过程。


![](https://images2015.cnblogs.com/blog/671185/201607/671185-20160720111104763-644678711.jpg)

同样，分析Spring MVC的源码我们从Listener开始 ！！！

### 一、ContextLoaderListener ###

对于Spring MVC 功能实现的分析，我们首先从web.xml开始，在web.xml文件中我们首先配置的就是ContextLoaderListener，那么它所提供的功能有哪些又是如何实现的呢？

当使用编程方式的时候我们可以直接将Spring配置信息作为参数传入Spring容器中，如 ApplicationContext ac = new ClassPathXmlApplicationContext（"applicationContext.xml"）。但是在Web下，我们需要更多的是与Web环境相互结合，通常的办法是将路径以 context-param 的方式注册并使用ContextLoaderListener进行监听读取。


![](http://hi.csdn.net/attachment/201009/17/0_1284726147uq4u.gif)


ContextLoderListener（根web应用环境上下文加载监听器类）的作用就是启动web容器时，自动装配ApplicationContext的配置信息。ContextLoderListener（根web应用环境上下文加载监听器类）实现了Servlet规范中定义的Servlet环境监听器用以处理初始化事件和销毁事件。ContextLoaderListener初始化的上下文和DispatcherServlet初始化的上下文关系如下图所示：


![](https://images2015.cnblogs.com/blog/671185/201607/671185-20160720164251513-1769278820.jpg)



在web.xml配置这个监听器，启动容器时，就会默认执行ContextLoderListener实现的上下文初始化方法。核心代码如下：


	/**
	 * Initialize the root web application context.
	 */
	@Override
	public void contextInitialized(ServletContextEvent event) {
		initWebApplicationContext(event.getServletContext());
	}


	/**
	 * Close the root web application context.
	 */
	@Override
	public void contextDestroyed(ServletContextEvent event) {
		closeWebApplicationContext(event.getServletContext());
		ContextCleanupListener.cleanupAttributes(event.getServletContext());
	}

在ServletContextListener中的核心逻辑便是初始化WebApplicationContext（在Web应用中，我们会用到WebApplicationContext，WebApplicationContext继承自ApplicationContext，在ApplicationContext的基础上又追加了一些特定于web的操作及属性，非常类似于我们通过编程方式使用Spting时使用的ClassPathXmlApplicationContext类提供的功能。）实例并存放至ServletContext中。继续追踪代码：

		
		public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
				//如果已经存在了根共享Web应用程序环境，则抛出异常提示客户
				if (servletContext.getAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE) != null) {
					throw new IllegalStateException(
							"Cannot initialize context because there is already a root application context present - " +
							"check whether you have multiple ContextLoader* definitions in your web.xml!");
				}
		
				Log logger = LogFactory.getLog(ContextLoader.class);
				servletContext.log("Initializing Spring root WebApplicationContext");
				if (logger.isInfoEnabled()) {
					logger.info("Root WebApplicationContext: initialization started");
				}
				//记录创建根Web应用程序环境的开始时间
				long startTime = System.currentTimeMillis();
		
				try {
					// Store context in local instance variable, to guarantee that
					// it is available on ServletContext shutdown.
					if (this.context == null) {
						//创建根Web应用程序环境
						this.context = createWebApplicationContext(servletContext);
					}
					if (this.context instanceof ConfigurableWebApplicationContext) {
						ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
						if (!cwac.isActive()) {
							// The context has not yet been refreshed -> provide services such as
							// setting the parent context, setting the application context id, etc
							if (cwac.getParent() == null) {
								// The context instance was injected without an explicit parent ->
								// determine parent for root web application context, if any.
								ApplicationContext parent = loadParentContext(servletContext);
								cwac.setParent(parent);
							}
							configureAndRefreshWebApplicationContext(cwac, servletContext);
						}
					}
					servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);
		
					ClassLoader ccl = Thread.currentThread().getContextClassLoader();
					if (ccl == ContextLoader.class.getClassLoader()) {
						currentContext = this.context;
					}
					else if (ccl != null) {
						currentContextPerThread.put(ccl, this.context);
					}
		
					if (logger.isDebugEnabled()) {
						logger.debug("Published root WebApplicationContext as ServletContext attribute with name [" +
								WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE + "]");
					}
					if (logger.isInfoEnabled()) {
						long elapsedTime = System.currentTimeMillis() - startTime;
						logger.info("Root WebApplicationContext: initialization completed in " + elapsedTime + " ms");
					}
		
					return this.context;
				}
				catch (RuntimeException ex) {
					logger.error("Context initialization failed", ex);
					servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, ex);
					throw ex;
				}
				catch (Error err) {
					logger.error("Context initialization failed", err);
					servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, err);
					throw err;
				}
			}



initWebApplicationContext函数主要是体现了创建WebApplicationContext实例的一个功能架构，从函数中我们看到了初始化的大致步骤。

（1）WebApplicationContext存在性的验证。

在配置中只允许声明一次ServletContextListener，多次声明会扰乱Spring的执行逻辑，所以这里首先做的就是对此验证，在Spring中如果创建WebApplicationContext实例会记录在ServletContext中以方便全局调用，而使用的Key就是WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE，所以验证的方式就是查看ServletContext实例中是否有对应的key的属性。


（2）创建WebApplicationContext实例。

如果通过验证，则Spring将创建WebApplicationContext实例的工作委托给了 createWebApplicationContext 函数。



	protected WebApplicationContext createWebApplicationContext(ServletContext sc) {
			//取得配置的Web应用程序环境类，如果没有配置，则使用缺省的类XmlWebApplicationContext
			Class<?> contextClass = determineContextClass(sc);
			//如果配置的Web应用程序环境类不是可配置的Web应用程序环境的子类，则抛出异常，停止初始化
			if (!ConfigurableWebApplicationContext.class.isAssignableFrom(contextClass)) {
				throw new ApplicationContextException("Custom context class [" + contextClass.getName() +
						"] is not of type [" + ConfigurableWebApplicationContext.class.getName() + "]");
			}
			return (ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass);
		}

（3）将实例记录在servletContext中。

（4）映射当前的类加载器与创建的实例到全局变量cuttentContextPerThread中。




### 二、Spring MVC核心组件 ###


Spring MVC 是由若干组件组成的，这些组件相互独立又相互协调工作共同完成 Spring MVC 工作流。每个组件都有清晰的接口定义，接口后面都有一个设计良好的类实现体系结构，清晰的抽象出公用的逻辑并且实现在通用的抽象类里，同时提供常用的具体实现类。 进而实现一个清晰的，高可扩展的，可插拔的 Web MVC 体系结构。

#### 1.DispatcherServlet ####

在Spring中，ContextLoaderListener只是辅助功能，用于创建WebApplicationContext类型实例，而真正的逻辑实现其实是在DispatcherServlet中进行的，DispatcherServlet是Spring MVC中最核心的组件，它是一个实现Servlet接口的实现类。从类的继承角度来看，DispatcherServlet 最终继承自 HttpServlet 对象。通过若干个抽象类的划分，使DispatcherServlet 的类体系接口清晰明了，任务分明，容易扩展。

![](http://hi.csdn.net/attachment/201009/17/0_12847161188tPK.gif)

初始化时， 它通过内部的 Spring Web 应用程序环境，找到相应的 Spring MVC 的各个组件，如果这些组件没有显式配置， Spring MVC 将会根据默认加载策略初始化各个模块的默认实现。
 
服务时，它通过一套注册的处理器映射对象 (HandlerMapping) 找到一个处理器对象 (Handler) ，然后，从一套注册的处理器适配器对象集（ HandlerAdapter ）中找到一个支持这个处理器对象 (Handler) 的处理器适配器对象（ HandlerAdapter ），并且通过它把控制流转发给这个处理器对象 (Hanlder) ，处理器对象 (Handler) 结束业务逻辑的调用后把模型数据和逻辑视图传回DispatcherServlet。DispatcherServlet再通过视图解析器对象 (ViewResolver) 得到真正的视图对象（ View ），把控制权交给视图对象（ View ），同时传入模型数据，视图对象（ View ）会按照一定的视图层定义展现这些数据到用户响应对象里。



#### 2.HandlerMapping ####

处理器映射对象 (HandlerMapping) 是用于映射一个请求对象 (Request) 到一个处理器对象 (Handler) 。一个 Spring Web MVC 实例可能会包含多个处理器映射对象 (HandlerMapping) 。按照处理器映射对象所在的顺序，第一个返回非空处理器执行链对象 (HandlerExecutionChain) 的处理器映射，会被当作为有效的处理器映射对象。处理器执行链对象 (HandlerExecutionChain) 包含一个处理器对象 (Hanlder) 和一组能够应用在处理器对象 (Handler) 上的拦截器对象 (Interceptor).

![](http://hi.csdn.net/attachment/201009/17/0_1284705364rqKR.gif)

在这个接口中，通过输入一个 HTTP 请求对象 (HttpServletRequest) 给方法 (getHandler) ，这个方法就会输出一个处理器执行链对象 (HandlerExecutionChain) ，进而我们能够从处理器执行链对象 (HandlerExecutionChain) 得到一个处理器对象 (Handler) 。


#### 3.HandlerAdaptor ####


处理器适配器 (HandlerAdaptor) 是用来转接一个控制流到一个指定类型的处理器对象（ Handler ）。一个类型的处理器对象（ Handler ）通常会对应一个处理器适配器对象 (HandlerAdaptor) 的一个实现。处理器适配器对象 (HandlerAdaptor) 能够判断自己是否支持某个处理器对象（ Hanlder ）。如果一个处理器适配器对象 (HandlerAdaptor) 支持这种类型的处理器 (Handler) ，那么这个处理器适配器对象 (HandlerAdaptor) 就可以使用这个处理器对象（ handler ）处理当前的这个 HTTP 请求。然而，处理器对象 (Hanlder) 是一个通用的 Java 对象类型 , 这样做是允许 Spring MVC 可以任意的去集成其他框架的处理器对象 (handler), 只需要为这个需要支持的处理器对象 (Handler) 提供一个客户化的处理器适配器对象 (HanlderAdaptor), 就可以在不改变任何上层派遣器 Servlet 对象 (DispatcherServlet) 代码以及下层控制器对象 (Controller) 代码的前提下，实现任意集成。后来证明，基于注释的控制器，以及 Spring Web Flow 都是通过这样的适配器的实现来完成和 Spring MVC 工作流集成的，我们将在后面章节详细讨论。

![](http://hi.csdn.net/attachment/201009/17/0_12847053753ZGC.gif)

在这个接口中，通过输入一个处理器对象（ Handler ）给 supports() 方法，这个方法就会返回是否支持这个处理器。通过传入一个 HttpServletRequese ， HttpServletResponse 和一个 Hanlder 对象给 handle() 方法，这个方法就会就会传递控制权给处理器对象 (Handler) 。当处理器对象（ Handler ）处理这个 HTTP 请求后返回数据模型和逻辑视图名称的组合对象，处理器适配器会把这个返回结果进而返回给派遣器 Servlet(DispatcherServlet) 。




#### 4.Handler - Controller ####

处理器对象（ Handler ）是用于处理业务逻辑的一个基本单元。它通过传入的 HTTP 请求对象 (HttpServletRequese) 来决定如何处理业务逻辑和执行哪些服务，执行服务后返回相应的模型数据和逻辑视图名。处理器对象 (Handler) 是一个通用的对象类型。控制流是由处理器适配器 (HandlerAdaptor) 传递给处理器的。所以，一个类型的处理器对象 (Hanlder) 一定会对应一个支持的处理器适配器 (HandlerAdaptor) 。这样可以实现，处理器适配器（ HandlerAdaptor ）和处理器 (Handler) 的任意插拔。

控制器是 Spring Web MVC 中最简单的一个处理器。这个处理器有清晰的接口定义，通常这个类型的处理器对象 (Handler) 是通过一个简单控制处理适配器对象 (SimpleControllerHandlerAdapter) 来传递控制的。

![](http://hi.csdn.net/attachment/201009/17/0_1284705380RB7Y.gif)

在这个接口中，通过输入一个 HTTP 请求对象（ HttpServletRequese ）和 HTTP 响应对象 (HttpServletResponse) 给方法 handleRequest() 方法，这个方法就会返回一个处理后的模型数据和逻辑视图名的组合对象。这个对象将通过处理器适配器 (HandlerAdapter) 进而返回给派遣器 Servlet(DispatcherServlet) 。

Spring MVC 之所以具有高可扩展性，在于处理器适配器（ HandlerAdapter ）和处理器 (Handler) 的设计。这样可以让任意的一个处理器对象插入到 Spring MVC 的体系结构中，是它具有无限的扩展性。一个例子就是自从 Spring 2.5 新引进的注释方法处理器适配器对象 (AnnotationMethodHandlerAdapter) 和注释处理器。

#### 5.ViewResolver ####

视图解析器对象 (ViewResolver) 是用于映射一个逻辑视图名称到一个真正的视图对象（ View ）。当控制器对象 (Controller) 处理业务逻辑之后，通常会返回需要显示的数据模型对象和视图的逻辑名称，这样就需要一个支持的视图解析器对象 (ViewResolver) 解析出一个真正的视图对象 (View) ，然后传递控制流给这个视图对象。
 
![](http://hi.csdn.net/attachment/201009/17/0_1284705395Z1S9.gif)

在这个接口中，通过输入一个逻辑视图名称和本地对象 (Locale) 给 resolveViewName 方法，这个方法就会返回一个事实上的物理视图对象（ View ）。


#### 6.View ####

View 是用来把模型数据通过某种显示方式反馈给客户，通常是通过执行 JSP 页面来完成的。也可以通过其他的更复杂的显示技术来完整这个展示过程，例如， JSTL 视图， Tiles 视图，报表视图和二进制文件视图等等。

![](http://hi.csdn.net/attachment/201009/17/0_1284705402Zr5L.gif)

在这个接口中，通过输入 HTTP 请求对象（ HttpServletRequese ） , HTTP 响应对象 (HttpServletResponse) 和一个模型 Map 对象，这个组件就会把模型 Map 通过一定的显示方式包含到客户的 HTTP 响应对象 (HttpServletResponse) ，这就是提供给用户请求的最终响应。 


**总结：组件间的协调通信**

众所周知，一个 HTTP 请求发送到 Web 容器， Web 容器就会封装一个 HTTP 请求对象 (HttpServletRequest) ，这个对象包含所有的 HTTP 请求信息，例如， HTTP 参数以及参数值， HTTP 请求头的各种元数据。同时， Web 容器会创建一个 HTTP 响应对象（ HttpServletResponse ），用以发送 HTTP 响应给客户端用户。然后， Web 容器传递 HTTP 请求对象 (HttpServletRequest) 和 HTTP 响应对象（ HttpServletResponse ）给 Servlet 对象的 service() 方法。实际上， Spring Web MVC 的入口就是一个客户化的 Servlet ，称为派遣器 Servlet 对象 (DispatcherServlet) 。这个派遣器 Servlet 对象（ DispatcherServlet ）得到 HTTP 请求对象 (HttpServletRequest) 和 HTTP 响应对象（ HttpServletResponse ）后，一个典型的 Spring Web MVC 工作流就开始了。

![](http://hi.csdn.net/attachment/201009/17/0_1284705410Ii4I.gif)




- 派遣器 Servlet 对象（ DispatcherServlet ）首先查找所有注册的处理器映射器对象（ HandlerMapping ），然后，遍历所有的处理器映射器对象（ HandlerMapping ），直到一个处理器映射器对象（ HandlerMapping ）返回一个非空的处理器执行链对象 (HandlerExecutionChain) 。那么，处理器执行链对象 (HandlerExecutionChain) 就包含着一个需要处理当前 HTTP 请求的一个处理器对象（ Handler ）。如图第 1 步。这里，一个处理器对象 (Handler) 被设计成了一个通用的对象类型，所以，这里需要一个处理器适配器（ HandlerAdaptor ）去派遣这个控制流到一个处理器对象 (Handler) ，因为只有支持这种类型的处理器（ Handler ）的处理器适配器 (HandlerAdapter) 才知道如何去传递控制流给这个类型的处理器（ Hanlder ）。
 


- 拿到了处理器对象 (Hanlder) 以后，派遣器 Servlet 对象（ DispatcherServlet ）则查找所有注册的处理器适配器对象（ HandlerAdapter ），然后，遍历所有的处理器适配器对象（ HandlerAdapter ）查询是否有一个处理器适配器对象（ HandlerAdapter ）支持这个处理器对象（ Handler ）。如图第 2 步。
 


- 如果有这样的一个处理器适配器对象 (HandlerAdapter) ，则派遣器 Servlet 对象（ DispatcherServlet ）将控制权转交给这个派遣器适配器对象 (HandlerAdapter) 。如图第 3 步。派遣器适配器对象（ HandlerAdapter ）和真正的处理器对象 (Handler) 是成对出现的，所以，这个支持的处理器适配器对象（ HandlerAdapter ）则知道如何去使用这个处理器 (Handler) 去处理这个请求。最简单的一个处理器则是控制器对象（ Controller ）。处理器适配器对象 (HandlerAdapter) 将传递 HTTP 请求对象（ HttpServletRequest ）和 HTTP 响应对象（ HttpServletResponse ）给控制器，并且期待控制器返回模型和视图对象（ ModelAndView ）。如图第 3.1 步。这个模型和视图对象（ ModelAndView ）对象包含着一组模型数据和视图逻辑名称，并且最终返回给派遣器 Servlet 对象（ DispatcherServlet ） .
 


- 派遣器 Servlet 对象（ DispatcherServlet ）然后查找所有注册的视图解析器对象，并且遍历所有的视图解析器对象（ ViewResolver ），直到一个视图解析器对象（ ViewResolver ）返回一个物理的视图对象（ View ）。如图 第 4 步。
 


- 最后，派遣器 Servlet （ DispatcherServlet ）把得到的一组模型数据传递给得到的物理视图对象（ View ）。如图第 5 步。然后，视图对象则会使用响应的表现层技术，把模型数据展现成 UI 界面，并且通过 HTTP 响应对象（ HttpServletResponse ）发送给 HTTP 客户。


### 三、DispatcherServlet ###

DispatcherServlet是一个经过多个层次最终继承自 Servlet规范中的 HttpServlet, 进而实现 Servlet规范中定义的 Servlet接口。这些继承和实现组成了一个复杂的树形结构，在树形结构中的每个层次的类完成一个特定的初始化功能，服务功能或者清理资源的功能，每个层次的类之间分工合理，易于扩展。

![](http://hi.csdn.net/attachment/201009/17/0_12847161188tPK.gif)

从上图可以看出，前三个类被标识为灰色，因为他们都是 Servlet规范中规定接口或者实现，并不是 Spring的实现。而后面的所有类都是 Spring Web MVC的实现类。


- Servlet是 Servlet规范中规定的一个服务器组件的接口，任何一个可以处理用户请求的服务器组件需要实现这个接口， Web容器就是根据 URL到 Servlet的映射派遣一个 HTTP请求到这个 Servlet组件的实现，进而对这个 HTTP请求进行处理，并且产生 HTTP响应。
 


- 通用 Servlet(GenericServlet)是 Servlet的一个抽象实现。这个实现是和协议无关的。它提供了 Servlet应该具有的基础功能。例如，保存 Servlet配置，为后来的操作提供初始化参数和信息等等。
 


- HTTP Servlet(HttpServlet)是针对 HTTP协议对通用 Servlet的实现。它实现了 HTTP协议的一些基本操作。例如，根据 HTTP请求的方法分发不同类型的 HTTP请求到不同的方法进行处理。对于简单的 HTTP方法 (HEAD, OPTIONS, TRACE)提供了通用的实现，这些实现在子类中通常是不需要重写的。而对其他的业务服务类型的方法 (GET, POST, PUT, DELETE)提供了占位符方法。子类应该有选择的根据业务逻辑重写这些服务类型方法的实现（ Spring Web MVC就是通过实现这些占位符方法来派遣 HTTP请求到 Spring Web MVC的控制器组件方法的）。

![](http://hi.csdn.net/attachment/201009/17/0_1284716766EVnP.gif) 


- HTTP Servlet Bean(HttpServletBean)是 Spring Web MVC的一个抽象实现。它提供了一个特殊的功能，可以将 Servlet配置的初始化参数作为 Bean的属性，自动赋值给 Servlet的属性。子类 Servlet的很多属性都是通过这个功能进行配置的。
 


- Framework Servlet(FrameworkServlet)也是一个抽象的实现。它提供了加载一个对应的 Web应用程序环境的功能。这个 Web应用程序环境可以存在一个根环境，这个根环境可以是共享的也可以是这个 Servlet或者几个 Servlet专用的。它也提供了功能将 HTTP GET, POST, PUT, DELETE方法统一派遣到 Spring Web MVC的控制器方法进行派遣。在派遣前到处请求环境等信息到线程的局部存储。
 


- 派遣器 Servlet(DispatcherServlet)是这个继承链中最后一个类，它是 Spring Web MVC的核心实现类，它在框架 Servlet中加载的 Web应用程序环境中查找一切 Spring Web MVC所需要的并且注册的组件，如果一个需要的 Spring Web MVC组件没有注册，则通过缺省策略的配置创建并且初始化这些组件。在一个 HTTP请求被派遣的时候，它使用得到的 Spring Web MVC组件进行处理和响应。
 
![](https://images2015.cnblogs.com/blog/671185/201607/671185-20160720172838763-1330150960.jpg) 

介绍完了DispatcherServlet的继承体系，是时候开始进入DispatcherServlet之旅了。我们知道Servlet的生命周期是由Servlet的容器来控制的，它可以分为3个阶段：初始化、运行和销毁。

#### 1.DispatcherServlet的初始化 ####

熟悉Servlet的人都清楚，在Servlet初始化阶段会调用其init方法，所以我们首先要查看DispatcherServlet中是否重写了init方法。我们在其父类HttpServletBean中找到了该方法。

		/**
			 * Map config parameters onto bean properties of this servlet, and
			 * invoke subclass initialization.
			 * 
			 */
			@Override
			public final void init() throws ServletException {
				if (logger.isDebugEnabled()) {
					logger.debug("Initializing servlet '" + getServletName() + "'");
				}
		
				// Set bean properties from init parameters.
				try {
					//解析init-param并封装至pvs中
					PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), this.requiredProperties);
					//将当前的Servlet类转化为一个BeanWrapper，从而能够以Spring的方式来对init-param的值进行注入
					BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
					ResourceLoader resourceLoader = new ServletContextResourceLoader(getServletContext());
					//注册自定义属性编辑器，一旦遇到Resource类型的属性就会使用ResourceEditor进行解析
					bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, getEnvironment()));
					//空实现，留给子类覆盖
					initBeanWrapper(bw);
					//属性注入
					bw.setPropertyValues(pvs, true);
				}
				catch (BeansException ex) {
					logger.error("Failed to set bean properties on servlet '" + getServletName() + "'", ex);
					throw ex;
				}
		
				// Let subclasses do whatever initialization they like.（留给子类扩展）
				initServletBean();
		
				if (logger.isDebugEnabled()) {
					logger.debug("Servlet '" + getServletName() + "' configured successfully");
				}
			}

DispatcherServlet的初始化过程主要是通过将当前的Servlet类型实例转化为BeanWrapper类型实例，以便使用Spring中提供的注入功能进行对应属性的注入。这些属性如contextAttribute、contextClass、nameSpace、contextConfigLocation等，都可以在web.xml文件中以初始化参数的方式配置在Servlet的声明中。从上面的源码可以看出，HttpServletBean初始化自己特殊的资源以后，留了另外一个方法initServletBean()，这个方法提供子类初始化的机会。而在这个类体系结构中的下一个实现类是FrameworkServlet，FrameworkServlet提供的主要功能就是加载一个Web应用程序环境，这就是通过实现父类的占位符方法initServletBean()实现的。并且重写HttpServlet中的占位符方法，派遣HTTP请求到统一的Spring MVC的控制器方法，进而派遣器Servlet(DispatcherServlet)派遣这个HTTP请求到不同的处理器进行处理和响应。


			//重写了Http Servlet Bean的初始化占位符方法initServletBean()，进而初始化框架Serlvet所需的资源，在这里就是Web应用程序环境  
			@Override
			protected final void initServletBean() throws ServletException {
				//打印初始化信息到Servlet容器的日志  
				getServletContext().log("Initializing Spring FrameworkServlet '" + getServletName() + "'");
				if (this.logger.isInfoEnabled()) {
					this.logger.info("FrameworkServlet '" + getServletName() + "': initialization started");
				}
				long startTime = System.currentTimeMillis();
		
				try {
					this.webApplicationContext = initWebApplicationContext();
					//设计为子类覆盖（这个方法在派遣器Servlet中并没有覆盖）
					initFrameworkServlet();
				}
				catch (ServletException ex) {
					this.logger.error("Context initialization failed", ex);
					throw ex;
				}
				catch (RuntimeException ex) {
					this.logger.error("Context initialization failed", ex);
					throw ex;
				}
		
				if (this.logger.isInfoEnabled()) {
					long elapsedTime = System.currentTimeMillis() - startTime;
					this.logger.info("FrameworkServlet '" + getServletName() + "': initialization completed in " +
							elapsedTime + " ms");
				}
			}

从上面代码可以看出，作为关键的初始化逻辑实现委托给了initWebApplicationContext().initWebApplicationContext()函数的主要工作就是创建或者刷新WebApplicationContext实例并对Servlet功能所使用的变量进行初始化。

		/**
			 * Initialize and publish the WebApplicationContext for this servlet.
			 */
			protected WebApplicationContext initWebApplicationContext() {
				//
				WebApplicationContext rootContext =
						WebApplicationContextUtils.getWebApplicationContext(getServletContext());
				WebApplicationContext wac = null;
		
				if (this.webApplicationContext != null) {
					// A context instance was injected at construction time -> use it
					//context实例在构造函数中被注入
					wac = this.webApplicationContext;
					if (wac instanceof ConfigurableWebApplicationContext) {
						ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) wac;
						if (!cwac.isActive()) {
							// The context has not yet been refreshed -> provide services such as
							// setting the parent context, setting the application context id, etc
							if (cwac.getParent() == null) {
								// The context instance was injected without an explicit parent -> set
								// the root application context (if any; may be null) as the parent
								cwac.setParent(rootContext);
							}
							//刷新上下文环境
							configureAndRefreshWebApplicationContext(cwac);
						}
					}
				}
				if (wac == null) {
					// No context instance was injected at construction time -> see if one
					// has been registered in the servlet context. If one exists, it is assumed
					// that the parent context (if any) has already been set and that the
					// user has performed any initialization such as setting the context id
					//根据contextAttribute属性加载WebApplicationContext
					wac = findWebApplicationContext();
				}
				if (wac == null) {
					// No context instance is defined for this servlet -> create a local one
					wac = createWebApplicationContext(rootContext);
				}
		
				if (!this.refreshEventReceived) {
					// Either the context is not a ConfigurableApplicationContext with refresh
					// support or the context injected at construction time had already been
					// refreshed -> trigger initial onRefresh manually here.
					onRefresh(wac);
				}
		
				if (this.publishContext) {
					// Publish the context as a servlet context attribute.
					String attrName = getServletContextAttributeName();
					getServletContext().setAttribute(attrName, wac);
					if (this.logger.isDebugEnabled()) {
						this.logger.debug("Published WebApplicationContext of servlet '" + getServletName() +
								"' as ServletContext attribute with name [" + attrName + "]");
					}
				}
		
				return wac;
			}

对于本函数中的初始化主要包含几个部分：


- （1）寻找或创建对应的WebApplicationContext实例。WebApplicationContext的寻找及创建包括以下几个步骤：
	- I、通过构造函数的注入进行初始化。
	- II、通过contextAttribute进行初始化。通过在web.xml文件中配置的servlet参数contextAttribute来查找ServletContext中对应的属性，默认为WebApplicationContext.class.getName()+".ROOT",也就是在ContextLoaderListener加载时会创建WebApplicationContent实例，并将实例以WebApplicationContext.class.getName()+".ROOT"为key放入ServletContext中。
	- III、重新创建WebApplicationContext实例。如果通过以上两种方式并没有找到任何突破，那就没办法了，只能在这里重新创建新的实例了。


- （2）configureAndRefreshWebApplicationContext

		protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac) {
				if (ObjectUtils.identityToString(wac).equals(wac.getId())) {
					// The application context id is still set to its original default value
					// -> assign a more useful id based on available information
					if (this.contextId != null) {
						wac.setId(this.contextId);
					}
					else {
						// Generate default id...
						wac.setId(ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX +
								ObjectUtils.getDisplayString(getServletContext().getContextPath()) + "/" + getServletName());
					}
				}
		
				wac.setServletContext(getServletContext());
				wac.setServletConfig(getServletConfig());
				wac.setNamespace(getNamespace());
				wac.addApplicationListener(new SourceFilteringListener(wac, new ContextRefreshListener()));
		
				// The wac environment's #initPropertySources will be called in any case when the context
				// is refreshed; do it eagerly here to ensure servlet property sources are in place for
				// use in any post-processing or initialization that occurs below prior to #refresh
				ConfigurableEnvironment env = wac.getEnvironment();
				if (env instanceof ConfigurableWebEnvironment) {
					((ConfigurableWebEnvironment) env).initPropertySources(getServletContext(), getServletConfig());
				}
		
				postProcessWebApplicationContext(wac);
				applyInitializers(wac);
				//加载配置文件及整合parent到wac
				wac.refresh();
			}

无论调用方式如何变化，只要是使用ApplicationContext所提供的功能最后都免不了使用公共父类AbstractApplicationContext提供的refresh()进行配置文件加载。



- （3）刷新

onRefresh是FrameworkServlet类中提供的模板方法，在其子类DispatcherServlet中进行了重写，主要用于刷新Spring在web功能实现中所必须使用的全局变量。下面我们介绍在DispatcherServlet中onRefresh的具体实现：

		protected void onRefresh(ApplicationContext context) {
				initStrategies(context);
			}
		
		
		/**
			 * Initialize the strategy objects that this servlet uses.
			 * <p>May be overridden in subclasses in order to initialize further strategy objects.
			 */
			protected void initStrategies(ApplicationContext context) {
				initMultipartResolver(context);
				initLocaleResolver(context);
				initThemeResolver(context);
				initHandlerMappings(context);
				initHandlerAdapters(context);
				initHandlerExceptionResolvers(context);
				initRequestToViewNameTranslator(context);
				initViewResolvers(context);
				initFlashMapManager(context);
			}


派遣器Servlet（DispatcherServlet）通过监听事件得知Servlet的Web应用程序环境初始化或者刷新后，首先在加载的Web应用程序环境(包括主环境和子环境)中查找是不是已经注册了相应的组件，如果查找到注册的组件，就会使用这些组件。如果没有查找到已经注册的相应的组件，派遣器Servlet将会加载缺省的配置策略，这些缺省的配置策略保存在一个属性文件里，这个属性文件和派遣器Servlet在同一个目录里，文件名是DispatcherServlet.properties，通过读取不同组件配置的实现类名，实例化并且初始化这些组件的实现。

![](http://hi.csdn.net/attachment/201009/17/0_1284726932PYMR.gif)

Spring MVC的组件通过数量分为可选组件，单值组件和多值组件。



- 可选组件是在整个流程里可能需要也可能不需要。例如，MultipartResolver。

		private void initMultipartResolver(ApplicationContext context) {  
		    try {  
		        //从配置的Web应用程序环境中查找多部请求解析器  
		        this.multipartResolver = context.getBean(MULTIPART_RESOLVER_BEAN_NAME, MultipartResolver.class);  
		        if (logger.isDebugEnabled()) {  
		            logger.debug("Using MultipartResolver [" + this.multipartResolver + "]");  
		        }  
		    }  
		    catch (NoSuchBeanDefinitionException ex) {  
		        this.multipartResolver = null;  
		        //如果没有多部请求解析器在Web应用程序环境中注册，忽略这种情况，毕竟不是所有的应用程序都需要使用它，多部请求通常会应用到文件上传的情况  
		        if (logger.isDebugEnabled()) {  
		            logger.debug("Unable to locate MultipartResolver with name '" + MULTIPART_RESOLVER_BEAN_NAME +  
		                    "': no multipart request handling provided");  
		        }  
		    }  
		}  

- 单值组件是在整个流程里只需要一个这样的组件。例如，ThemeResolver, LocaleResolver和RequestToViewNameTranslator。


		private void initLocaleResolver(ApplicationContext context) {  
		    try {  
		        //从配置的Web应用程序环境中查找地域请求解析器  
		        this.localeResolver = context.getBean(LOCALE_RESOLVER_BEAN_NAME, LocaleResolver.class);  
		        if (logger.isDebugEnabled()) {  
		            logger.debug("Using LocaleResolver [" + this.localeResolver + "]");  
		        }  
		    }  
		    catch (NoSuchBeanDefinitionException ex) {  
		        //如果没有地域请求解析器在Web应用程序环境中注册，则查找缺省配置的策略，并且根据配置初始化缺省的地域请求解析器，后面将代码注解如何加载默认策略  
		        this.localeResolver = getDefaultStrategy(context, LocaleResolver.class);  
		        if (logger.isDebugEnabled()) {  
		            logger.debug("Unable to locate LocaleResolver with name '" + LOCALE_RESOLVER_BEAN_NAME +  
		                    "': using default [" + this.localeResolver + "]");  
		        }  
		    }  
		}  


（nitThemeResolver()和initRequestToViewNameTranslator()同样是对单值组件进行初始化，他们的实现和initLocaleResolver()具有相同的实现，这里将不重复。）


- 多值组件是在整个流程里可以配置多个这样的实现组件。运行时，轮询查找哪个组件支持当前的HTTP请求，如果找到一个支持的组件则使用这个组件进行处理。



		private void initHandlerMappings(ApplicationContext context) {  
		    this.handlerMappings = null;  
		  
		    if (this.detectAllHandlerMappings) {  
		        //如果配置成为自动检测所有的处理器映射，则在加载的Web应用程序环境中查找所有实现HandlerMapping接口的Bean  
		        Map<String, HandlerMapping> matchingBeans =  
		                BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerMapping.class, true, false);  
		        if (!matchingBeans.isEmpty()) {  
		            this.handlerMappings = new ArrayList<HandlerMapping>(matchingBeans.values());  
		              
		            // 根据这些Bean所实现的Order接口进行排序  
		            OrderComparator.sort(this.handlerMappings);  
		        }  
		    }  
		    else {  
		        //如果没有配置成为自动检测所有的处理器映射，则在Web应用程序环境中查找指定名字为”handlerMapping”的Bean作为处理器映射  
		        try {  
		            HandlerMapping hm = context.getBean(HANDLER_MAPPING_BEAN_NAME, HandlerMapping.class);  
		              
		            //构造单个Bean的集合  
		            this.handlerMappings = Collections.singletonList(hm);  
		        }  
		        catch (NoSuchBeanDefinitionException ex) {  
		            //忽略异常，后面将使用对象引用是否为空判断是否查找成功  
		        }  
		    }  
		  
		    if (this.handlerMappings == null) {  
		        //如果仍然没有查找到注册的HanlderMapping的实现，则使用缺省的配置策略加载处理器映射  
		        this.handlerMappings = getDefaultStrategies(context, HandlerMapping.class);  
		        if (logger.isDebugEnabled()) {  
		            logger.debug("No HandlerMappings found in servlet '" + getServletName() + "': using default");  
		        }  
		    }  
		}  


（initHandlerAdapters(), initHandlerExceptionResolvers()和initViewResolvers ()同样是对多值组件进行初始化，他们和initHandlerMappings()具有相同的实现，这里将不重复。）


#### 2.DispatcherServlet的逻辑处理 ####

根据之前的分析，我们知道在HttpServlet类中分别提供了相应的服务方法，他们是doDelete(),doGet(),doOptions(),doPost()和doTrace(),它会根据请求的不同形式将程序引导至对应的函数进行处理。而FrameWorkServlet重写了HttpServlet中的这些占位符方法，派遣HTTP请求到统一的Spring MVC的控制器方法，进而派遣器Servlet(DispatcherServlet)派遣这个HTTP请求到不同的处理器进行处理和响应。



		/**
			 * Delegate GET requests to processRequest/doService.
			 */
			@Override
			protected final void doGet(HttpServletRequest request, HttpServletResponse response)
					throws ServletException, IOException {
		
				processRequest(request, response);
			}
		
			/**
			 * Delegate POST requests to {@link #processRequest}.
			 */
			@Override
			protected final void doPost(HttpServletRequest request, HttpServletResponse response)
					throws ServletException, IOException {
		
				processRequest(request, response);
			}
		
			/**
			 * Delegate PUT requests to {@link #processRequest}.
			 */
			@Override
			protected final void doPut(HttpServletRequest request, HttpServletResponse response)
					throws ServletException, IOException {
		
				processRequest(request, response);
			}
		
			/**
			 * Delegate DELETE requests to {@link #processRequest}.
			 */
			@Override
			protected final void doDelete(HttpServletRequest request, HttpServletResponse response)
					throws ServletException, IOException {
		
				processRequest(request, response);
			}
		
			......


对于不同的方法，Spring并没有做特殊处理，而是统一将程序再一次地引导至 processRequest(request,response)中。


		/**
			 * Process this request, publishing an event regardless of the outcome.
			 */
			protected final void processRequest(HttpServletRequest request, HttpServletResponse response)
					throws ServletException, IOException {
		
				long startTime = System.currentTimeMillis();
				Throwable failureCause = null;
		
				LocaleContext previousLocaleContext = LocaleContextHolder.getLocaleContext();
				LocaleContext localeContext = buildLocaleContext(request);
		
				RequestAttributes previousAttributes = RequestContextHolder.getRequestAttributes();
				ServletRequestAttributes requestAttributes = buildRequestAttributes(request, response, previousAttributes);
		
				WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
				asyncManager.registerCallableInterceptor(FrameworkServlet.class.getName(), new RequestBindingInterceptor());
		
				initContextHolders(request, localeContext, requestAttributes);
		
				try {
					doService(request, response);
				}
				catch (ServletException ex) {
					failureCause = ex;
					throw ex;
				}
				catch (IOException ex) {
					failureCause = ex;
					throw ex;
				}
				catch (Throwable ex) {
					failureCause = ex;
					throw new NestedServletException("Request processing failed", ex);
				}
		
				finally {
					resetContextHolders(request, previousLocaleContext, previousAttributes);
					if (requestAttributes != null) {
						requestAttributes.requestCompleted();
					}
		
					if (logger.isDebugEnabled()) {
						if (failureCause != null) {
							this.logger.debug("Could not complete request", failureCause);
						}
						else {
							if (asyncManager.isConcurrentHandlingStarted()) {
								logger.debug("Leaving response open for concurrent processing");
							}
							else {
								this.logger.debug("Successfully completed request");
							}
						}
					}
		
					publishRequestHandledEvent(request, response, startTime, failureCause);
				}
			}

函数中已经开始了对请求的处理，虽然把细节转移到了doService函数中实现，但是我们不难看出处理请求前后所做的准备与处理工作。



- （1）为了保证当前线程的LocaleContext以及RequestAttributes可以在当前请求后还能恢复，提取当前线程的两个属性。
- （2）根据当前request创建对应的LocaleContext和RequestAttributes，并绑定到当前线程。
- （3）委托给doService方法进一步处理。
- （4）请求处理结束后恢复线程到原始状态。
- （5）请求处理结束后无论成功与否发布事件进行通知。

继续查看DispatcherServlet类中的doService()方法.

		/**
			 * Exposes the DispatcherServlet-specific request attributes and delegates to {@link #doDispatch}
			 * for the actual dispatching.
			 */
			@Override
			protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {
				if (logger.isDebugEnabled()) {
					String resumed = WebAsyncUtils.getAsyncManager(request).hasConcurrentResult() ? " resumed" : "";
					logger.debug("DispatcherServlet with name '" + getServletName() + "'" + resumed +
							" processing " + request.getMethod() + " request for [" + getRequestUri(request) + "]");
				}
		
				// Keep a snapshot of the request attributes in case of an include,
				// to be able to restore the original attributes after the include.
				Map<String, Object> attributesSnapshot = null;
				if (WebUtils.isIncludeRequest(request)) {
					attributesSnapshot = new HashMap<String, Object>();
					Enumeration<?> attrNames = request.getAttributeNames();
					while (attrNames.hasMoreElements()) {
						String attrName = (String) attrNames.nextElement();
						if (this.cleanupAfterInclude || attrName.startsWith("org.springframework.web.servlet")) {
							attributesSnapshot.put(attrName, request.getAttribute(attrName));
						}
					}
				}
		
				// Make framework objects available to handlers and view objects.
				request.setAttribute(WEB_APPLICATION_CONTEXT_ATTRIBUTE, getWebApplicationContext());
				request.setAttribute(LOCALE_RESOLVER_ATTRIBUTE, this.localeResolver);
				request.setAttribute(THEME_RESOLVER_ATTRIBUTE, this.themeResolver);
				request.setAttribute(THEME_SOURCE_ATTRIBUTE, getThemeSource());
		
				FlashMap inputFlashMap = this.flashMapManager.retrieveAndUpdate(request, response);
				if (inputFlashMap != null) {
					request.setAttribute(INPUT_FLASH_MAP_ATTRIBUTE, Collections.unmodifiableMap(inputFlashMap));
				}
				request.setAttribute(OUTPUT_FLASH_MAP_ATTRIBUTE, new FlashMap());
				request.setAttribute(FLASH_MAP_MANAGER_ATTRIBUTE, this.flashMapManager);
		
				try {
					doDispatch(request, response);
				}
				finally {
					if (!WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
						// Restore the original attribute snapshot, in case of an include.
						if (attributesSnapshot != null) {
							restoreAttributesAfterInclude(request, attributesSnapshot);
						}
					}
				}
			}


我们猜想对请求处理至少应该包括一些诸如寻找handler并进行页面跳转之类的逻辑处理，但是，在doService中我们并没有看到想看的逻辑，相反却同样是一些准备工作，但是这些准备工作却是必不可少的。Spring将已经初始化的功能辅助工具变量，比如localResolver、themeResover等设备在request属性中，而这些属性会在接下来的处理中派上用场。

![](http://hi.csdn.net/attachment/201009/17/0_1284725014KmLU.gif)


经过层层的准备工作，终于在doDispatch函数中看到了完整的请求处理逻辑过程。下面我们重点来看下这段核心处理逻辑：

		/**
			 * Process the actual dispatching to the handler.
			 */
			protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
				HttpServletRequest processedRequest = request;
				HandlerExecutionChain mappedHandler = null;
				boolean multipartRequestParsed = false;
		
				WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
		
				try {
					ModelAndView mv = null;
					Exception dispatchException = null;
		
					try {
						//如果是HTTP多部请求，则转换并且封装成一个简单的HTTP请求  
						processedRequest = checkMultipart(request);
						multipartRequestParsed = (processedRequest != request);
		
						// Determine handler for the current request.
						// 根据处理器映射的配置，取得处理器执行链对象
						mappedHandler = getHandler(processedRequest);
						if (mappedHandler == null || mappedHandler.getHandler() == null) {
							//如果没有发现任何处理器,发送错误信息  
							noHandlerFound(processedRequest, response);
							return;
						}
		
						// Determine handler adapter for the current request.
						//查找支持的处理器适配器 
						HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
		
						// Process last-modified header, if supported by the handler.
						String method = request.getMethod();
						boolean isGet = "GET".equals(method);
						if (isGet || "HEAD".equals(method)) {
							long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
							if (logger.isDebugEnabled()) {
								logger.debug("Last-Modified value for [" + getRequestUri(request) + "] is: " + lastModified);
							}
							if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
								return;
							}
						}
		
						if (!mappedHandler.applyPreHandle(processedRequest, response)) {
							return;
						}
		
						// Actually invoke the handler.
						mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
		
						if (asyncManager.isConcurrentHandlingStarted()) {
							return;
						}
		
						applyDefaultViewName(request, mv);
						mappedHandler.applyPostHandle(processedRequest, response, mv);
					}
					catch (Exception ex) {
						dispatchException = ex;
					}
					processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
				}
				catch (Exception ex) {
					triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
				}
				catch (Error err) {
					triggerAfterCompletionWithError(processedRequest, response, mappedHandler, err);
				}
				finally {
					if (asyncManager.isConcurrentHandlingStarted()) {
						// Instead of postHandle and afterCompletion
						if (mappedHandler != null) {
							mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
						}
					}
					else {
						// Clean up any resources used by a multipart request.
						// 清除Multipart资源
						if (multipartRequestParsed) {
							cleanupMultipart(processedRequest);
						}
					}
				}
			}




看完代码，就知道了doDispath方法中处理http请求的大概流程了：

![](https://images2015.cnblogs.com/blog/808517/201511/808517-20151113104035619-604403700.png)



**总结：**

至此，DispatcherServlet整个逻辑处理过程依然结束。借用两幅图来概括下：


![](https://images2015.cnblogs.com/blog/671185/201607/671185-20160720161437779-899505891.jpg)



![](https://images2015.cnblogs.com/blog/671185/201607/671185-20160720172639154-1984148465.jpg)




















