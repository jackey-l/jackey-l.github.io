---
title: (四)容器化实战--eureka改造
comments: true
index_img: 'https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/docker_k8s.png'
banner_img: >-
  https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/background/vilige.jpg
tags: [容器化,Docker,eureka]
date: 2021-06-04 16:20:56
updated: 2021-11-04 16:20:56
categories: 容器化技术
---

### 本章我们主要解决3个问题

- **docker 镜像私仓的搭建**

  上文提到，我们的源码直接推到dockerhub公网仓上是不安全的，我们需要一个私仓来放置我们的镜像，让k8s使用私仓各个版本的镜像去编排、发布微服务。如果在容器化的实践中，已经有了镜像仓库，那么这个步骤可以省略，在k8s的资源文件、及推送镜像的步骤当中，自行把本文提到的私仓地址替换为对应的仓库地址即可。

- **registry 、微服务集群无法互相注册。**

  原因为POD的IP不是固定的，随着每次Deployment、StatefulSet、Service的销毁、更新、回滚等，都会生成新的POD及IP，那么我们把IP写死在eureka以及微服务的配置文件中里肯定就无法寻址与路由了。k8s了Service的设计其实是用来提供类似eureka的功能的，但是由于本次改造时间有限，如果改动微服务架构，那么工作量太大，因此我们先继续沿用之前的微服务架构，在后面可以继续优化为。一种比较好的实践是k8s+SpringBoot+RPC调用的架构。

- **registry服务的状态问题。**
  registry目前仍然是无状态的，虽然我们使用了StatefulSet，但是并没有对他的持久化策略进行设置，我们的微服务日志需要保存在一个固定的卷，与POD的生命周期解耦。这个K8s也有对应的功能支持，我们在这个章节把这个问题解决。

---

### 问题1

首先我们来搭建一个docker私仓

>这里采用Docker提供的现成解决方案--Docker Registry
>
>官方文档：https://docs.docker.com/registry/

我们这里使用master01节点来搭建，这里我们就不做HA了。有兴趣的同学可以自己研究。

#### 1、	获取registry镜像

```shell
[root@master01 docker]# docker pull registry:2
```

#### 2、	运行registry容器

```shell
[root@dockerA ~]# docker run  -itd --name registry --restart=always  -p 5000:5000 -v /opt/registry:/var/lib/registry registry:2

//创建一个registry容器来运行registry服务；
//-p：端口映射（前面是宿主机端口：后面是容器暴露的端口）；
//-v：挂载目录（前面是宿主机的目录：后面的是容器的目录）自动创建宿主机的目录；
//--restart=always：随docker服务的启动而启动！
```

#### 3、	验证

往私仓上推送镜像

```shell
#按照规则重命名要推送的镜像，
#注：私有仓库镜像的命名规则：192.168.45.129:5000/XXX（宿主机的IP:5000端口/镜像名称:版本）
[root@master01 docker]# docker tag nginx:latest 192.168.60.128:5000/my_nginx:1.0
#推送
[root@master01 docker]# docker push 192.168.60.128:5000/my_nginx
#查看私藏镜像清单
[root@master01 docker]# curl -XGET http://192.168.60.128:5000/v2/_catalog
{"repositories":["my_hello","my_nginx"]}
```

在其他客户端下载镜像

```shell
[root@node02 ~]# docker pull 192.168.60.128:5000/my_nginx
Using default tag: latest
Error response from daemon: Get https://192.168.60.128:5000/v2/: http: server gave HTTP response to HTTPS client
#报错https请求错误，设置跳过https验证
[root@master01 docker]# cat /etc/docker/daemon.json
{
  "insecure-registries":["192.168.60.128:5000"],
  "registry-mirrors": ["http://192.168.60.128:5000"],
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
[root@node02 ~]# systemctl daemon-reload && systemctl restart docker

#成功拉取
[root@node02 ~]# docker pull 192.168.60.128:5000/my_nginx:1.0
1.0: Pulling from my_nginx
Digest: sha256:61191087790c31e43eb37caa10de1135b002f10c09fdda7fa8a5989db74033aa
Status: Downloaded newer image for 192.168.60.128:5000/my_nginx:1.0
```

#### 4、	如何删除Docker Registry

```shell
[root@master01 docker]# docker container stop registry && docker container rm -v registry
registry
```

---

### 问题2

#### 背景

对于一般的后端微服务来说，在k8s中同时起多个相同的服务来做负载均衡，只需要简单的修改deployment的replicas，增加pod数量，然后通过对外暴露一个service来代理这些pod。

而对于eureka来说，要实现eureka的高可用，那就不是修改replicas这么方便了。由于部署的多个eureka之间需要将自己注册到彼此，因此要做一些特殊改动。

主要是用到了StatefulSet和headless service这两个k8s对象

#### StatefulSet、Headless Service简介

##### StatefulSet

StatefulSet是为了解决有状态服务的问题（对应Deployments和ReplicaSets是为无状态服务而设计），其应用场景包括

稳定的持久化存储，即Pod重新调度后还是能访问到相同的持久化数据，基于PVC来实现

稳定的网络标志，即Pod重新调度后其PodName和HostName不变，基于Headless Service（即没有Cluster IP的Service）来实现

有序部署，有序扩展，即Pod是有顺序的，在部署或者扩展的时候要依据定义的顺序依次依次进行（即从0到N-1，在下一个Pod运行之前所有之前的Pod必须都是Running和Ready状态），基于init containers来实现

有序收缩，有序删除（即从N-1到0）

StatefulSet中每个Pod的DNS格式为

```cpp
statefulSetName-{0..N-1}.serviceName.namespace.svc.cluster.local
```

> serviceName为Headless Service的名字
> 0..N-1为Pod所在的序号，从0开始到N-1
> statefulSetName为StatefulSet的名字
> namespace为服务所在的namespace，Headless Service和StatefulSet必须在相同的namespace
> cluster.local为Cluster Domain

##### Headless Service

Headless Service 和普通service的一个显著的区别是，Headless Service的对应的每一个Endpoints，即每一个Pod，都会有对应的DNS域名
 例如：我们可以用过这种域名来访问某个具体的pod：

```css
statefulSetName-0.serviceName.namespace.svc.cluster.local
```

