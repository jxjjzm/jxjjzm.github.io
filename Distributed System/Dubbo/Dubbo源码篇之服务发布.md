# Dubbo源码篇之服务发布 #
***

### 一、ServiceConfig------>ProxyFactory------>Invoker ###


	<bean id="demoService"class="com.alibaba.dubbo.demo.provider.DemoServiceImpl" />
	<dubbo:service interface="com.alibaba.dubbo.demo.DemoService" ref=”demoService”/>

由前篇分析可知，Dubbo基于sping 扩展schema 利用 DubboNamespaceHandler 实现对自定义schema的解析，将各种不同的配置信息解析成Spring容器中的具体Bean对象。


- application ------> ApplicationConfig
- module------> ModuleConfig
- registry------> RegistryConfig
- monitor------> MonitorConfig
- provider------> ProviderConfig
- consumer------> ConsumerConfig
- protocol------> ProtocolConfig
- service------> ServiceBean
- reference------> ReferenceBean
- annotation------> AnnotationBean



在上面的配置中，Spring容器在启动的过程中会解析自定义的schema元素<dubbo:service/>转换成实际的配置实现ServiceBean。

![](https://upload-images.jianshu.io/upload_images/1041678-9a91b103c7c98b9d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)


Dubbo服务暴露就是由ServiceBean开始处理逻辑。如上图所示，ServiceBean继承了ServiceConfig并实现了InitializingBean、ApplicationListener等一系列的Spring接口。由于ServiceBean实现了InitializingBean接口，Spring实例化ServiceBean后会调用其afterPropertiesSet()方法：


	@SuppressWarnings({ "unchecked", "deprecation" })
		public void afterPropertiesSet() throws Exception {
	        ......
			......(省略了一些属性设置操作)
			......
	        if (! isDelay()) {
	            export();
	        }
	    }


从上面代码可以看出，ServiceBean<T>类在设置属性后，如果不是延迟暴露则会调用export函数来暴露服务，当然你会问如果是延迟暴露又是什么情况呢？别急，心急吃不了热豆腐，继续往下看，ServiceBean除了实现了Spring的InitializingBean接口外还实现了ApplicationListener事件监听机制，相信熟悉Spring的读者都不会对这个事件机制感到陌生。当Bean初始化完成之后ServiceBean的onApplicationEvent (ApplicationEvent event)方法会被其触发：

	public void onApplicationEvent(ApplicationEvent event) {
	        if (ContextRefreshedEvent.class.getName().equals(event.getClass().getName())) {
	        	if (isDelay() && ! isExported() && ! isUnexported()) {
	                if (logger.isInfoEnabled()) {
	                    logger.info("The service ready on spring started. service: " + getInterface());
	                }
	                export();
	            }
	        }
	    }


在onApplicationEvent(ApplicationEvent event)方法中同样会调用export函数来暴露服务。因此，归根结底，ServiceConfig的export()是Dubbo服务暴露的入口方法。接下来老司机开始开车了，我们来看export()方法：

	 public synchronized void export() {
	        if (provider != null) {
	            if (export == null) {
	                export = provider.getExport();
	            }
	            if (delay == null) {
	                delay = provider.getDelay();
	            }
	        }
	        if (export != null && ! export.booleanValue()) {
	            return;
	        }
	        if (delay != null && delay > 0) {
	            Thread thread = new Thread(new Runnable() {
	                public void run() {
	                    try {
	                        Thread.sleep(delay);
	                    } catch (Throwable e) {
	                    }
	                    doExport();
	                }
	            });
	            thread.setDaemon(true);
	            thread.setName("DelayExportServiceThread");
	            thread.start();
	        } else {
	            doExport();
	        }
	    }


不停留，我们继续往下看doExport(),最终会调用到doExportUrls():

	private void doExportUrls() {
	        List<URL> registryURLs = loadRegistries(true);
	        for (ProtocolConfig protocolConfig : protocols) {
	            doExportUrlsFor1Protocol(protocolConfig, registryURLs);
	        }
	    }


我们一直说Dubbo框架是基于URL为总线的方式来作为配置信息的统一格式的，在此也有体现。因为Dubbo支持多协议配置，所以这里需要遍历所有的协议，分别根据不用的协议把服务export到不同的注册中心上去，具体实现逻辑体现在doExportUrlsFor1Protocol(protocolConfig, registryURLs)：


	private void doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL> registryURLs) {
	        ......
			......（省略了一些构建暴露服务的统一数据模型URL）
			......
	        String scope = url.getParameter(Constants.SCOPE_KEY);
	        //配置为none不暴露
	        if (! Constants.SCOPE_NONE.toString().equalsIgnoreCase(scope)) {
	
	            //配置不是remote的情况下做本地暴露 (配置为remote，则表示只暴露远程服务)
	            if (!Constants.SCOPE_REMOTE.toString().equalsIgnoreCase(scope)) {
	                exportLocal(url);
	            }
	            //如果配置不是local则暴露为远程服务.(配置为local，则表示只暴露本地服务)
	            if (! Constants.SCOPE_LOCAL.toString().equalsIgnoreCase(scope) ){
	                if (logger.isInfoEnabled()) {
	                    logger.info("Export dubbo service " + interfaceClass.getName() + " to url " + url);
	                }
	                if (registryURLs != null && registryURLs.size() > 0
	                        && url.getParameter("register", true)) {
	                    for (URL registryURL : registryURLs) {
	                        url = url.addParameterIfAbsent("dynamic", registryURL.getParameter("dynamic"));
	                        URL monitorUrl = loadMonitor(registryURL);
	                        if (monitorUrl != null) {
	                            url = url.addParameterAndEncoded(Constants.MONITOR_KEY, monitorUrl.toFullString());
	                        }
	                        if (logger.isInfoEnabled()) {
	                            logger.info("Register dubbo service " + interfaceClass.getName() + " url " + url + " to registry " + registryURL);
	                        }
	                        Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(Constants.EXPORT_KEY, url.toFullString()));
	
	                        Exporter<?> exporter = protocol.export(invoker);
	                        exporters.add(exporter);
	                    }
	                } else {
	                    Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, url);
	
	                    Exporter<?> exporter = protocol.export(invoker);
	                    exporters.add(exporter);
	                }
	            }
	        }
	        this.urls.add(url);
	    }


