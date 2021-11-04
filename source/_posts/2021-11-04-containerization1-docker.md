---
title: (一)容器化实战--docker探索
comments: true
index_img: 'https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/docker_k8s.png'
banner_img: >-
  https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/background/vilige.jpg
tags: [容器化,Docker]
date: 2021-06-04 16:18:56
updated: 2021-11-04 16:18:56
categories: 容器化技术
---

## 前言

近日，某甲方爸爸心血来潮，要求项目使用某PAAS采用容器化方式部署，没办法只能开整。大概看了一眼，这个PAAS是基于Kubernates之上进行了一层封装，由于之前对容器化方面经验并不是很多。经过2个月左右的学习与探索，总算一个人把传统项目给进行了容器化改造。这里也准备用一个小系列将这个过程记录下来，方便将来复盘与研究。

**注：由于原文在项目组内部发表，本系列已经过脱敏，行文润色不周之处请谅解。**

#### 上下文

>容器化技术采用Docker+Kubernates
>
>项目抽象为一个典型的微服务项目
>
>前后端分离Nginx+Tomcat
>
>注册中心Eureka
>
>网关Zuul
>
>数据库 Mysql+Redis
>
>2B项目，并发量不大。
>
>待改造的节点大致如下
>
>| 微服务               | 数据库 | 前端      |
>| -------------------- | ------ | --------- |
>| Registry（eureka）   | Mysql  | Front-end |
>| Core(zuul)           | Redis  |           |
>| SM（系统管理微服务） |        |           |



## 1.Docker是什么？

Docker是一种容器化技术,也是目前业界运行时主流的选择。从它的两句口号便可理解它的思想。

>#### **Build,ship and run.**
>
>#### **Build once,run anywhere.**

它可以看做一个超轻量级的虚拟机。把系统资源与应用打包成了一个镜像，大大降低了部署的难度。解决了软件与IAAS层耦合严重的问题，docker可以让开发工程师可以更专注于软件开发，而无须关注硬件层面问题。它还具有分层构建镜像、文件挂载等优秀特性。

关于Docker的优秀特性这里就不再赘述，同学们可以自行研究。https://www.docker.com/

## 2.	安装Docker

### 2.1	环境准备

安装了Centos7的Linux

推荐安装DVD以上的版本，小版本容易缺少组件导致报错。

环境查看

```shell
#系统内核是3.10以上的
[root@localhost /]# uname -r
3.10.0-327.el7.x86_64
```



```shell
#系统版本
[root@localhost /]# cat /etc/os-release
NAME="CentOS Linux"
VERSION="7 (Core)"
ID="centos"
ID_LIKE="rhel fedora"
VERSION_ID="7"
PRETTY_NAME="CentOS Linux 7 (Core)"
ANSI_COLOR="0;31"
CPE_NAME="cpe:/o:centos:centos:7"
HOME_URL="https://www.centos.org/"
BUG_REPORT_URL="https://bugs.centos.org/"

CENTOS_MANTISBT_PROJECT="CentOS-7"
CENTOS_MANTISBT_PROJECT_VERSION="7"
REDHAT_SUPPORT_PRODUCT="centos"
REDHAT_SUPPORT_PRODUCT_VERSION="7"
```

### 2.2	安装

#### 2.2.1 联网安装docker

[帮助文档]: https://docs.docker.com/engine/install/