在实际使用中，将service的clusterIP设置成None，就表明这个service是一个Headless Service。

##### StatefulSet和Headless Service的结合

通过 StatefulSet，我们得到了一些列pod，每个pod的name为statefulSetName-{0..N-1}，
 加入我们创建了一个名称叫eureka的StatefulSet，并且设置replicas =3，那么部署到k8s后，k8s会为我们生成三个名称依次为eureka-0，eureka-1，eureka-2的pod。
 通过Headless Service，我们可以通过pod名称来访问某个pod，

例如，我们在namespace=test的命名空间下创建了一个名称为register-server的service，并且关联了之前StatefulSet创建的pod，那么我们可以在集群内任意地方
 通过eureka-0.register-server.test.svc.cluster.local这个域名访问到eureka-0这个pod。

#### 搭建

有了前面的基础，现在部署eureka集群的方式就逐渐清晰了。

首先明确部署eureka的关键点：需要让每个eureka注册到另外的eureka上。
 也就是eureka.client.serviceUrl.defaultZone这个配置，是一组eureka的地址。
 通过StatefulSet，我们可以明确知道生成的每个eureka的名称，
 通过Headless Service，我们又可以访问到每个eureka，所以eureka.client.serviceUrl.defaultZone的值就是

```bash
"http://registry-0.registry-service:8022/eureka,http://registry-0.registry-service:8022/eureka"
```

> 由于这两个个pod在同一个命名空间内，可以省略.namespace.svc.cluster.local

有个这个配置，那么我们部署StatefulSet，和Headless Service
 那么我们能基本能得到一个可用的eureka集群
 除了会有以下问题：
 红框中的可用副本（available-replicas）会出现在不可用unavailable-replicas中

![](https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/image-20211104173002335.png)

原因是我们默认是通过ip的方式来注册eureka（eureka.instance.prefer-ip-address配置默认为true），但是eureka的注册地址又是域名的形式，两者不一致。
 要解决这个问题，还需做一些额外的配置。

1.在application.yaml中，将eureka.instance.prefer-ip-address设置成false。

```bash
 eureka:
     instance:
        prefer-ip-address: false
```

2.registry-sts.yml中，增加环境变量配置，将pod的名称绑定到环境变量

```yml
...    
    spec:
      containers:
        - name: registry
          image: 192.168.60.128:5000/registry:1.0.0 
          imagePullPolicy: Always
          ports:
            - containerPort: 8022
          env: # 环境变量设置
            - name: EUREKA_URL_LIST 
              value: "http://registry-0.registry-service:8022/eureka,http://registry-0.registry-service:8022/eureka"
            - name: REGISTRY_PORT
              value: 8022
            - name: EUREKA_INSTANCE_HOSTNAME #环境变量，这个变量在我们项目的配置文件中有配置，作用是指定注册到eureka集群中的hostname
              value: ${MY_POD_NAME}.registry-service
            - name: MY_NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            - name: MY_POD_NAMESPACE # POD当前命名空间
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: MY_POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: MY_IN_SERVICE_NAME # 因为pod 通过域名互相访问，需要使用headless 服务名称
              value: registry-service
            - name: EUREKA_APPLICATION_NAME
              value: "registry"
            - name: EUREKA_REPLICAS
              value: "2"
...
```

3.在application.yml中配置

```yml
spring:
  application:
    name: ${EUREKA_APPLICATION_NAME:registry}
server:
  port: ${REGISTRY_PORT:8022}
eureka:
  client:
    service-url:
      defaultZone: ${EUREKA_URL_LIST:http://localhost:8022/eureka} # 指定服务中心 eureka server的地址
    register-with-eureka: ${BOOL_REGISTER:true} # 是否把服务中心本身当做eureka client 注册。默认为true
    fetch-registry: ${BOOL_FETCH:true} # 是否拉取 eureka server 的注册信息。 默认为true
  instance:
    hostname: ${MY_POD_NAME:peer1} #服务主机名
    appname: ${spring.application.name} #服务名称，默认为 unknow 这里直接取 spring.application.name
    prefer-ip-address: false #默认为true 表示为使用IP注册 可用副本（available-replicas）会出现在不可用unavailable-replicas中，因此设为false
```

这里我们选择让镜像打包脚本把这个文件替换到registry的依赖目录下

把`application.yml`放在config目录下

![](https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/image-20210630191106294.png)

修改打包脚本中的`modify_projectName_registry_service`函数

```shell
modify_projectName_registry_service()
{
	echo "INFO" "begin modify registry service ..."
    application=${CONF_PATH}/application.yml
    application_common=${CONF_PATH}/application-common.yml
    application_broker=${CONF_PATH}/application-broker.yml
    application_path=${REGISTRY_USER_HOME}/tomcat/webapps/${projectName_REGISTRY_SERVICE}/WEB-INF/classes/

    \sudo cp -rf ${application} ${application_path}
    sudo cp ${application_common} ${application_path}
    sudo cp ${application_broker} ${application_path}
	  file_application_yml=${REGISTRY_USER_HOME}/tomcat/webapps/${projectName_REGISTRY_SERVICE}/WEB-INF/classes/application.yml
    sudo su - ${REGISTRY_USER} <<EOF

    if [[ ! -f ${file_application_yml} ]];then
        echo "[ file ${file_application_yml} ] is not exist,pls check~"
        return 1
    fi

EOF
  sudo chown -R ${REGISTRY_USER}:${REGISTRY_USER_GROUP_NAME} ${REGISTRY_USER_HOME}
	echo "INFO" "successful modify registry service ..."

}
```

重新打包镜像，重命名并推送至我们之前搭建好的私仓，注意版本要与我们资源文件中的设置一致。

k8s的registry资源配置文件`registry-sts.yml`

