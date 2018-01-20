### Spring入门篇之Spring IOC ###
***

### 一、Spring IOC 概述 ###

#### 1.IOC背景 ####

![](https://i.imgur.com/Z2nVBIi.png)

如果我们打开机械式手表的后盖，就会看到与上面类似的情形，各个齿轮分别带动时针、分针和秒针顺时针旋转，从而在表盘上产生正确的时间。图1中描述的就是 这样的一个齿轮组，它拥有多个独立的齿轮，这些齿轮相互啮合在一起，协同工作，共同完成某项任务。我们可以看到，在这样的齿轮组中，如果有一个齿轮出了问题，就可能会影响到整个齿轮组的正常运转。

![](https://i.imgur.com/mP6t42N.png)

齿轮组中齿轮之间的啮合关系,与软件系统中对象之间的耦合关系非常相似。我们都知道，在采用面向对象方法设计的软件系统中，它的底层实现都是由N个对象组成的，所有的对象通过彼此的合作，最终实现系统的业务逻辑。对象之间的耦合关系是无法避免的，也是必要的，这是协同工作的基础。现在，伴随着工业级应用的规模越来越庞大，对象之间的依赖关系也越来越复杂，经常会出现对象之间的多重依赖性关系，因此，架构师和设计师对于系统的分析和设计，将面临更大的挑战。对象之间耦合度过高的系统，必然会出现牵一发而动全身的情形。

耦合关系不仅会出现在对象与对象之间，也会出现在软件系统的各模块之间，以及软件系统和硬件系统之间。如何降低系统之间、模块之间和对象之间的耦合度，是软件工程永远追求的目标之一。为了解决对象之间的耦合度过高的问题，1996年，软件专家Michael Mattson在一篇有关探讨面向对象框架的文章中，首先提出了IOC这个理论，用来实现对象之间的“解耦”，目前这个理论已经被成功地应用到实践当中，很多的J2EE项目均采用了IOC框架产品Spring。IOC理论提出的观点大体是这样的：借助于“第三方”实现具有依赖关系的对象之间的解耦，如下图：

![](https://i.imgur.com/rRdAC5Q.jpg)

大家看到了吧，由于引进了中间位置的“第三方”，也就是IOC容器，使得A、B、C、D这4个对象没有了耦合关系，齿轮之间的传动全部依靠“第三方”了， 全部对象的控制权全部上缴给“第三方”IOC容器，所以，IOC容器成了整个系统的关键核心，它起到了一种类似“粘合剂”的作用，把系统中的所有对象粘合 在一起发挥作用，如果没有这个“粘合剂”，对象与对象之间会彼此失去联系，这就是有人把IOC容器比喻成“粘合剂”的由来。我们再来做个试验：把上图中间的IOC容器拿掉，然后再来看看这套系统：

![](https://i.imgur.com/6I9ru8i.png)

我们现在看到的画面，就是我们要实现整个系统所需要完成的全部内容。这时候，A、B、C、D这4个对象之间已经没有了耦合关系，彼此毫无联系，这样的话， 当你在实现A的时候，根本无须再去考虑B、C和D了，对象之间的依赖关系已经降低到了最低程度。所以，如果真能实现IOC容器，对于系统开发而言，这将是 一件多么美好的事情，参与开发的每一成员只要实现自己的类就可以了，跟别人没有任何关系！

IOC是Inversion of Control的缩写，多数书籍翻译成“控制反转”，还有些书籍翻译成为“控制反向”或者“控制倒置”。我们再来看看，控制反转(IOC)到底为什么要起这么个名字？我们来对比一下：软件系统在没有引入IOC容器之前，如图1所示，对象A依赖于对象B，那么对象A在初始化或者运行到某一点的时候，自己必须主动去创建对象B或者使用已经创建的对象B。无论是创建还是使用对象B，控制权都在自己手上；软件系统在引入IOC容器之后，这种情形就完全改变了，如图3所示，由于IOC容器的加入，对象A与对象B之间失去了直接联系，所以，当对象A运行到需要对象B的时候，IOC容器会主动创建一个对象B注入到对象A需要的地方。通过前后的对比，我们不难看出来：对象A获得依赖对象B的过程,由主动行为变为了被动行为，控制权颠倒过来了，这就是“控制反转”这个名称的由来。



#### 2.IOC 简介 ####

Ioc（Inversion of Control，即“控制反转”），不是什么技术，而是一种设计思想。在Java开发中，Ioc意味着将你设计好的对象交给容器控制，而不是传统的在你的对象内部直接控制。如何理解好Ioc呢？理解好Ioc的关键是要明确“谁控制谁，控制什么，为何是反转（有反转就应该有正转了），哪些方面反转了”，那我们来深入分析一下：


- **谁控制谁，控制什么**：传统Java SE程序设计，我们直接在对象内部通过new进行创建对象，是程序主动去创建依赖对象；而IoC是有专门一个容器来创建这些对象，即由Ioc容器来控制对象的创建；谁控制谁？当然是IoC 容器控制了对象；控制什么？那就是主要控制了外部资源获取（不只是对象包括比如文件等）。
- **为何是反转，哪些方面反转了**：有反转就有正转，传统应用程序是由我们自己在对象中主动控制去直接获取依赖对象，也就是正转；而反转则是由容器来帮忙创建及注入依赖对象；为何是反转？因为由容器帮我们查找及注入依赖对象，对象只是被动的接受依赖对象，所以是反转；哪些方面反转了？依赖对象的获取被反转了。

Spring所倡导的开发方式就是如此，所有的类都会在spring容器中登记，告诉spring你是个什么东西，你需要什么东西，然后spring会在系统运行到适当的时候，把你要的东西主动给你，同时也把你交给其他需要你的东西。所有的类的创建、销毁都由 spring来控制，也就是说控制对象生存周期的不再是引用它的对象，而是spring。对于某个具体的对象而言，以前是它控制其他对象，现在是所有对象都被spring控制，所以这叫控制反转。


#### 3.IOC和DI ####

DI（Dependency Injection），即“依赖注入”：组件之间依赖关系由容器在运行期决定，形象的说，即由容器动态的将某个依赖关系注入到组件之中。依赖注入的目的并非为软件系统带来更多功能，而是为了提升组件重用的频率，并为系统搭建一个灵活、可扩展的平台。通过依赖注入机制，我们只需要通过简单的配置，而无需任何代码就可指定目标需要的资源，完成自身的业务逻辑，而不需要关心具体的资源来自何处，由谁实现。理解DI的关键是：“谁依赖谁，为什么需要依赖，谁注入谁，注入了什么”，那我们来深入分析一下：


- **谁依赖于谁**：当然是应用程序依赖于IoC容器；
- **为什么需要依赖**：应用程序需要IoC容器来提供对象需要的外部资源；
- **谁注入谁**：很明显是IoC容器注入应用程序某个对象，应用程序依赖的对象；
- **注入了什么**：就是注入某个对象所需要的外部资源（包括对象、资源、常量数据）。

IoC和DI由什么关系呢？其实它们是同一个概念的不同角度描述，由于控制反转概念比较含糊（可能只是理解为容器控制对象这一个层面，很难让人想到谁来维护对象关系），所以2004年大师级人物Martin Fowler又给出了一个新的名字：“依赖注入”，相对IoC 而言，“依赖注入”明确描述了“被注入对象依赖IoC容器配置依赖对象”。

对于Spring Ioc这个核心概念，我相信每一个学习Spring的人都会有自己的理解。这种概念上的理解没有绝对的标准答案，仁者见仁智者见智。 理解了IoC和DI的概念后，一切都将变得简单明了，剩下的工作只是在框架中堆积木而已。

#### 4.IOC 好处 ####

我们还是从USB的例子说起，使用USB外部设备比使用内置硬盘，到底带来什么好处？
　　


- 第一、USB设备作为电脑主机的外部设备，在插入主机之前，与电脑主机没有任何的关系，只有被我们连接在一起之后，两者才发生联系，具有相关性。所以，无论两者中的任何一方出现什么的问题，都不会影响另一方的运行。这种特性体现在软件工程中，就是可维护性比较好，非常便于进行单元测试，便于调试程序和诊断故障。代码中的每一个Class都可以单独测试，彼此之间互不影响，只要保证自身的功能无误即可，这就是组件之间低耦合或者无耦合带来的好处。
- 第二、USB设备和电脑主机的之间无关性，还带来了另外一个好处，生产USB设备的厂商和生产电脑主机的厂商完全可以是互不相干的人，各干各事，他们之间唯一需要遵守的就是USB接口标准。这种特性体现在软件开发过程中，好处可是太大了。每个开发团队的成员都只需要关心实现自身的业务逻辑，完全不用去关心 其它的人工作进展，因为你的任务跟别人没有任何关系，你的任务可以单独测试，你的任务也不用依赖于别人的组件，再也不用扯不清责任了。所以，在一个大中型 项目中，团队成员分工明确、责任明晰，很容易将一个大的任务划分为细小的任务，开发效率和产品质量必将得到大幅度的提高。
- 第三、同一个USB外部设备可以插接到任何支持USB的设备，可以插接到电脑主机，也可以插接到DV机，USB外部设备可以被反复利用。在软件工程中，这种特性就是可复用性好，我们可以把具有普遍性的常用组件独立出来，反复利用到项目中的其它部分，或者是其它项目，当然这也是面向对象的基本特征。显然，IOC不仅更好地贯彻了这个原则，提高了模块的可复用性。符合接口标准的实现，都可以插接到支持此标准的模块中。
- 第四、同USB外部设备一样，模块具有热插拔特性。IOC生成对象的方式转为外置方式，也就是把对象生成放在配置文件里进行定义，这样，当我们更换一个实现子类将会变得很简单，只要修改配置文件就可以了，完全具有热插拨的特性。
　　

（以上几点好处，难道还不足以打动你吗......）


### 二、Spring IOC 使用 ###

#### 1、Spring容器和Bean管理 ####

**I、Spring 容器**

在Spring中，任何的Java类和JavaBean都被当成Bean处理，这些Bean通过容器管理和应用。Spring提供了两种容器类型：BeanFactory和ApplicationContext。



- BeanFactory —— 基础类型IoC容器，提供完整的IoC服务支持。如果没有特殊指定，默认采用延迟初始化策略（lazy-load）。只有当客户端对象需要访问容器中的某个受管对象的时候，才对该受管对象进行初始化以及依赖注入操作。所以，相对来说，容器启动初期速度较快，所需要的资源有限。对于资源有限，并且功能要求不是很严格的场景，BeanFactory是比较合适的IoC容器选择。


- ApplicationContext（推荐使用） —— ApplicationContext在BeanFactory的基础上构建（ApplicationContext间接继承自BeanFactory，所以说它是构建于BeanFactory之上的IoC容器。），是相对比较高级的容器实现，除了拥有BeanFactory的所有支持，ApplicationContext还提供了其他高级特性，比如事件发布、国际化信息支持等。ApplicationContext所管理的对象，在该类型容器启动之后，默认全部初始化并绑定完成。所以，相对于BeanFactory来说，ApplicationContext要求更多的系统资源，同时，因为在启动时就完成所有初始化，容器启动时间较之BeanFactory也会长一些。在那些系统资源充足，并且要求更多功能的场景中，ApplicationContext类型的容器是比较合适的选择。

		//加载文件系统中的配置文件实例化
		String conf = "C:\applicationContext.xml";
		ApplicationContext ac = new FileSystemXmlApplicationContext(conf);
		//加载工程classpath下的配置文件实例化
		String conf = "applicationContext.xml";
		ApplicationContext ac = new ClassPathXmlApplicationContext(conf);


从本质上讲，BeanFactory和ApplicationContext仅仅只是一个维护Bean定义以及相互依赖关系的高级工厂接口。通过BeanFactory和ApplicationContext我们可以访问bean定义，首先在容器配置文件applicationContext.xml中添加Bean定义，然后再创建BeanFactory和ApplicationContext容器对象后调用getBean（） 方法获取Bean的实例即可。

**II、Bean管理**

**（1）Bean的实例化**

Spring容器创建Bean对象的方法有以下3种：

- 使用构造器来实例化

		//id或name属性用于指定Bean名称，用于从Spring中查找这个Bean对象；class用于指定Bean类型，会自动调用无参数构造器创建对象
		<bean id="calendarObj1" class="java.util.GregorianCalendar"/>
		<bean id="calendarObj2" class="java.util.GregorianCalendar"/>

- 使用静态工厂方法实例化

		//id属性用于指定Bean名称；class属性用于指定Be工厂类型；factory-method属性用于指定工厂中创建Bean对象的方法，必须用static修饰的方法。
		<bean id="calendarObj2" class="java.util.Calendar" factory-method="getInstance"/>


- 使用实例工厂方法实例化

		//id用于指定Bean名称；factory-bean属性用于指定工厂Bean对象;factory-method属性用于指定工厂中创建Bean对象的方法
		<bean id="calendarObj3" class="java.util.GregorianCalendar"/>
		<bean id="dateObj" factory-bean="calendarObj3" factory-method="getTime">


（

- 在Spring容器中，每个Bean都需要有名字，该名字可以用id或name属性指定，id属性比name严格，要求具有唯一性，不允许用“/”等特殊字符。
- Bean的别名:为已定义好的Bean，在增加另外一个名字引用

		<alias name="fromName" alias="toName">


）


**（2）Bean的作用域**

Spring中为Bean定义了5中作用域，分别为singleton（单例）、prototype（原型）、request、session和global session，5种作用域说明如下：

- singleton：单例模式，Spring IoC容器中只会存在一个共享的Bean实例，无论有多少个Bean引用它，始终指向同一对象。Singleton作用域是Spring中的缺省作用域，也可以显示的将Bean定义为singleton模式，配置为：

		<bean id="userDao" class="com.ioc.UserDaoImpl" scope="singleton"/>

- prototype:原型模式，每次通过Spring容器获取prototype定义的bean时，容器都将创建一个新的Bean实例，每个Bean实例都有自己的属性和状态，而singleton全局只有一个对象。根据经验，对有状态的bean使用prototype作用域，而对无状态的bean使用singleton作用域。


- request：在一次Http请求中，容器会返回该Bean的同一实例。而对不同的Http请求则会产生新的Bean，而且该bean仅在当前Http Request内有效。


- session：在一次Http Session中，容器会返回该Bean的同一实例。而对不同的Session请求则会创建新的实例，该bean实例仅在当前Session内有效。


- global Session：在一个全局的Http Session中，容器会返回该Bean的同一个实例，仅在使用portlet context时有效。  



**（3）Bean的生命周期**

我们知道一个对象的生命周期：创建（实例化-初始化）-使用-销毁，而在Spring中，Bean对象周期当然遵从这一过程，但是Spring提供了许多对外接口，允许开发者对三个过程（实例化、初始化、销毁）的前后做一些操作。 （说明：在Spring Bean中，实例化是为bean对象开辟空间（具体可以理解为构造函数的调用），初始化则是对属性的初始化）


- 指定初始化回调方法

		<bean id="exampleBean" class="com.foo.ExampleBean" init-method="init"></bean>

- 指定销毁回调方法，仅适用于singleton模式的bean

		<bean id="exampleBean" class="com.foo.ExampleBean" destroy-method="destroy"></bean>

(在顶级元素beans中的default-init-method属性可以为容器所有bean指定初始化回调方法，在顶级元素beans中的default-destroy-method属性可以为容器所有bean指定销毁回调方法)

- 在ApplicationContext实现的默认行为就是在启动时将所有singleton bean 提前进行实例化，如果不想让一个singleton bean在ApplicationContext初始化时被提前实例化，可以使用bean元素的lazy-init="true"属性改变。（一个延迟初始化bean将在第一次被用到时实例化）

		<bean id="exampleBean" lazy-init="true" class="com.foo.ExampleBean"></bean>

(在顶级元素beans中的default-lazy-init属性可以为容器所有bean指定延迟实例化特性)

- 当一个bean对另一个bean存在依赖关系时，可以利用bean元素的depends-on属性指定，当一个bean对多个bean存在依赖关系时，depends-on属性可以指定多个bean名，同逗号隔开。

		<bean id="exampleBean" class="com.foo.ExampleBean" depends-on="beanOne,beanTwo"></bean>

实际上，Spring Bean的生命周期远远没有我们想象的那么简单，感兴趣的可以去阅读下其源码部分，这里不再详述。（引用网上的一副流程图）

![](https://images0.cnblogs.com/i/580631/201405/181453414212066.png)
![](https://images0.cnblogs.com/i/580631/201405/181454040628981.png)


#### 2、DI注入方式 ####

Spring IOC容器的依赖有两层含义：Bean依赖容器和容器注入Bean的依赖资源：


- a、Bean依赖容器：也就是说Bean要依赖于容器，这里的依赖是指容器负责创建Bean并管理Bean的生命周期，正是由于由容器来控制创建Bean并注入依赖，也就是控制权被反转了，这也正是IOC名字的由来，此处的有依赖是指Bean和容器之间的依赖关系。
- b、容器注入Bean的依赖资源：容器负责注入Bean的依赖资源，依赖资源可以是Bean、外部文件、常量数据等，在Java中都反映为对象，并且由容器负责组装Bean之间的依赖关系，此处的依赖是指Bean之间的依赖关系，可以认为是传统类与类之间的“关联”、“聚合”、“组合”关系。

Spring IOC容器注入依赖资源主要有以下两种基本实现方式：


- 构造器注入：就是容器实例化Bean时注入那些依赖，通过在在Bean定义中指定构造器参数进行注入依赖，包括实例工厂方法参数注入依赖，但静态工厂方法参数不允许注入依赖；
- setter注入：通过setter方法进行注入依赖；


**I、构造器注入**

使用构造器注入通过配置构造器参数实现，构造器参数就是依赖。除了构造器方式，还有静态工厂、实例工厂方法可以进行构造器注入。

![](https://i.imgur.com/5YDwnRM.png)

构造器注入可以根据参数索引注入、参数类型注入或Spring3支持的参数名注入，但参数名注入是有限制的，需要使用在编译程序时打开调试模式（即在编译时使用“javac –g:vars”在class文件中生成变量调试信息，默认是不包含变量调试信息的，从而能获取参数名字，否则获取不到参数名字）或在构造器上使用@ConstructorProperties（java.beans.ConstructorProperties）注解来指定参数名。

（1）根据参数索引（顺序）注入，使用标签“<constructor-arg index="1" value="1"/>”来指定注入的依赖，其中“index”表示索引，从0开始，即第一个参数索引为0，“value”来指定注入的常量值，配置方式如下：

![](https://i.imgur.com/k66Eval.jpg)


（2）根据参数类型进行注入，使用标签“<constructor-arg type="java.lang.String" value="Hello World!"/>”来指定注入的依赖，其中“type”表示需要匹配的参数类型，可以是基本类型也可以是其他类型，如“int”、“java.lang.String”，“value”来指定注入的常量值，配置方式如下：

![](https://i.imgur.com/SHPhHZm.jpg)

（3）根据参数名进行注入，使用标签“<constructor-arg name="message" value="Hello World!"/>”来指定注入的依赖，其中“name”表示需要匹配的参数名字，“value”来指定注入的常量值，配置方式如下：

![](https://i.imgur.com/uPfZMUD.jpg)


附：构造器方式引入其他Bean

1）.通过” <constructor-arg>”标签的ref属性来引用其他Bean，这是最简化的配置：

![](https://i.imgur.com/EAIJcn9.jpg)


2）.通过”<constructor-arg>”标签的子<ref>标签来引用其他Bean，使用bean属性来指定引用的Bean

![](https://i.imgur.com/V8Q8IBq.jpg)


**II、Setter注入**

setter注入，是通过在通过构造器、静态工厂或实例工厂实例好Bean后，通过调用Bean类的setter方法进行注入依赖，setter注入方式只有一种根据setter名字进行注入。如图所示：

![](https://i.imgur.com/HzKIVib.jpg)


（1）注入基本值

![](https://i.imgur.com/BGMGJLe.png)

或者

![](https://i.imgur.com/KAoHyzC.png)

注意两种情况：

- 注入空字符串

![](https://i.imgur.com/vt902eQ.png)

- 注入null

![](https://i.imgur.com/ShX2Xnq.png)

（2）注入对象

注入Bean对象，定义格式有内部Bean和外部Bean两种：

- 注入内部Bean

![](https://i.imgur.com/D6S52Um.png)

- 注入外部Bean

![](https://i.imgur.com/ZZoYQsQ.png)

（3）注入List集合

![](https://i.imgur.com/H7j9T2Q.png)

或者引用方式List集合注入（Set Map Properties都可以采用引用方式注入）

![](https://i.imgur.com/fk6Y95n.png)


（4）注入Set集合

![](https://i.imgur.com/lOQGsKY.png)

（5）注入Map集合

![](https://i.imgur.com/1zAFhWt.png)

（6）注入Properties集合

![](https://i.imgur.com/o7Xnzzp.png)

（7）注入Spring表达式值

Spring引入了一种表达式语言，这和统一的EL在语法上很相似，这种表达式语言可以用于定义基于XML和注解配置的Bean，注入一个properties文件信息。

![](https://i.imgur.com/pW5M5qR.png)


#### 3.Spring IOC 注解方式 ####


**I、组件扫描**

指定一个包路径，Spring会自动扫描该包及其子包所有组件类，当发现组件类定义前有特定的注解标记时，就将该组件纳入到Spring容器。等价于原有XML配置中的<bean>定义功能。

- 组件扫描可以替代大量XML配置的<bean>定义
- 使用组件扫描，首先需要在XML配置中指定扫描类路径。

		//容器实例化时会自动扫描org.example包及其子包下所有组件类
		<context:component-scan base-package="org.example">


- 指定扫描类路径后，并不是该路径下所有组件类都扫描到Spring容器的，只有在组件类定义前面有以下注解标记时，才会扫描到Spring容器。

	![](https://i.imgur.com/muGahVj.png)

- 当一个组件在扫描过程中被检测到时，会生成一个默认id值，默认id为小写开头的类名。也可以在注解标记中自定义id.
	
	![](https://i.imgur.com/TxetQhs.png)

- 通常受Spring管理的组件，默认的作用域是"singleton".如果需要其他的作用域可以使用@Scope注解，只要在注解中提供作用域的名称即可。

	![](https://i.imgur.com/JJrv3MY.png)


**II、注解注入**

（1）@Autowired/@Qualifier（不推荐使用，建议使用@Resource）

- @Autowired 注解标记可以用在字段定义或setter方法定义前面，默认按类型匹配注入。
- @Autowired当遇到多个匹配Bean时注入会发生错误，可以使用@qualifier配合@Autowired来指定名称。

	![](https://i.imgur.com/USsPM8T.png)



（2）@Resource

@Resource 注解标记也可以用在字段定义或setter方法定义前面，默认首先按名称匹配注入，然后类型匹配注入。

JSR-250标准注解，推荐使用它来代替Spring专有的@Autowired注解。@Resource的作用相当于@Autowired，只不过 @Autowired按byType自动注入，而@Resource默认按byName自动注入罢了。@Resource有两个属性是比较重要的，分别是 name和type，Spring将 @Resource注解的name属性解析为bean的名字，而type属性则解析为bean的类型。所以如果使用name属性，则使用byName的自动注入策略，而使用type属性时则使用byType自动注入策略。如果既不指定name也不指定type属性，这时将通过反射机制使用byName自动注入策略。

@Resource装配顺序： 
    

- a.如果同时指定了name和type，则从Spring上下文中找到唯一匹配的bean进行装配，找不到则抛出异常 
- b.如果指定了name，则从上下文中查找名称（id）匹配的bean进行装配，找不到则抛出异常 
- c.如果指定了type，则从上下文中找到类型匹配的唯一bean进行装配，找不到或者找到多个，都会抛出异常 
- d.如果既没有指定name，又没有指定type，则自动按照byName方式进行装配（见b）；如果没有匹配，则回退为一个原始类型（UserDao）进行匹配，如果匹配则自动装配； 




（3）@Inject/@Named

@Inject注解标记是Spring3.0开始增添的对JSR-330标准的支持，使用前需要添加JSR-330的jar包，使用方法与@Autowired相似。@Inject当遇到多个匹配Bean时注入会发生错误，可以使用@Named指定名称限定。

![](https://i.imgur.com/9lVUx2s.png)


附：@Value

@Value注解可以注入Spring表达式值，使用方法：

- 首先在XML配置中指定要注入的properties文件

	![](https://i.imgur.com/Op8tvCB.png)

- 然后在setter方法前使用@Value注解

	![](https://i.imgur.com/5KVx7iU.png)
