```shell
# 1、卸载docker
yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
#删除资源 
rm -rf /var/lib/docker
# /var/lib/docker 是docker的默认工作路径！

# 2、安装脚本工具
yum install -y yum-utils
# 3、设置镜像仓库
yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
# 上述配置默认是国外docker官方的，速度慢不推荐

#推荐使用国内的如阿里的
yum-config-manager \
    --add-repo \
    https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

#更新yum软件包索引
yum makecache fast

#查看docker版本清单
yum list docker-ce --showduplicates|sort -r

# 4、安装docker 有CE社区版和EE企业版之分，这里使用ce版即可
# 指定安装版本
yum install docker-ce-18.06.0.ce docker-ce-cli-18.06.0.ce containerd.io -y
# 安装最新版
yum install docker-ce docker-ce-cli containerd.io

# 5、启动docker&设置开机自启动
systemctl start docker & systemctl enable docker

# 6、使用docker version查看是否安装成功
docker version
[root@master01 /]# docker version
Client:
 Version:           18.06.0-ce
 API version:       1.38
 Go version:        go1.10.3
 Git commit:        0ffa825
 Built:             Wed Jul 18 19:08:18 2018
 OS/Arch:           linux/amd64
 Experimental:      false

Server:
 Engine:
  Version:          18.06.0-ce
  API version:      1.38 (minimum version 1.12)
  Go version:       go1.10.3
  Git commit:       0ffa825
  Built:            Wed Jul 18 19:10:42 2018
  OS/Arch:          linux/amd64
  Experimental:     false
  
# 7、配置docker 使用阿里云镜像加速
[root@master01 ~]# mkdir -p /etc/docker
[root@master01 ~]# tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://v16stybc.mirror.aliyuncs.com"]
}
EOF
# 	如果安装K8s 为了与高版本兼容顺便设置cgroup driver
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "registry-mirrors": ["https://v16stybc.mirror.aliyuncs.com"],
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

#	重载配置文件&重启服务
[root@master01 ~]# systemctl daemon-reload && systemctl restart docker
```

#### 2.2.2	离线安装Docker

下载离线安装包

https://download.docker.com/linux/static/stable/x86_64/

```shell
#1、解压上传的压缩包，解压后当前路径会生成一个docker目录，命令如下：

[root@118 install]# tar -zxvf docker-19.03.4.tgz

#2、将docker服务注册为service

[root@118 install]# cp docker/* /usr/bin/

[root@118 install]# vim /etc/systemd/system/docker.service

#将如下内容粘贴到docker.service

#ps：需要root权限，如果权限被限制，可以通过ln -s 软连接指到 /etc/systemd/system/docker.service一样可以

```

```shell
[Unit]

Description=Docker Application Container Engine

Documentation=https://docs.docker.com

After=network-online.target firewalld.service

Wants=network-online.target

[Service]

Type=notify

# the default is not to use systemd for cgroups because the delegate issues still

# exists and systemd currently does not support the cgroup feature set required

# for containers run by docker

ExecStart=/usr/bin/dockerd

ExecReload=/bin/kill -s HUP $MAINPID

# Having non-zero Limit*s causes performance problems due to accounting overhead

# in the kernel. We recommend using cgroups to do container-local accounting.

LimitNOFILE=infinity

LimitNPROC=infinity

LimitCORE=infinity

# Uncomment TasksMax if your systemd version supports it.

# Only systemd 226 and above support this version.

#TasksMax=infinity

TimeoutStartSec=0

# set delegate yes so that systemd does not reset the cgroups of docker containers

Delegate=yes

# kill only the docker process, not all processes in the cgroup

KillMode=process

# restart the docker process if it exits prematurely

Restart=on-failure

StartLimitBurst=3

StartLimitInterval=60s

 

[Install]

WantedBy=multi-user.target
```



```shell
#4、启动

[root@118 install]# chmod +x /etc/systemd/system/docker.service             #添加文件权限并启动docker

[root@118 install]# systemctl daemon-reload                                                       #重载unit配置文件

[root@118 install]# systemctl start docker                                                             #启动Docker

[root@118 install]# systemctl enable docker.service                                           #设置开机自启

#5、验证

[root@118 install]# docker version                                                         #查看Docker版本
```



### 2.3	运行helloworld镜像

```shell
# 测试helloworld
[root@localhost ~]# docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
b8dfde127a29: Already exists
Digest: sha256:5122f6204b6a3596e048758cabba3c46b1c937a46b5be6225b835d091b90e46c
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

### 2.4	卸载Docker

```shell
# 1、卸载docker
yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
#删除资源 
rm -rf /var/lib/docker
# /var/lib/docker 是docker的默认工作路径！
```

## 3.	使用Docker构建镜像

> 这里我们的公共镜像层设计如下
>
> 系统镜像：OS
>
> Tomcat镜像：OS+JDK1.8+Tomcat
>
> 数据库镜像：OS+Mysql、OS+Redis
>
> 前端镜像：OS+Nginx

![](https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/image-20211104174847655.png)

### 3.1	下载系统镜像层

### 3.2	构建基础镜像

这里以构建centos+jdk+tomcat为例，Docker registry 地址为192.168.60.128:5000

把压缩包上传至/home/docker 目录

解压缩并重命名文件夹

![](https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/965a6ca74ca2ce615c100e72517acd91_274x183.png@900-0-90-f.png)

```shell
#拉取centos7镜像
[root@master01 docker]# docker pull centos:7
[root@master01 docker]# docker build -t 192.168.60.128:5000/centos-jdk-tomcat:0.0.1 .
#参数-t name为设置标签名,格式为 仓库/标签名：版本
#最后需要加上"."表示使用当前目录下的Dockerfile
Sending build context to Docker daemon    355MB
Step 1/11 : FROM centos:7
 ---> 8652b9f0cb4c