```shell
apiVersion: v1
kind: Namespace
metadata:
  name: projectName
---
# 有状态pod，会自动根据metadata.name及实例数量生产xxx-0,xxx-1的pod
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: registry
  namespace: projectName
spec:
  serviceName: registry-service
  replicas: 2
  selector:
    matchLabels:
      app: registry
  template:
    metadata:
      labels:
        app: registry
    spec:
      containers:
        - name: registry
          image: 192.168.60.128:5000/registry:1.0.0 # 配置镜像路径，也就是我们刚才push好的镜像
          imagePullPolicy: Always #配置总是拉取镜像，可以配置成如有本地有就不拉取。
          ports:
            - containerPort: 8022
          env: # 环境变量设置
            - name: MY_POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name # 取值metadata.name，这里注意，statefulset类型的对象取到的是有-0,-1序号后缀的
            - name: EUREKA_URL_LIST #环境变量，这个变量在我们项目的配置文件中有配置,作用是配置eureka集群的所有地址，注意相同namespace底下使用dns访问时，不需要配置全路径（全路径为podname.headless-name.namespace.cluster.svc.local),只要到podname.headless-name
              value: "http://registry-0.registry-service:8022/eureka,http://registry-0.registry-service:8022/eureka"
            - name: REGISTRY_PORT
              value: 8022
            - name: EUREKA_INSTANCE_HOSTNAME #环境变量，这个变量在我们项目的配置文件中有配置，作用是指定注册到eureka集群中的hostname
              value: ${MY_POD_NAME}.registry-service
            - name: MY_NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            - name: MY_POD_NAMESPACE # POD当前命名空间
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: MY_POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: MY_IN_SERVICE_NAME # 因为pod 通过域名互相访问，需要使用headless 服务名称
              value: registry-service
            - name: EUREKA_APPLICATION_NAME
              value: "registry"
            - name: EUREKA_REPLICAS
              value: "3"
  podManagementPolicy: "Parallel" # 以并行方式创建pod，默认是串行的
---
# file name eureka.yaml
# headless service
apiVersion: v1
kind: Service
metadata:
  name: registry-service
  namespace: projectName
  labels:
    app: registry
spec:
  clusterIP: None
  ports:
    - port: 8022
      name: registry-svc-port
      targetPort: 8022
  selector:
    app: registry
```

执行 kubectl create命令后，便可以得到一个eureka集群。

```shell
[root@master01 k8s-projectName]# kubectl create -f registry-sts.yml
namespace/projectName created
statefulset.apps/registry created
service/registry-service created
```

#### 验证

```shell
#可以看到我们2个POD都是健康状态了
[root@master01 k8s-projectName]# kubectl get pods -n projectName
NAME                   READY   STATUS    RESTARTS   AGE
registry-0   1/1     Running   0          7h37m
registry-1   1/1     Running   0          7h37m
#进入任意POD 查看一下eureka portal
[root@master01 k8s-projectName]# kubectl exec -it registry-0 -n projectName /bin/bash
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
[registry@registry-0 /]$ curl localhost:8022
...
        <tbody>
              <tr>
                <td><b>REGISTRY</b></td>
                <td>
                    <b>n/a</b> (2)
                </td>
                <td>
                    <b></b> (2)
                </td>
                <td>
                    <b>UP</b> (2) -
                        <a href="http://registry-1.registry-service:8022/actuator/info" target="_blank">registry-1.registry-service.projectName.svc.cluster.local:registry:8022</a>
,
                        <a href="http://registry-0.registry-service:8022/actuator/info" target="_blank">registry-0.registry-service.projectName.svc.cluster.local:registry:8022</a>

                </td>
              </tr>

        </tbody>
      </table>
...
```

##### 可以看到我们两个eureka实例已经相互注册了，那么微服务的注册功能我们后面还需要对微服务进行微调。

#### 微服务注册

将一般的微服务注册到eureka集群中，可以通过eureka的service来访问eureka，即：将eureka.client.serviceUrl.defaultZone设置成register-server.test.svc.cluster.local，使用了k8s的service负载均衡，将服务注册到任意一个活着的eureka上，然后eureka集群内部会做同步，最终注册到eureka集群内部所有eureka上

---

```shell
[registry@registry-0 logs]$ cat catalina.log
2021-06-30 03:44:59,768 [main] WARN  org.apache.catalina.security.SecurityListener- No umask setting was found in system property [org.apache.catalina.security.SecurityListener.UMASK]. However, it appears Tomcat is running on a platform that supports umask. The system property is typically set in CATALINA_HOME/bin/catalina.sh. The Lifecycle listener org.apache.catalina.security.SecurityListener (usually configured in CATALINA_BASE/conf/server.xml) expects a umask at least as restrictive as [0027]
2021-06-30 03:44:59,770 [main] INFO  org.apache.catalina.core.AprLifecycleListener- The APR based Apache Tomcat Native library which allows optimal performance in production environments was not found on the java.library.path: /usr/java/packages/lib/amd64:/usr/lib64:/lib64:/lib:/usr/lib
2021-06-30 03:45:00,506 [main] INFO  org.apache.coyote.http11.Http11NioProtocol- Initializing ProtocolHandler ["http-nio-8022"]
2021-06-30 03:45:00,554 [main] INFO  org.apache.tomcat.util.net.NioSelectorPool- Using a shared selector for servlet write/read
2021-06-30 03:45:00,561 [main] INFO  org.apache.catalina.startup.Catalina- Initialization processed in 2475 ms
2021-06-30 03:45:00,603 [main] INFO  org.apache.catalina.core.StandardService- Starting service Catalina
2021-06-30 03:45:00,603 [main] INFO  org.apache.catalina.core.StandardEngine- Starting Servlet Engine:
2021-06-30 03:45:02,199 [localhost-startStop-1] WARN  org.apache.catalina.startup.ContextConfig- catalina annotation was enabled
2021-06-30 03:45:58,222 [main] INFO  org.apache.coyote.http11.Http11NioProtocol- Starting ProtocolHandler ["http-nio-8022"]
2021-06-30 03:45:58,266 [main] INFO  org.apache.catalina.startup.Catalina- Server startup in 57704 ms

```

![](https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/image-20210713095957693.png)

```shell
apiVersion: v1
kind: Service
metadata:
  name: registry-service-o
  namespace: projectName
  labels:
    app: registry
spec:
  type: NodePort
  ports:
    - port: 8022
      name: registry-svc-port
      targetPort: 8022
      nodePort: 30022
  selector:
    app: registry
```

```shell
[root@master01 k8s-projectName]# kubectl get svc -A
...
projectName              registry-service            ClusterIP   None          <none>        8022/TCP                 16h
projectName              registry-service-o          NodePort    10.1.42.201   <none>        8022:30022/TCP           6m33s

```

### 问题3

