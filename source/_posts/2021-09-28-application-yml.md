---
title: application.yml配置文件优先级
comments: true
index_img: /img/girl01.jpg
banner_img: /img/background/vilige.jpg
date: 2021-06-20 09:12:32
updated: 2021-09-28 09:12:32
tags: [Spring,配置,JAVA]
categories: JAVA
---

在我们的微服务资源文件夹下，用于解耦配置信息的有多个`application.yml`文件，如

```shell
application.yml
application-common.yml
application-broker.yml
application-dev.yml
application-sit.yml
application-uat.yml
application-prod.yml
```

那么他们之间有什么区别？各文件的生效优先级又是怎么样的呢？

首先我们来了解一下各环境的定义

>dev：本地开发环境，主要供开发人员使用
>
>sit：测试环境，主要供测试人员使用
>
>uat：用户测试环境，一般使用部分生产数据，模拟生产环境，测试人员为真实用户
>
>prod：生产环境，商用环境，用户正在使用的环境。

环境分层以后，我们软件的交付流程、研发人员的职责也更加清晰。

如果有多个环境，那么可以通过让程序读取不同的配置文件来做到环境、资源隔离，如开发环境数小二连接的是开发本地的数据库、调用的开发环境ABM，不会影响测试环境及测试人员的工作。

从设计模式上来看，也可以用其他自定义后缀如`application-common.yml`文件来抽象公共的配置，如

```yml
#feign 与ribbon 超时等设置
feign:
  hystrix:
    enabled: true
ribbon:
  OkToRetryOnAllOperations: true
  ConnectTimeout: 2000
  ReadTimeout: 8000
  MaxAutoRetriesNextServer: 0
  MaxAutoRetries: 0
  eager-load:
    enabled: true #开启饥饿加载
    clients: asset,as,sm,core
#其他需要生效的application-XXX.yml文件
spring:
  profiles:
    active: common,broker
```

那么怎么怎么使这些文件协调地工作，key相同时value不冲突并且达到我们想要的效果呢。

我们需要指定生效的规则

```yml
#其他需要生效的application-XXX.yml文件
spring:
  profiles:
    active: common,broker
```



我们使用一个简单的SpringBoot Demo来验证一下

![](image-20210803145617104.png)

3个配置文件

`application.yml`

```yml
env: origin
```

`application-sit.yml`

```yml
env: sit
attr1: sit
```

`application-prod.yml`

```yml
env: prod
attr1: prod
attr2: prod
```



```java
//controller类
...
@RestController
class controller {
	//这种方式可以读取到配置文件中key为env的value
    @Value("${env}")
    String env;
	//":"后跟的是缺省值
    @Value("${attr1:none1}")
    String attr1;

    @Value("${attr2:none2}")
    String attr2;

    @RequestMapping("/test")
    public String test() {
        System.out.println("env:"+env);
        System.out.println("attr1:"+attr1);
        System.out.println("attr2:"+attr2);
        return env;

    }
}
```

执行后控制台打印

```java
env:origin
attr1:none1
attr2:none2
```

可以看出，`application-sit.yml`与`application-prod.yml`中定义的值并没有生效。

因为我们设置spring读取其他后缀的文件，因此只有`application,yml`文件生效了。

我们在`application,yml`中添加

```yml
spring:
  profiles:
   active: sit
```

控制台输出

```java
env:sit
attr1:sit
attr2:none2
```

`application.yml`的env值被sit中覆盖了,attr1也由缺省值变为了sit。

修改为

```yml
spring:
  profiles:
   active: sit,prod
```

控制台输出

```java
env:prod
attr1:prod
attr2:prod
```

prod文件覆盖了其他所有的文件的值

调整下active value的顺序

```yml
spring:
  profiles:
   active: prod,sit
```

控制台输出

```java
env:sit
attr1:sit
attr2:prod
```

可以看出，active中写的顺序越靠后，覆盖的优先级越高。



#### **结论**：

在值没有冲突的情况下，需要在spring.profiles.active中指定文件后缀来使对应配置生效。

在值有冲突的情况下，active中书写顺序越靠后的优先级越高。





