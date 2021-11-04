---
title: (六)容器化实战--前端Nginx改造
comments: true
index_img: 'https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/docker_k8s.png'
banner_img: >-
  https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/background/vilige.jpg
tags: [容器化,Docker,Nginx]
date: 2021-06-04 16:22:56
updated: 2021-11-04 16:22:56
categories: 容器化技术
---

### 本章，我们来进行前端服务的容器化改造

首先来看一下我们原来的组网拓扑：

---

![](https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/image-20211104171031523.png)



---

可以发现，我们的Nginx目前承担的职责主要有：

- 动静分离，响应静态资源请求，动态资源请求反向代理至CORE进一步路由。（何为动静分离请浏览文末）

- https拦截层
- 后端网关服务的负载均衡器
- 网关熔断器

我们这里镜像采用centos+Nginx基础层、frontend应用层的分层设计，目的是一方面是可以解耦、一方面也可以大大提升镜像的构建速度、应用的发布速度。

![](https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/32fcbc3ed3d559bdf392e2a2152ce451_392x245.png@900-0-90-f.png)

**由于原Nginx在部署时配置了大量的配置，在改造过程中，我们需要把我们原来的运行时环境变量，完整、准确地通过shell脚本写入到我们的镜像当中。这个过程需要十分耐心，任何一个步骤的遗漏或者出错，都会导致镜像运行失败。**

但即便如此，我们还是遇到了不少问题



### 遇到的问题

1. 在之前的基础镜像中没有创建安全用户`maint`与引入安全秘钥文件，而我们Nginx启用了Https，启动需要对应的安全证书，无法找到安装证书导致Nginx启动失败

2. Nginx启动脚本中有sudo命令，执行sudo命令默认需要输入密码，会导致启动脚本执行失败。

3. 安全送检反馈要求我们的Nginx宿主机只暴露必要Nginx端口，例如80与443，未用到端口不开放，防止绕过Nginx直接访问后端以及其他的。

   目前采用iptables配置来解决此问题，需要适配镜像OS层环境。

4. 原Nginx服务启动的方式为执行`operat_nginx.sh`，这个线程是后台运行的，下一步我们将开发前端镜像的Kubernates资源文件，而POD需要线程以前台运行的方式才能对容器的健康状态进行监控。

5. Nginx的负载均衡设置中，原先的后端IP是写死的，现在我们的后端容器的IP是不固定的，如何路由。

那么让我们来逐个攻破把，已经走到这里了，没有什么可以难倒我们的，不是吗。



### 问题1

#### 分析

构建镜像当中，nginx的配置文件nginx.conf采用了先拷贝模板文件，再使用sed命令对变量进行修改的方式。

仔细观察后发现，在https的相关配置当中，有一个名为security的目录，在这个目录下，存放着安全相关的正式与key,而在data01目录下，我们并没有这个文件夹与证书文件，所以会导致nginx启动失败。

```shell
 ....
 # https server  443
    server {
        access_log  /var/log/nginx/log/host.access.log  main;
        listen 127.0.0.0:443 ssl backlog=10000;
        server_name localhost;

        # https server
        ssl_certificate_key_password    "0000000100000001CF63A8E8E06D0FE814A7F33FD704F8478B5495FFDE40F4CD408842732CADCC9164D65A77CE2B967A4EE3D01404A8951D" /data01/security/root.key /data01/security/common_shared.key;
        ssl_certificate           /etc/nginx/conf/nginx.crt;
        ssl_certificate_key       /etc/nginx/conf/nginx.key;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_session_cache shared:SSL:10m;
...
```

我们需要准备这些文件。

---

![](https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/5447b32117321407a7b64d18ce62a028_402x388.png@900-0-90-f.png)

经过查找，我们发现原部署脚本目录当有现有一个名为`install_projectName_maint.sh`的脚本，它创建了maint这个用户，并且将原码仓当中的一个security的目录拷贝到了虚拟机上，这个步骤应该就是我们想要的了。