> k8s有几种容器持久化方案，这里我们使用的是基于nfs的PV/PVC，其余的方案这里暂时不讨论了。
>
> 这块我们放在K8s里面介绍，链接待更新...

#### 安装NFS

在作为NFS服务的节点执行

```shell
[root@localhost ~]# yum -y install nfs-utils  rpcbind
```

#### 创建共享目录

```shell
[root@localhost ~]# mkdir /nfsdata
```

#### **创建共享目录的权限**

```shell
[root@master ~]# vi /etc/exports
/nfsdata *(rw,sync,no_root_squash)
```

#### 开启nfs和rpcbind

```shell
[root@master ~]# systemctl start nfs-server.service 
[root@master ~]# systemctl start rpcbind
```

#### 测试一下

```shell
[root@localhost /]# showmount -e
Export list for localhost.localdomain:
/nfsdata *
```

#### 在master 和其他节点执行

```shell
yum install -y nfs-utils
```

#### 设置tomcat的日志输出路径

更改tomcat的日志输出格式通过pod名和pod命名空间变量，唯一标识输出文件名。

给所有的log前缀加上 `${my.pod.name}_${my.pod.namespace}_`

```xml
[vincent@node02 conf]$ cat log4j.xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE log4j:configuration SYSTEM "log4j.dtd">

<log4j:configuration xmlns:log4j='http://jakarta.apache.org/log4j/' >

        <!--  tomcat log-->
        <appender name="CATALINA" class="com.huawei.openas.common.log.appender.RollingFileAppenderMod">
                <param name="File" value="${catalina.base}/logs/${my.pod.name}_${my.pod.namespace}_catalina.log" />
                <param name="Append" value="true" />
                <param name="Encoding" value="UTF-8" />
                <param name="MaxFileSize" value="10MB" />
                <param name="MaxBackupIndex" value="10" />
                <layout class="org.apache.log4j.PatternLayout">
                        <param name="ConversionPattern" value="%d [%t] %-5p %c- %m%n" />
                </layout>
        </appender>

        <appender name="STD_OUT_REDIRECTOR" class="com.huawei.openas.common.log.appender.LazyStdOutRollingFileAppender">
     <errorHandler class="com.huawei.openas.common.log.appender.CatalinaErrorhandler"/>
                <param name="File" value="${catalina.base}/logs/${my.pod.name}_${my.pod.namespace}_catalina.out" />
                <param name="Append" value="true" />
                <param name="Encoding" value="UTF-8" />
                <param name="MaxFileSize" value="10MB" />
                <param name="MaxBackupIndex" value="10" />
                <layout class="org.apache.log4j.PatternLayout">
                        <param name="ConversionPattern" value="%d [%t] %-5p %m%n" />
                </layout>
        </appender>

        <appender name="LOCALHOST" class="com.huawei.openas.common.log.appender.RollingFileAppenderMod">
                <param name="File" value="${catalina.base}/logs/${my.pod.name}_${my.pod.namespace}_localhost.log" />
                <param name="Append" value="true" />
                <param name="Encoding" value="UTF-8" />
                <param name="MaxFileSize" value="10MB" />
                <param name="MaxBackupIndex" value="10" />
                <layout class="org.apache.log4j.PatternLayout">
                        <param name="ConversionPattern" value="%d [%t] %-5p %c- %m%n" />
                </layout>
        </appender>

        <appender name="CONSOLE" class="org.apache.log4j.ConsoleAppender">
                <param name="Encoding" value="UTF-8" />
                <layout class="org.apache.log4j.PatternLayout">
                        <param name="ConversionPattern"
                                value="%d %n%p: %m%n" />
                </layout>
        </appender>

        <appender name="SWITCHABLE_CONSOLE_APPENDER" class="com.huawei.openas.common.log.appender.SwitchableConsoleAppender" />

        <appender name="accesslog" class="com.huawei.openas.common.log.appender.RollingFileAppenderMod">
                <param name="File" value="${catalina.base}/logs/${my.pod.name}_${my.pod.namespace}_localhost_access.log" />
                <param name="Append" value="true" />
                <param name="Encoding" value="UTF-8" />
                <param name="MaxFileSize" value="100MB" />
                <param name="MaxBackupIndex" value="10" />
                <layout class="org.apache.log4j.PatternLayout">
                        <param name="ConversionPattern" value="%d [%t] %-5p %c- %m%n" />
                </layout>
        </appender>

        <!--  DBPool log (Custom Appender)-->
        <appender name="dbpool" class="com.huawei.openas.common.log.appender.LazyRollingFileAppenderMod">
                <param name="ModuleCheckerClass" value="com.huawei.bme.dbconnector.dbpool.PoolDataSource" />
                <param name="File" value="${catalina.base}/logs/dbpool/${my.pod.name}_${my.pod.namespace}_dbpool.log" />
                <param name="Append" value="true" />
                <param name="Encoding" value="UTF-8" />
                <param name="MaxFileSize" value="10MB" />
                <param name="MaxBackupIndex" value="10" />
                <layout class="org.apache.log4j.PatternLayout">
                        <param name="ConversionPattern" value="%d{yyyy-MM-dd HH:mm:ss.SSS}|%-5p|%t|JTA > %m (%l)%n" />
                </layout>
        </appender>

        <!--  xa transaction recovery log (Custom Appender)-->
        <appender name="xa_transaction_recovery" class="com.huawei.openas.common.log.appender.RollingFileAppenderMod">
                <param name="File" value="${catalina.base}/logs/dbpool/${my.pod.name}_${my.pod.namespace}_xa_transaction.log" />
                <param name="Append" value="true" />
                <param name="Encoding" value="UTF-8" />
                <param name="MaxFileSize" value="10MB" />
                <param name="MaxBackupIndex" value="10" />
                <layout class="org.apache.log4j.PatternLayout">
                        <param name="ConversionPattern" value="%d{yyyy-MM-dd HH:mm:ss.SSS}|%-5p|%t|JTA > %m (%l)%n" />
                </layout>
        </appender>

        <!--  Mbean log-->
        <appender name="mbTool" class="com.huawei.openas.common.log.appender.RollingFileAppenderMod">
                <param name="File" value="${catalina.base}/logs/mbeanTool/${my.pod.name}_${my.pod.namespace}_mbeanTool.log" />
                <param name="Append" value="true" />
                <param name="Encoding" value="UTF-8" />
                <param name="MaxFileSize" value="10MB" />
                <param name="MaxBackupIndex" value="10" />
                <layout class="org.apache.log4j.PatternLayout">
                        <param name="ConversionPattern" value="%d{yyyy-MM-dd HH:mm:ss.SSS}|%-5p|%t|MBeanTool > %m (%l)%n" />
                </layout>
        </appender>


        <!--  monitor log-->
        <appender name="monitor" class="com.huawei.openas.common.log.appender.RollingFileAppenderMod">
                <param name="File" value="${catalina.base}/logs/monitor/${my.pod.name}_${my.pod.namespace}_monitor.log" />
                <param name="Append" value="true" />
                <param name="Encoding" value="UTF-8" />
                <param name="MaxFileSize" value="10MB" />
                <param name="MaxBackupIndex" value="10" />
                <layout class="org.apache.log4j.PatternLayout">
                        <param name="ConversionPattern" value="%d{yyyy-MM-dd HH:mm:ss.SSS}|%-5p|%t|Monitor > %m (%l)%n" />
                </layout>
        </appender>

        <!--  sessionkeeper log-->
        <!--<appender name="sessionkeeper" class="org.apache.log4j.RollingFileAppender">
                <param name="File" value="${catalina.base}/logs/sessionkeeper/sessionkeeper.log" />
                <param name="Append" value="true" />
                <param name="Encoding" value="UTF-8" />
                <param name="MaxFileSize" value="10MB" />
                <param name="MaxBackupIndex" value="10" />
                <layout class="org.apache.log4j.PatternLayout">
                        <param name="ConversionPattern" value="%d{yyyy-MM-dd HH:mm:ss.SSS}|%-5p|%t|Monitor > %m (%l)%n" />
                </layout>
        </appender>-->

        <!--  dcs log-->
        <!--<appender name="dcs" class="org.apache.log4j.RollingFileAppender">
                <param name="File" value="${catalina.base}/logs/dcs/dcs.log" />
                <param name="Append" value="true" />
                <param name="Encoding" value="UTF-8" />
                <param name="MaxFileSize" value="10MB" />
                <param name="MaxBackupIndex" value="10" />
                <layout class="org.apache.log4j.PatternLayout">
                        <param name="ConversionPattern" value="%d{yyyy-MM-dd HH:mm:ss.SSS}|%-5p|%t|Monitor > %m (%l)%n" />
                </layout>
        </appender>-->
        <!--  coherence log-->
        <!--<appender name="coherenceMonitor" class="org.apache.log4j.RollingFileAppender">
                <param name="File" value="${catalina.base}/logs/coherence/coherence.log" />
                <param name="Append" value="true" />
                <param name="Encoding" value="UTF-8" />
                <param name="MaxFileSize" value="10MB" />
                <param name="MaxBackupIndex" value="10" />
                <layout class="org.apache.log4j.PatternLayout">
                        <param name="ConversionPattern" value="%d{yyyy-MM-dd HH:mm:ss.SSS}|%-5p|%t|Monitor > %m (%l)%n" />
                </layout>
        </appender>-->
...
```

