---
title: 从Servlet聊聊JAVA
comments: true
index_img: /img/geek.jpeg
banner_img: /img/background/vilige.jpg
date: 2021-09-29 18:09:15
updated: 2021-09-29 18:09:15
tags: [JAVA,Servlet,J2EE]
categories: JAVA
---

### 导言

“一流企业做标准、二流企业做品牌、三流企业做产品！”				

---

近日，许多小伙伴对于JAVA的基础知识不太了解，比较迷茫，这里分享一下。

在学习JAVA之初，老师曾赠与我2条slogan，受益良多。

1. JAVA程序员终其一生只做两件事，一是在写Servlet，二是与互联网通讯打交道。
2. 学习JAVA就是学习13规范，Servlet是其中之一。

那么什么是Servlet呢？

---

先让我们从J2EE说起。

>1991年，James Gosling博士的孵化了JAVA语言的前身：Oak。Oak最初的定位是开发一种能够在各种消费性电子产品（机顶盒、冰箱、收音机等设备）上运行的程序架构。
>
>1995年，随着互联网潮流的兴起，Oak迅速找到了最适合自己发展的市场定位并迅速蜕变成为JAVA语言。
>
>1998年，JDK迎来了一个里程碑式的重要版本：工程代号为Playground的JDK1.2，在这个版本中，JAVA的技术体系拆分为3个方向。
>
>- 面向桌面应用开发的J2SE
>- 面向手机等移动设备的J2ME
>- 面向企业级应用开发的J2EE
>
>J2EE的全称是Java 2 Platform Enterprise Edition，它是由SUN公司领导、各厂家共同制定并得到广泛认可的工业标准，或者说，它是在SUN公司领导下，多家公司参与共同制定的企业级[分布式应用程序](https://baike.baidu.com/item/分布式应用程序/9854429)开发规范。J2EE是市场上主流的企业级分布式应用平台的解决方案 。
>
>J2EE平台由一整套服务（services）、[应用程序接口](https://baike.baidu.com/item/应用程序接口/10418844)（APIs）和协议构成，它对开发基于Web的多层应用提供了功能支持，下面对J2EE中的主要技术规范进行简单的描述。

在了解J2EE 13规范之前，我们思考一下。

什么是规范？J2EE为何要制定13个规范？

---

这里举一个很简单的例子，你购买了一个U盘，回到家里，可以插到你电脑的USB接口上面使用。

在这个例子里面，有3个角色：

- USB协议制定机构
- U盘生产商
- 电脑生产商

USB就是一种协议，也是一种规范。USB协议的制定机构并不生产U盘或者电脑。而U盘与电脑的生产商只需遵循该协议，分别生产拥有USB插头和接口的设备，这些设备就可以基于USB协议进行通信。

回到JAVA中，拿我们经常用的JDBC（Java Database Connectivity）协议来映射便是：

- JDBC规范制定机构：Sun公司，现在已被Oracle收购。
- 程序的开发商：如果自己实现JDBC功能，别左右看了，就是你。如果引用封装了JDBC代码的第三方框架，那么就是如Mybatis等厂商。
- 数据库开发商：Mysql、Oracle等开发数据库的公司。

>Java数据库连接，（Java Database Connectivity，简称JDBC）是Java语言中用来规范客户端程序如何来访问数据库的，提供了诸如查询和更新数据库中数据的方法。

可以看出，程序员基于JDBC开发程序，数据库厂商基于JDBC开发数据库。程序跑起来之后就能实现访问数据库的功能。而无需关心程序是谁写的，也不用关心数据库厂家是谁。

JDBC是JAVA程序与数据库服务器之间通讯的规范，同样地，如果我们需要与Web服务器如Tomcat进行交互呢？如果我们需要与邮件服务器进行交互呢？

这些问题Sun公司早就想到了，因此我们有了13个各式各样J2EE规范，足以解决我们日常开发的大部分问题。

Servlet：JAVA程序与Web服务器通讯的规范

JavaMail：与邮件服务器间通讯的规范

....

现在再让我们来仔细看下这13个规范

### 1 , JDBC ( JavaDatabase Connectivity )

JDBC 是以统一方式访问数据库的 API .
它提供了独立于平台的数据库访问,也就是说,当我有了 JDBC API 的时候,就不必再为访问 Oracle 数据库专门写一个程序,为访问 Sybase 数据库又专门写一个程序等等,只需要用 JDBC API 写一个程序就够了,它可以向相应数据库发送 SQL 调用. JDBC 是 Java 应用程序与各种不同数据库之间进行对话的方法的机制.简单地说,它做了三件事:与数据库建立连接–发送操作数据库的语句–处理结果.

### 2 , JNDI ( JavaName and Directory Interface )

JNDI 是一组在 Java 应用中访问命名和目录服务的 API .
JNDI 为开发人员提供了查找和访问各种命名和目录服务的通用,统一的接口,利用 JNDI 的命名与服务功能可满足企业级 API 对命名与服务的访问,诸如 EJB , JMS , JDBC 2.0 以及 IIOP 上的 RMI 通过 JNDI 来使用 CORBA 的命名服务.
在这儿,想多说一点: JNDI 和 JDBC 类似,都是构建在抽象层上.因为它提供了标准的独立于命名系统的 API ,这些 API 构建在命名系统之上.这一层有助于将应用与实际数据源分离,因此不管是访问的 LDAP , RMI 还是 DNS .也就是说, JNDI 独立于目录服务的具体实现,只要有目录的服务提供接口或驱动,就可以使用目录.

### 3 , EJB ( Enterprise JavaBean )

J2EE 将业务逻辑从客户端软件中抽取出来,封装在一个组件中.这个组件运行在一个独立的服务器上,客户端软件通过网络调用组件提供的服务以实现业务逻辑,而客户端软件的功能只是负责发送调用请求和显示处理结果.
在 J2EE 中,这个运行在一个独立的服务器上,并封装了业务逻辑的组件就是 EJB 组件.其实就是把原来放到客户端实现的代码放到服务器端,并依靠 RMI 进行通信.

### 4 , RMI ( Remote MethodInvoke )

是一组用户开发分布式应用程序的 API .
这一协议调用远程对象上的方法使用了序列化的方式在客户端和服务器之间传递数据,使得原先的程序在同一操作系统的方法调用,变成了不同操作系统之间程序的方法调用,即 RMI 机制实现了程序组件在不同操作系统之间的通信.它是一种被 EJB 使用的更底层的协议.
RMI/JNI : RMI 可利用标准 Java 本机方法接口与现有的和原有的系统相连接
RMI/JDBC : RMI 利用标准 JDBC 包与现有的关系数据库连接
就实现了与非 Java 语言的现有服务器进行通信.

### 5 , JavaIDL/CORBA ( Common Object Request BrokerArchitecture )

Java 接口定义语言/公用对象请求代理程序体系结构
在 JavaIDL 的支持下,开发人员可以将 Java 和 CORBA 集成在一起.他们可以创建 Java 对象并使之可以在 CORBA ORB 中展开,或者他们还可以创建 Java 类并作为和其它 ORB 一起展开的 CORBA 对象的客户.后一种方法提供了另外一种途径,通过 Java 可以被用于将新的应用和旧的系统相集成.
CORBA 是面向对象标准的第一步,有了这个标准,软件的实现与工作环境对用户和开发者不再重要,可以把精力更多地放在本地系统的实现与优化上.

### 6 , JSP ( Java Server Pages )

JSP 页面 = HTML + Java ,其根本是一个简化的 Servlet 设计.
服务器在页面被客户端请求后,对这些 Java 代码进行处理,然后将执行结果连同原 HTML 代码生成的新 HTML 页面返回给客户端浏览器.

### 7 , Java Servlet

Servlet 是一种小型的 Java 程序,扩展了 Web 服务器的功能,作为一种服务器的应用,当被请求时开始执行. Servlet 提供的功能大多和 JSP 类似,不过, JSP 通常是大多数的 HTML 代码中嵌入少量的 Java 代码,而 Servlet 全部由 Java 写成并生成 HTML .

### 8 , XML

XML 是一个用来定义其它标记语言的语言,可用作数据共享. XML 的发展和 Java 是相互独立的.不过,它和 Java 具有的相同目标就是跨平台.通过将 Java 与 XML 结合,我们可以得到一个完全与平台无关的解决方案.

### 9 , JMS ( JavaMessage Service )

它是一种与厂商无关的 API ,用来访问消息收发系统消息.它类似于 JDBC . JDBC 是可以用来访问不同关系数据库的 API ,而 JMS 则提供同样与厂商无关的访问消息收发服务的方法,这样就可以通过消息收发服务实现从一个 JMS 客户机向另一个 JMS 客户机发送消息,所需要的是厂商支持 JMS .
换句话说, JMS 是 Java 平台上有关面向消息中间件的技术规范.

### 10 , JTA ( JavaTransaction API )

定义了一种标准 API ,应用程序由此可以访问各种事务监控.它允许应用程序执行分布式事务处理–在两个或多个网络计算机资源上访问并且更新数据. JTA 和 JTS 为 J2EE 平台提供了分布式事务服务.
JTA 事务比 JDBC 事务更强大,一个 JTA 事务可以有多个参与者,而一个 JDBC 事务则被限定在一个单一的数据库连接.

### 11 , JTS ( JavaTransaction Service )

JTS 是 CORBA OTS 事务监控器的一个基本实现. JTS 指定了一个事务管理器的实现（ Transaction Manager ）,这个管理器在一个高级别上支持 JTA 规范,并且在一个低级别上实现了 OMGOTS 规范的 Java 映射.一个 JTS 事务管理器为应用服务器,资源管理器, standalone 应用和通信资源管理器提供事务服务.

### 12 , JavaMail

用于访问邮件服务器的 API ,提供了一套邮件服务器的抽象类.

### 13 , JAF ( JavaBeansActivation Framework )

JAF 是一个专用的数据处理框架,它用于封装数据,并为应用程序提供访问和操作数据的接口.也就是说, JAF 让 Java 程序知道怎么对一个数据源进行查看,编辑,打印等.
JavaMail 利用 JAF 来处理 MIME 编码的邮件附件.

---



​	熟悉了这些规范，JAVA程序员便可以更加专注程序的业务实现，而不用把精力放在如明天新出了一款性能好的数据库，要如何适配这种问题上。

​	因此，每一个J2EE程序员有必要一开始就熟悉这13个规范，至少了解有这么一个东西，防止重复造轮子。

​	最后，让我们回到开头的导言，JAVA是因为这些标准才流行起来的吗？JAVA的生命力与此有关吗？JAVA到底何时才会被取代呢（JAVA程序员何时会失业呢）:D？这个可能见仁见智，欢迎大家留言发表自己的看法。

>《深入理解JAVA虚拟机》 周志明 著
>
>[[J2EE 之 13 个规范总结](https://www.cnblogs.com/zll-0405/p/12534127.html)] https://www.cnblogs.com/zll-0405/p/12534127.html
>
>j2ee词条 百度百科 https://baike.baidu.com/item/j2ee/110838?fr=aladdin
