---
layout: post
title:  "OAuth2.0认证流程是如何实现的？"
categories: java
tags:  java basic
author: Franklinfang
---

* content
{:toc}


# 导读

大家也许都有过这样的体验，我们登录一些不是特别常用的软件或网站的时候可以使用QQ、微信或者微博等账号进行授权登陆。例如我们登陆豆瓣网的时候，如果不想单独注册豆瓣网账号的话，就可以选择用微博或者微信账号进行授权登录。这样的场景还有很多，例如登录微博、头条等网站。




<img src="https://user-images.githubusercontent.com/29160332/67388605-8df01b00-f5cb-11e9-9b3e-7f5aebb10f93.png" width = "80%" height = "57%" div align="center" />

那么这样的第三方登陆方式到底是怎么实现的呢？难道是腾讯把我们QQ或者微信的账户信息卖给了这些网站不成？很显然，腾讯是不会这么干的，而这种登录方式的实现就是我们这篇文章中要给大家介绍的`OAuth2.0`的认证方式。类似地，在公司内部，如果公司有多套不同的软件系统，例如我们日常使用的GitLab代码管理、公司内网系统、财务报销系统以及很多业务系统的管理后台，是不是也可以实现一个员工账号就能授权访问所有以上这些系统，而不需要每个系统都开通单独的账号，设置独立的密码才行呢？实际上在很多公司都是可以实现这种效果的，这也就是我们通常所说的`SSO`单点登录系统。

在接下来的内容中，会先给大家具体介绍下OAuth2.0的基本原理，然后再通过Spring Boot实现一套遵循OAuth2.0规范的SSO单点登录系统！



## 什么是OAuth2.0?

OAuth2.0是一种允许第三方应用程序使用资源所有者的`凭据`获得对资源有限访问权限的一种授权协议。例如在前面的例子中，通过微信登录豆瓣网的过程，就相当于微信允许豆瓣网作为第三方应用程序在经过微信用户授权后，通过`微信颁发的授权凭证`有限地访问用户的微信头像、手机号，性别等受限制的资源，从而来构建自身的登录逻辑。

  
需要说明的是在OAuth2.0协议中第三方应用程序获取的凭证并不等同于资源拥有者持有的用户名和密码，以上面例子来说微信是不会直接将用户的用户名、密码等信息作为凭证返回给豆瓣的。这种`授权访问凭证`一般来说就是一个表示特定范围、生存周期和其访问权限的一个由字符串组成的访问令牌，也就是我们常说的`token`。在这种模式下OAuth2.0协议中通过引入一个授权层来将第三方应用程序与资源拥有者进行分离，而这个`授权层`也就是我们常说的`“auth认证服务/sso单点登录服务器”`。


**为了便于理清认证流程中的各个角色，在OAuth2.0协议中定义了以下四个角色：**

- 1、resource owner（资源拥有者）
 
即能够有权授予对保护资源访问权限的实体。例如我们使用通过微信账号登陆豆瓣网，而微信账号信息的实际拥有者就是微信用户，也被称为最终用户。

- 2、resource server（资源服务器）
 
承载受保护资源的服务器，能够接收使用访问令牌对受保护资源的请求并响应，它与授权服务器可以是同一服务器，也可以是不同服务器。在上述例子中该角色就是微信服务器。

- 3、client（客户端）

代表资源所有者及其授权发出对受保护资源请求的应用程序。在上面的例子中豆瓣网就是这样的角色。

- 4、authorization server（授权服务器）

认证服务器，即服务提供商专门用来处理认证授权的服务器。例如微信开放平台提供的认证服务的服务器。

## OAuth2.0协议流程

在了解了OAuth2.0协议的基本概念后，接下来让我们一起以程序员的视角（NB点的叫法又叫上帝视角）来分析下OAuth2.0的运行流程。还是以微信授权登陆豆瓣网举例，流程如下：

