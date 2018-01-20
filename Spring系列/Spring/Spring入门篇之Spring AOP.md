### Spring入门篇之Spring AOP ###

***

### 一、AOP概述 ###

AOP（Aspect-OrientedProgramming，面向方面编程），可以说是OOP（Object-Oriented Programing，面向对象编程）的补充和完善。OOP引入封装、继承和多态性等概念来建立一种对象层次结构，用以模拟公共行为的一个集合。当我们需要为分散的对象引入公共行为的时候，OOP则显得无能为力。也就是说，OOP允许你定义从上到下的关系，但并不适合定义从左到右的关系。例如日志功能。日志代码往往水平地散布在所有对象层次中，而与它所散布到的对象的核心功能毫无关系。对于其他类型的代码，如安全性、异常处理和透明的持续性也是如此。这种散布在各处的无关的代码被称为横切（cross-cutting）代码，在OOP设计中，它导致了大量代码的重复，而不利于各个模块的重用。
 
而AOP技术则恰恰相反，它利用一种称为“横切”的技术，剖解开封装的对象内部，并将那些影响了多个类的公共行为封装到一个可重用模块，并将其名为“Aspect”，即方面。所谓“方面”，简单地说，就是将那些与业务无关，却为业务模块所共同调用的逻辑或责任封装起来，便于减少系统的重复代码，降低模块间的耦合度，并有利于未来的可操作性和可维护性。AOP代表的是一个横向的关系，如果说“对象”是一个空心的圆柱体，其中封装的是对象的属性和行为；那么面向方面编程的方法，就仿佛一把利刃，将这些空心圆柱体剖开，以获得其内部的消息。而剖开的切面，也就是所谓的“方面”了。然后它又以巧夺天功的妙手将这些剖开的切面复原，不留痕迹。

- OOP（面向对象编程）针对业务处理过程的实体及其属性和行为进行抽象封装，以获得更加清晰高效的逻辑单元划分。
- AOP（面向方面编程）针对业务处理过程中的切面进行提取，它所面对的是处理过程中的某个步骤或阶段，以获得逻辑过程中各部分之间低耦合性的隔离效果。


