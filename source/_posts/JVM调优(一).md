---
title: JVM调优（一）
date: 2021-09-24 09:23:42
tags: [ JVM,Tomcat,调优 ] 
categories: JVM
comments: true
index_img: https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/JVM.png
banner_img: https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/background/vilige.jpg
---



​	  近日，随着项目的演进，微服务越来越多，在前后分离、后端双节点的部署架构下，每个微服务都运行在一个tomcat当中，单节点的负载能力竟已达到瓶颈。在计算能力如此强大的今天，没有节制地使用虚拟机资源果然还是不行，呵呵。

![](https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/image-20210923135927456.png)



## 问题

#### 上下文

> 虚拟机规格为4U16G x86   linux操作系统
>
> 内网项目，微服务大概10个左右，包括注册中心、网关等，并发量不大。
>
> web容器为tomcat
>
> jdk1.8

**首先说说什么是RES和VIRT？**

>RES：resident memory usage 常驻内存
>
>（1）进程当前使用的内存大小，但不包括swap out
>
>（2）包含其他进程的共享
>
>（3）如果申请100m的内存，实际使用10m，它只增长10m，与VIRT相反
>
>（4）关于库占用内存的情况，它只统计加载的库文件所占内存大小
>
>RES = CODE + DATA
>
>VIRT：virtual memory usage
>
>（1）进程“需要的”虚拟内存大小，包括进程使用的库、代码、数据等
>
>（2）假如进程申请100m的内存，但实际只使用了10m，那么它会增长100m，而不是实际的使用量
>
>VIRT = SWAP + RES

安装部署项目之后使用free -h和top命令查看CPU和内存的使用情况

![](https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/image-20210923140858686.png)

1、可以看出，虚拟机16G的内存只剩下可怜197M内存，真是在翻车边缘疯狂试探，这与测试反映的系统偶尔瘫痪的表象也是吻合的。

2、单个微服务的RES常驻内存在1.2G~1.6G之间，而在ARM服务器上甚至达到了3G左右，10个微服务基本把16G的内存用差不多了。以当前项目的负载需求来看，并没有大并发的场景，也不存在大数据量的接口，这个负载是偏高的，比如一个10个左右服务的注册中心Eureka，不包含任何业务代码也占用了如上的内存，是不合理的。

## 分析

我们以注册中心Eureka来入手，它没有业务代码，减少变量利于我们快速地定位问题。

这里我们先排除内存溢出或内存泄露的问题，当然也不是绝对排除。为什么这么说呢，如果某个微服务有代码存在内存溢出或泄露隐患，不会导致所有微服务内存占用都高，而且类似注册中心这种微服务几乎是零业务代码的。其次，这类问题会在程序运行一段时间过后突然down掉，从项目运行的情况来看，暂时没有发现这种情况。因此，内存溢出或泄露的问题可以放在后面再进一步定位。

通过本人的给服务器把脉问诊，初步分析原因是服务器君过于娇纵后宫，导致后宫争宠，时间被各妃子日程占满，从而导致身体吃不消了。

简单来说就是没有为各Tomcat设置JVM内存参数，各个微服务无节制地使用服务器内存导致的。

我们通过几个简单的参数`Xms`、`Xmx`、`Xss`来验证一下。

>-Xms 是指程序启动时占用内存大小。一般来讲，大点，程序会启动的快一点，但是也可能会导致机器暂变慢，程序在内存不够时会进行扩展，而扩展也是会影响效率的，可能会导致程序突然卡顿一下，因此许多的项目中这个值一步到位设置和最大值Xmx一样。
>
>-Xmx 是指程序运行期间最大可占用的内存大小。如果程序运行需要占用更多的内存，超出了这个设置值，就会抛出OutOfMemory异常。
>
>-Xss 是指设定每个线程的堆栈大小。这个就要依据你的程序，看一个线程大约需要占用多少内存，可能会有多少线程同时运行等。默认JDK1.4中是256K，JDK1.5+中是1M。

由于项目当初是使用的是一个未知的定制版的Tomcat，已经改的面目全非，没有文档支持，花费了很大功夫才找到加JAVA启动参数的地方。不具备参考意义，我们以开源版的Tomcat为例。

这次 ，我们使用java自带的jvm监控工具jvisualvm来监控jvm的启动参数、内存占用等信息，使用前我们需要给JVM加上启动参数来打开调试接口。

我们在Tomcat的bin目录下的catalina.sh启动文件中添加如下内容（windows下请修改catalina.bat）。

![](https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/image-20210923161704099.png)

```shell
export JAVA_OPTS="-Djava.rmi.server.hostname=xxx.xxx.xxx.xxx -Dcom.sun.management.jmxremote.port=8899 -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false"
```

>-Djava.rmi.server.hostname：服务器IP，修改为自己的虚拟机的IP即可
>
>-Dcom.sun.management.jmxremote.port ：开放端口，这里自定义即可，笔者用的是8899，只要端口没有冲突即可。
>
>-Dcom.sun.management.jmxremote.ssl=false、-Dcom.sun.management.jmxremote.authenticate=false：是否启用SSL安全协议、安全认证，主要是为了安全，这里一般设置为false即可，生产环境中我们一般也不会打开调试端口。

打开jvisualvm工具

1、添加远程服务器，Remote右键 Add Remote Host...，填入上面配置的服务器IP

![](https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/image-20210923163132389.png)



![](https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/image-20210923163347853.png)



2、添加JXM连接

![](https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/image-20210923163549140.png)

按照下图配置

![](https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/image-20210923163638670.png)

双击建立好的连接可以实时查看当前程序的运行状况和堆栈信息等

![](https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/image-20210923164143750.png)

可以看到，我们jvm虚拟机的参数列表并没有内存限制相关参数。

我们为它加上参数再连接试试，将刚才的刚才的JAVA_OPTS参数修改为：

```shell
export JAVA_OPTS="-Xms512m -Xmx512m -Xss256m -Djava.rmi.server.hostname=xxx.xxx.xxx.xxx -Dcom.sun.management.jmxremote.port=8899 -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false"
```

![](https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/image-20210923165424944.png)

可以看到我们jvm已经有内存参数了，这时我们再看下这个eureka注册中心进程的内存占用。

![](https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/image-20210923165520445.png)

已经稳定在518M左右了，为什么是这个数呢，而不是我们设置的512M呢？因为我们的JVM进程除了heap内存，还有一些堆外内存。

## 总结

我们定位到了tomcat内存高的原因，也找到了解决的方法，但是这个问题我们目前只解决了一半。

我们仍然需要回答Xms或其他内存参数到底该设置多少才合理，这个数字不能拍脑袋，需要结合具体的业务需求，还需要与我们的测试小哥哥一起压测才能得出结论。