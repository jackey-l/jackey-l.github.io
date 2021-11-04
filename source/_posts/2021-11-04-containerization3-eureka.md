---
title: (三)容器化实战--eureka改造
comments: true
index_img: 'https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/docker_k8s.png'
banner_img: >-
  https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/background/vilige.jpg
tags: [容器化,Docker,eureka]
date: 2021-06-04 16:19:56
updated: 2021-11-04 16:19:56
categories: 容器化技术
---

上面两节我们对docker与k8s已经有了一定的了解，那么我们基于知识上下文，来对微服务进行改造。

本小节我们以registry服务镜像改造作为起点，开始我们的容器化改造工作。路漫漫而修远兮，吾独上下而求索，与君共勉。

首先，为什么选用registry呢，因为它是一个eruka注册中心，结构相对简单，代码量少，不涉及数据库，不依赖其他服务，可以独立运行。

在改造之前，我们需要先考虑一些问题：

- dockerhub是公网仓库，往外网传输源码涉及信息安全，需要搭建docker私仓来存放我们的镜像。
- registry目前的测试运行环境与容器环境有差异需要适配。
- registry部署态中部署脚本进行了大量的配置性设置，需要配置到镜像当中。
- registry中的服务注册发现目前是基于服务器IP去路由寻址的，在k8s集群中容器的IP是随着POD的生命周期而改变，如何保证当在任何一套物理环境部署、及部署之后registry本地IP随时可能发生改变的情况下，保证registry的功能不受影响。
- k8s中的pod是无状态的，随着pod的销毁、弹性伸缩，registry的日志也会丢失，在多个registry实例同时产生和销毁的场景下，如何分别对它们 的日志进行保存，并保证彼此之间没有冲突。

---



那么我们一步步来

首先我的思路是自己在linux上安装一遍registry服务，看下过程中有哪些步骤是与正常的部署有区别的，把这些步骤写入到Dockefile中就能完成了。没想到一上来就掉进了坑里。


我们的安装包使用的是公司定制的安全加固版的tomcat和jdk，猜想一定和平时用的开源版有许多区别，所以一定要小心谨慎，否则出了问题再没有文档的情况下很难解决。

![](https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/84912118cdf0bb1321e85eff1ac6c5ed_351x563.png@900-0-90-f.png)


在测试机器上安装了jdk和把OpenAS解压到home目录下tomcat文件夹下后用tomcat试着启动了一下，没有启动起来。提示报错是当前使用root用户登陆，猜测安全加固版不允许root账户执行。改为使用其他用户执行后此报错消失。接着一直报错umask当前值为0022，安全要求最低的umask值为0027。

>简单介绍下umask，就是unix系统中默认创建文件的最小值。linux中的权限可以用9个位数来表示 000 000 000 前三位是owner的读、写、执行权限，后6位分别是组、其他用户的对应权限，读写执行可以分别用4、2、1来表示，如果为1个文件设置所有用户都有读写执行权限，可以用chmod 777 文件名 来设置，7=4+2+1（读加写加执行）。如果当前用户的umask值是027，那么会默认减去对应的权限值，也就是上一条命令实际设置的文件权限是750，每个用户又可以设置自己的umask值。

一番操作后设置了各种umask值还是不行，原来home目录下的tomcat文件夹与tomcat用户的默认文件夹同名了，linux中创建用户会默认在home文件夹下创建同名目录，其中./bashrc是用来给这个用户配置环境变量和其他配置，以此来达到用户隔离的作用。所以在解压tomcat的时候覆盖了tomcat用户的./bashrc文件，导致我们设置umask值没有生效。

真是让人哭笑不得，在其他文件夹下解压、用非root用户启动终于启动正常。

这里也让我发现，自己去摸索这个不熟悉的安全框架可能还要花费大量时间。

所以改变思路，我决定直接把所有安装脚本的内容研究清楚，再把这些逻辑写到镜像的制作文件Dockerfile中去，然后再写一个脚本执行docker build 就可以了。但是这样Dockerfile会变得又臭又长，不能忍受，必须解耦，复用。所以最终的方案是：

把构建需要的文件和脚本放到各自的目录下，再使用docker 的ADD 和COPY命令，把这些文件和脚本一股脑丢到容器中，让脚本把jdk、tomcat安装好、环境配置好，再把自己删了....后面环境有变，我们只需要更改同目录下的脚本即可，基本无须改动Dockerfile，完美。

这里省略了基础镜像centos7_jdk8的制作过程，只放一下dockerfile，制作基础镜像的目的是使用docker的优秀特性分层构建。这里就不展开了。

**基础镜像Dockerfile**