Step 2/11 : MAINTAINER Vincent "vicentlee@email.com"
 ---> Running in bf344a418629
Removing intermediate container bf344a418629
 ---> 117e4ac1e435
Step 3/11 : RUN mkdir -p /opt/java/jre1.8.0_232
 ---> Running in 0b56b3cf1adc
Removing intermediate container 0b56b3cf1adc
 ---> c7730a840847
Step 4/11 : ADD jdk /opt/java/_jre1.8.0_232
 ---> f03745da59bd
Step 5/11 : RUN mkdir -p /opt/java/OpenAS_Tomcat_7
 ---> Running in 2924c5125a05
Removing intermediate container 2924c5125a05
 ---> ddac7fd13ad6
Step 6/11 : ADD tomcat /opt/java/OpenAS_Tomcat_7
 ---> 0f83265f4f3d
Step 7/11 : ENV JAVA_HOME /opt/java/_jre1.8.0_232
 ---> Running in c66572d4a875
Removing intermediate container c66572d4a875
 ---> 560f2e8a82cb
Step 8/11 : ENV CATALINA_HOME /opt/java/OpenAS_Tomcat_7
 ---> Running in c626d143b379
Removing intermediate container c626d143b379
 ---> 2253de1e357b
Step 9/11 : ENV PATH $PATH:$JAVA_HOME/bin:$CATALINA_HOME/bin
 ---> Running in 310b6924ab32
Removing intermediate container 310b6924ab32
 ---> 89dd4d21f44a
Step 10/11 : EXPOSE 8080
 ---> Running in c876584c135d
Removing intermediate container c876584c135d
 ---> 23fe401b8e9e
Step 11/11 : CMD ["/opt/java/OpenAS_Tomcat_7/bin/catalina.sh","run"]
 ---> Running in 3f7ecde682c5
Removing intermediate container 3f7ecde682c5
 ---> 6a569cbc874c
Successfully built 6a569cbc874c
Successfully tagged 192.168.60.128:5000/centos-jdk-tomcat:0.0.1
```



### 3.3	DockerFile构建镜像

微服务镜像

```shell
#基于Tomcat:alpine
FROM tomcat:alpine
#作者
MAINTAINER young<yyggmmshen@foxmail.com>
#声明变量
ENV TOMCATPATH /usr/local/tomcat
#设置工作目录
WORKDIR ${TOMCATPATH}/webapps
#删除ROOT目录里的所有文件
RUN rm -rf *
#导入war包
COPY hello.war ./hello.war
#创建ROOT目录
RUN mkdir ROOT
#将war包解压到ROOT目录下
RUN unzip ./hello.war -d ./ROOT
#删除war包
RUN rm -rf hello.war
#暴露接口
EXPOSE 8080
#启动Tomcat
CMD ["../bin/catalina.sh","run"]

```



### 3.4	使用DockerHub

假设注册DockerHub账号为tom

```shell
#注册或拉取私人镜像都需要登录
[root@node02 ~]# docker login
Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
Username: tom
Password:
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded

#镜像tag需要按照  	用户名/镜像名：版本号的格式命名
[root@node01 ~]# docker tag hello-world pperfectt/helloworld_m
#push 镜像
[root@node01 ~]# docker push tom/helloworld_m
The push refers to repository [docker.io/pperfectt/helloworld_m]
f22b99068db9: Mounted from library/hello-world
latest: digest: sha256:1b26826f602946860c279fce658f31050cff2c596583af237d971f4629b57792 size: 525
#在另一节点docker login之后拉取镜像
[root@node02 ~]# docker pull tom/helloworld_m:latest
latest: Pulling from pperfectt/helloworld_m
Digest: sha256:1b26826f602946860c279fce658f31050cff2c596583af237d971f4629b57792
Status: Downloaded newer image for pperfectt/helloworld_m:latest
```

## 4.	搭建私仓Docker Registry

实际情况中，大多数项目并不会将自己的镜像放在docker官方镜像仓，这种情况，我们需要搭建自己的镜像私仓。

#### 4.1	获取registry镜像

```shell
[root@master01 docker]# docker pull registry:2
```

#### 4.2	运行registry容器

```shell
[root@dockerA ~]# docker run  -itd --name registry --restart=always  -p 5000:5000 -v /opt/registry:/var/lib/registry registry:2