```shell
#!/bin/bash
######################部署脚本用到的配置#########################################
SCRIPT_NAME="install_shuxiaoer_maint.sh"
CUR_DIR=$(pwd)
FILE_LOG="${CUR_DIR}/install_`date +%Y-%m-%d`.log" # 日志文件
CONF_PATH=${CUR_DIR}/../config

cdir=$(cd `dirname $0`; pwd)
. ${cdir}/comm_lib

install_shuxiaoer_maint()
{
    LOG "INFO" "begin install maint service ..."

    maint_path=${CUR_DIR}/../security
    if [[ ! -d ${maint_path} ]];then
        LOG "ERROR" "[ the ${maint_path} is not exit,pls check~]"
        return 1
    fi

    sudo cp -r ${maint_path}/*  ${MAINT_USER_HOME}
    if [[ $? -ne 0 ]];then
        LOG "ERROR" "[ copy ${maint_path} to ${MAINT_USER_HOME} ] failed"
        return 1
    fi

    sudo chmod -R 750 ${MAINT_USER_HOME}
    sudo chown -R ${MAINT_USER}:${MAINT_USER_GROUP_NAME} ${MAINT_USER_HOME}
}

main()
{
    echo "======================================================================="
    LOG "INFO" "------ begin install shuxiaoer maint root -----"

	#判断用户是否存在
    id ${MAINT_USER} > /dev/null 2>&1
    if [[ $? -ne 0 ]];then
    	#用户不存在，需要创建用户
      create_user "maint" ${MAINT_USER} ${MAINT_USER_GROUP_NAME} ${MAINT_USER_HOME} || return 1
    fi


    install_shuxiaoer_maint  || return 1

    LOG "INFO" "------ end install shuxiaoer maint root -----"
    echo "======================================================================="
}
main "$@"
```

#### 开发

由于其他微服务可能也使用到了这个安全框架相关的内容，我们将这部分代码写入我们的centos基础镜像的构建脚本当中，并重新构建所有应用的基础OS层。

![](https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/9c0c0e4f6c1d1b886842e91b50463867_1187x570.png@900-0-90-f.png)

新的centos基础镜像我们命名为`centos_base:1.0.0`,用来替代我们原来的基础镜像层`centos:7`记得修改其他的微服务Dockfile中的FROM 值再进行构建。

#### 验证

构建之后，进入centos_base镜像内部，发现我们需要的目录、用户、安全文件都已经准备好了。

```shell
[root@723d1d9b48db /]# cd data01/
[root@723d1d9b48db data01]# ll
total 0
drwxr-x---. 6 maint    shuxiaoer 148 Aug 14 02:15 security
drwxr-x---. 1 frontend shuxiaoer  90 Aug 14 06:52 shuxiaoer-frontend
[root@723d1d9b48db data01]# cd security/
[root@723d1d9b48db security]# ll
total 8
-rwxr-x---. 1 maint shuxiaoer 1024 Aug 14 02:15 common_shared.key
drwxr-x---. 3 maint shuxiaoer   20 Aug 14 02:15 etc
drwxr-x---. 2 maint shuxiaoer   36 Aug 14 02:15 logs
-rwxr-x---. 1 maint shuxiaoer 1024 Aug 14 02:15 root.key
drwxr-x---. 3 maint shuxiaoer   54 Aug 14 02:15 scc
drwxr-x---. 6 maint shuxiaoer   54 Aug 14 02:15 sec
```

### 问题2

在linux的系统当中，非root用户执行sudo命令，需要输入密码，这个过程会导致我们的自动脚本执行失败，我们需要跳过此步骤。

sudo的配置在`/etc/sudoers`这个文件中

其中

如将　%admin ALL=(ALL) ALL　修改为　%admin ALL=(ALL) NOPASSWD: ALL

意思是属于admin组的用户可以不需要输入密码执行sudo

如果去除%，则仅仅表示admin这个用户不需要输入密码

 为某个用户设置可设置为

username ALL=(ALL) NOPASSWD: ALL

我们为projectName组设置免密。

在nginx基础镜像构建脚本中添加

![](https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/308c05f15d78ac60f3c1a5e40405f7b1_1329x630.png@900-0-90-f.png)

 ```shell
sudo -i sed -i '/\#includedir \/etc\/sudoers.d/a\%shuxiaoer ALL=(ALL) NOPASSWD: ALL' /etc/sudoers
 ```

### 问题3