```dockerfile
#使用的基础镜像 暂时使用centos7开源，后期可更改，注意如无法访问公网，本地需要提前准备
FROM centos:7
#作者信息  姓名 邮箱
#MAINTAINER name "xxxx@xxx.com"

#安装命令工具（可选）
RUN yum -y install unzip && yum -y install sudo

#声明变量
ENV JAVA_HOME=/opt/JRE
ENV INSTALL_HOME=/opt/docker_install
ENV SCRIPT_NAME=install_jdk.sh

#创建安装目录
RUN mkdir -p /data01 && mkdir -p ${INSTALL_HOME} && mkdir -p ${JAVA_HOME}

#复制安装脚本
COPY ${SCRIPT_NAME} ${INSTALL_HOME}
COPY java.policy ${INSTALL_HOME}

###############构建JDK层######################
#把当前目录下的jdk文件夹添加到镜像
ADD jre-8u232-linux-x64.tar.gz ${JAVA_HOME}

#运行安装脚本
RUN sh ${INSTALL_HOME}/${SCRIPT_NAME}
#删除安装目录
RUN rm -rf ${INSTALL_HOME}

#Docker build命令参考：docker build -t centos7_hwjdk8 .
```



**registry的Dockerfile**

```dockerfile
######################################################
#
FROM centos7_hwjdk8
#作者信息  姓名 邮箱
#MAINTAINER name "xxxx@xxx.com"
#/data01/registry/tomcat/webapps/registry/
#声明变量
ENV INSTALL_HOME=/opt/docker_install
ENV SCRIPT_NAME=install_projectName_registry.sh
ENV REGISTRY_HOME=/data01/registry
ENV TOMCATPATH=${REGISTRY_HOME}/tomcat

############构建tomcat&应用层######################
RUN mkdir -p ${INSTALL_HOME}/config
#复制文件
COPY OpenAS_Tomcat-7.zip ${INSTALL_HOME}
COPY registry.war ${INSTALL_HOME}
COPY ${SCRIPT_NAME} ${INSTALL_HOME}
COPY config ${INSTALL_HOME}/config
#运行安装脚本
RUN sh ${INSTALL_HOME}/${SCRIPT_NAME}

#删除安装目录
RUN rm -rf ${INSTALL_HOME}

#启动tomcat
EXPOSE 8022
USER registry
#启动时运行tomcat
CMD ["/data01/registry/tomcat/bin/catalina.sh","run"]
#docker build -t registry .
```

在写脚本之前，我们需要对目前的部署脚本结构、实现有一个整体的了解。

首先我们来了解一下我们的部署脚本结构

![](https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/100126e1efcaaa305d50bb43523d8259_308x818.png@900-0-90-f.png)



核心是`install.sh`这个脚本，在main函数里可以发现，它执行了其它同目录下的其它sh,这里我们以registry为例，看下这个服务到底是如何部署的。

打开`install.sh`

我们只看核心部分,其它几个脚本的作用这里只简单注释下

```shell
...
main() {
  echo ""
  LOG "INFO" "=============================projectName install Begin==================================="
  LOG "INFO" "projectName all install exec date: date "+%Y-%m-%d %H:%M:%S""
  #检查安装包完整性
  prepare_install || return 1
  #检查ssh密码（后面用于连接其它节点进行部署）
  ssh_with_password || return 1
  #修改application.yml
  modify_application_yml || return 1
  #复制安装包到其他节点
  scp_package_vm || return 1
  #创建安装目录
  install_maint || return 1
  #安装jdk
  install_jdk || return 1
  #数据库相关操作
  refresh_sql || return 1
  #核心实现
  install_registry || return 1
  ...
...  
install_registry() {
  ## registry install
  for ((i = 0; i < ${#registry_install_ip[@]}; i++)); do
    vm1=${registry_install_ip[$i]}
    vm=$(eval echo '$'"${vm1}")
    LOG "INFO" "================= install projectName registry in ${vm} ================="
    #执行了install_projectName_registry.sh
    ssh ${setup_user}@${vm} "cd ${CUR_DIR}; sh install_projectName_registry.sh ${vm}"
  done
}
```

打开部署脚本`install_projectName_registry.sh`

查看主函数,这里我们有些配置我们要去掉，有些要修改，需要根据我们对未来运行环境的情况来自己配置生成自己的打包脚本。这里仅供参考