![](https://i.imgur.com/hgpPgaq.png)


使用“横切”技术，AOP把软件系统分为两个部分：核心关注点和横切关注点。业务处理的主要流程是核心关注点，与之关系不大的部分是横切关注点。横切关注点的一个特点是，他们经常发生在核心关注点的多处，而各处都基本相似。比如权限认证、日志、事务处理。Aop 的作用在于分离系统中的各种关注点，将核心关注点和横切关注点分离开来。正如Avanade公司的高级方案构架师Adam Magee所说，AOP的核心思想就是“将应用程序中的商业逻辑同对其提供支持的通用服务进行分离。”
 
（插播一条个人感言：个人觉得之所以AOP让许多人觉得难以理解的一个关键点是它的概念术语比较多，而且坑爹的是，这些概念经过了中文翻译后, 变得面目全非, 相同的一个术语, 在不同的翻译下, 含义总有着各种莫名其妙的差别. 鉴于此, 个人采用自己的语言来阐述下AOP相关的核心术语。）

核心术语：


- 方面（Aspect）：切面是指封装共同处理的组件，该组件被作用到其他目标组件方法上。
- 目标对象（Traget Object）：目标对象是指被一个或多个方面所作用的对象。
- 切入点（Pointcut）：切入点是用于指定哪些组件和方法使用方面功能，在Spring中利用一个表达式指定切入目标对象。Spring提供以下常用的切入点表达式：
	- 方法限定表达式： execution(修饰符？ 返回类型 方法名（参数） throws 异常类型？)
	- 类型限定表达式： within(包名.类型)
	- Bean名称限定表达式： bean("Bean的id或name属性值")

- 通知（Advice）：通知是用于指定方面组件和目标组件作用的时机。例如方面功能在目标方法之前或之后执行等时机。Spring框架提供以下几种类型的通知：
	- 前置通知：先执行方面功能再执行目标功能.(@Before)
	- 后置通知：先执行目标功能再执行方面功能（目标无异常才执行方面）.(AfterReturning)
	- 最终通知：先执行目标功能再执行方面功能（目标有无异常都执行方面）.(After)
	- 异常通知：先执行目标，抛出后执行方面.(@AfterThrowing)
	- 环绕通知：先执行方面前置部分，然后执行目标，最后再执行方面后置部分.(@Around)

![](https://i.imgur.com/rqcDUio.png)



### 二、AOP基本使用 ###

#### 1.XML配置方式 ####



- （1）创建方面组件：创建一个类，充当方面组件，实现通用业务逻辑
- （2）声明方面组件：在applicationContext.xml中声明方面组件
- （3）使用方面组件：在applicationContext.xml中奖方面组件作用到目标组件的方法上，并设置通知类型以确认方面组件调用的时机。




#### 2.注解方式 ####

- （1）创建方面组件：创建一个类，充当方面组件，实现通用业务逻辑
- （2）声明方面组件：
	- 在applicationContext.xml中开启AOP注解扫描 ： “<aop:aspectj-autoproxy proxy-target="true"/>”
	- 使用@Component注解标识这个类，将其声明为组件。
	- 使用@Aspect注解标识这个类，将其声明为方面组件。
- （3）使用方面组件：在组件的方法上，使用注解将方面组件作用到目标组件的方法上，不设置通知类型以确认方面组件调用的时机。


下面举个个人实际项目中写过的例子以供参考：

	package com.twodfire.wechat.aspect;
	
	import com.alibaba.fastjson.JSON;
	import com.dfire.consumer.util.LoggerUtil;
	import com.twodfire.wechat.common.logger.LoggerMarkers;
	import com.twodfire.wechat.common.logger.WXLoggerFactory;
	
	import org.apache.commons.lang.time.StopWatch;
	import org.aspectj.lang.ProceedingJoinPoint;
	import org.aspectj.lang.annotation.Around;
	import org.aspectj.lang.annotation.Aspect;
	
	import java.io.Serializable;
	
	/**
	 * MethodTimeAdvice
	 *
	 * 
	 */
	@Aspect
	public class MethodTimeAdvice {
	
	    private int longTimeThreshold;
	
	    public MethodTimeAdvice(int longTimeThreshold) {
	        this.longTimeThreshold = longTimeThreshold;
	    }
	
	    @Around("execution(public * com.twodfire.wechat.service..*.*(..))")
	    public Object aroundWechatService(ProceedingJoinPoint pjp) throws Throwable {
	        return around(pjp, longTimeThreshold);
	    }
	
	    @Around("execution(public * com.dfire.soa..*.*(..))")
	    public Object aroundSoaService(ProceedingJoinPoint pjp) throws Throwable {
	        return around(pjp, longTimeThreshold);
	    }
	
	    /**
	     * 记录方法的时间
	     *
	     * @param pjp      ProceedingJoinPoint
	     * @param longTime 超时需记录日志的时间
	     * @return 执行结果
	     * @throws Throwable 异常
	     */
	    private Object around(ProceedingJoinPoint pjp, int longTime) throws Throwable {
	
	        StopWatch clock = new StopWatch();
	        clock.start(); //计时开始
	        Object obj = pjp.proceed();
	        clock.stop();  //计时结束
	
	        String methodName = pjp.getSignature().getName();
	
	        long time = clock.getTime();
	
	        // 超过指定时间时记录到另外一个文件中
	        if (time >= longTime) {
	            StringBuilder sb = new StringBuilder(200);
	            sb.append(" ");
	            sb.append(methodName);
	            sb.append("(");
	
	            // 添加方法的参数, 方便调试
	            Object[] params = pjp.getArgs();
	            if (params != null && params.length > 0) {
	                StringBuilder sbParam = new StringBuilder();
	                for (Object param : params) {
	                    sbParam.append(",");
	                    if (param instanceof Serializable) {
	                        sbParam.append(JSON.toJSONString(param));
	                    } else {
	                        sbParam.append(param);
	                    }
	                }
	                sb.append(sbParam.substring(1));
	            }
	
	            sb.append(") ");
	            sb.append(time);
	            LoggerUtil.warn(WXLoggerFactory.TIME_OUT_LOGGER, LoggerMarkers.TIME_OUT, sb.toString());
	        }
	
	        return obj;
	    }
	}



### 三、AOP实现原理 ###

Spring AOP实现主要是基于动态代理技术。当Spring采用AOP配置后，Spring容器返回的目标对象实质上是Spring利用动态代理技术生成一个代理类型。代理类重写了原目标组件方法的功能，在代理类中调用方面对象功能和目标对象功能。Spring框架采用了两种动态代理实现：

- CGLIB动态代理：目标没有接口时采用此方法，代理类是利用继承方法生成一个目标子类。
- JDK Proxy：目标有接口时采用此方法，代理类是采用实现目标接口方法生成一个类。

![](https://i.imgur.com/kATwqEu.png)







































































































































