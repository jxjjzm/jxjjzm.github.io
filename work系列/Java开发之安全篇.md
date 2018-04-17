### Java开发之安全篇 ###
***

参考链接：[https://mp.weixin.qq.com/s/QIqGGFJd8bcrHfJoSiwViA](https://mp.weixin.qq.com/s/QIqGGFJd8bcrHfJoSiwViA)

http://k.2dfire.net/pages/viewpage.action?pageId=12353638

### 一、SQL注入 ###

#### 1.SQL注入攻击及原理 ####

所谓SQL注入,就是通过把SQL命令伪装成正常的HTTP请求参数,传递到服务端,欺骗服务器最终执行恶意的SQL命令,达到入侵目的。攻击者可以利用SQL注入漏洞,查询非授权信息, 修改数据库服务器的数据,改变表结构,甚至是获取服务器root权限。总而言之,SQL注入漏洞的危害极大,攻击者采用的SQL指令,决定攻击的威力。当前涉及到大批量数据泄露的攻击事件,大部分都是通过利用SQL注入来实施的。


假设有个网站的登录页面,如下所示:

![](https://mmbiz.qpic.cn/mmbiz_png/6fuT3emWI5LBQ5DSsdXHAjg1Gxkk69G870e7fznt9EjaUEYicDVmsPTTwxsCfYLjLWcCzq1ZOJ1nnrZoxgpkl2A/640?tp=webp&wxfrom=5&wx_lazy=1)

假设用户输入nick为zhangsan,密码为password1,则验证通过,显示用户登录:

![](https://mmbiz.qpic.cn/mmbiz_png/6fuT3emWI5LBQ5DSsdXHAjg1Gxkk69G8icyYgtvIDmSTyiceUneK4OUaNdqECsjUXUmDWg7R4ickVeABZ9msJEdHw/640?tp=webp&wxfrom=5&wx_lazy=1)

否则,显示用户没有登录:

![](https://mmbiz.qpic.cn/mmbiz_png/6fuT3emWI5LBQ5DSsdXHAjg1Gxkk69G875icOudIKSBk4affuSEyib050jS3iblrE1xic2aCGQXmGkzZzicibibJCXwcA/640?tp=webp&wxfrom=5&wx_lazy=1)


下面是一段普通的JDBC的Java代码，这段代码就可以被利用，进行SQL注入攻击：

		Connection conn = getConnection();
		String sql = "select * from hhuser where nick = '" + nickname + "'" + " and passwords = '" + password + "'"; 
		Statement st = (Statement) conn.createStatement();
		ResultSet rs = st.executeQuery(sql);
		List<UserInfo> userInfoList = new ArrayList<UserInfo>(); 
		while (rs.next()) {
		UserInfo userinfo = new UserInfo();         
		userinfo.setUserid(rs.getLong("userid"));       
		userinfo.setPasswords(rs.getString("passwords"));       
		userinfo.setNick(rs.getString("nick"));         
		userinfo.setAge(rs.getInt("age"));      
		userinfo.setAddress(rs.getString("address"));       
		userInfoList.add(userinfo);

当用户输入nick为zhangsan,密码为' or '1'='1的时候,意想不到的事情出现了,页面显示为login状态:

![](https://mmbiz.qpic.cn/mmbiz_png/6fuT3emWI5LBQ5DSsdXHAjg1Gxkk69G8icyYgtvIDmSTyiceUneK4OUaNdqECsjUXUmDWg7R4ickVeABZ9msJEdHw/640?tp=webp&wxfrom=5&wx_lazy=1)

（
当用户在密码栏输入“' or '1'='1”后，代码中的SQL语句就会被拼接成：
"select * from hhuser user where nick = 'zhangsan' and passwords = '' or '1'='1'"，因为or后面的1=1是恒为true的，所以，该语句的执行结果就会有正常的数据返回。从而绕过密码校验。
）

以上便是一次简单的、典型的SQL注入攻击。当然,SQL注入的危害不仅如此,假设用户输入用户名zhangsan,在密码框输入' ;drop table aaa;-- 会发生什么呢?

#### 2.SQL注入的防御 ####

（1）使用预编译语句

预编译语句PreparedStatement是java.sql中的一个接口,继承自Statement接口。通过 Statement对象执行SQL语句时,需要将SQL语句发送给DBMS,由DBMS先进行编译后再执行。而预编译语句和Statement不同,在创建PreparedStatement对象时就指定了SQL语句,该语句立即发送给DBMS进行编译,当该编译语句需要被执行时,DBMS直接运行编译后的SQL语句,而不需要像其他SQL语句那样首先将其编译。 

（
前面介绍过,引发SQL注入的根本原因是恶意用户将SQL指令伪装成参数传递到后端数据库执行, 作为一种更为安全的动态字符串的构建方法,预编译语句使用参数占位符来替代需要动态传入的参数,这样攻击者无法改变SQL语句的结构,SQL语句的语义不会发生改变,即便用户传入类似于前面' or '1'='1这样的字符串,数据库也会将其作为普通的字符串来处理。
）



（2）使用ORM框架

由上文可见,防止SQL注入的关键在于对一些关键字符进行转义,而常见的一些ORM框架,如 ibatis、hibernate等,都支持对相应的关键字或者特殊符号进行转义,可以通过简单的配置, 很好的预防SQL注入漏洞,降低了普通的开发人员进行安全编程的门槛。

以ibatis框架为例，通过#符号配置的变量,ibatis能够对输入变量的一些关键字进行转义,防止SQL注入攻击。先看一段ibatis的xml配置：

	<select id="queryByAccountId" parameterClass="java.util.Map" resultMap="ApplicationInstanceResult">
	    SELECT * FROM product where account_id = $accountId$
	</select>

以上的sql中存在一个问题，$accountId$是变量替换的形式, 容易引入sql注入, 例如$accountId$是前台用户输入的"';select * from admin--", 那么数据库端就会执行两个sql, 所以需要改成#accountId#, 进行预编译处理。修改后的配置如下:

	<select id="queryByAccountId" parameterClass="java.util.Map" resultMap="ApplicationInstanceResult">
	    SELECT * FROM product where account_id = #accountId#
	</select>

再看一段ibatis的xml配置：

	<select id="queryByAccountId" parameterClass="java.util.Map" resultMap="ApplicationInstanceResult">
	    SELECT * FROM product where account_id = #accountId# ORDER BY $columnName$ $sortType$ LIMIT #start#, #rowNum#
	</select>

可能由于某种需要搜索结果的排序很灵活, sql中有ORDER BY后$columnName$和$sortType$两个变量,在这里没法用变量绑定, 只能用$符号去替换,所以也存在隐患（其实如果这两个变量是程序中指定的, 那么是没有风险的）但是如果这两个变量是web前端选择填入的,那么就可以被利用构造sql注入的value, 就一定存在安全隐患, 碰到这样的情况, 我们需要映入安全开发的ibaits版本：

	<dependency>
	  <groupId>com.alibaba.external</groupId>
	  <artifactId>sourceforge.ibatis</artifactId>
	  <version>2.3.4.726-patch</version>
	</dependency>

然后是我们可以指定$columnName$和$sortType$的元数据类型, 如:$columnName:METADATA$, $sortType:SQLKEYWORD$, 这样ibatis就会做检查, 杜绝风险.修改后的配置如下:

	<select id="queryByAccountId" parameterClass="java.util.Map" resultMap="ApplicationInstanceResult">
	    SELECT * FROM product where account_id = #accountId# ORDER BY $columnName:METADATA$ $sortType:SQLKEYWORD$ LIMIT #start#, #rowNum#
	</select>


（
补充说明：


- 1.避免密码明文存放：对存储的密码进行单向Hash,如使用MD5对密码进行摘要,而非直接存储明文密码,这样的好处就是万一用户信息泄露,即圈内所说的被“拖库”,黑客无法直接获取用户密码,而只能得到一串跟密码相差十万八千里的Hash码。


- 2.处理好相应的异常：后台的系统异常,很可能包含了一些如服务器版本、数据库版本、编程语言等等的信息,甚至是数据库连接的地址及用户名密码,攻击者可以按图索骥,找到对应版本的服务器漏洞或者数据库漏洞进行攻击,因此,必须要处理好后台的系统异常,重定向到相应的错误处理页面,而不是任由其直接输出到页面上。
）


### 二.上传文件漏洞 ###


在上网的过程中,我们经常会将一些如图片、压缩包之类的文件上传到远端服务器进行保存, 文件上传攻击指的是恶意攻击者利用一些站点没有对文件的类型做很好的校验这样的漏洞, 上传了可执行的文件或者脚本,并且通过脚本获得服务器上相应的权利,或者是通过诱导外 部用户访问或者下载上传的病毒或者木马文件,达到攻击目的。

为了防范用户上传恶意的可执行文件和脚本,以及将文件上传服务器当做免费的文件存储服务器使用,需要对上传的文件类型进行白名单(非黑名单,这点非常重要)校验,并且限制上传文件的大小,上传的文件,需要进行重新命名,使攻击者无法猜测到上传文件的访问路径。如对于上传的图片, 我们需对其进行重格式化, 可以去掉多余的meta信息；对于上传的其他文件, 进行病毒扫描.

（
对于上传的文件来说,不能简单的通过后缀名称来判断文件的类型,因为恶意攻击可以将可执行文件的后缀名称改成图片或者其他的后缀类型,诱导用户执行。因此,判断文件类型需要使用更安全的方式。很多类型的文件,起始的几个字节内容是固定的,因此,根据这几个字节的内容,就可以确定文件类型,这几个字节也被称为魔数(magic number)。
）

### 三.跨站脚本攻击XSS(Cross Site Scripting) ###

#### 1.XSS攻击 ####

XSS（为了不和层叠样式表 (Cascading Style Sheets,CSS)的缩写混淆,故将跨站脚本攻击缩写为XSS）攻击的全称是跨站脚本攻击(Cross Site Scripting),是WEB应用程序中最常见到的攻击手段之一。跨站脚本攻击指的是攻击者在网页中嵌入恶意脚本程序, 当用户打开该网页时,脚本程序便开始在客户端的浏览器上执行,以盗取客户端cookie、 盗取用户名密码、下载执行病毒木马程序等等。

有一种场景,用户在表单上输入一段数据后,提交给服务端进行持久化,其他页面上需要从服务端将数据取出来展示。还是使用之前那个表单nick,用户输入昵称之后,服务端会将nick保存,并在新的页面展现给用户,当普通用户正常输入hollis,页面会显示用户的 nick为hollis:

	<body> 
	   hollis
	</body>

但是,如果用户输入的不是一段正常的nick字符串,而是<script>alert("haha")</script>, 服务端会将这段脚本保存起来,当有用户查看该页面时,页面会出现如下代码:

	<body> 
	   <script>
	       alert("haha")
	   </script>
	</body>

其影响就是可能会在页面直接弹出对话框。

![](https://mmbiz.qpic.cn/mmbiz_png/6fuT3emWI5LBQ5DSsdXHAjg1Gxkk69G8hA4OAsr3iczsC9QYTRgqNWQxC2LqX9pKJrkiatvQdoJkbmHjMh4CBKTQ/640?tp=webp&wxfrom=5&wx_lazy=1)

#### 2.XSS防御 ####

XSS之所以会发生,是因为用户输入的数据变成了代码。因此,我们需要对用户输入的数据进行HTML转义处理,将其中的“尖括号”、“单引号”、“引号” 之类的特殊字符进行转义编码。

![](https://mmbiz.qpic.cn/mmbiz_png/6fuT3emWI5LBQ5DSsdXHAjg1Gxkk69G8VfaI49F8grISTv9hsNnhr0gM8ibhr6YtGaY7ymaf93KANfkTxDWkCwQ/640?tp=webp&wxfrom=5&wx_lazy=1)

如今很多开源的开发框架本身默认就提供HTML代码转义的功能,如流行的jstl、Struts等等,不需要开发人员再进行过多的开发。使用jstl标签进行HTML转义,将变量输出,代码 如下:

	<c:out value="${nick}" escapeXml="true"></c:out>

只需要将escapeXml设置为true, jstl就会将变量中的HTML代码进行转义输出。


### 四.跨站请求伪造CSRF(cross site request forgery) ###

#### 1.CSRF攻击 ####

CSRF攻击的全称是跨站请求伪造(cross site request forgery), 是一种对网站的恶意利用,你可以这么理解CSRF攻击:攻击者盗用了你的身份,以你的名义向第三方网站发送恶意请求。CRSF能做的事情包括利用你的身份发邮件、发短信、进行交易转账等等,甚至盗取你的账号。（尽管听起来跟XSS跨站脚本攻击有点相似,但事实上CSRF与XSS差别很大,XSS利用的是站点内的信任用户,而CSRF则是通过伪装来自受信任用户的请求来利用受信任的网站。）

假设某银行网站A,他以GET请求来发起转账操作,转账的地址为www.xxx.com/transfer.do?accountNum=10001&money=10000,accountNum参数表示转账的目的账户,money参数表示转账金额。 而某大型论坛B上,一个恶意用户上传了一张图片,而图片的地址栏中填的并不是图片的地址,而是前面所说的转账地址:

	<img src="http://www.xxx.com/transfer.do?accountNum=10001&money=10000">

当你登陆网站A后,没有及时登出,这个时候你访问了论坛B,不幸的事情发生了,你会发现你的账户里面少了10000块...... 

为什么会这样呢,在你登陆银行A的时候,你的浏览器端会生成银行A的cookie,而当你访问论坛B的时候,页面上的<img>标签需要浏览器发起一个新的HTTP请求,以获得图片资源, 当浏览器发起请求的时候,请求的却是银行A的转账地址www.xxx.com/transfer.do?accoun tNum=10001&money=10000,并且会带上银行A的cookie信息,结果银行的服务器收到这个请求后,会认为是你发起的一次转账操作,因此你的账户里边便少了10000块。

![](https://mmbiz.qpic.cn/mmbiz_png/6fuT3emWI5LBQ5DSsdXHAjg1Gxkk69G8bj2I53qBz1bmFTRKsUp3uGyfnFiceZUoSotERFdoyJjxDlckP8eDgpQ/640?tp=webp&wxfrom=5&wx_lazy=1)


#### 2.CSRF的防御 ####



- **cookie设置为HttpOnly**

CSRF攻击很大程度上是利用了浏览器的cookie,为了防止站内的XSS漏洞盗取cookie,需要在cookie中设置"HttpOnly"属性,这样通过程序(如JavascriptS脚本、Applet等)就无法读取到cookie信息,避免了攻击者伪造cookie的情况出现。



- **增加token**

CSRF攻击之所以能够成功,是因为攻击者可以伪造用户的请求,该请求中所有的用户验证信息都存在于cookie中,因此攻击者可以在不知道用户验证信息的情况下直接利用用户的cookie来通过安全验证。由此可知,抵御CSRF攻击的关键在于:在请求中放入攻击者所不能伪造的信息,并且该信息不存在于cookie之中。鉴于此,系统开发人员可以在HTTP请求中以参数的形式加入一个随机产生的token,并在服务端进行token校验,如果请求中没有token或者token内容不正确,则认为是CSRF攻击而拒绝该请求。



- **通过Referer识别**

根据HTTP协议,在HTTP头中有一个字段叫Referer,它记录了该HTTP请求的来源地址。在通常情况下,访问一个安全受限页面的请求都来自于同一个网站。比如某银行的转账是通过用户访问http://www.xxx.com/transfer.do页面完成,用户必须先登录www.xxx.com,然后通过点击页面上的提交按钮来触发转账事件。当用户提交请求时,该转账请求的Referer值就会是提交按钮所在页面的URL(本例为www.xxx.com/transfer.do)。如果攻击者要对银行网站实施CSRF攻击,他只能在其他的网站构造请求,当用户通过其他网站发送请求到银行时,该请求的Referer的值是其他网站的地址,而不是银行转账页面的地址。因此,要防御CSRF攻击,银行网站只需要对于每一个转账请求验证其Referer值,如果是以www.xxx.com域名开头的地址,则说明该请求是来自银行网站自己的请求,是合法的。如果 Referer是其他网站的话,就有可能是CSRF攻击,则拒绝该请求。


### 五.分布式拒绝服务攻击DDoS(Distributed Denial of Service) ###

DDoS(Distributed Denial of Service),即分布式拒绝服务攻击,是指攻击者利用大量“肉鸡”对攻击目标发动大量的正常或非正常请求、耗尽目标主机资源或网络资源，从而使被攻击的主机不能为正常用户提供服务。

做一个形象的比喻，我们可以把攻击者视为无赖，被攻击者视为商场。无赖为了让一家商场无法正常营业，无赖们扮作普通客户一直拥挤在商场，赖着不走，真正的购物者却无法进入；或者总是和营业员有一搭没一搭的东扯西扯，让工作人员不能正常服务客户；也可以为商铺的经营者提供虚假信息，商铺的上上下下忙成一团之后却发现都是一场空，最终跑了真正的大客户，损失惨重。一个无赖去胡闹，就是 DoS攻击，而一群无赖去胡闹，就是 DDoS攻击(DoS（拒绝服务，Denial of Service）就是利用合理的服务请求来占用过多的服务资源，从而使合法用户无法得到服务的响应。这是早期非常基本的网络攻击方式。)

![](https://mmbiz.qpic.cn/mmbiz_jpg/6fuT3emWI5LzVUS063jrU0JtctjQoibWh5eLMonIdKfgicIiaYSZxba9fcDazDJpCmQQFuZwOXfmHnhHs3LIRAXcw/640?tp=webp&wxfrom=5&wx_lazy=1)

在信息安全的三要素——保密性、完整性和可用性中，DoS（Denial of Service）针对的目标正是可用性。该攻击方式利用目标系统网络服务功能缺陷或者直接消耗其系统资源，使得该目标系统无法提供正常的服务。一般来说，DDoS 攻击可以具体分成两种形式：带宽消耗型以及资源消耗型。它们都是透过大量合法或伪造的请求占用大量网络以及器材资源，以达到瘫痪网络以及系统的目的。其中，DDoS 带宽消耗攻击可以分为两个不同的层次：洪泛攻击或放大攻击。

当服务器被DDos攻击时，一般会出现以下现象：

- 被攻击主机上有大量等待的TCP连接；
- 网络中充斥着大量的无用的数据包；
- 受害主机无法正常和外界通讯；
- 受害主机无法处理所有正常请求；
- 严重时会造成系统死机。

对于用户来说，在常见的现象就是网站无法访问。

![](https://mmbiz.qpic.cn/mmbiz_jpg/6fuT3emWI5LzVUS063jrU0JtctjQoibWhqbDW4tUR8jdhjDSjE4XSNDRicNqd5ticaEwjNTQQ80AUNAV82HHHOyzw/640?tp=webp&wxfrom=5&wx_lazy=1)

参考链接：[GitHub遭受的DDoS攻击到底是个什么鬼？](https://mp.weixin.qq.com/s?__biz=MzI3NzE0NjcwMg==&mid=2650120826&idx=1&sn=bc7dc7bd910ae019e60e14b90ce2842b&chksm=f36bbf5bc41c364d0e20263d27c8c31899385d856067bf12ee97536e770abaa08df5c41bdd6d&scene=21#wechat_redirect)

DDoS的攻击有很多种类型,如依赖蛮力的ICMP Flood、UDP Flood等等,随着硬件性能的提升,需要的机器规模越来越大,组织大规模的攻击越来越困难,现在已经不常见,还有就是依赖协议特征以及具体的软件漏洞进行的攻击,如Slowloris攻击,Hash碰撞攻击等等,这类攻击主要利用协议以及软件漏洞发起攻击,需要在特定环境下才会出现,更多的攻击者采用的是前面两种的混合方式,即利用了协议、系统的缺陷,又具备了海量的流量, 如SYN Flood、DNS Query Flood等等。


### 六.其他 ###

除了上面列举出来的常见的Web安全漏洞外，还有一些常见的细节需要注意：

- 密码安全：DB中存放的密码必须是加密的，客户端的密码需要加密之后再传给服务端，服务端返回给客户端的数据中不要包含密码。
- Cookie：防止cookie中用户的敏感信息泄漏, 我们需要对cookie进行高强度的加密, 同时将设置httpOnly的属性。


































