```shell
main()
{
    echo "======================================================================="
    LOG "INFO" "------ begin install projectName registry root -----"
	#创建用户、组等，后面的registry应用的操作都基于这个权限，保留
    create_user "registry" ${REGISTRY_USER} ${REGISTRY_USER_GROUP_NAME} ${REGISTRY_USER_HOME} || return 1
    #设置环境变量 保留
    set_env "registry" ${REGISTRY_USER_HOME} ${REGISTRY_USER} || return 1
	#安装JDK 这里重复安装了2个JDK 我们后面使用一个就可以了
    install_jdk "registry" ${REGISTRY_USER} ${REGISTRY_USER_GROUP_NAME} ${REGISTRY_USER_HOME} || return 1
	#安装tomcat
    install_openas "registry" ${REGISTRY_USER} ${REGISTRY_USER_GROUP_NAME} ${REGISTRY_USER_HOME} || return 1
	#设置配置 
    install_projectName_registry_service  || return 1
    #设置tomcat配置
    modify_tomcat_config "registry" ${REGISTRY_USER} ${REGISTRY_USER_GROUP_NAME} ${REGISTRY_USER_HOME} ${local_ip} || return 1
	#设置tomcat端口
    modify_server_xml_port || return 1
	#tomcat https相关设置 我们的容器并没有对外暴露，去掉
    modify_https_license || return 1
    #配置服务设置
    modify_projectName_registry_service || return 1
    #启动服务 去掉，我们使用的是Dockerfild的CMD命令启动，服务必须运行于前台，因为后面要使用k8s等平台来编排。
    start_registry || return 1
    #增加定时启动任务，防止服务器重启后服务down掉，提高程序健壮性，去掉，我们使用k8s的自愈功能。
    add_crontab || return 1

    LOG "INFO" "------ end install projectName registry root -----"
    echo "======================================================================="
}
```

【注意】如果写完脚本还是运行不了，可以检查下看是不是省略了某些步骤。

> 此处省略1000字写脚本调试过程....

最终得到的脚本：