在Nginx镜像改造过程中，发现nginx是通过脚本`operat_nginx.sh`启动的，而并非我们常用的nginx启动命令。

这一段脚本的开发人员已经找不到了，没有注释，仔细研究后发现它将宿主机的端口限制在了80与443，应该是出于安全方面的考虑，我们将注释补上。

```shell
#!/bin/sh
source /etc/profile
if [ -e ~/.bashrc ];then
  source ~/.bashrc
fi

CUR_PATH=$(cd `dirname $0`; pwd)

function open_iptables_port()
{
    sudo iptables -t filter -L INPUT -n | grep "ACCEPT.*tcp dpt:"$1 &>/dev/null
    if [ $? -ne 0 ]; then
        sudo iptables -A INPUT -t filter -p tcp -m state --state ESTABLISHED --dport $1 -j ACCEPT &>/dev/null
        if [ $? -ne 0 ]; then
            return 1
        fi
    fi
    return 0
}
#安全要求规定需仅放行必要tcp到80与443端口的请求
open_iptables_port 80
open_iptables_port 443

#授权nginx文件可以以非root用户开启1024以下端口
sudo setcap cap_net_bind_service=+eip $CUR_PATH/nginx

if [[ $# -gt 0 ]]; then
	#启动nginx，指定配置文件
    $CUR_PATH/nginx $* -c $CUR_PATH/../conf/nginx.conf -p $CUR_PATH/..
    if [[ $? -ne 0 ]]; then
        echo "failed to operate nginx"
        exit 1
    fi
else
    $CUR_PATH/nginx -c $CUR_PATH/../conf/nginx.conf -p $CUR_PATH/..
    if [[ $? -ne 0 ]]; then
        echo "failed to start nginx"
        exit 1
    fi
fi

if [[ -f $CUR_PATH/../nginx.pid ]]; then
    chmod 600 $CUR_PATH/../nginx.pid > /dev/null 2>&1
fi

```

然后发现这段脚本执行还有些问题：

**在配置了sudo免密执行之后，我们的nginx启动脚本仍然报错**

```shell
ERROR: problem running iptables: iptables v1.6.1: can't initialize iptables table `filter': Permission denied (you must be root)
Perhaps iptables or your kernel needs to be upgraded.
```

网上搜索了一下之后发现，需要在docker启动命令上加上一行参数`--privileged`。

如

```shell
docker run -it --privileged 32jo13jl1 bash
```

https://blog.csdn.net/Magic_Ninja/article/details/88432140

添加后启动正常，在我们的K8s资源文件中也需要添加相应的设置

### 问题4

在nginx启动命令中加入参数`-g "daemon off;"`来使我们的nginx线程处于前台运行。

在我们的镜像构建脚本中，修改nginx启动脚本`operat_nginx.sh`中的nginx启动命令

`install_nginx.sh`中添加

```shell
···
install_nginx()
{
	...
    #设置nginx启动脚本前台启动
    sed -i "s#-p \$CUR_PATH\/\.\.#-p \$CUR_PATH\/\.\. -g \"daemon off\;\"#g" ${FRONTEND_USER_HOME}/nginx-run/sbin/operat_nginx.sh
···
```

修改后`operat_nginx.sh`

```she
···
if [[ $# -gt 0 ]]; then
    $CUR_PATH/nginx $* -c $CUR_PATH/../conf/nginx.conf -p $CUR_PATH/.. -g "daemon off;"
    if [[ $? -ne 0 ]]; then
        echo "failed to operate nginx"
        exit 1
    fi
else
    $CUR_PATH/nginx -c $CUR_PATH/../conf/nginx.conf -p $CUR_PATH/.. -g "daemon off;"
    if [[ $? -ne 0 ]]; then
        echo "failed to start nginx"
        exit 1
    fi
fi
···
```



### 问题5

处理完了以上问题之后，试着运行一下frontend镜像，将宿主机的443端口映射到容器的443端口

```shell
[root@storage01 ~]# docker run -it -p 443:443 --privileged--rm fd2c8e2ae783 bash
```

在浏览器输入https://+ 宿主机ip

https://192.168.4.118

OK，我们本节的内容就结束了。下一节中我们将来解决容器化的另外一个重要问题---日志采集。

