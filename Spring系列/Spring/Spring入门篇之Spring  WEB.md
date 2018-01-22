### Spring入门篇之Spring WEB ###
***

### 一、Spring MVC 概述 ###

#### 1.MVC模式简介 ####

![](http://img.blog.csdn.net/20151216181152293?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

- M-Model 模型 ： Model的职责是负责业务逻辑。包含两层：业务数据和业务处理逻辑。
- V-View 视图： View的职责是负责显示界面和用户交互。
- C-controller 控制器 ： Controller是模型层M和视图层V之间的桥梁，用于控制流程。

#### 2.Spring MVC 简介 ####


Spring MVC是一种基于Java的实现了MVC模式的请求驱动（请求驱动指的就是使用请求-响应模型）类型的轻量级Web框架，框架的目的就是帮助我们简化开发。Spring MVC也是服务到工作者（服务到工作者：Front Controller + Application Controller + Page Controller + Context）模式的实现。前端控制器是DispatcherServlet；应用控制器其实拆为处理器映射器(HandlerMapping)进行处理器管理和视图解析器(ViewResolver)进行视图管理；页面控制器/动作/处理器为Controller接口（仅包含ModelAndView 、handleRequest(request, response) 方法）的实现；支持本地化（Locale）解析、主题（Theme）解析及文件上传等；提供了非常灵活的数据验证、格式化和数据绑定机制；提供了强大的约定大于配置（惯例优先原则）的契约式编程支持。


**（1）Spring MVC 核心组件**

![](https://i.imgur.com/7ff7z97.png)


- DispatcherServlet（Servlet控制器，请求入口）
- HandlerMapping（映射处理器，请求派发）
- Controller（应用控制器，请求处理流程）
- ModelAndView（模型，封装业务处理结果和视图）
- ViewResolver（视图，视图显示处理器） 



**（2）Spring MVC 处理流程**

![](https://i.imgur.com/Ev03Ubi.png)

- 浏览器向Spring发出请求，请求交给前端控制器DispatcherServlet处理
- 控制器通过HandlerMapping找到相应的Controller组件处理请求
- 执行Controller组件约定方法处理请求，在约定方法调用模型组件完成业务处理。约定方法可以返回一个ModelView对象，封装了处理结果数据和视图名称信息
- 控制器接收ModelAndView之后，调用ViewResolver组件，定位View并传递数据信息，生成响应界面结果



**（3）Spring MVC 能帮我们做什么？**



- 让我们能非常简单的设计出干净的Web层和薄薄的Web层；
- 进行更简洁的Web层的开发；
- 天生与Spring框架集成（如IoC容器、AOP等）；
- 提供强大的约定大于配置的契约式编程支持；
- 能简单的进行Web层的单元测试；
- 支持灵活的URL到页面控制器的映射；
- 非常容易与其他视图技术集成，如Velocity、FreeMarker等等，因为模型数据不放在特定的API里，而是放在一个Model里（Map数据结构实现，因此很容易被其他框架使用）；
- 非常灵活的数据验证、格式化和数据绑定机制，能使用任何对象进行数据绑定，不必实现特定框架的API；
- 提供一套强大的JSP标签库，简化JSP开发；
- 支持灵活的本地化、主题等解析；
- 更加简单的异常处理；
- 对静态资源的支持；
- 支持Restful风格。


**（4）Spring MVC 优势**



- 1）、清晰的角色划分：前端控制器（DispatcherServlet）、处理器映射（HandlerMapping）、处理器适配器（HandlerAdapter）、视图解析器（ViewResolver）、处理器或页面控制器（Controller）、验证器（Validator）等。
- 2）、分工明确，而且扩展点相当灵活，可以很容易扩展，虽然几乎不需要；
- 3）、由于命令对象就是一个POJO，无需继承框架特定API，可以使用命令对象直接作为业务对象；
- 4）、和Spring 其他框架无缝集成，是其它Web框架所不具备的；
- 5）、可适配，通过HandlerAdapter可以支持任意的类作为处理器；
- 6）、可定制性，HandlerMapping、ViewResolver等能够非常简单的定制；
- 7）、功能强大的数据验证、格式化、绑定机制；
- 8）、利用Spring提供的Mock对象能够非常简单的进行Web层单元测试；
- 9）、本地化、主题的解析的支持，使我们更容易进行国际化和主题的切换。
- 10）、强大的JSP标签库，使JSP编写更容易。
- 11）、还有RESTful风格的支持、简单的文件上传、约定大于配置的契约式编程支持、基于注解的零配置支持等等。




### 二、Spring MVC 实战###

#### 1.xml常见配置 ####

（1）DispatcherServlet控制器配置

![](https://i.imgur.com/sRydfH4.png)

#### 2.Spring MVC 常用注解 ####

**(1) @RequestMapping**

 ![](https://i.imgur.com/PDZYf3W.png) 

- @RequestMapping是一个用来处理请求地址映射的注解，可用于类或方法上。用于类上，表示类中的所有响应请求的方法都是以该地址作为父路径。
- 开启 @RequestMapping注解映射，需要在Spring的XML配置文件中定义RequestMappingHandlerMapping（类定义前）和 RequestMappingHandlerAdapter（方法定义前）两个bean组件
- Spring 3.1版本之前需要定义 DefaultAnnotationHandlerMapping和AnnotationMethodHandlerAdapter两个组件。

	![](https://i.imgur.com/xnROn1a.png)

从Spring 3.2版本开始可以使用下面XML配置简化RequestMappingHandlerMapping和RequestMappingHandlerAdapter定义：

![](https://i.imgur.com/u1MWdGz.png)

- @RequestMapping 属性：
	- value，method
		- value：     指定请求的实际地址，指定的地址可以是URI Template 模式（后面将会说明）；
		- method：  指定请求的method类型， GET、POST、PUT、DELETE等；
	- consumes,produces
		- consumes： 指定处理请求的提交内容类型（Content-Type），例如application/json, text/html;
		- produces:    指定返回的内容类型，仅当request请求头中的(Accept)类型中包含该指定类型才返回；
	- params,headers
		- params： 指定request中必须包含某些参数值是，才让该方法处理。
		- headers： 指定request中必须包含某些指定的header值，才能让该方法处理请求。



#### 3.Spring MVC 接收请求参数 ####

- I.使用HttpServletRequest获取
- II.使用 注解@RequestParam绑定请求参数
- III.使用自动机制封装成Bean对象（前提：前台属性的名称和java bean的属性的名称保持一致）
- IV. Spring会自动将表单参数注入到方法参数（前提：方法参数与表单的name属性保持一致）（说明：如果前台有相同name值的数据传递则要使用数组来接收，相同名称的可能是有这样的数值但更多的是多选框，所以我们要使用数组来接收这样的值）
- V.  通过@PathVariabl注解获取路径中传递的参数值 


#### 4.Spring MVC 向页面传值 ####

- I.使用HttpServletRequest 和 Session  然后setAttribute()，就和Servlet中一样
- II.使用ModelAndView对象
- III.使用ModelMap对象
- IV.使用@ModelAttribute注解






### 三、REST 与 RESTful ###


TODO：



































































































































































