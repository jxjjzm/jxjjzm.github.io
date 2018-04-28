# Dubbo源码篇之服务引用 #
***

Dubbo服务引用是服务的消费方向注册中心订阅服务提供方提供的服务地址后向服务提供方引用服务的过程。我们先来看下Dubbo服务引用方在Spirng的配置实例：

	<dubbo:referenceid="demoService"interface="com.alibaba.dubbo.demo. DemoService"/>


同前面一样，Dubbo基于sping 扩展schema 利用 DubboNamespaceHandler 实现对自定义schema的解析，将各种不同的配置信息解析成Spring容器中的具体Bean对象。

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

在上面的配置中，Spring容器在启动的过程中会解析自定义的schema元素<dubbo:reference/>转换成实际的配置实现ReferenceBean。

![](https://img-blog.csdn.net/20141201194135362?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXVob25nd2VpX3poYW5xaXU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)


	public void afterPropertiesSet() throws Exception {
	        ......
			......(省略了一些属性设置操作)
			......
	        Boolean b = isInit();
	        if (b == null && getConsumer() != null) {
	            b = getConsumer().isInit();
	        }
	        if (b != null && b.booleanValue()) {
	            getObject();
	        }
	    }


	public Object getObject() throws Exception {
	        return get();
	    }


这里刚开始的逻辑和服务暴露基本一致，因为ReferenceBean继承了ReferenceConfig外也实现了Spring一系列的接口。Spring完成Bean初始化操作后会触发afterPropertiesSet（）的调用，下面继续看ReferenceConfig：

	 public synchronized T get() {
	        if (destroyed){
	            throw new IllegalStateException("Already destroyed!");
	        }
	    	if (ref == null) {
	    		init();
	    	}
	    	return ref;
	    }
	
	
	
	 private void init() {
		    ......
			......(省略了一些消费者配置操作)
			......
	        ref = createProxy(map);
	    }



	private T createProxy(Map<String, String> map) {
			URL tmpUrl = new URL("temp", "localhost", 0, map);
			final boolean isJvmRefer;
	        if (isInjvm() == null) {
	            if (url != null && url.length() > 0) { //指定URL的情况下，不做本地引用
	                isJvmRefer = false;
	            } else if (InjvmProtocol.getInjvmProtocol().isInjvmRefer(tmpUrl)) {
	                //默认情况下如果本地有服务暴露，则引用本地服务.
	                isJvmRefer = true;
	            } else {
	                isJvmRefer = false;
	            }
	        } else {
	            isJvmRefer = isInjvm().booleanValue();
	        }
			
			if (isJvmRefer) {
				URL url = new URL(Constants.LOCAL_PROTOCOL, NetUtils.LOCALHOST, 0, interfaceClass.getName()).addParameters(map);
				invoker = refprotocol.refer(interfaceClass, url);
	            if (logger.isInfoEnabled()) {
	                logger.info("Using injvm service " + interfaceClass.getName());
	            }
			} else {
	            if (url != null && url.length() > 0) { // 用户指定URL，指定的URL可能是对点对直连地址，也可能是注册中心URL
	                String[] us = Constants.SEMICOLON_SPLIT_PATTERN.split(url);
	                if (us != null && us.length > 0) {
	                    for (String u : us) {
	                        URL url = URL.valueOf(u);
	                        if (url.getPath() == null || url.getPath().length() == 0) {
	                            url = url.setPath(interfaceName);
	                        }
	                        if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
	                            urls.add(url.addParameterAndEncoded(Constants.REFER_KEY, StringUtils.toQueryString(map)));
	                        } else {
	                            urls.add(ClusterUtils.mergeUrl(url, map));
	                        }
	                    }
	                }
	            } else { // 通过注册中心配置拼装URL
	            	List<URL> us = loadRegistries(false);
	            	if (us != null && us.size() > 0) {
	                	for (URL u : us) {
	                	    URL monitorUrl = loadMonitor(u);
	                        if (monitorUrl != null) {
	                            map.put(Constants.MONITOR_KEY, URL.encode(monitorUrl.toFullString()));
	                        }
	                	    urls.add(u.addParameterAndEncoded(Constants.REFER_KEY, StringUtils.toQueryString(map)));
	                    }
	            	}
	            	if (urls == null || urls.size() == 0) {
	                    throw new IllegalStateException("No such any registry to reference " + interfaceName  + " on the consumer " + NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion() + ", please config <dubbo:registry address=\"...\" /> to your spring config.");
	                }
	            }
	
	            if (urls.size() == 1) {
	                invoker = refprotocol.refer(interfaceClass, urls.get(0));
	            } else {
	                List<Invoker<?>> invokers = new ArrayList<Invoker<?>>();
	                URL registryURL = null;
	                for (URL url : urls) {
	                    invokers.add(refprotocol.refer(interfaceClass, url));
	                    if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
	                        registryURL = url; // 用了最后一个registry url
	                    }
	                }
	                if (registryURL != null) { // 有 注册中心协议的URL
	                    // 对有注册中心的Cluster 只用 AvailableCluster
	                    URL u = registryURL.addParameter(Constants.CLUSTER_KEY, AvailableCluster.NAME); 
	                    invoker = cluster.join(new StaticDirectory(u, invokers));
	                }  else { // 不是 注册中心的URL
	                    invoker = cluster.join(new StaticDirectory(invokers));
	                }
	            }
	        }
	
	        Boolean c = check;
	        if (c == null && consumer != null) {
	            c = consumer.isCheck();
	        }
	        if (c == null) {
	            c = true; // default true
	        }
	        if (c && ! invoker.isAvailable()) {
	            throw new IllegalStateException("Failed to check the status of the service " + interfaceName + ". No provider available for the service " + (group == null ? "" : group + "/") + interfaceName + (version == null ? "" : ":" + version) + " from the url " + invoker.getUrl() + " to the consumer " + NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion());
	        }
	        if (logger.isInfoEnabled()) {
	            logger.info("Refer dubbo service " + interfaceClass.getName() + " from url " + invoker.getUrl());
	        }
	        // 创建服务代理
	        return (T) proxyFactory.getProxy(invoker);
	    }