![image](https://user-images.githubusercontent.com/29160332/67389443-3ce12680-f5cd-11e9-94a0-5af9576164fe.png)

上图通过对微信授权登录豆瓣网过程的分析，已经比较细节的描述了OAuth2.0的运行流程了，大致说明如下：


- 1.首先微信用户点击豆瓣网微信授权登录按钮后，豆瓣网会将请求通过URL重定向的方式跳转至微信用户授权界面；

- 2.此时微信用户实际上是在微信上进行身份认证，与豆瓣网并无交互了，这一点非常类似于-我们使用网银支付的场景；

- 3.用户使用微信客户端扫描二维码认证或者输入用户名密码后，微信会验证用户身份信息的正确性，如正确，则认为用户确认授权微信登录豆瓣网，此时会先生成一个临时凭证，并携带此凭证通过用户浏览器将请求重定向回豆瓣网在第一次重定向时携带的callBackUrl地址；

- 4.之后用户浏览器会携带临时凭证code访问豆瓣网服务，豆瓣网则通过此临时凭证再次调用微信授权接口，获取正式的访问凭据access_token；

- 5.在豆瓣网获取到微信授权访问凭据access_token后，此时用户的授权基本上就完成了，后续豆瓣网要做的只是通过此token再访问微信提供的相关接口，获取微信允许授权开发的用户信息，如头像，昵称等，并据此完成自身的用户逻辑及用户登录会话逻辑；

 
在上述流程中比较关键的动作是用户怎么样才能给Client（豆瓣网）授权，因为只有有了这个授权，Client角色才可以获取访问令牌（access_token），进而通过令牌访问其他资源接口。而关于客户端如何获得授权的问题，在OAuth2.0中定义了四种授权方式，目前微信授权登录使用的是其中一种比较常用的模式`authorization_code`模式。

## OAuth2.0客户端授权模式

OAuth2.0定义了四种授权模式，它们分别是：

- 授权码模式（authorization code）

- 简化模式（implicit）

- 密码模式（resource owner password credentials）

- 客户端模式（client credentials）

下面我们就分别来分析下这四种客户端授权模式的流程和特点吧！
 
**授权码模式（authorization code）**

授权码模式（authorization code）是功能最完整、流程最严密的授权模式。它的特点是通过客户端的后台服务器，与“服务提供商”的认证服务进行互动（如微信开放平台），我们前面以微信账号登录豆瓣网的流程就是授权码模式的实现。这种模式下授权代码并不是客户端直接从资源所有者获取，而是通过授权服务器（authorization server）作为中介来获取，授权认证的过程也是资源所有者直接通过授权服务器进行身份认证，避免了资源所有者身份凭证与客户端共享的可能，因此是十分安全的。

例如，微信用户授权登录豆瓣网的过程中，微信授权认证是直接在微信的网站上进行的，即便是输入用户名密码也只有微信授权认证服务器可以获取，因此豆瓣网是感知不到的，从而避免了微信用户账号信息在第三方网站泄露的风险。

由于授权码模式的认证方式与前面微信登录豆瓣网的过程是一致的，因此就不在单独进行流程叙述了！

**简化模式（implicit grant type）**

简化模式是对授权码模式的简化，用于在浏览器中使用脚本语言如JS实现的客户端中，它的特点是不通过客户端应用程序的服务器，而是直接在浏览器中向认证服务器申请令牌，跳过了“授权码临时凭证”这个步骤。其所有的步骤都在浏览器中完成，令牌对访问者是可见的，且客户端不需要认证。如果我们使用此种授权方式来实现微信登录豆瓣网的过程的话，流程如下：


![image](https://user-images.githubusercontent.com/29160332/67389803-f3dda200-f5cd-11e9-95ae-6e2601db1143.png)

从上面的流程中可以看到在第4步用户完成授权后，认证服务器是直接返回了access_token令牌至用户浏览器端，而并没有先返回授权码，然后由客户端的后端服务去通过授权码再去获取`access_token`令牌，从而省去了一个跳转步骤，提高了交互效率。


但是由于这种方式访问令牌`access_token`会在URL片段中进行传输，因此可能会导致访问令牌被其他未经授权的第三方截取，所以安全性上并不是那么的强壮。

**密码模式（resource owner password credentials）** 

在密码模式中，用户需要向客户端提供自己的用户名和密码，客户端使用这些信息向“服务提供商”索要授权。这相当于在豆瓣网中使用微信登录，我们需要在豆瓣网输入微信的用户名和密码，然后由豆瓣网使用我们的微信用户名和密码去向微信服务器获取授权信息。

这种模式一般用在用户对客户端高度信任的情况下，因为虽然协议规定客户端不得存储用户密码，但是实际上这一点并不是特别好强制约束。就像在豆瓣网输入了微信的用户名和密码，豆瓣网存储不存储我们并不是很清楚，所以安全性是非常低的。因此一般情况下是不会考虑使用这种模式进行授权的。

假设微信登录豆瓣网的过程使用了此种客户端授权模式，其流程如下：

![image](https://user-images.githubusercontent.com/29160332/67390005-52a31b80-f5ce-11e9-86db-08cc9ff26883.png)

以上模式逻辑主要在服务器端实现，无法避免客户端应用程序对授权方身份信息的存储，所以一般情况下是不会采用这种授权模式的。

**客户端模式（client credentials）**

客户端模式是指客户端以自己的名义，而不是以用户的名义，向“服务提供方”进行认证。严格地说，客户端模式并不属于OAuth2.0协议所要解决的问题。在这种模式下，用户并不需要对客户端授权，用户直接向客户端注册，客户端以自己的名义要求“服务提供商”提供服务。

 

这种类似于之前共享单车比较火的情况下，出现的全能车的模式，即用户向全能车注册，全能车以自己平台设立的共享账户去开各种品牌的单车。


综上所述，虽然在OAuth2.0协议中定义了四种客户端授权认证模式，但是实际上大部分实际应用场景中使用的都是授权码`（authorization code）`的模式，如微信开放平台、微博开放平台等使用的基本都是授权码认证模式。

  