```shell
#!/bin/bash
######################部署脚本用到的配置#########################################
SCRIPT_NAME="install_projectName_registry.sh"
CUR_DIR=/opt/docker_install
SOFT_PATH=${CUR_DIR}/../../../SoftWarePackage
CONF_PATH=${CUR_DIR}/config
JRE_PATH=/opt/JRE
common_key=2db799094c8101ded579be4435d5a75d7bb36a57ec8822f35437585551847c788919a5b5e76d92037a6483a5a0be061c
tomcat_name="OpenAS_Tomcat-7.zip"

REGISTRY_USER_HOME=/data01/registry
REGISTRY_USER=registry
REGISTRY_USER_GROUP_NAME=projectName
projectName_REGISTRY_SERVICE=registry
projectName_war_path=${CUR_DIR}/registry.war
projectName_registry_http_port=8022
port1=8023

install_projectName_registry_service()
{
    echo "INFO" "begin install registry service ..."

    if [[ ! -f ${projectName_war_path} ]];then
        echo "ERROR" "[ file ${projectName_war_path} ] is not exist,pls check~"
        return 1
    fi

    registry_path=${REGISTRY_USER_HOME}/tomcat/webapps/${projectName_REGISTRY_SERVICE}
    if [[ ! -d ${registry_path} ]];then
        sudo su - ${REGISTRY_USER} -c "mkdir -p ${registry_path}"
        if [[ $? -ne 0 ]];then
            echo "ERROR" "[ mkdir -p ${registry_path} ] failed"
            return 1
        fi
    fi

    sudo unzip -q ${projectName_war_path} -d ${registry_path}
    if [[ $? -ne 0 ]];then
        echo "ERROR" "[ unzip ${projectName_war_path} ] failed"
        return 1
    fi
  sudo chmod -R 750 ${registry_path}/META-INF
    echo "INFO" "end install registry service ..."
}

function modify_server_xml_port
{
    echo "INFO" "begin modify server.xml port openas ..."

    sudo su - ${REGISTRY_USER} <<EOF
    server_path=${REGISTRY_USER_HOME}/tomcat/conf/server.xml
    if [[ ! -f ${server_path} ]];then
        echo "[ file ${server_path} ] is not exist,pls check~"
        return 1
    fi
    #修改端口号


    sed -i -e s/8384/${port1}/g ${server_path}
    # https 端口号
    #sed -i -e s/8542/${projectName_registry_port}/g ${server_path}
    # http 端口号

    sed -i -e s/19998/${projectName_registry_http_port}/g ${server_path}

    #sed -i "s#path=\".*\"\\(.*docBase.*\\)#path=\"\/register\"\\1#g" ${server_path}

    sed -i "s#docBase=\".*\"\\(.*reloadable.*\\)#docBase=\"${projectName_REGISTRY_SERVICE}\"\\1#g" ${server_path}
    exit
EOF

    echo "INFO" "successful modify server.xml port openas ..."
}

modify_projectName_registry_service()
{
	echo "INFO" "begin modify registry service ..."
    application_common=${CONF_PATH}/application-common.yml
    application_broker=${CONF_PATH}/application-broker.yml
    application_path=${REGISTRY_USER_HOME}/tomcat/webapps/${projectName_REGISTRY_SERVICE}/WEB-INF/classes/
    sudo cp ${application_common} ${application_path}
    sudo cp ${application_broker} ${application_path}

	  file_application_yml=${REGISTRY_USER_HOME}/tomcat/webapps/${projectName_REGISTRY_SERVICE}/WEB-INF/classes/application.yml
    sudo su - ${REGISTRY_USER} <<EOF

    if [[ ! -f ${file_application_yml} ]];then
        echo "[ file ${file_application_yml} ] is not exist,pls check~"
        return 1
    fi
    ## set spring application name
    sed -i "s/    name: .*/    name: ${projectName_REGISTRY_SERVICE}/" ${file_application_yml}

    sed -i "s/  port: .*/  port: ${projectName_registry_http_port}/" ${file_application_yml}

EOF
  sudo chown -R ${REGISTRY_USER}:${REGISTRY_USER_GROUP_NAME} ${REGISTRY_USER_HOME}
	echo "INFO" "successful modify registry service ..."

}


create_user()
{
    COMPONENT_NAME=$1
    USER_NAME=$2
    GROUP_NAME=$3
    USER_HOME=$4

    if [[ "X" == "X${COMPONENT_NAME}" ]] || [[ "X" == "X${USER_NAME}" ]] || [[ "X" == "X${GROUP_NAME}" ]] || [[ "X" == "X${USER_HOME}" ]];then
        echo "ERROR" "${COMPONENT_NAME} or ${USER_NAME} or ${GROUP_NAME} or ${USER_HOME}is null,pls check !"
        return 1
    fi

	echo "INFO" "begin create projectName ${COMPONENT_NAME} user ..."

	#判断用户是否存在
    flag_create="No"
    id ${USER_NAME} > /dev/null 2>&1
    if [[ $? -ne 0 ]];then
    	#用户不存在，需要创建用户
        flag_create="Yes"
    fi

	#用户存在,返回
    if [[ "xYes" != "x${flag_create}" ]];then
        is_shell_matched=$(cat /etc/passwd | grep ${USER_NAME} | awk -F ':' '{print $7}'| grep -E 'ksh|bash|csh')
        if [[ "x" == "x${is_shell_matched}" ]];then
            echo "ERROR" "${USER_NAME} is existed,pls check or change another user name~"
            return 1
        fi

        echo "ERROR" "${USER_NAME} is existed"
        return 1
    fi

	#判断用户组是否存在，存在直接使用，不存在则重新创建
    ret=$(cat /etc/group | grep -w "^${GROUP_NAME}")
    if [[ "x" != "x${ret}" ]];then
        echo "WARN" "[ GROUP_NAME:${GROUP_NAME} ] is already exist, use it directly!"
    else
        echo "INFO" "GROUP_NAME:${GROUP_NAME}"

        sudo groupadd ${GROUP_NAME}
        if [[ $? -ne 0 ]];then
            echo "ERROR" "[ groupadd ${GROUP_NAME} ] failed"
            return 1
        fi
    fi

    user_path="/data01"
    sudo chmod 755 ${user_path}

    linux_bash="/bin/bash"
    sudo useradd -m -d ${USER_HOME} ${USER_NAME} -s ${linux_bash} -g ${GROUP_NAME} >/dev/null 2>/dev/null
    if [[ $? -ne 0 ]];then
        echo "ERROR" "[ useradd -m -d ${USER_HOME} ${USER_NAME} -s ${linux_bash} -g ${GROUP_NAME} ] failed"
        return 1
    fi
    sudo chmod -R 750 ${USER_HOME}
    encrypt_user_password ${USER_NAME} || return 1

    echo "INFO" "successful create projectName ${COMPONENT_NAME} user ..."

    return 0
}

function encrypt_user_password
{
    PASSWORD_USER_NAME=$1
    passwd_user=${PASSWORD_USER_NAME}
    if [[ "X" == "X${PASSWORD_USER_NAME}" ]];then
        echo "ERROR" "${PASSWORD_USER_NAME} is null,pls check !"
        return 1
    fi
    echo "INFO" "begin encrypt user password ..."

    encrypt_user_password=`echo -n "${passwd_user}" | openssl enc -aes-128-ecb -a -e -pass pass:"indicator" -nosalt`
    if [[ $? -ne 0 ]];then
        echo "WARN" "[ encrypt user password {openssl enc -aes-128-ecb -a -e -pass pass:"indicator" -nosalt} ] failed"
        encrypt_user_password=${passwd_user}
    fi

    if [[ "x" == "x${encrypt_user_password}" ]];then
        echo "WARN" "[ encrypt user password=${encrypt_user_password},it is null~ ] failed"
    fi

    (echo -E "${encrypt_user_password}";sleep 1;echo -E "${encrypt_user_password}") | sudo passwd "${PASSWORD_USER_NAME}" >/dev/null 2>/dev/null
    if [[ $? -ne 0 ]]; then
        echo "WARN" "Failed to change user passwd by aes"
    fi

    echo "INFO" "end encrypt user password ..."
}

function set_env
{
    COMPONENT_NAME=$1
    USER_HOME=$2
    USER_NAME=$3

    if [[ "X" == "X${COMPONENT_NAME}" ]] || [[ "X" == "X${USER_HOME}" ]];then
        echo "ERROR" "${COMPONENT_NAME} or ${USER_HOME} is null,pls check !"
        return 1
    fi

	echo "INFO" "begin set ${COMPONENT_NAME} umask and JDK env ..."

	file_bash=${USER_HOME}/.bashrc
    if [[ ! -f ${file_bash} ]];then
        sudo touch ${file_bash}
        if [[ $? -ne 0 ]];then
            echo "ERROR" "[ touch ${file_bash} ] failed"
            return 1
        fi
        sudo chown ${USER_NAME}:projectName ${file_bash}
    fi
    sudo su - ${USER_NAME} <<EOF
    echo "export APP_HOME=${USER_HOME}" >> ${file_bash}
    echo "export JAVA_HOME=/opt/JRE/jre1.8.0_232" >> ${file_bash}
    echo 'export JRE_HOME=\${JAVA_HOME}' >> ${file_bash}
    echo 'export  PATH=\$JAVA_HOME/bin:\$PATH' >> ${file_bash}
    echo 'export CLASSPATH=\$CLASSPATH:\$JAVA_HOME/lib' >> ${file_bash}
    echo "umask 027" >> ${file_bash}
EOF


    if [[ -d ${USER_HOME}/perl5 ]];then
          sudo chmod 750 -R ${USER_HOME}/perl5
    fi

    if [[ -d ${USER_HOME}/.local ]];then
        sudo chmod 750 -R ${USER_HOME}/.local
    fi

    echo "INFO" "successful set ${COMPONENT_NAME} umask and JDK env ..."
}

install_jdk()
{
    COMPONENT_NAME=$1
    USER_NAME=$2
    GROUP_NAME=$3
    USER_HOME=$4

    if [[ "X" == "X${COMPONENT_NAME}" ]] || [[ "X" == "X${USER_NAME}" ]] || [[ "X" == "X${GROUP_NAME}" ]] || [[ "X" == "X${USER_HOME}" ]];then
        echo "ERROR" "${COMPONENT_NAME} or ${USER_NAME} or ${GROUP_NAME} or ${USER_HOME} is null,pls check !"
        return 1
    fi

	echo "INFO" "begin install ${COMPONENT_NAME} JRE ..."

    sudo chown -R ${USER_NAME}:${GROUP_NAME} ${JRE_PATH}

    sudo su - ${USER_NAME} -c "source etc/profile;java -version" >/dev/null 2>/dev/null
	if [[ $? -ne 0 ]];then
        echo "ERROR" "[ su - ${USER_NAME} -c \"java -version\" ] failed"
        return 1
    fi
    sudo chmod 750 -R ${JRE_PATH}
	echo "INFO" "successful install ${COMPONENT_NAME} JDK ..."
}

install_openas()
{
    COMPONENT_NAME=$1
    USER_NAME=$2
    GROUP_NAME=$3
    USER_HOME=$4

    if [[ "X" == "X${COMPONENT_NAME}" ]] || [[ "X" == "X${USER_NAME}" ]] || [[ "X" == "X${GROUP_NAME}" ]] || [[ "X" == "X${USER_HOME}" ]];then
        echo "ERROR" "${COMPONENT_NAME} or ${USER_NAME} or ${GROUP_NAME} or ${USER_HOME} is null,pls check !"
        return 1
    fi

	echo "INFO" "begin install ${COMPONENT_NAME} openas ..."


	file_openas=${CUR_DIR}/${tomcat_name}
    if [[ ! -f ${file_openas} ]];then
        echo "ERROR" "[ file ${file_openas} ] is not exist,pls check~"
        return 1
    fi

	sudo unzip -q ${file_openas} -d ${USER_HOME}/tomcat
    if [[ $? -ne 0 ]];then
        echo "ERROR" "[ unzip ${file_openas} -d ${USER_HOME}/tomcat ] failed"
        return 1
    fi

 	sudo chown -R ${USER_NAME}:${GROUP_NAME} ${USER_HOME}/tomcat
      sudo chmod 750 ${USER_HOME}/tomcat
    echo "INFO" "successful install ${COMPONENT_NAME} openas ..."

}

function modify_tomcat_config
{
    COMPONENT_NAME=$1
    USER_NAME=$2
    GROUP_NAME=$3
    USER_HOME=$4
    LOCAL_IP=$5

    if [[ "X" == "X${COMPONENT_NAME}" ]] || [[ "X" == "X${USER_NAME}" ]] || [[ "X" == "X${GROUP_NAME}" ]] || [[ "X" == "X${USER_HOME}" ]];then
        echo "ERROR" "${COMPONENT_NAME} or ${USER_NAME} or ${GROUP_NAME} or ${USER_HOME} is null,pls check !"
        return 1
    fi

    echo "INFO" "begin modify ${COMPONENT_NAME} server.xml openas ..."

    server_path=${USER_HOME}/tomcat/conf/server.xml


    server_path_template=${CUR_DIR}/config/server.xml
    if [[ ! -f ${server_path_template} ]];then
        echo "ERROR" "[ file ${server_path_template} ] is not exist,pls check~"
        return 1
    fi

    sudo cp ${server_path_template} ${server_path}
    if [[ $? -ne 0 ]];then
        echo "ERROR" "[ cp ${server_path_template} ${server_path}] failed"
        return 1
    fi

    #修改https ip地址
    sudo su - ${USER_NAME} <<EOF

    #修改加解密地址
    sed -i "s#keystoreFile=\".*\"\\(.*keystorePass.*\\)#keystoreFile=\"${USER_HOME}/tls/romaServerKeyStore\"\\1#g" ${server_path}
    sed -i "s#truststoreFile=\".*\"\\(.*truststorePass.*\\)#truststoreFile=\"${USER_HOME}/tls/romaServerTrustStore\"\\1#g" ${server_path}
    sed -i "s#clientAuth=\".*\"\\(.*keystoreFile.*\\)#clientAuth=\"true\"\\1#g" ${server_path}

    sed -i "s#keystorePass=\".*\"\\(.*ciphers.*\\)#keystorePass=\"${keystorePass}\"\\1#g" ${server_path}
    sed -i "s#truststorePass=\".*\"\\(.*/>.*\\)#truststorePass=\"${truststorePass}\"\\1#g" ${server_path}
    #修改setvmargs.sh java prop
    chmod  -R 700 ${USER_HOME}/tomcat/bin
    echo "JAVA_OPTS=\"\$JAVA_OPTS -Dapp_home=${USER_HOME}\"" >> ${USER_HOME}/tomcat/bin/setvmargs.sh
    chmod  -R 0550 ${USER_HOME}/tomcat/bin
EOF



    #copy web.xml to user home openas config
    sudo cp ${CONF_PATH}/web.xml ${USER_HOME}/tomcat/conf
    if [[ $? -ne 0 ]];then
        echo "ERROR" "[ cp ${CONF_PATH}/web.xml ${USER_HOME}/tomcat/conf ] failed"
        return 1
    fi

    #copy catalina.properties to user home openas config
    sudo cp ${CONF_PATH}/catalina.properties ${USER_HOME}/tomcat/conf
    if [[ $? -ne 0 ]];then
        echo "ERROR" "[ cp ${CONF_PATH}/catalina.properties ${USER_HOME}/tomcat/conf ] failed"
        return 1
    fi

    #copy setkidenv.sh to user home openas config
    sudo cp ${CONF_PATH}/setkidenv.sh  ${USER_HOME}/tomcat/bin/kiddle/
    if [[ $? -ne 0 ]];then
        echo "ERROR" "[ cp ${CONF_PATH}/setkidenv.sh  ${USER_HOME}/tomcat/bin/kiddle ] failed"
        return 1
    fi

    # modify secretkey.properties

    secretkey_properties=${USER_HOME}/tomcat/conf/secretkey.properties
    sudo -i sed -i "s#common.key=.*#common.key=${common_key}#g" ${secretkey_properties}

    sudo chown -R ${USER_NAME}:${GROUP_NAME} ${USER_HOME}

    echo "INFO" "successful modify ${COMPONENT_NAME} server.xml openas ..."
}

main()
{
    echo "======================================================================="
    echo "INFO" "------ begin build projectName registry image -----"

    create_user "registry" ${REGISTRY_USER} ${REGISTRY_USER_GROUP_NAME} ${REGISTRY_USER_HOME} || return 1

    set_env "registry" ${REGISTRY_USER_HOME} ${REGISTRY_USER} || return 1

    install_jdk "registry" ${REGISTRY_USER} ${REGISTRY_USER_GROUP_NAME} ${REGISTRY_USER_HOME} || return 1

    install_openas "registry" ${REGISTRY_USER} ${REGISTRY_USER_GROUP_NAME} ${REGISTRY_USER_HOME} || return 1

    install_projectName_registry_service  || return 1

    modify_tomcat_config "registry" ${REGISTRY_USER} ${REGISTRY_USER_GROUP_NAME} ${REGISTRY_USER_HOME} ${local_ip} || return 1

    modify_server_xml_port || return 1

    modify_projectName_registry_service || return 1


    echo "INFO" "------ end build projectName registry image -----"
    echo "======================================================================="
}
main "$@"
```