这段代码可以体现Dubbo服务消费者消费一个服务的大致过程：ReferenceConfig------>Protocol------>Invoker------->ProxyFactory------>ref 。开始划重点了，我们重点看下 procotol.refer(interface,url):

(1)DubboProtocol.refer

	 public <T> Invoker<T> refer(Class<T> serviceType, URL url) throws RpcException {
	        // create rpc invoker.
	        DubboInvoker<T> invoker = new DubboInvoker<T>(serviceType, url, getClients(url), invokers);
	        invokers.add(invoker);
	        return invoker;
	    }
	    
	    private ExchangeClient[] getClients(URL url){
	        //是否共享连接
	        boolean service_share_connect = false;
	        int connections = url.getParameter(Constants.CONNECTIONS_KEY, 0);
	        //如果connections不配置，则共享连接，否则每服务每连接
	        if (connections == 0){
	            service_share_connect = true;
	            connections = 1;
	        }
	        
	        ExchangeClient[] clients = new ExchangeClient[connections];
	        for (int i = 0; i < clients.length; i++) {
	            if (service_share_connect){
	                clients[i] = getSharedClient(url);
	            } else {
	                clients[i] = initClient(url);
	            }
	        }
	        return clients;
	    }


这里根据Url创建长连接，将底层封装成ExchangeClient对象，构建远程调用Invoker。

(2)RegistryProtocol.refer



	public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
        url = url.setProtocol(url.getParameter(Constants.REGISTRY_KEY, Constants.DEFAULT_REGISTRY)).removeParameter(Constants.REGISTRY_KEY);
        Registry registry = registryFactory.getRegistry(url);
        if (RegistryService.class.equals(type)) {
        	return proxyFactory.getInvoker((T) registry, type, url);
        }

		//根据配置的group分组
        // group="a,b" or group="*"
        Map<String, String> qs = StringUtils.parseQueryString(url.getParameterAndDecoded(Constants.REFER_KEY));
        String group = qs.get(Constants.GROUP_KEY);
        if (group != null && group.length() > 0 ) {
            if ( ( Constants.COMMA_SPLIT_PATTERN.split( group ) ).length > 1
                    || "*".equals( group ) ) {
                return doRefer( getMergeableCluster(), registry, type, url );
            }
        }
        return doRefer(cluster, registry, type, url);
    }


	private <T> Invoker<T> doRefer(Cluster cluster, Registry registry, Class<T> type, URL url) {
			//创建注册服务目录RegistryDirectory并设置注册器
	        RegistryDirectory<T> directory = new RegistryDirectory<T>(type, url);
	        directory.setRegistry(registry);
	        directory.setProtocol(protocol);
			//构建订阅服务的subscribeUrl
	        URL subscribeUrl = new URL(Constants.CONSUMER_PROTOCOL, NetUtils.getLocalHost(), 0, type.getName(), directory.getUrl().getParameters());
	        if (! Constants.ANY_VALUE.equals(url.getServiceInterface())
	                && url.getParameter(Constants.REGISTER_KEY, true)) {
	            registry.register(subscribeUrl.addParameters(Constants.CATEGORY_KEY, Constants.CONSUMERS_CATEGORY,
	                    Constants.CHECK_KEY, String.valueOf(false)));
	        }
	        directory.subscribe(subscribeUrl.addParameter(Constants.CATEGORY_KEY, 
	                Constants.PROVIDERS_CATEGORY 
	                + "," + Constants.CONFIGURATORS_CATEGORY 
	                + "," + Constants.ROUTERS_CATEGORY));
	        return cluster.join(directory);
	    }


![](https://upload-images.jianshu.io/upload_images/1041678-5c94322b3aa14d9a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)



### 总结： ###

![](https://upload-images.jianshu.io/upload_images/1041678-358abe45d733a109.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)




- Dubbo服务引用时序图：

![](https://img-blog.csdn.net/20141201194410349?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXVob25nd2VpX3poYW5xaXU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

- Dubbo服务引用活动图：

![](https://img-blog.csdn.net/20141201194421004?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXVob25nd2VpX3poYW5xaXU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)






















































