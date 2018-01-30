### Spring源码篇之Spring的整体架构 ###
***

### 一、Spring整体架构 ###

![](http://static.oschina.net/uploads/img/201403/19135321_1cQf.png)

Spring 总共大约有20个模块，由1300多个不同的文件构成。而这些组件被分别整合在以下几个个模块集合中：

#### （1）Core Container ####

Core Container(核心容器)包含有Core、Beans、Context和Expression Language模块 ，Core和Beans模块是框架的核心模块，提供IoC(控制反转)和DI(依赖注入)特性。

- core模块主要包含Spring框架基本的核心工具类，Spring的其他组件都要使用到这个包里的类，Core模块是其他组件的基本核心。（我们也可以在自己的应用系统中使用这些工具类）
- Beans模块是所有应用都要用到的，它包含访问配置文件、创建和管理bean以及进行IoC(控制反转)和DI(依赖注入)操作相关的所有类。其中BeanFactory接口是Spring框架中的核心接口，它是工厂模式的具体实现。它提供对Factory模式的经典实现来消除对程序性单例模式的需要，并真正地允许你从程序逻辑中分离出依赖关系和配置。
- Context模块构建于Core和Beans模块基础之上，提供了一种类似于JNDI注册器的框架式的对象访问方法。Context模块继承了Beans的特性，为Spring核心提供了大量扩展，添加了对国际化(如资源绑定)、事件传播、资源加载和对Context的透明创建的支持。ApplicationContext接口是Context模块的关键。
- Expression Language模块提供了一个强大的表达式语言用于在运行时查询和操纵对象，该语言支持设置/获取属性的值，属性的分配，方法的调用，访问数组上下文、容器和索引器、逻辑和算术运算符、命名变量以及从Spring的IoC容器中根据名称检索对象


#### （2）Data Access/Integration ####

数据访问及集成（Data Access/Integration）层包含有JDBC、ORM、OXM、JMS和Transactions几个模块：

- JDBC模块是Spring 提供的JDBC抽象框架的主要实现模块，用于简化Spring JDBC。主要是提供JDBC模板方式、关系数据库对象化方式、SimpleJdbc方式、事务管理来简化JDBC编程，主要实现类是JdbcTemplate、SimpleJdbcTemplate以NamedParameterJdbcTemplate。
- ORM模块为流行的对象-关系映射API，如JPA、JDO、Hibernate、iBatis等，提供了一个交互层，利用ORM封装包，可以混合使用所有Spring提供的特性进行O/R映射。
- JMS模块主要包含了一些制造和消费消息的特性。
- OXM模块提供了一个对Object/XML映射实现的抽象层，Object/XML映射实现包括JAXB、Castor、XMLBeans、JiBX和XStream。
- Transaction模块支持编程和声明性的事务管理，这些事务类必须实现特定的接口，并且对所有的POJO都使用。

#### （3）Web ####

Web上下文模块建立在应用程序上下文模块之上，为基于Web的应用程序提供了上下文，所以Spring框架支持与Jakarta Struts的集成。Web模块还简化了处理多部分请求以及将请求参数绑定到域对象的工作。Web层包含了Web、Web-Servlet、Web-Struts和Web、Porlet模块：


- Web模块：提供了基础的面向Web的集成特性，例如，多文件上传、使用Servlet listeners初始化IoC容器以及一个面向Web的应用上下文，它还包含了Spring远程支持中Web的相关部分
- Web-Servlet模块web.servlet.jar：该模块包含Spring的model-view-controller(MVC)实现，Spring的MVC框架使得模型范围内的代码和web forms之间能够清楚地分离开来，并与Spring框架的其他特性集成在一起。
- Web-Struts模块：该模块提供了对Struts的支持，使得类在Spring应用中能够与一个典型的Struts Web层集成在一起。（注意，该支持在Spring 3.0中是deprecated的）。
- Web-Porlet模块：提供了用于Portlet环境和Web-Servlet模块的MVC的实现


#### （4）AOP ####

AOP模块提供了一个符合AOP联盟标准的面向切面编程的实现，它让你可以定义例如方法拦截器和切点，从而将逻辑代码分开，降低它们之间的耦合性，利用source-level的元数据功能，还可以将各种行为信息合并到你的代码中。通过配置管理特性，Spring AOP模块直接将面向切面的编程功能集成到了Spring框架中，所以可以很容易地使Spring框架管理的任何对象支持AOP。Spring AOP模块为基于Spring的应用程序中的对象提供了事务管理服务。通过使用Spring AOP，不用依赖EJB组件，就可以将声明性事务管理集成到应用程序中。


- aspects模块提供了对AspectJ的集成支持。
- instrumentation模块提供了class instrumentation支持和classloader实现，使得可以在特定的应用服务器上使用。


#### （5）Test ####

Test模块支持使用Junit和TestNG对Spring组件进行测试。


### 二、Spirng 源码结构 ###

#### 1.源码结构图 ####

![](https://i.imgur.com/V2Fa54o.png)

#### 2.模块导图 ####

![](https://i.imgur.com/ExtI7Ep.png)


#### 3.Spring的骨骼架构 ####

虽然Spring总共有十几个组件，但是我们不难发现Spring框架中的核心组件只有三个：Core、Context和Beans，它们构建起了整个Spring的骨骼架构，没有它们就不可能有AOP、WEB等上层的特性功能。如果再在他们三个中选出核心的话，那就非Beans组件莫属了，个人理解其实Spring 就是面向Bean的编程（BOP,Bean Oriented Programming），Bean在Spring 中才是真正的主角。Bean 在 Spring 中作用就像 Object 对 OOP 的意义一样，没有对象的概念就像没有面向对象编程，Spring 中没有 Bean 也就没有 Spring 存在的意义。

Bean 包装的是 Object，而 Object 必然有数据，如何给这些数据提供生存环境就是 Context 要解决的问题，对 Context 来说他就是要发现每个 Bean 之间的关系，为它们建立这种关系并且要维护好这种关系。所以 Context 就是一个 Bean 关系的集合，这个关系集合又叫 Ioc 容器，一旦建立起这个 Ioc 容器后 Spring 就可以为你工作了。那 Core 组件又有什么用武之地呢？其实 Core 就是发现、建立和维护每个 Bean 之间的关系所需要的一些列的工具，从这个角度看来，Core 这个组件叫 Util 更能让人理解。

请允许我借用一个比喻让你更好地理解这三者之间的关系，如果把Bean 比作一场演出中的演员的话，那 Context 就是这场演出的舞台背景，而 Core 应该就是演出的道具了。只有他们在一起才能具备能演出一场好戏的最基本的条件。当然有最基本的条件还不能使这场演出脱颖而出，还要他表演的节目足够的精彩，这些节目就是 Spring 能提供的特色功能了。

![](https://www.ibm.com/developerworks/cn/java/j-lo-spring-principle/image002.gif)

下面针对这三个核心组件做个简单的概述，详细请参考后续Spring源码系列。


**I、Bean组件概述**

Bean 组件在 Spring 的 org.springframework.beans 包下。这个包下的所有类主要解决了三件事：Bean 的定义、Bean 的创建以及对 Bean 的解析。

- Bean的定义

![](https://www.ibm.com/developerworks/cn/java/j-lo-spring-principle/image004.png)

Bean 的定义就是完整的描述了在 Spring 的配置文件中你定义的 <bean/> 节点中所有的信息，包括各种子节点。当 Spring 成功解析你定义的一个 <bean/> 节点后，在 Spring 的内部他就被转化成 BeanDefinition 对象。以后所有的操作都是对这个对象完成的。


- Bean的解析

![](https://www.ibm.com/developerworks/cn/java/j-lo-spring-principle/image005.png)

Bean 的解析主要就是对 Spring 配置文件的解析。Bean 的解析过程非常复杂，功能被分的很细，因为这里需要被扩展的地方很多，必须保证有足够的灵活性，以应对可能的变化。当然这里必然涉及到对默认标签和自定义标签的解析。


- Bean的创建

![](https://www.ibm.com/developerworks/cn/java/j-lo-spring-principle/origin_image009.png)

Spring Bean 的创建是典型的工厂模式，他的顶级接口是 BeanFactory 。BeanFactory 有三个子类：ListableBeanFactory、HierarchicalBeanFactory 和 AutowireCapableBeanFactory。但是通过分析源码我们可以发现最终的默认实现类是DefaultListableBeanFactory，他实现了所有的接口，是整个bean加载的核心部分。那为何要定义这么多层次的接口呢？查阅这些接口的源码和说明发现，每个接口都有他使用的场合，它主要是为了区分在 Spring 内部在操作过程中对象的传递和转化过程中，对对象的数据访问所做的限制。例如 ListableBeanFactory 接口表示这些 Bean 是可列表的，而 HierarchicalBeanFactory 表示的是这些 Bean 是有继承关系的，也就是每个 Bean 有可能有父 Bean。AutowireCapableBeanFactory 接口定义 Bean 的自动装配规则。这四个接口共同定义了 Bean 的集合、Bean 之间的关系、以及 Bean 行为。





**II、Context组件**


Context 在 Spring 的 org.springframework.context 包下，它实际上就是给 Spring 提供一个运行时的环境，用以保存各个对象的状态。ApplicationContext接口是Context模块的关键，Context 的顶级父类，他除了能标识一个应用环境的基本信息外，他还继承了五个接口，这五个接口主要是扩展了 Context 的功能。

![](https://www.ibm.com/developerworks/cn/java/j-lo-spring-principle/origin_image006.png)

从图中我们可以看出，ApplicationContext继承了 BeanFactory，这也进一步说明了 Spring 容器中运行的主体对象是 Bean。另外 ApplicationContext 继承了 ResourceLoader 接口，使得 ApplicationContext 可以访问到任何外部资源。ApplicationContext 的子类主要包含两个方面：

- ConfigurableApplicationContext 表示该 Context 是可修改的，也就是在构建 Context 中用户可以动态添加或修改已有的配置信息，它下面又有多个子类，其中最经常使用的是可更新的 Context，即 AbstractRefreshableApplicationContext 类。
- WebApplicationContext 顾名思义，就是为 web 准备的 Context 他可以直接访问到 ServletContext，通常情况下，这个接口使用的少。

再往下分就是按照构建 Context 的文件类型，接着就是访问 Context 的方式。这样一级一级构成了完整的 Context 等级层次。总体来说 ApplicationContext 必须要完成以下几件事：

- 标识一个应用环境
- 利用 BeanFactory 创建 Bean 对象
- 保存对象关系表
- 能够捕获各种事件

Context 作为 Spring 的 Ioc 容器，基本上整合了 Spring 的大部分功能，或者说是大部分功能的基础。


**III、Core组件**

Core 组件作为 Spring 的核心组件，他其中包含了很多的关键类，其中一个重要组成部分就是定义了资源的访问方式。这种把所有资源都抽象成一个接口的方式很值得在以后的设计中拿来学习。Core 组件中还有很多类似的方式，下面以资源Resource为例。


- Resource

![](https://www.ibm.com/developerworks/cn/java/j-lo-spring-principle/origin_image007.png)

从上图可以看出 Resource 接口封装了各种可能的资源类型，也就是对使用者来说屏蔽了文件类型的不同。对资源的提供者来说，如何把资源包装起来交给其他人用这也是一个问题，我们看到 Resource 接口继承了 InputStreamSource 接口，这个接口中有个 getInputStream 方法，返回的是 InputStream 类。这样所有的资源都被可以通过 InputStream 这个类来获取，所以也屏蔽了资源的提供者。另外还有一个问题就是加载资源的问题，也就是资源的加载者要统一，从上图中可以看出这个任务是由 ResourceLoader 接口完成，他屏蔽了所有的资源加载者的差异，只需要实现这个接口就可以加载所有的资源，他的默认实现是 DefaultResourceLoader。

- Context & Resource 

![](https://www.ibm.com/developerworks/cn/java/j-lo-spring-principle/image008.png)

从上图可以看出，Context 是把资源的加载、解析和描述工作委托给了 ResourcePatternResolver 类来完成，他相当于一个接头人，他把资源的加载、解析和资源的定义整合在一起便于其他组件使用。


**总结：**

Spring Ioc 容器实际上就是 Context 组件结合其他两个组件（Bean和Core）共同构建的一个 Bean 关系网。



详细参详：[Spring 框架的设计理念与设计模式分析](https://www.ibm.com/developerworks/cn/java/j-lo-spring-principle/ "Spring 框架的设计理念与设计模式分析")























































