有了这个脚本，我们docker build还是要手动敲命令，那么再来一个脚本。

`build_registry_image.sh`

```shell
#! /bin/bash

CUR_DIR=$(pwd)
cdir=$(cd `dirname $0`; pwd)
. ${cdir}/comm_lib

IMAGE_NAME=${REGISTRY_USER}

WORK_DIR=$(cd ../DockerFile/projectName_registry;pwd)

check_dictionary()
{
    LOG "INFO" "BEGIN CHECK DICTIONARY..."


	  if [[ ! -d ${WORK_DIR} ]];then
        LOG "ERROR" "[ dictionary ${WORK_DIR} ] is not exist,pls check~"
        return 1
    fi

    LOG "INFO" "DICTIONARY IS READY"
}

check_files()
{
    LOG "INFO" "BEGIN CHECK FILES..."

  	dockfile_path=${WORK_DIR}/Dockerfile
	  if [[ ! -f ${dockfile_path} ]];then
        LOG "ERROR" "[ file ${dockfile_path} ] is not exist,pls check~"
        return 1
    fi

    tomcat_path=${WORK_DIR}/OpenAS_Tomcat-7.zip
	  if [[ ! -f ${tomcat_path} ]];then
        LOG "ERROR" "[ file ${tomcat_path} ] is not exist,pls check~"
        return 1
    fi

    war_path=${WORK_DIR}/registry.war
	  if [[ ! -f ${war_path} ]];then
        LOG "ERROR" "[ file ${war_path} ] is not exist,pls check~"
        return 1
    fi

    sh_path=${WORK_DIR}/install_projectName_registry.sh
	  if [[ ! -f ${sh_path} ]];then
        LOG "ERROR" "[ file ${sh_path} ] is not exist,pls check~"
        return 1
    fi
}

buildImage(){

    cd ${WORK_DIR}

    docker build -t ${IMAGE_NAME} . || return 1

    if [[ $? -ne 0 ]];then
        LOG "ERROR" "${IMAGE_NAME}Image Build  failed"
        return 1
    fi

}

function main()
{
    echo "======================================================================="

    check_dictionary || return 1

    check_files || return 1

    buildImage || return 1

    echo "======================================================================="
}

main "$@"
```