通过容器传入的变量在tomcat中并不能直接使用,需要在tomcat的启动脚本中，在java的启动命令中加上-D选项。找到所有有-Dcatalina.base="\"$CATALINA_BASE\"" 的地方加上

`-Dmy.pod.name="$MY_POD_NAME"
-Dmy.pod.namespace="$MY_POD_NAMESPACE"`

```shell
...
#例如
elif [ "$1" = "configtest" ] ; then
    JAVA_OPTS="$JAVA_OPTS "-Dopenas.catalina.out.log.file.control=off
    eval "\"$_RUNJAVA\"" $JAVA_OPTS \
      -D$ENDORSED_PROP="\"$JAVA_ENDORSED_DIRS\"" \
      -classpath "\"$CLASSPATH\"" \
      -Dcatalina.base="\"$CATALINA_BASE\"" \
      -Dcatalina.home="\"$CATALINA_HOME\"" \
      -Dmy.pod.name="$MY_POD_NAME" \
      -Dmy.pod.namespace="$MY_POD_NAMESPACE" \

...
```

#### 修改构建脚本

将`catalina.sh`与`log4j.xml`文件夹添加到dockerfile根目录下的config目录

在之前的镜像安装脚本`install_projectName_registry.sh`中添加

```shell
 ...
  #copy log4j.xml to user home openas config
    sudo cp ${CONF_PATH}/log4j.xml ${USER_HOME}/tomcat/conf
    if [[ $? -ne 0 ]];then
        echo "ERROR" "[ cp ${CONF_PATH}/web.xml ${USER_HOME}/tomcat/conf ] failed"
        return 1
    fi

    #copy catalina.sh to user home openas bin
    sudo cp ${CONF_PATH}/catalina.sh ${USER_HOME}/tomcat/bin
    if [[ $? -ne 0 ]];then
        echo "ERROR" "[ cp ${CONF_PATH}/web.xml ${USER_HOME}/tomcat/bin ] failed"
        return 1
    fi
...
```

为了方便，把push动作也放到镜像打包脚本`build_registry_image.sh`中,镜像版本和URL解耦为变量。

```shell
pushdImage(){

    docker tag ${IMAGE_NAME}:${IMAGE_VERSION} ${DOCKER_REGISTRY_URL}/${IMAGE_NAME}:${IMAGE_VERSION} || return 1

    if [[ $? -ne 0 ]];then
       LOG "ERROR" "${DOCKER_REGISTRY_URL}/${IMAGE_NAME}:${IMAGE_VERSION}Image tag  failed"
        return 1
    fi

    docker push ${DOCKER_REGISTRY_URL}/${IMAGE_NAME}:${IMAGE_VERSION} || return 1

    if [[ $? -ne 0 ]];then
        LOG "ERROR" "${DOCKER_REGISTRY_URL}/${IMAGE_NAME}:${IMAGE_VERSION}Image push  failed"
        return 1
    fi

}
...
    pushdImage || return 1

    echo "======================================================================="
}
```

重新打包并push镜像

#### 修改k8s资源文件

为registry服务添加PVC和PV，这里我们抽象出一个公共资源文件`k8s-env.yml`用于准备命名空间和pv等。