//创建一个registry容器来运行registry服务；
//-p：端口映射（前面是宿主机端口：后面是容器暴露的端口）；
//-v：挂载目录（前面是宿主机的目录：后面的是容器的目录）自动创建宿主机的目录；
//--restart=always：随docker服务的启动而启动！
```

#### 4.3	验证

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

#### 4.4	删除Docker Registry

```shell
[root@master01 docker]# docker container stop registry && docker container rm -v registry
registry
```



## 5.	Docker常用命令

### 帮助命令

```she
docker version	#显示docker的版本
docker info		#显示docker的系统信息，包括镜像和容器的数量
docker --help	#帮助命令
```

帮助文档的地址：https://docs.docker.com/

### 镜像命令

#### 查看所有本地的主机上的镜像

**docker images**

```she
[root@localhost ~]# docker images
REPOSITORY    TAG       IMAGE ID       CREATED        SIZE
hello-world   latest    d1165f221234   2 months ago   13.3kB
[root@localhost ~]#

#解释
REPOSITORY	镜像的仓库源
TAG			镜像的标签
IMAGE ID	镜像的id
CREATED 	镜像的创建时间
SIZE		镜像的大小

#可选项
Options:
  -a, --all             # 列出所有镜像
  -f, --filter filter   # 过滤镜像
  -q, --quiet          只显示镜像ID
  
#显示所有镜像的ID
[root@localhost ~]# docker images -aq
d1165f221234

```

#### 搜索镜像

DockerHub https://www.docker.com/products/docker-hub

**docker search** 

```shell
[root@localhost ~]# docker search mysql
NAME                              DESCRIPTION                                     STARS     OFFICIAL   AUTOMATED
mysql                             MySQL is a widely used, open-source relation…   10921     [OK]
mariadb                           MariaDB Server is a high performing open sou…   4125      [OK]
mysql/mysql-server                Optimized MySQL Server Docker images. Create…   810                  [OK]
percona                           Percona Server is a fork of the MySQL relati…   539  

# --filter=STARS=3000 #过滤，搜索出来的镜像收藏STARS数量大于3000的
Options:
  -f, --filter filter   Filter output based on conditions provided
      --format string   Pretty-print search using a Go template
      --limit int       Max number of search results (default 25)
      --no-trunc        Don't truncate output
      
[root@iz2zeak7sgj6i7hrb2g862z ~]# docker search mysql --filter=STARS=3000
NAME        DESCRIPTION         STARS            OFFICIAL        AUTOMATED
mysql       MySQL IS ...        9520             [OK]                
mariadb     MariaDB IS ...      3456             [OK] 
```

#### 下载镜像

**docker pull** 

```shell
# 下载镜像 docker pull 镜像名[:tag]
➜ ~ docker pull tomcat:8
8: Pulling from library/tomcat #如果不写tag，默认就是latest
90fe46dd8199: Already exists #分层下载： docker image 的核心 联合文件系统
35a4f1977689: Already exists
bbc37f14aded: Already exists
74e27dc593d4: Already exists
93a01fbfad7f: Already exists
1478df405869: Pull complete
64f0dd11682b: Pull complete
68ff4e050d11: Pull complete
f576086003cf: Pull complete
3b72593ce10e: Pull complete
Digest: sha256:0c6234e7ec9d10ab32c06423ab829b32e3183ba5bf2620ee66de866df640a027
# 签名 防伪
Status: Downloaded newer image for tomcat:8
docker.io/library/tomcat:8 #真实地址
#等价于
docker pull tomcat:8
docker pull docker.io/library/tomcat:8