```shell
#其中这个镜像名我们配在了common_lib中。
IMAGE_NAME=${REGISTRY_USER}
```

好了，写了半天，是骡子是，拉出来跑跑。

```shell
[root@node02 bin]# cd /opt/projectName/SetupScripts/projectNameScripts/bin/docker/bin
[root@node02 bin]# sh build_registry_image.sh
=======================================================================
[2021-06-28 23:08:39][INFO][][root][BEGIN CHECK DICTIONARY...]

[2021-06-28 23:08:39][INFO][][root][DICTIONARY IS READY]

[2021-06-28 23:08:39][INFO][][root][BEGIN CHECK FILES...]

Sending build context to Docker daemon  52.65MB
Step 1/15 : FROM centos7_hwjdk8
 ---> 7addb65ed3f6
Step 2/15 : ENV INSTALL_HOME=/opt/docker_install
 ---> Running in 436e36f4c713
Removing intermediate container 436e36f4c713
 ---> 9c6cf4df82f4
Step 3/15 : ENV SCRIPT_NAME=install_projectName_registry.sh
 ---> Running in 9d47bdf95dc0
Removing intermediate container 9d47bdf95dc0
 ---> 4457cc6a3b5b
Step 4/15 : ENV REGISTRY_HOME=/data01/registry
 ---> Running in 1d0573560912
Removing intermediate container 1d0573560912
 ---> 7a7d94d7b1ad
Step 5/15 : ENV TOMCATPATH=${REGISTRY_HOME}/tomcat
 ---> Running in 4318ee1fd9ab
Removing intermediate container 4318ee1fd9ab
 ---> f60b711e3fee
Step 6/15 : RUN mkdir -p ${INSTALL_HOME}/config
 ---> Running in 0bafcf3362ed
Removing intermediate container 0bafcf3362ed
 ---> 203039be9ed7
Step 7/15 : COPY OpenAS_Tomcat-7.zip ${INSTALL_HOME}
 ---> c3dada09286a
Step 8/15 : COPY registry.war ${INSTALL_HOME}
 ---> 863b6a19f960
Step 9/15 : COPY ${SCRIPT_NAME} ${INSTALL_HOME}
 ---> 3c9ea6dabbea
Step 10/15 : COPY config ${INSTALL_HOME}/config
 ---> b6e1a20d1d6a
Step 11/15 : RUN sh ${INSTALL_HOME}/${SCRIPT_NAME}
 ---> Running in 834d4299cb17
=======================================================================
INFO ------ begin build projectName registry image -----
INFO begin create projectName registry user ...
INFO GROUP_NAME:projectName
INFO begin encrypt user password ...
INFO end encrypt user password ...
INFO successful create projectName registry user ...
INFO begin set registry umask and JDK env ...
INFO successful set registry umask and JDK env ...
INFO begin install registry JRE ...
INFO successful install registry JDK ...
INFO begin install registry openas ...
INFO successful install registry openas ...
INFO begin install registry service ...
INFO end install registry service ...
INFO begin modify registry server.xml openas ...
Last login: Tue Jun 29 03:08:50 UTC 2021
INFO successful modify registry server.xml openas ...
INFO begin modify server.xml port openas ...
Last login: Tue Jun 29 03:08:51 UTC 2021
INFO successful modify server.xml port openas ...
INFO begin modify registry service ...
Last login: Tue Jun 29 03:08:51 UTC 2021
INFO successful modify registry service ...
INFO ------ end build projectName registry image -----
=======================================================================
Removing intermediate container 834d4299cb17
 ---> 2075f2e83001
Step 12/15 : RUN rm -rf ${INSTALL_HOME}
 ---> Running in a51f9db8b758
Removing intermediate container a51f9db8b758
 ---> 817edeead831
Step 13/15 : EXPOSE 8022
 ---> Running in 34b9b76c7ba5
Removing intermediate container 34b9b76c7ba5
 ---> 1c1ee23b204e
Step 14/15 : USER registry
 ---> Running in 110bc62aca7b
Removing intermediate container 110bc62aca7b
 ---> b565e892cca1
Step 15/15 : CMD ["/data01/registry/tomcat/bin/catalina.sh","run"]
 ---> Running in e0af975e3aa9
Removing intermediate container e0af975e3aa9
 ---> 82b43c9ecdda
Successfully built 82b43c9ecdda
Successfully tagged registry:latest
=======================================================================

[root@node02 bin]# docker images
REPOSITORY                               TAG                 IMAGE ID            CREATED             SIZE
registry                       latest              82b43c9ecdda        3 minutes ago       1.1GB

```