```shell
apiVersion: v1
kind: Namespace
metadata:
  name: projectName
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: registry-nfs-pv
  namespace: projectName
  labels:
    pv: registry-nfs-pv
spec:
  capacity: #pv容量的大小
    storage: 1024Mi
  accessModes: #访问pv的模式
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: nfs  #持久资源的类型
  nfs:
    # FIXME: use the right IP
    server: 192.168.60.144
    path: "/nfsdata/registry/tomcat/"
---
apiVersion: v1 #这里先定义好core的pv，后面我们会用到
kind: PersistentVolume
metadata:
  name: core-nfs-pv
  namespace: projectName
  labels:
    pv: core-nfs-pv
spec:
  capacity: #pv容量的大小
    storage: 1024Mi
  accessModes: #访问pv的模式
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: nfs
  nfs:
    # FIXME: use the right IP
    server: 192.168.60.144
    path: "/nfsdata/core/"
```

在`regitsry-sts.yml`添加

```shell
kind: PersistentVolumeClaim  #pvc 
apiVersion: v1
metadata:
  name: registry-nfs-pvc
  namespace: projectName
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1000Mi   #不能大于绑定pv的容量
  storageClassName: nfs #需要和pv的sc类型一致，否则会绑定失败
  selector:
    matchLabels:
      pv: registry-nfs-pv
---
# 有状态pod，会自动根据metadata.name及实例数量生产xxx-0,xxx-1的pod
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: registry
  namespace: projectName
spec:
  serviceName: registry-service
  replicas: 2
  selector:
    matchLabels:
      app: registry
  template:
    metadata:
      labels:
        app: registry
    spec:
      containers:
        - name: registry
          image: 192.168.60.128:5000/registry:1.0.0 # 配置镜像路径，也就是我们刚才push好的镜像
          imagePullPolicy: Always
          ports:
            - containerPort: 8022
          volumeMounts:
            - mountPath: /data01/registry/tomcat/logs
              name: registry-data
          env: # 环境变量设置
            - name: MY_POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name # 取值metadata.name，这里注意，statefulset类型的对象取到的是有-0,-1序号后缀的
            - name: EUREKA_URL_LIST #环境变量，这个变量在我们项目的配置文件中有配置,作用是配置eureka集群的所有地址，注意相同namespace底下使用dns访问时，不需要配置全路径（全路径为podname.headless-name.namespace.cluster.svc.local),只要到podname.headless-name
              value: "http://registry-0.registry-service:8022/eureka,http://registry-0.registry-service:8022/eureka"
            - name: REGISTRY_PORT
              value: "8022"
            - name: EUREKA_INSTANCE_HOSTNAME #环境变量，这个变量在我们项目的配置文件中有配置，作用是指定注册到eureka集群中的hostname
              value: ${MY_POD_NAME}.registry-service
            - name: MY_NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            - name: MY_POD_NAMESPACE # POD当前命名空间
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: MY_POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: MY_IN_SERVICE_NAME # 因为pod 通过域名互相访问，需要使用headless 服务名称
              value: registry-service
            - name: EUREKA_APPLICATION_NAME
              value: "registry"
            - name: EUREKA_REPLICAS
              value: "2"
      volumes:
        - name: registry-data
          persistentVolumeClaim:
            claimName: registry-nfs-pvc
  podManagementPolicy: "Parallel" # 以并行方式创建pod，默认是串行的
---
# file name eureka.yaml
# headless service
apiVersion: v1
kind: Service
metadata:
  name: registry-service
  namespace: projectName
  labels:
    app: registry
spec:
  clusterIP: None
  ports:
    - port: 8022
      name: registry-svc-port
      targetPort: 8022
  selector:
    app: registry

```

清除全部资源并重新创建k8s资源，注意依赖关系先删除`registry-sts.yml`。

```shell
[root@master01 k8s-projectName]kubectl delete -f registry-sts.yml
[root@master01 k8s-projectName]kubectl delete -f k8s-env.yml
[root@master01 k8s-projectName]kubectl create k8s-env.yml
[root@master01 k8s-projectName]kubectl create registry-sts.yml
```

#### 检查

##### pv

```shell
[root@master01 k8s-projectName]# kubectl get pv -A
NAME                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                                  STORAGECLASS   REASON   AGE
core-nfs-pv       1Gi        RWX            Recycle          Available                                          nfs                     23m
registry-nfs-pv   1Gi        RWX            Recycle          Bound       projectName/registry-nfs-pvc   nfs                     23m
[root@master01 k8s-projectName]# kubectl describe pv/registry-nfs-pv -n projectName
Name:            registry-nfs-pv
Labels:          pv=registry-nfs-pv
Annotations:     pv.kubernetes.io/bound-by-controller: yes
Finalizers:      [kubernetes.io/pv-protection]
StorageClass:    nfs
Status:          Bound
Claim:           projectName/registry-nfs-pvc
Reclaim Policy:  Recycle
Access Modes:    RWX
VolumeMode:      Filesystem
Capacity:        1Gi
Node Affinity:   <none>
Message:
Source:
    Type:      NFS (an NFS mount that lasts the lifetime of a pod)
    Server:    192.168.60.144
    Path:      /nfsdata/registry/tomcat/
    ReadOnly:  false
Events:        <none>
```

##### pvc

```shell
[root@master01 k8s-projectName]# kubectl get pvc -A
NAMESPACE   NAME                         STATUS   VOLUME                      CAPACITY   ACCESS MODES   STORAGECLASS   AGE
projectName   registry-nfs-pvc   Bound    registry-nfs-pv   1Gi        RWX            nfs            26m
[root@master01 k8s-projectName]# kubectl describe pvc/registry-nfs-pvc -n projectName
Name:          registry-nfs-pvc
Namespace:     projectName
StorageClass:  nfs
Status:        Bound
Volume:        registry-nfs-pv
Labels:        <none>
Annotations:   pv.kubernetes.io/bind-completed: yes
               pv.kubernetes.io/bound-by-controller: yes
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:      1Gi
Access Modes:  RWX
VolumeMode:    Filesystem
Mounted By:    registry-0
               registry-1
Events:        <none>
```

##### pod

