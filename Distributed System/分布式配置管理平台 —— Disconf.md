### 分布式配置管理平台 —— Disconf ###

***

![](https://img0.tuicool.com/7jqiuq.jpg!web)

Disconf —— Distributed Configuration Management Platform(分布式配置管理平台)专注于各种分布式系统配置管理的通用组件和通用平台, 提供统一的配置管理服务。


### 1.重要功能特点 ###

![](https://img2.tuicool.com/yYnuIru.jpg!web)

- 支持配置（配置项+配置文件）的分布式化管理
- 配置发布统一化
	- 配置发布、更新统一化（云端存储、发布）:配置存储在云端系统，用户统一在平台上进行发布、更新配置。
	- 配置更新自动化：用户在平台更新配置，使用该配置的系统会自动发现该情况，并应用新配置。特殊地，如果用户为此配置定义了回调函数类，则此函数类会被自动调用。
- 配置异构系统管理
	- 异构包部署统一化：这里的异构系统是指一个系统部署多个实例时，由于配置不同，从而需要多个部署包（jar或war）的情况（下同）。使用 Disconf后，异构系统的部署只需要一个部署包，不同实例的配置会自动分配。特别地，在业界大量使用部署虚拟化（如JPAAS系统，SAE，BAE） 的情况下，同一个系统使用同一个部署包的情景会越来越多，Disconf可以很自然地与他天然契合。
	- 异构主备自动切换：如果一个异构系统存在主备机，主机发生挂机时，备机可以自动获取主机配置从而变成主机。
	- 异构主备机Context共享工具：异构系统下，主备机切换时可能需要共享Context。可以使用Context共享工具来共享主备的Context。
- 极简的使用方式（注解式编程 或 XML代码无代码侵入模式）：我们追求的是极简的、用户编程体验良好的编程方式。目前支持两种开发模式：基于XML配置或才基于注解，即可完成复杂的配置分布式化。
- 低侵入性或无侵入性、强兼容性：
	- 低侵入性：通过极少的注解式代码撰写，即可实现分布式配置。
	- 无侵入性：通过XML简单配置，即可实现分布式配置。
	- 强兼容性：为程序添加了分布式配置注解后，开启Disconf则使用分布式配置；若关闭Disconf则使用本地配置；若开启Disconf后disconf-web不能正常Work，则Disconf使用本地配置。
- 支持配置项多个项目共享，支持批量处理项目配置。
- 需要Spring编程环境
- 配置监控：平台提供自校验功能（进一步提高稳定性），可以定时校验应用系统的配置是否正确。


### 2.模块架构 ###

![](https://img2.tuicool.com/fiya6j6.jpg!web)

- disconf-core : 分布式配置基础包模块
	- 分布式通知模块：支持配置更新的实时化通知
	- 路径管理模块：统一管理内部配置路径URL
- disconf-client : 分布式配置客户端模块, 依赖disconf-core包。 用户程序使用它作为Jar包进行分布式配置编程。（[Disconf-client详细设计文档](http://disconf.readthedocs.io/zh_CN/latest/design/src/disconf-client%E8%AF%A6%E7%BB%86%E8%AE%BE%E8%AE%A1%E6%96%87%E6%A1%A3.html)）
	- 配置仓库容器模块：统一管理用户实例中本地配置文件和配置项的内存数据存储
	- 配置reload模块：监控本地配置文件的变动，并自动reload到指定bean
	- 扫描模块：支持扫描所有disconf注解的类和域
	- 下载模块：restful风格的下载配置文件和配置项
	- watch模块：监控远程配置文件和配置项的变化
	- 主备分配模块：主备竞争结束后，统一管理主备分配与主备监控控制
	- 主备竞争模块：支持分布式环境下的主备竞争

			<dependency>
			    <groupId>com.baidu.disconf</groupId>
			    <artifactId>disconf-client</artifactId>
			    <version>2.6.20</version>
			</dependency>

- disconf-tool : 分布式配置工具包，依赖disconf-core包。 Disconf-tool是disconf的辅助工具类。
- [disconf-web](https://github.com/knightliao/disconf/tree/master/disconf-web) : 分布式配置平台服务模块, 依赖disconf-core包。采用SpringMvc+纯HTML方式实现。 用户使用它来进行日常的分布式配置管理。（[Disconf-web详细设计文档](http://disconf.readthedocs.io/zh_CN/latest/design/src/disconf-web%E8%AF%A6%E7%BB%86%E8%AE%BE%E8%AE%A1%E6%96%87%E6%A1%A3.html)）


[分布式配置管理平台Disconf](http://disconf.readthedocs.io/zh_CN/latest/design/src/%E5%88%86%E5%B8%83%E5%BC%8F%E9%85%8D%E7%BD%AE%E7%AE%A1%E7%90%86%E5%B9%B3%E5%8F%B0Disconf.html)



















































































