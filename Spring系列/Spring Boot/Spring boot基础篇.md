#Spring boot 基础篇
***

### 引言 ###

正式进入正文前，我们首先来回顾下Spring 配置相关的简史：



- 第一阶段：XML配置

在Spring 1.x时代，使用Spring开发满眼都是Xml配置的Bean，随着项目的扩大，我们需要把XML配置文件分放到不同的配置文件里，那时候需要频繁地在开发的类和配置文件之间切换。



- 第二阶段：注解配置

在Spring 2.x时代，随着JDK 1.5带来的注解支持，Spring提供了声明Bean的注解（如@Service @Component），大大减少了配置量。这时Spring圈子里存在着一种争论：注解配置和Xml配置究竟哪个更好？我们的常态选择是应用的基本配置（如数据库配置）用Xml，业务配置用注解。



- 第三阶段：Java配置

从Spring 3.x到现在，Spring提供了Java配置的能力，使用Java配置可以让你更理解你配置的Bean。Spring 4.x和SpringBoot都推荐使用Java配置。

Java配置是通过@Configuration和@Bean来实现的。（@configuration声明当前类是一个配置类，相当于一个Spring配置的Xml文件；@Bean注解在方法上，声明当前方法的返回值为一个Bean）

（说明：何时使用Java配置或注解配置呢？主要的原则是：全局配置使用Java配置（如数据库相关配置），业务Bean的配置使用注解配置（@Service、@Component））。



### 一、Spring boot简介 ###

#### 1.什么是Spring boot ####

随着动态语言的流行（Ruby、Groovy、Scala...），Java 的开发显得格外的笨重：繁多的配置、低下的开发效率、复杂的部署流程以及第三方技术集成难度大。在上述环境下，Spirng boot应运而生。它使用“习惯优于配置”的理念让你的项目快速运行起来。使用Spring boot很容易创建一个独立运行（运行jar,内嵌Servlet容器）、准生产级别的基于Spring框架的项目，使用Spring boot你可以不用或者只需要很少的Spring配置。


#### 2.为什么选择Spring boot ####

- Spring Boot 使编码变简单
- Spring Boot 使配置变简单
- Spring Boot 使部署变简单
- Spring Boot 使监控变简单


#### 3.Spring boot 核心功能 ####

- 独立运行（可以以Jar包的形式）的Spring项目
- 内嵌Servlet容器
- 提供Starter简化Maven配置
- 自动配置Spring
- 准生产的应用监控
- 无代码生成和Xml配置


#### 4.Spring boot 优缺点 ####


**优点：**

- 快速构建项目
- 对主流开发框架的无配置集成
- 项目可独立运行，无须外部依赖Servlet容器
- 提供运行时的应用监控
- 极大地提高了开发、部署效率
- 与云计算的天然集成



**缺点：**

- 书籍文档较少且不够深入
- 如果你不认同Spring框架，这也许是它的缺点，但建议你一定要使用Spring框架


### 二、Spring boot快速初始化与运行 ###

#### （一）快速初始化 ####

Spring Initializr有几种用法。


- 通过Web界面使用。
- 通过Spring Tool Suite使用。
- 通过IntelliJ IDEA使用。
- 使用Spring Boot CLI使用。

（以Web界面、IntelliJ IDEA为例）

**方式一：通过Web界面使用** 



- 1.访问：http://start.spring.io/
- 2.选择构建工具Maven Project、Spring Boot版本以及一些工程基本信息，可参考下图所示:

![](http://7xqch5.com1.z0.glb.clouddn.com/springboot1-1.png)



- 3.点击Generate Project下载项目压缩包
- 4.导入工程。以IDEA为例：
	
	- a.菜单中选择File–>New–>Project from Existing Sources...
	- b.选择解压后的项目文件夹，点击OK
	- c.点击Import project from external model并选择Maven，点击Next到底为止。
	- d.若你的环境有多个版本的JDK，注意到选择Java SDK的时候请选择Java 7以上的版本

**方式二：通过IntelliJ IDEA使用**

IntelliJ IDEA是非常流行的IDE，IntelliJ IDEA 14.1已经支持Spring Boot了。创建Spring Boot操作步骤如下：在File菜单里面选择 New > Project,然后选择Spring Initializr，接着如下图一步步操作即可。

![](http://7xqch5.com1.z0.glb.clouddn.com/springboot1-2.png)

![](http://7xqch5.com1.z0.glb.clouddn.com/springboot1-3.png)

![](http://7xqch5.com1.z0.glb.clouddn.com/springboot1-4.png)

![](http://7xqch5.com1.z0.glb.clouddn.com/springboot1-5.png)

![](http://7xqch5.com1.z0.glb.clouddn.com/springboot1-6.png)


#### （二）运行 ####

- 从IDE中运行
- 作为一个打包后的应用运行：如果使用Spring Boot Maven或Gradle插件创建一个可执行jar，你可以使用 java -
jar 运行应用。
- 使用Maven插件运行：Spring Boot Maven插件包含一个 run 目标，可用来快速编译和运行应用程序，并且跟在IDE运行一样支持热加载。
- 使用Gradle插件运行：Spring Boot Gradle插件也包含一个 bootRun 任务，可用来运行你的应用程序。无论你何时import spring-boot-gradle-plugin ， bootRun 任务总会被添加进
去。


### 三、Spring boot核心概要 ###

#### 1.基本配置 ####

（1）入口类和@SpringBootApplication

Spring Boot 通常有一个名为 *Application的入口类，入口类里有一个main方法，这个main方法其实就是一个标准的Java应用的入口方法。在main方法中使用SpringApplication.run(*Application.class,args),启动Spring Boot应用项目。

@springBootApplication注解主要组合了@Configuration、@EnableAutoConfiguration 、@ComponentScan。（若不使用@SpringBootApplication注解，则可以在入口类上直接使用@Configuration、@EnableAutoConfiguration 、@ComponentScan）


- @Configuration ： 标明该类使用Spring基于Java的配置。
- @EnableAutoConfiguration ： 启用自动配置，让Spring Boot根据类路径中的jar包依赖为当前项目进行自动配置。如：我们添加了spring-boot-starter-web的依赖，项目中也就会引入SpringMVC的依赖，Spring Boot就会自动配置tomcat和SpringMVC。
- @ComponentScan ： 启用组件扫描，这样你写的Web控制类和其他组件才能被自动发现并注册为Spring应用程序上下文里的Bean。


（2）起步依赖

Spring Boot起步依赖大大简化了项目构建说明中的依赖配置，因为常用的依赖聚合于更粗粒
度的依赖。你的构建项目会传递解析到起步依赖中声明的其他依赖。起步依赖不仅能让构建说明中的依赖配置更简单，还根据提供给应用程序的功能将它们组织到一起。起步依赖本质上是一个Maven项目对象模型（POM），定义了对其他库的传递依赖，这些东西加在一起即支持某项功能（很多起步依赖的命名都暗示了它们提供的某种或某类功能）。除官方的start pom 外，还有第三方为Spring Boot所写的starter pom.

起步依赖和项目里的其他依赖没什么区别。也就是说，你可以通过构建工具中的功能，选择性地覆盖它们引入的传递依赖的版本号，排除传递依赖，当然还可以为那些Spring Boot起步依赖没有涵盖的库指定依赖。

（3）自动配置

Spring Boot 自动配置（auto-configuration）尝试根据添加的jar依赖自动配置你的Spring应用。实现自动配置有两种方式，分别是将@EnableAutoConfiguration 或 @SpringBootApplication注解到@Configuration类上。（注：你应该只添加一个@EnableAutoConfiguration注解，通常建议将它添加到主配置类（primary @Configuration）上）

在向应用程序加入Spring Boot时，有个名为spring-boot-autoconfigure的JAR文件，其中包含了
很多配置类。每个配置类都在应用程序的Classpath里，都有机会为应用程序的配置添砖加瓦。这
些配置类里有用于Thymeleaf的配置，有用于Spring Data JPA的配置，有用于Spiring MVC的配置，还有很多其他东西的配置，你可以自己选择是否在Spring应用程序里使用它们。所有这些配置如此与众不同，原因在于它们利用了Spring的条件化配置，这是Spring 4.0引入的新特性。条件化配置允许配置存在于应用程序中，但在满足某些特定条件之前都忽略这个配置。Spring Boot定义了很多更有趣的条件，并把它们运用到了配置类上，这些配置类构成了Spring Boot的自动配置。Spring Boot运用条件化配置的方法是，定义多个特殊的条件化注解，并将它们用到配置类上。


- @ConditionalOnBean 配置了某个特定Bean
- @ConditionalOnMissingBean 没有配置特定的Bean
- @ConditionalOnClass Classpath里有指定的类
- @ConditionalOnMissingClass Classpath里缺少指定的类
- @ConditionalOnExpression 给定的Spring Expression Language（SpEL）表达式计算结果为true
- @ConditionalOnJava Java的版本匹配特定值或者一个范围值
- @ConditionalOnJndi 参数中给定的JNDI位置必须存在一个，如果没有给参数，则要有JNDI
InitialContext
- @ConditionalOnProperty 指定的配置属性要有一个明确的值
- @ConditionalOnResource Classpath里有指定的资源
- @ConditionalOnWebApplication 这是一个Web应用程序
- @ConditionalOnNotWebApplication 这不是一个Web应用程序


自动配置会做出诸如以下配置决策：



- 因为Classpath 里有H2 ， 所以会创建一个嵌入式的H2 数据库Bean ， 它的类型是
javax.sql.DataSource，JPA实现（Hibernate）需要它来访问数据库。
- 因为Classpath里有Hibernate（Spring Data JPA传递引入的）的实体管理器，所以自动配置
会配置与Hibernate 相关的Bean ， 包括Spring 的LocalContainerEntityManager-
FactoryBean和JpaVendorAdapter。
- 因为Classpath里有Spring Data JPA，所以它会自动配置为根据仓库的接口创建仓库实现。
- 因为Classpath里有Thymeleaf，所以Thymeleaf会配置为Spring MVC的视图，包括一个
Thymeleaf的模板解析器、模板引擎及视图解析器。视图解析器会解析相对于Classpath根
目录的/templates目录里的模板。
- 因为Classpath 里有Spring MVC （ 归功于Web 起步依赖）， 所以会配置Spring 的
DispatcherServlet并启用Spring MVC。
- 因为这是一个Spring MVC Web应用程序，所以会注册一个资源处理器，把相对于Classpath
根目录的/static目录里的静态内容提供出来。（这个资源处理器还能处理/public、/resources
和/META-INF/resources的静态内容。）
- 因为Classpath里有Tomcat（通过Web起步依赖传递引用），所以会启动一个嵌入式的Tomcat
容器，监听8080端口。


自动配置（Auto-configuration）是非侵入性的，任何时候你都可以定义自己的配置类来替换自动配置的特定部分。（如果需要查看当前应用启动了哪些自动配置项，你可以在运行应用时打开 --
debug 开关，这将为核心日志开启debug日志级别，并将自动配置相关的日志输出
到控制台。）如果发现启用了不想要的自动配置项，你也可以使用 @EnableAutoConfiguration 注解的exclude属性禁用它们。


#### 2.自定义配置 ####

Spring Boot应用程序有多种设置途径。Spring Boot能从多种属性源获得属性，包括如下几处：

- (1) 命令行参数
- (2) java:comp/env里的JNDI属性
- (3) JVM系统属性
- (4) 操作系统环境变量
- (5) 随机生成的带random.*前缀的属性（在设置其他属性时，可以引用它们，比如${random.
long}）
- (6) 应用程序以外的application.properties或者appliaction.yml文件
- (7) 打包在应用程序内的application.properties或者appliaction.yml文件
- (8) 通过@PropertySource标注的属性源
- (9) 默认属性

这个列表按照优先级排序，也就是说，任何在高优先级属性源里设置的属性都会覆盖低优先
级的相同属性。例如，命令行参数会覆盖其他属性源里的属性。其中，application.properties和application.yml文件能放在以下四个位置：


- (1) 外置，在相对于应用程序运行目录的/config子目录里。
- (2) 外置，在应用程序运行的目录里。
- (3) 内置，在config包内。
- (4) 内置，在Classpath根目录。

同样，这个列表按照优先级排序。也就是说，/config子目录里的application.properties会覆盖
应用程序Classpath里的application.properties中的相同属性。此外，如果你在同一优先级位置同时有application.properties和application.yml，那么application.
yml里的属性会覆盖application.properties里的属性。


（Spring boot常见配置(诸如常规属性配置，日志配置，Profile配置，数据源配置...)此处不一一列出，请参考Spring boot官网）




### 四、Spring boot运行原理 ###

我们开发任何一个Spring boot项目，都会用到如下的启动类：


	@SpringBootApplication
	public class Application {
	    public static void main(String[] args) {
	        SpringApplication.run(Application.class, args);
	    }
	}

从上面的代码可以看出，@SpringBootApplication 和 SpringApplication.run 最为耀眼，所以要揭开SpringBoot的神秘面纱，我们还是得回归到@SpringBootApplication注解和SpringApplication 上来。

#### 1.@SpringBootApplication ####

	@Target({ElementType.TYPE})
	@Retention(RetentionPolicy.RUNTIME)
	@Documented
	@Inherited
	@SpringBootConfiguration
	@EnableAutoConfiguration
	@ComponentScan(
	    excludeFilters = {@Filter(type = FilterType.CUSTOM,classes = {TypeExcludeFilter.class}),
		 @Filter(type = FilterType.CUSTOM,classes = {AutoConfigurationExcludeFilter.class}
	)}
	)
	public @interface SpringBootApplication {
	    ...
	}


虽然定义使用了多个Annotation进行了原信息标注，但实际上重要的只有三个Annotation：

- @Configuration 
- @EnableAutoConfiguration 
- @ComponentScan

**(1)@Configuration**

这里的@Configuration对我们来说不陌生，它就是Java配置（JavaConfig）形式的Spring Ioc容器的配置类使用的那个@Configuration，SpringBoot社区推荐使用基于JavaConfig的配置形式，所以，这里的启动类标注了@Configuration之后，本身其实也是一个IoC容器的配置类。

![](https://i.imgur.com/63WZbAV.png)



**(2)@ComponentScan**

@ComponentScan的功能其实就是自动扫描并加载符合条件的组件（比如@Component和@Repository等）或者bean定义，最终将这些bean定义加载到IoC容器中。

**(3)@EnableAutoConfiguration**

	@Target({ElementType.TYPE})
	@Retention(RetentionPolicy.RUNTIME)
	@Documented
	@Inherited
	@AutoConfigurationPackage
	@Import({AutoConfigurationImportSelector.class})
	public @interface EnableAutoConfiguration {
	    String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";
	
	    Class<?>[] exclude() default {};
	
	    String[] excludeName() default {};
	}

其中，最关键的要属@Import({AutoConfigurationImportSelector.class} ，而EnableAutoConfigurationImportSelector 类中最为关键的是 getCandidateConfigurations 方法中通过 SpringFactoriesLoader.loadFactoryNames 扫描SpringBoot的autoconfigure依赖包中的 META-INF/spring.factories 文件。

![](http://7xqch5.com1.z0.glb.clouddn.com/springboot3-1.png)

总而言之，@EnableAutoConfiguration自动配置的魔法骑士就变成了：从classpath中搜寻所有的META-INF/spring.factories配置文件，并将其中org.springframework.boot.autoconfigure.EnableutoConfiguration对应的配置项通过反射（Java Refletion）实例化为对应的标注了@Configuration的JavaConfig形式的IoC容器配置类，然后汇总为一个并加载到IoC容器。


#### 2.SpringApplication.run ####

这里请允许我借用网上一副流程图来概括SpringApplication.run方法的大致执行流程：

![](http://7xqch5.com1.z0.glb.clouddn.com/springboot3-3.jpg)




#### 附：资料链接 ####

- [Spirngboot 源码](https://github.com/spring-projects/spring-boot "Spirng boot 源码")
- [Springboot 官网](http://projects.spring.io/spring-boot/)
- [Springboot 资料大合集](https://github.com/CFMystery/awesome-spring-boot)












































