```shell
[root@master01 k8s-projectName]# kubectl get pods -n projectName
NAME                   READY   STATUS    RESTARTS   AGE
registry-0   1/1     Running   0          22m
registry-1   1/1     Running   0          27m
[root@master01 k8s-projectName]# kubectl describe pods -n projectName
Name:         registry-0
Namespace:    projectName
Priority:     0
Node:         node01/192.168.60.129
Start Time:   Sat, 03 Jul 2021 04:25:30 -0400
Labels:       app=registry
              controller-revision-hash=registry-69fb95c8c6
              statefulset.kubernetes.io/pod-name=registry-0
Annotations:  <none>
Status:       Running
IP:           10.244.1.25
IPs:
  IP:           10.244.1.25
Controlled By:  StatefulSet/registry
Containers:
  registry:
    Container ID:   docker://1270c57122d4f9effa1e3aa494648942efe1ea050f5b9a51045bdb5d77da90d8
    Image:          192.168.60.128:5000/registry:1.0.0
    Image ID:       docker-pullable://192.168.60.128:5000/registry@sha256:385e3dffdb979469797150c41d537883c8271308c522208a41d81be6d854299e
    Port:           8022/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Sat, 03 Jul 2021 04:25:33 -0400
    Ready:          True
    Restart Count:  0
    Environment:
      MY_POD_NAME:               registry-0 (v1:metadata.name)
      EUREKA_URL_LIST:           http://registry-0.registry-service:8022/eureka,http://registry-0.registry-service:8022/eureka
      REGISTRY_PORT:             8022
      EUREKA_INSTANCE_HOSTNAME:  ${MY_POD_NAME}.registry-service
      MY_NODE_NAME:               (v1:spec.nodeName)
      MY_POD_NAMESPACE:          projectName (v1:metadata.namespace)
      MY_POD_IP:                  (v1:status.podIP)
      MY_IN_SERVICE_NAME:        registry-service
      EUREKA_APPLICATION_NAME:   registry
      EUREKA_REPLICAS:           2
    Mounts:
      /data01/registry/tomcat/logs from registry-data (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-zrfkg (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  registry-data:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  registry-nfs-pvc
    ReadOnly:   false
  default-token-zrfkg:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-zrfkg
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                 node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  22m   default-scheduler  Successfully assigned projectName/registry-0 to node01
  Normal  Pulling    22m   kubelet            Pulling image "192.168.60.128:5000/registry:1.0.0"
  Normal  Pulled     22m   kubelet            Successfully pulled image "192.168.60.128:5000/registry:1.0.0" in 48.902535ms
  Normal  Created    22m   kubelet            Created container registry
  Normal  Started    22m   kubelet            Started container registry


Name:         registry-1
Namespace:    projectName
Priority:     0
Node:         node02/192.168.60.130
Start Time:   Sat, 03 Jul 2021 04:20:38 -0400
Labels:       app=registry
              controller-revision-hash=registry-69fb95c8c6
              statefulset.kubernetes.io/pod-name=registry-1
Annotations:  <none>
Status:       Running
IP:           10.244.2.19
IPs:
  IP:           10.244.2.19
Controlled By:  StatefulSet/registry
Containers:
  registry:
    Container ID:   docker://35ca2fd2a67d37b373c37de9ab7ae52bc91c4d1abab87ce3e3b99bc5a5d3bb1a
    Image:          192.168.60.128:5000/registry:1.0.0
    Image ID:       docker-pullable://192.168.60.128:5000/registry@sha256:385e3dffdb979469797150c41d537883c8271308c522208a41d81be6d854299e
    Port:           8022/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Sat, 03 Jul 2021 04:21:10 -0400
    Ready:          True
    Restart Count:  0
    Environment:
      MY_POD_NAME:               registry-1 (v1:metadata.name)
      EUREKA_URL_LIST:           http://registry-0.registry-service:8022/eureka,http://registry-0.registry-service:8022/eureka
      REGISTRY_PORT:             8022
      EUREKA_INSTANCE_HOSTNAME:  ${MY_POD_NAME}.registry-service
      MY_NODE_NAME:               (v1:spec.nodeName)
      MY_POD_NAMESPACE:          projectName (v1:metadata.namespace)
      MY_POD_IP:                  (v1:status.podIP)
      MY_IN_SERVICE_NAME:        registry-service
      EUREKA_APPLICATION_NAME:   registry
      EUREKA_REPLICAS:           2
    Mounts:
      /data01/registry/tomcat/logs from registry-data (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-zrfkg (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  registry-data:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  registry-nfs-pvc
    ReadOnly:   false
  default-token-zrfkg:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-zrfkg
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                 node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  27m   default-scheduler  Successfully assigned projectName/registry-1 to node02
  Normal  Pulling    27m   kubelet            Pulling image "192.168.60.128:5000/registry:1.0.0"
  Normal  Pulled     27m   kubelet            Successfully pulled image "192.168.60.128:5000/registry:1.0.0" in 25.890524219s
  Normal  Created    27m   kubelet            Created container registry
  Normal  Started    27m   kubelet            Started container registry

```

检查两个registry节点日志是否按照设定格式存入nfs服务器对应目录下

```shell
[root@localhost tomcat]# pwd
/nfsdata/registry/tomcat
[root@localhost tomcat]# ll
total 76
drwxr-x---. 2 1000 1000   120 Jul  3 04:21 dbpool
drwxr-x---. 2 1000 1000   110 Jul  3 04:21 mbeanTool
drwxr-x---. 2 1000 1000   106 Jul  3 04:21 monitor
-rw-r-----. 1 1000 1000  3942 Jul  3 04:26 registry-0_projectName_catalina.log
-rw-r-----. 1 1000 1000 55446 Jul  3 04:51 registry-0_projectName_localhost_access.log
-rw-r-----. 1 1000 1000  1397 Jul  3 04:26 registry-0_projectName_localhost.log
-rw-r-----. 1 1000 1000  1722 Jul  3 04:22 registry-1_projectName_catalina.log
-rw-r-----. 1 1000 1000   211 Jul  3 04:22 registry-1_projectName_localhost_access.log
-rw-r-----. 1 1000 1000   351 Jul  3 04:21 registry-1_projectName_localhost.log
```

可以看到已经日志已经与pod解耦了，我们使用`kubectl delete`命令模拟下pod挂掉的情况，看之前的日志是否能保留

