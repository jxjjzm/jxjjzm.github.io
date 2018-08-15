### 授权认证Oauth2.0 ###
***

最近这段时间由于公司内部组织架构调整，我开始接触第三方应用授权这块（微信小程序授权、支付宝小程序授权、淘宝开放平台...），在此针对这个话题做一个小小的总结。

### 一、Oauth 2.0 概述 ###

相信大部分读者对Oauth 2.0（OAuth是一个关于授权（authorization）的开放网络标准，在全世界得到广泛应用，目前的版本是2.0版。）并不陌生，在认证和授权的过程中涉及的三方包括：


- 服务提供方，用户使用服务提供方来存储受保护的资源，如照片，视频，联系人列表。
- 用户，存放在服务提供方的受保护的资源的拥有者。
- 客户端，要访问服务提供方资源的第三方应用，通常是网站，如提供照片打印服务的网站。在认证过程之前，客户端要向服务提供者申请客户端标识。

使用OAuth进行认证和授权的过程如下所示:


- 用户想操作存放在服务提供方的资源。
- 用户登录客户端向服务提供方请求一个临时令牌。
- 服务提供方验证客户端的身份后，授予一个临时令牌。
- 客户端获得临时令牌后，将用户引导至服务提供方的授权页面请求用户授权。在这个过程中将临时令牌和客户端的回调连接发送给服务提供方。
- 用户在服务提供方的网页上输入用户名和密码，然后授权该客户端访问所请求的资源。
- 授权成功后，服务提供方引导用户返回客户端的网页。
- 客户端根据临时令牌从服务提供方那里获取访问令牌。
- 服务提供方根据临时令牌和用户的授权情况授予客户端访问令牌。
- 客户端使用获取的访问令牌访问存放在服务提供方上的受保护的资源。


至于Oauth 2.0详细细节这里不再重复赘述（笔者推荐了两篇文章仅供参考）。


- [理解OAuth 2.0](http://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html)
- [一张图搞定OAuth2.0](https://www.cnblogs.com/flashsun/p/7424071.html)




### 二、第三方应用授权案例 ###

#### 1.微信公众平台 ####

![](https://res.wx.qq.com/op_res/g360EANvw_kVk3WCt-rRVP5UNFVJ2pYjH6gQCmxVL58lWhow97U8wYXpB4gw-I-d)

- [微信公众平台](https://mp.weixin.qq.com/wiki?t=resource/res_main&id=mp1445241432)
- [微信开放平台](https://open.weixin.qq.com/cgi-bin/index?t=home/index&lang=zh_CN)

#### 2.蚂蚁金服开放平台 ####

![](https://gw.alipayobjects.com/os/skylark-tools/public/files/d1aa377ec1777ba2c53fcefeb3f96d8a)


- [蚂蚁金服开放平台](https://open.alipay.com/platform/home.htm)


#### 3.淘宝开放平台 ####

![](http://img.alicdn.com/top/i1/TB1CBIbHpXXXXaIXpXXSutbFXXX.jpg)

- [淘宝开放平台](http://open.taobao.com/doc.htm?docId=73&docType=1)