这里不得不提及两点：


- （1）为什么会有本地暴露和远程暴露呢?

在dubbo中我们一个服务可能既是Provider,又是Consumer,因此就存在他自己调用自己服务的情况,如果再通过网络去访问,那自然是舍近求远,因此它是有本地暴露服务的这个设计.从这里我们就知道这个两者的区别：本地暴露是暴露在JVM中,不需要网络通信；远程暴露是将ip,端口等信息暴露给远程客户端,调用时需要网络通信。



- （2）这里我们会发现，实际上发布服务的是protocol,不同的协议export的实现也各不相同（默认的protocol是DubboProtocol，下面我们以DubboProtocol为例）。


### 二、Invoker------>Protocol------>Exporter ###


接前一部分，以DubboProtocol为例（RegistryProtocol有时间待续），我们看下DubboProtocol中的export方法（这才是Dubbo服务暴露的核心所在）：

	public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
	       
	        ......
	
	        openServer(url);
	        
	        ......
	    }


	private void openServer(URL url) {
	        // find server.
	        String key = url.getAddress();
	        //client 也可以暴露一个只有server可以调用的服务。
	        boolean isServer = url.getParameter(Constants.IS_SERVER_KEY,true);
	        if (isServer) {
	        	ExchangeServer server = serverMap.get(key);
	        	if (server == null) {
	        		serverMap.put(key, createServer(url));
	        	} else {
	        		//server支持reset,配合override功能使用
	        		server.reset(url);
	        	}
	        }
	    }



	private ExchangeServer createServer(URL url) {
	         ......
	         ......
	         ......
	        ExchangeServer server;
	        try {
	            server = Exchangers.bind(url, requestHandler);
	        } catch (RemotingException e) {
	            throw new RpcException("Fail to start server(url: " + url + ") " + e.getMessage(), e);
	        }
	         ......
	         ......
	         ......
	        return server;
	    }


我们可以看到，在openServer和createServer中，最终调用的是： server = Exchangers.bind(url, requestHandler); 


	public static ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
	        if (url == null) {
	            throw new IllegalArgumentException("url == null");
	        }
	        if (handler == null) {
	            throw new IllegalArgumentException("handler == null");
	        }
	        url = url.addParameterIfAbsent(Constants.CODEC_KEY, "exchange");
			//getExchanger方法追究下去实际上调用的是ExtensionLoader.getExtensionLoader(Exchanger.class).getExtension(type)
	        return getExchanger(url).bind(url, handler);
	    }


这里Exchanger的默认实现只有一个：HeaderExchanger ，我们直接转到HeaderExchanger:


	public ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
	        return new HeaderExchangeServer(Transporters.bind(url, new DecodeHandler(new HeaderExchangeHandler(handler))));
	    }

HeaderExchangeServe需要一个Server类型的参数，来自Transporters.bind(URL url, ChannelHandler... handlers)：

	 public static Server bind(URL url, ChannelHandler... handlers) throws RemotingException {
	        if (url == null) {
	            throw new IllegalArgumentException("url == null");
	        }
	        if (handlers == null || handlers.length == 0) {
	            throw new IllegalArgumentException("handlers == null");
	        }
	        ChannelHandler handler;
	        if (handlers.length == 1) {
	            handler = handlers[0];
	        } else {
	            handler = new ChannelHandlerDispatcher(handlers);
	        }
	        return getTransporter().bind(url, handler);
	    }

	public static Transporter getTransporter() {
	        return ExtensionLoader.getExtensionLoader(Transporter.class).getAdaptiveExtension();
	    }


getTransporter()获取的Transporter实例来源于扩展配置，默认返回一个NettyTransporter：

![](https://upload-images.jianshu.io/upload_images/1041678-be841049c17f05b5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

	public class NettyTransporter implements Transporter {
	
	    public static final String NAME = "netty";
	    
	    public Server bind(URL url, ChannelHandler listener) throws RemotingException {
	        return new NettyServer(url, listener);
	    }
	
	    public Client connect(URL url, ChannelHandler listener) throws RemotingException {
	        return new NettyClient(url, listener);
	    }
	
	}




### 总结： ###


![](https://upload-images.jianshu.io/upload_images/1041678-e21933b0fa25ffa8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

- Dubbo服务暴露时序图：

![](https://upload-images.jianshu.io/upload_images/1041678-f90a214d6083df42.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)



- Dubbo服务暴露活动图：

![](https://img-blog.csdn.net/20141201192637144?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXVob25nd2VpX3poYW5xaXU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)


其实，Dubbo服务暴露整体来看也就是两大部分（1）Service To Invoker （2）Invoker To Export ，简单概括如下图所示：

![](https://i.imgur.com/jU1dpmZ.png)




















































