我们先看下`registry-0`节点的日志

```shell
[root@localhost tomcat]# cat registry-0_projectName_catalina.log
...
2021-07-03 08:25:35,412 [main] INFO  org.apache.coyote.http11.Http11NioProtocol- Initializing ProtocolHandler ["http-nio-8022"]
2021-07-03 08:25:35,437 [main] INFO  org.apache.tomcat.util.net.NioSelectorPool- Using a shared selector for servlet write/read
2021-07-03 08:25:35,441 [main] INFO  org.apache.catalina.startup.Catalina- Initialization processed in 1308 ms
2021-07-03 08:25:35,468 [main] INFO  org.apache.catalina.core.StandardService- Starting service Catalina
2021-07-03 08:25:35,469 [main] INFO  org.apache.catalina.core.StandardEngine- Starting Servlet Engine:
2021-07-03 08:25:36,051 [localhost-startStop-1] WARN  org.apache.catalina.startup.ContextConfig- catalina annotation was enabled
2021-07-03 08:25:55,509 [main] INFO  org.apache.coyote.http11.Http11NioProtocol- Starting ProtocolHandler ["http-nio-8022"]
2021-07-03 08:25:55,535 [main] INFO  org.apache.catalina.startup.Catalina- Server startup in 20094 ms
```

> 这里我们意外发现了个小问题，节点上的时间还没有统一，这个放在后面解决

我们delete一个节点，然后等待k8s重新拉起。

```shell
[root@master01 k8s-projectName]# kubectl delete pod/registry-0 -n projectName
pod "registry-0" deleted
[root@master01 k8s-projectName]# kubectl get pods -n projectName
NAME                   READY   STATUS        RESTARTS   AGE
registry-0   0/1     Terminating   0          32m
registry-1   1/1     Running       0          37m
[root@master01 k8s-projectName]# kubectl get pods -n projectName
NAME                   READY   STATUS    RESTARTS   AGE
registry-0   1/1     Running   0          5s
registry-1   1/1     Running   0          37m

```

nfs日志

```shell
[root@localhost tomcat]# cat registry-0_projectName_catalina.log
...
2021-07-03 08:25:35,412 [main] INFO  org.apache.coyote.http11.Http11NioProtocol- Initializing ProtocolHandler ["http-nio-8022"]
2021-07-03 08:25:35,437 [main] INFO  org.apache.tomcat.util.net.NioSelectorPool- Using a shared selector for servlet write/read
2021-07-03 08:25:35,441 [main] INFO  org.apache.catalina.startup.Catalina- Initialization processed in 1308 ms
2021-07-03 08:25:35,468 [main] INFO  org.apache.catalina.core.StandardService- Starting service Catalina
2021-07-03 08:25:35,469 [main] INFO  org.apache.catalina.core.StandardEngine- Starting Servlet Engine:
2021-07-03 08:25:36,051 [localhost-startStop-1] WARN  org.apache.catalina.startup.ContextConfig- catalina annotation was enabled
2021-07-03 08:25:55,509 [main] INFO  org.apache.coyote.http11.Http11NioProtocol- Starting ProtocolHandler ["http-nio-8022"]
2021-07-03 08:25:55,535 [main] INFO  org.apache.catalina.startup.Catalina- Server startup in 20094 ms
2021-07-03 08:58:07,932 [Thread-12] INFO  org.apache.coyote.http11.Http11NioProtocol- Pausing ProtocolHandler ["http-nio-8022"]
2021-07-03 08:58:07,954 [Thread-12] INFO  org.apache.catalina.core.StandardService- Stopping service Catalina
2021-07-03 08:58:27,131 [Thread-12] INFO  org.apache.coyote.http11.Http11NioProtocol- Stopping ProtocolHandler ["http-nio-8022"]
2021-07-03 08:58:27,159 [Thread-12] INFO  org.apache.coyote.http11.Http11NioProtocol- Destroying ProtocolHandler ["http-nio-8022"]
2021-07-03 08:58:35,284 [main] WARN  org.apache.catalina.security.SecurityListener- No umask setting was found in system property [org.apache.catalina.security.SecurityListener.UMASK]. However, it appears Tomcat is running on a platform that supports umask. The system property is typically set in CATALINA_HOME/bin/catalina.sh. The Lifecycle listener org.apache.catalina.security.SecurityListener (usually configured in CATALINA_BASE/conf/server.xml) expects a umask at least as restrictive as [0027]
2021-07-03 08:58:35,286 [main] INFO  org.apache.catalina.core.AprLifecycleListener- The APR based Apache Tomcat Native library which allows optimal performance in production environments was not found on the java.library.path: /usr/java/packages/lib/amd64:/usr/lib64:/lib64:/lib:/usr/lib
2021-07-03 08:58:35,609 [main] INFO  org.apache.coyote.http11.Http11NioProtocol- Initializing ProtocolHandler ["http-nio-8022"]
2021-07-03 08:58:35,638 [main] INFO  org.apache.tomcat.util.net.NioSelectorPool- Using a shared selector for servlet write/read
2021-07-03 08:58:35,641 [main] INFO  org.apache.catalina.startup.Catalina- Initialization processed in 1124 ms
2021-07-03 08:58:35,671 [main] INFO  org.apache.catalina.core.StandardService- Starting service Catalina
2021-07-03 08:58:35,672 [main] INFO  org.apache.catalina.core.StandardEngine- Starting Servlet Engine:
2021-07-03 08:58:36,261 [localhost-startStop-1] WARN  org.apache.catalina.startup.ContextConfig- catalina annotation was enabled
```

可以看到，之前的日志并没有丢失，那么假如我们把pvc和pv也删除掉，日志会不会丢失呢？这里卖个关子，有兴趣的同学可以自己测试一下。

细心的同学可能会发现，我们的nfs是单点，那么如果考虑HA的话，我们还需要搭建nfs集群，nfs集群也有多种实现方式，限于篇幅，我们后面再讨论这个问题。



### OK，在下一章节中，我们继续改造其余的微服务。

> 引用
>
> k8s部署eureka集群 https://www.jianshu.com/p/a3829851a97d
>
> k8s存储方式的介绍及应用 https://blog.csdn.net/xgp666/article/details/107242598/