可以看到registry已经构建成功了，我们启动一下试试。

```shell
[root@node02 bin]# docker run -it -p 80:8022 --rm registry
...
2021-06-29 03:14:55.541  INFO 1 --- [      Thread-14] c.n.e.r.PeerAwareInstanceRegistryImpl    : Changing status to UP
2021-06-29 03:14:55.591  INFO 1 --- [      Thread-14] e.s.EurekaServerInitializerConfiguration : Started Eureka Server
2021-06-29 03:14:55.764  INFO 1 --- [ost-startStop-1] c.s.j.s.i.a.WebApplicationImpl           : Initiating Jersey application, version 'Jersey: 1.19.1 03/11/2016 02:08 PM'
2021-06-29 03:14:55.875  INFO 1 --- [ost-startStop-1] c.n.d.provider.DiscoveryJerseyProvider   : Using JSON encoding codec LegacyJacksonJson
2021-06-29 03:14:55.875  INFO 1 --- [ost-startStop-1] c.n.d.provider.DiscoveryJerseyProvider   : Using JSON decoding codec LegacyJacksonJson
2021-06-29 03:14:55.876  INFO 1 --- [ost-startStop-1] c.n.d.provider.DiscoveryJerseyProvider   : Using XML encoding codec XStreamXml
2021-06-29 03:14:55.876  INFO 1 --- [ost-startStop-1] c.n.d.provider.DiscoveryJerseyProvider   : Using XML decoding codec XStreamXml
2021-06-29 03:14:56,842
INFO: Starting ProtocolHandler ["http-nio-8022"]
2021-06-29 03:14:56,877
INFO: Server startup in 21631 ms
```