```

#### 构建镜像

```shell
docker build -t test .
```

#### 运行镜像

```shell
docker run -dit -p 80:8022 --rm test
# -dit 后台运行容器 -it 容器启动后进入容器
# --rm 容器停止后删除容器
# -p 宿主端口号：容器端口号 映射宿主机与容器端口
```

#### 删除镜像

**docker rmi** 

```shell
[root@localhost ~]# docker rmi -f 镜像id # 删除指定镜像
[root@localhost ~]# docker rmi -f 镜像id 镜像id 镜像id # 删除指定镜像
[root@localhost ~]# docker rmi -f $(docker images -aq)镜像id # 删除符合条件镜像（全部）
```

#### 导出镜像

```shell
[root@storage01 tmp]# docker save -o centos.tar centos:7
```

#### 导入镜像

```shell
[root@storage01 tmp]# docker load -i centos.tar
```

### 容器命令

```shell
docker run 		镜像id 新建容器并启动
docker ps 		列出所有运行的容器 docker container list
docker rm 		容器id 删除指定容器
docker start 	容器id #启动容器
docker restart	容器id #重启容器
docker stop 	容器id #停止当前正在运行的容器
docker kill 	容器id #强制停止当前容器
```

> 说明：我们有了镜像才可以创建容器，Linux，下载centos镜像来学习

#### 容器生命周期管理命令

##### 创建容器

```shell
docker create
```

##### 启动容器

```shell
docker start
```

##### 运行容器

```shell
docker run [可选参数] image | docker container run [可选参数] image 
#参数说明
--name="Name"		#容器名字 tomcat01 tomcat02 用来区分容器
-d					#后台方式运行
-it 				#使用交互方式运行，进入容器查看内容
-p					#指定容器的端口 -p 8080(宿主机):8080(容器)
		-p ip:主机端口:容器端口
		-p 主机端口:容器端口(常用)
		-p 容器端口
		容器端口
-P(大写) 				随机指定端口
#创建一个nginx容器
[root@localhost ~]# docker run -d -p 80:80 --name nginx1 nginx
60d8cc8576ab56239eab28d106ff09758430fa4aeab2ad240dc23472ade9ebe3
[root@localhost ~]# docker ps
CONTAINER ID   IMAGE     COMMAND                  CREATED          STATUS          PORTS                               NAMES
60d8cc8576ab   nginx     "/docker-entrypoint.…"   16 seconds ago   Up 14 seconds   0.0.0.0:80->80/tcp, :::80->80/tcp   nginx1

#验证
[root@localhost ~]# curl localhost
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

##### 停止一个运行的容器

```shell
docker stop
```

##### 删除一个处于终止状态的容器

```shell
docker rm
```

##### 杀死容器进程

```shell
docker kill
```

#### 查询容器

```shell
docker ps [可选参数]

Options:
  -a, --all             列出所有容器
  -f, --filter filter   筛选过滤
  -n, --last int        列出最后创建的n个容器，包括所有状态的
  -l, --latest          列出最后创建的容器
  -q, --quiet           仅显示容器id
  -s, --size            显示文件大小
  
[root@localhost ~]# docker ps -a
CONTAINER ID   IMAGE     COMMAND                  CREATED         STATUS         PORTS                               NAMES
60d8cc8576ab   nginx     "/docker-entrypoint.…"   8 minutes ago   Up 8 minutes   0.0.0.0:80->80/tcp, :::80->80/tcp   nginx1
```

#### 操作容器

#### 进入容器

```shell
#进入容器方法1 attach命令，直接进入已启动的容器终端，不会启动新的进程。
docker attach [OPTIONS]CONTAINER

#进入容器方法2 exec命令，在容器中打开新的终端
docker exec [OPTIONS]CONTAINER COMMAND[ARG...]

#如果报错无法进入，一般的容器都可以执行/bin/bash，但也有部分容器没有，那么我们可以用sh来代替
[root@master01 docker]# docker exec -it 933cb70e96c6 /bin/bash
OCI runtime exec failed: exec failed: container_linux.go:348: starting container process caused "exec: \"/bin/bash\": stat /bin/bash: no such file or directory": unknown

[root@master01 docker]# docker exec -it 933cb70e96c6 sh
/ # ls
```

#### 从容器内拷贝文件到宿主机上

```shell
docker cp <containerId>:/file/path/within/container /host/path/target
```

#### 删除容器

```shell
docker rm -f [容器ID] 
docker rm -f b2c70821b7aa 
#-f 强制删除
```

```shell
# 清楚镜像缓存
docker system prune
```