没有问题。

```shell
[root@node02 ~]# curl localhost
<!doctype html>
<!--[if lt IE 7]>      <html class="no-js lt-ie9 lt-ie8 lt-ie7"> <![endif]-->
<!--[if IE 7]>         <html class="no-js lt-ie9 lt-ie8"> <![endif]-->
<!--[if IE 8]>         <html class="no-js lt-ie9"> <![endif]-->
<!--[if gt IE 8]><!--> <html class="no-js"> <!--<![endif]-->
  <head>
    <base href="/">
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <title>Eureka</title>
    <meta name="description" content="">
    <meta name="viewport" content="width=device-width">

    <link rel="stylesheet" href="eureka/css/wro.css">

  </head>

  <body id="one">
<nav class="navbar navbar-default" role="navigation">
  <div class="container">
    <div class="navbar-header">
      <a class="navbar-brand" href=""><span></span></a>
      <button type="button" class="navbar-toggle" data-toggle="collapse" data-target="#bs-example-navbar-collapse-1">
        <span class="sr-only">Toggle navigation</span>
...
#node1
[root@node01 docker]# curl 192.168.60.130
<!doctype html>
<!--[if lt IE 7]>      <html class="no-js lt-ie9 lt-ie8 lt-ie7"> <![endif]-->
<!--[if IE 7]>         <html class="no-js lt-ie9 lt-ie8"> <![endif]-->
<!--[if IE 8]>         <html class="no-js lt-ie9"> <![endif]-->
<!--[if gt IE 8]><!--> <html class="no-js"> <!--<![endif]-->
  <head>
    <base href="/">
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <title>Eureka</title>
    <meta name="description" content="">
    <meta name="viewport" content="width=device-width"
```

本地访问没有问题,其他节点访问也有问题。（注意关闭防火墙）

好了，到这里，我们已经完成了registry镜像的初步构建，那么只完成了1/3的工作，这个镜像仍然还无法使用。首先还没有推送到docker仓库里面，其他节点是无法拉取的，也无法通过k8s来进行编排部署。

其次，我们并没有解决registry集群相互注册的问题。

由于篇幅限制，带着这些问题，我们下一章再见。

