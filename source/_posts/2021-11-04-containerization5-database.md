---
title: (五)容器化实战--数据库改造
comments: true
index_img: 'https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/docker_k8s.png'
banner_img: >-
  https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/background/vilige.jpg
tags: [容器化,Docker,Mysql,Redis,数据库]
date: 2021-06-04 16:21:56
updated: 2021-11-04 16:21:56
categories: 容器化技术
---

>在本章，由于后面所有的微服务均需要与数据库交互，我们本章进行数据库的容器化改造，首先我们来改造项目的持久层，mysql和redis。在本次改造过程当中，由于目前项目k8s测试资源不足，无法搭建数据库高可用集群，并且GM项目中我们也将采用云数据库的解决方案，因此本章仅保证功能的可用，不进行深入的研究。数据库的容器化工作中，卷的挂载及高可用的实现等问题都相对更为复杂，且数据库的性能也会受到一定程度的影响，如果能够选择云存储或其他封装好的持久层解决方案，那么可以跳过本章，也是笔者比较建议的方式。

## 1.	Mysql

### 1.1    软件获取

获取数据库相关软件包，以下以mysql-5.7.23-linux-glibc2.12-x86_64.tar.gz这个版本为例

### 1.2    构建过程设计

这里我们还是采取把安装文件包传到容器当中，通过shell脚本的方式完成镜像的构建。

为了提升镜像构建的速度与提高复用性，我们把mysql基础镜像与项目数据库初始化分为两层镜像。

- `mysql_5.7_base`基础镜像
- `mysql`应用层镜像

**目录树设计**

>|--docker
>
>|   |--bin
>|   |   |--comm_lib										#公共函数抽象、环境变量设置文件	
>|   |   |--build_registry_image.sh				#registry镜像构建脚本
>|   |   |--build_mysql_base_image.sh    	 #mysql基础层构建脚本
>|   |   |--build_baseOS_HWJDK_image.sh  #OS+JDK层构建脚本
>|   |   |--buildImage.sh								  #主构建脚本	
>|   |   |--build_mysql_app_image.sh   		#mysql应用层构建脚本
>|   |   |--build_redis_images.sh 		   		#redis镜像构建脚本
>
>|   |--config													 #环境变量读取文件
>|   |   |--config.property
>
>|   |--DockerFile											 #DockerFile文件夹
>
>|   |   |--centos_hwjdk1.8_mysql5.7_liquibase3.6.2
>|   |   |   |--install											#安装包文件夹
>|   |   |   |   |--package								  #二进制安装文件夹
>|   |   |   |   |   |--liquibase-3.6.2-bin.zip
>|   |   |   |   |   |--mysql-5.7.23-linux-glibc2.12-x86_64.tar.gz
>|   |   |   |   |--config                      				#相关配置文件夹
>|   |   |   |   |--bin										   #构建脚本文件夹
>|   |   |   |   |   |--install_mysql_liquibase.sh
>|   |   |   |--libaio-0.3.109-13.el7.x86_64.rpm   #其他文件
>|   |   |   |--Dockerfile                                            #Dockerfile文件  
>
>|   |   |--projectName_mysql
>|   |   |   |--install
>|   |   |   |   |--config
>|   |   |   |   |   |--db_changelog
>|   |   |   |   |   |   |--projectName.init.xml
>|   |   |   |   |   |   |--projectName.changelog.master.xml
>|   |   |   |   |   |   |--projectName.changelog.21.RP2.xml
>|   |   |   |   |--bin
>|   |   |   |   |   |--init_database.sh
>|   |   |   |--run.sh
>|   |   |   |--Dockerfile
>|   |   |--projectName_redis
>|   |   |   |--install
>|   |   |   |   |--package
>|   |   |   |   |   |--redis-4.0.9.tar.gz
>|   |   |   |   |--bin
>|   |   |   |   |   |--install.sh
>|   |   |   |--Dockerfile



### 1.3	镜像构建文件开发

#### 1.3.1	mysql基础镜像层

##### **Dockerfile**

```dockerfile
#使用的基础镜像 暂时使用centos7开源，后期可更改，注意如无法访问公网，本地需要提前准备
FROM centos7_hwjdk8
#作者信息  姓名 邮箱
#MAINTAINER name "xxxx@xxx.com"


#声明变量
ENV INSTALL_HOME=/opt/install
ENV SCRIPT_NAME=install_mysql_liquibase.sh
ENV MYSQL_HOME=/opt/mysql

#创建安装目录
RUN mkdir -p /data01 && mkdir -p ${INSTALL_HOME}

#由于centos7缺少libaio依赖，会导致后面mysql安装失败，如已有依赖可忽略此步骤
COPY libaio-0.3.109-13.el7.x86_64.rpm ${INSTALL_HOME}
RUN rpm -ivh ${INSTALL_HOME}/libaio-0.3.109-13.el7.x86_64.rpm && yum install -y numactl

#复制安装脚本
COPY install ${INSTALL_HOME}

#运行安装脚本
RUN sh ${INSTALL_HOME}/bin/${SCRIPT_NAME}

```

##### **镜像构建脚本** 

`install_mysql_liquibase.sh`

```shell
#! /bin/bash
##########################################################################################
#function:      refresh_database
#author:        XXXXXXX
#date:          2021-07-19
##########################################################################################

######################基本配置####################################################
CUR_DIR=$(pwd)
jdk_path="/opt/JRE"
liquibase_package="liquibase-3.6.2-bin.zip"
package_dir="/opt/install/package"
db_changelog_dir="/opt/install/config/db_changelog"
mysql_pachage="/opt/install/package/mysql-5.7.23-linux-glibc2.12-x86_64.tar.gz"
mysql_dest="/opt/mysql-5.7.23-linux-glibc2.12-x86_64"
DB_ADDRESS="127.0.0.1"
DB_PORT="3306"
DB_NAME="data_waiter"
DB_USERNAME="root"
DB_PASSWORD="Huawei@123"
MYSQL_UID="3000"
MYSQL_GID="3000"


function install_mysql()
{
    DB_ADDRESS=$1
    DB_NAME=$2
    DB_PASSWORD=$3
    echo "INFO" "------ begin prepare mysql -----"
    #解压mysql安装文件
    tar -xzvf /opt/install/package/mysql-5.7.23-linux-glibc2.12-x86_64.tar.gz -C /opt
    #重命名 mysql目录
    mv /opt/mysql-5.7.23-linux-glibc2.12-x86_64 /opt/mysql

    if [[ ! -d /opt/mysql ]];then
        echo "ERROR" "[ file /opt/mysql ] is not exist,pls check~"
        return 1
    fi

    #设置数据库时区
    #Install MySQL
    #set timezone
    #rm -rf /etc/localtime
    #ln -s /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
    #Delete Old Mysql program
    #Disable Selinux

    #创建mysql用户与组,授权目录
    groupadd mysql -g ${MYSQL_GID}
    useradd -d /home/mysql -s /bin/bash -u ${MYSQL_UID} -g mysql -m mysql
    chown -R mysql:mysql /opt/mysql

    if [[ $? -ne 0 ]];then
        echo "ERROR" "user&group add  failed"
        return 1
    fi

    #创建文件目录
    mkdir -p /data01/mysql-data
    mkdir /data01/mysql-data/tmp
    mkdir /data01/mysql-data/log
    chown -R mysql:mysql /data01/mysql-data
    chgrp -R mysql /data01/mysql-data
    if [[ $? -ne 0 ]];then
        echo "ERROR" "directory /data01/mysql-data created failed"
        return 1
    fi

    #创建配置文件 my.cnf
    touch /opt/mysql/my.cnf
    cat >>/opt/mysql/my.cnf<<EOF
[mysqld]
basedir = /opt/mysql
#bind-address = ${DB_ADDRESS}
datadir = /data01/mysql-data/workdbs
tmpdir = /data01/mysql-data/tmp/
port = 3306
socket =/opt/mysql/lib/mysql.sock
lower_case_table_names=1
character-set-server = utf8
max_allowed_packet = 1000M
sql_mode=NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES,STRICT_ALL_TABLES
log-error=/data01/mysql-data/log/mysql_3306.log
max_connections=1000
event_scheduler=ON
[mysql]
default-character-set = utf8
socket =/opt/mysql/lib/mysql.sock
EOF
    chown mysql:mysql /opt/mysql/my.cnf
    \cp -rf /opt/mysql/my.cnf /etc/my.cnf
    if [[ $? -ne 0 ]];then
        echo "ERROR" "edit /opt/mysql/my.cnf failed"
        return 1
    fi
    #设置环境变量
    cat >>/etc/profile<<EOF
export PATH=$PATH:/opt/mysql/bin
export PATH=$PATH:/etc/init.d
EOF
    source /etc/profile
    if [[ $? -ne 0 ]];then
        echo "ERROR" "edit /etc/profile failed"
        return 1
    fi

    #初始化MySQL
    cp -a /opt/mysql/support-files/mysql.server /etc/init.d/mysql.server
    /opt/mysql/bin/mysqld --initialize --user=mysql --basedir=/opt/mysql/ --datadir=/data01/mysql-data/workdbs
    if [[ $? -ne 0 ]];then
        echo "ERROR" "initialize mysqld failed"
        return 1
    fi

    #创建软连接
    ln -s /opt/mysql /usr/local/mysql
    ln -s /opt/mysql/lib/mysql.sock /tmp/mysql.sock
    if [[ $? -ne 0 ]];then
        echo "ERROR" "create softlink failed"
        return 1
    fi

    #获取初始密码
    mysqlrootpwd=`cat /data01/mysql-data/log/mysql_3306.log|grep root|awk -F' ' 'NR==1 {print $NF}'`
    if [[ $? -ne 0 ]];then
        echo "ERROR get mysqlrootpwd== ${mysqlrootpwd} failed "
        return 1
    fi

    #启动MySQL
    /opt/mysql/support-files/mysql.server start
    if [[ $? -ne 0 ]];then
        echo "ERROR" "mysql.server start failed"
        return 1
    fi

    #以脚本的方式更改初始密码、授权登录IP、创建初始数据库data_waiter等
    cat >/tmp/mysql_sec_script<<EOF
grant all privileges on *.* to 'root'@"%" identified by "${DB_PASSWORD}" with grant option;
CREATE DATABASE \`${DB_NAME}\` CHARACTER SET 'utf8' COLLATE 'utf8_general_ci';
flush privileges;
EOF
    /opt/mysql/bin/mysqladmin -uroot -p${mysqlrootpwd} password ${DB_PASSWORD}
    /opt/mysql/bin/mysql -u root -p${DB_PASSWORD}  < /tmp/mysql_sec_script
    if [[ $? -ne 0 ]];then
        echo "ERROR" "mysql config changed failed"
        return 1
    fi
    rm -f /tmp/mysql_sec_script
    #将可执行文件复制至/usr/bin 下方便启动
    cp /opt/mysql/bin/mysql /usr/bin

    if [[ $? -ne 0 ]];then
    echo "==================MySQL 5.7 install faild!======================"
    exit 1
    else
    echo "==================MySQL 5.7 install success======================"
    echo -e "\n"
    fi
}

#准备liquibase安装文件
function install_liquibase()
{
    echo "INFO" "------ begin install liquibase -----"


    file_liquibase=${package_dir}/${liquibase_package}
	  if [[ ! -f ${file_liquibase} ]];then
        echo "ERROR" "[ file ${file_liquibase} ] is not exist,pls check~"
        return 1
    fi

    unzip -o ${file_liquibase} -d ${package_dir} >/dev/null 2>&1
	  if [[ $? -ne 0 ]];then
        echo "ERROR" "[ unzip ${file_liquibase} ] failed"
        return 1
    fi

    echo "INFO" "------ successful install liquibase -----"
}


    #设置JAVA环境变量
function set_env()
{
    echo "INFO" "------ begin set_env  -----"
    file_bash=/etc/profile
    root_bash=~/.bashrc
    sudo chmod 646 ${file_bash}
    echo "export JAVA_HOME=${jdk_path}/jre1.8.0_232" >> ${file_bash}
    echo 'export JRE_HOME=${JAVA_HOME}' >> ${file_bash}
    echo 'export PATH=$JAVA_HOME/bin:$PATH' >> ${file_bash}
    echo 'export CLASSPATH=$CLASSPATH:$JAVA_HOME/lib' >> ${file_bash}
    sudo chmod 644 ${file_bash}
    echo 'source /etc/profile' >> ${root_bash}
    source ${file_bash}
    java -version >/dev/null 2>/dev/null

	  if [[ $? -ne 0 ]];then
        echo "ERROR" "[ java -version ] failed"
        return 1
    fi
        echo "INFO" "------ successful set_env  -----"
}

function main()
{
    echo "INFO" "------ begin install projectName database -----"

    install_mysql ${DB_ADDRESS} ${DB_NAME} ${DB_PASSWORD} || return 1

    set_env || return 1

    install_liquibase || return 1


    echo "INFO" "------ end install projectName database root -----"
}

main "$@"
```



##### **打包镜像脚本**

`build_mysql_base_image.sh`

```shell
#! /bin/bash

#配置安装变量
CUR_DIR=$(pwd)
cdir=$(cd `dirname $0`; pwd)
. ${cdir}/comm_lib
IMAGE_NAME=mysql_5.7_base
DOCKER_REGISTRY_URL=${Docker_Registry_URL}
IMAGE_VERSION=1.0.0
WORK_DIR=$(cd ../DockerFile/centos_hwjdk1.8_mysql5.7_liquibase3.6.2;pwd)

#检查安装文件夹
check_dictionary()
{
    LOG "INFO" "BEGIN CHECK DICTIONARY..."


	  if [[ ! -d ${WORK_DIR} ]];then
        LOG "ERROR" "[ dictionary ${WORK_DIR} ] is not exist,pls check~"
        return 1
    fi

    LOG "INFO" "DICTIONARY IS READY"
}

#检查安装文件
check_files()
{
    LOG "INFO" "BEGIN CHECK FILES..."

  	dockfile_path=${WORK_DIR}/Dockerfile
	  if [[ ! -f ${dockfile_path} ]];then
        LOG "ERROR" "[ file ${dockfile_path} ] is not exist,pls check~"
        return 1
    fi

    mysql_path=${WORK_DIR}/install/package/mysql-5.7.23-linux-glibc2.12-x86_64.tar.gz
	  if [[ ! -f ${mysql_path} ]];then
        LOG "ERROR" "[ file ${mysql_path} ] is not exist,pls check~"
        return 1
    fi

    liquibase_path=${WORK_DIR}/install/package/liquibase-3.6.2-bin.zip
	  if [[ ! -f ${liquibase_path} ]];then
        LOG "ERROR" "[ file ${liquibase_path} ] is not exist,pls check~"
        return 1
    fi
    sh_path=${WORK_DIR}/install/bin/install_mysql_liquibase.sh
	  if [[ ! -f ${sh_path} ]];then
        LOG "ERROR" "[ file ${sh_path} ] is not exist,pls check~"
        return 1
    fi
}

#构建镜像
buildImage(){

    cd ${WORK_DIR}

    docker build -t ${IMAGE_NAME} . || return 1

    if [[ $? -ne 0 ]];then
        LOG "ERROR" "${IMAGE_NAME}Image Build  failed"
        return 1
    fi

}

#往镜像仓推送镜像
pushdImage(){

    docker tag ${IMAGE_NAME} ${DOCKER_REGISTRY_URL}/${IMAGE_NAME} || return 1

    if [[ $? -ne 0 ]];then
       LOG "ERROR" "${DOCKER_REGISTRY_URL}/${IMAGE_NAME}Image tag  failed"
        return 1
    fi

    docker push ${DOCKER_REGISTRY_URL}/${IMAGE_NAME} || return 1

    if [[ $? -ne 0 ]];then
        LOG "ERROR" "${DOCKER_REGISTRY_URL}/${IMAGE_NAME}Image push  failed"
        return 1
    fi

}

#主函数
function main()
{
    echo "======================================================================="

    check_dictionary || return 1

    check_files || return 1

    buildImage || return 1

    pushdImage || return 1

    echo "======================================================================="
}

main "$@"
```

#### 1.3.2	mysql应用镜像层

##### Dockerfile

```dockerfile
FROM mysql_5.7_base
#作者信息  姓名 邮箱
#MAINTAINER name "xxxx@xxx.com"

#声明变量
ENV INSTALL_HOME=/opt/install
ENV SCRIPT_NAME=init_database.sh
ENV MYSQL_HOME=/opt/mysql

#复制安装脚本
COPY install ${INSTALL_HOME}
COPY run.sh ${MYSQL_HOME}/bin

#运行安装脚本
RUN sh ${INSTALL_HOME}/bin/${SCRIPT_NAME}
#删除安装目录
RUN rm -rf ${INSTALL_HOME}

#更换用户、工作目录
USER mysql
WORKDIR ${MYSQL_HOME}/bin
CMD ["sh","run.sh"]
#Docker bu5  ild命令参考：docker build -t centos7_hwjdk8 .
```

##### 镜像构建脚本

`init_database.sh`

```shell
#! /bin/bash
##########################################################################################
#function:      refresh_database
#author:        XXXXXXX
#date:          2018-11-21
##########################################################################################

######################基本配置####################################################
CUR_DIR=$(pwd)
jdk_path="/opt/JRE"
package_dir="/opt/install/package"
db_changelog_dir="/opt/install/config/db_changelog"
DB_ADDRESS="127.0.0.1"
DB_PORT="3306"
DB_NAME="data_waiter"
DB_USERNAME="root"
DB_PASSWORD="Huawei@123"
run_file="/opt/mysql/bin/run.sh"

#初始化liquibase
function exec_liquibase()
{
  	if [[ ! -f ${run_file} ]];then
        echo "ERROR" "[ file ${run_file} ] is not exist,pls check~"
        return 1
    fi
    sudo chown mysql ${run_file}
    sudo chgrp mysql ${run_file}
#. lib/us.sh

#export CIPHER_TEXT=${DB_PASSWORD}

#sudo chmod 750 qUKwo8UXe61X5aqT.sh

#obfuscated_text=$(./qUKwo8UXe61X5aqT.sh 'CIPHER_TEXT')

#if [ $? -ne 0 ]; then
#  error_msg=$obfuscated_text
#  echo "$error_msg"
#  exit 1
#fi

#plaintext=$(dstring "$obfuscated_text")

#password=$plaintext

#unset obfuscated_text
#unset plaintext
#unset CIPHER_TEXT

    dir_liquibase=${package_dir}/liquibase-3.6.2-bin
    #echo "INFO" "password has been decrypted"
	  if [[ ! -d ${dir_liquibase} ]];then
        echo "ERROR" "[ dir ${dir_liquibase} ] is not exist,pls check~"
        return 1
    fi

    source /etc/profile
    sudo chmod 750 -R ${dir_liquibase}
    /opt/mysql/support-files/mysql.server start
    cd ${dir_liquibase};

       sh liquibase --driver=com.mysql.cj.jdbc.Driver \
       --classpath=${db_changelog_dir} \
       --changeLogFile=projectName.changelog.master.xml \
       --url="jdbc:mysql://${DB_ADDRESS}:${DB_PORT}/${DB_NAME}?serverTimezone=UTC&useSSL=false" \
       --username=${DB_USERNAME} \
       --password=${DB_PASSWORD} \
       update
    if [[ $? -ne 0 ]];then
        echo "ERROR" "failed execute liquibase update, please check error message."
        return 1
    fi

}

# 数据库密码加密函数
function dstring()
{
    ens='abcdefghijklmnopqrstuvwxyzABCDEFHIJKLMNOPQRSTUVWXYZ012345689`~!@$%^&*()-_=+|[{}];:'\''",<.>/? '
    des='p;)ILlF+rZUW6=smX-V3SEid:xaw2t[HC|</D,_To 9uMk0fK1N8}eJcqz4`gOb~.BRj!h$(Q'\''"]{^A5*&n@y?v%>YP'
    echo "$*" | tac | rev | sed "y#$des#$ens#" | sed "y/G7\\\\\#/\\\\\#G7/"
}


function main()
{
    echo "======================================================================="
    echo "INFO" "------ begin initialize projectName database -----"

    exec_liquibase || return 1

    echo "INFO" "------ end initialize projectName database root -----"
    echo "======================================================================="
}

main "$@"
```



##### 打包镜像脚本

`build_mysql_app_image.sh`

```shell
#! /bin/bash

CUR_DIR=$(pwd)
cdir=$(cd `dirname $0`; pwd)
. ${cdir}/comm_lib

IMAGE_NAME=mysql
DOCKER_REGISTRY_URL=${Docker_Registry_URL}
IMAGE_VERSION=1.0.0
WORK_DIR=$(cd ../DockerFile/projectName_mysql;pwd)

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

    sh_path=${WORK_DIR}/install/bin/install.sh
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

pushdImage(){

    docker tag ${IMAGE_NAME} ${DOCKER_REGISTRY_URL}/${IMAGE_NAME} || return 1

    if [[ $? -ne 0 ]];then
       LOG "ERROR" "${DOCKER_REGISTRY_URL}/${IMAGE_NAME}Image tag  failed"
        return 1
    fi

    docker push ${DOCKER_REGISTRY_URL}/${IMAGE_NAME} || return 1

    if [[ $? -ne 0 ]];then
        LOG "ERROR" "${DOCKER_REGISTRY_URL}/${IMAGE_NAME}Image push  failed"
        return 1
    fi

}

function main()
{
    echo "======================================================================="

    check_dictionary || return 1

    check_files || return 1

    buildImage || return 1

    pushdImage || return 1

    echo "======================================================================="
}

main "$@"
```



### 1.4	kubernetes资源文件开发

```yml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  namespace: projectName
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5G
  #storageClassName: nfs #需要和pv的sc类型一致，否则会绑定失败
  selector:
    matchLabels:
      pv: mysql-pv
---
# 有状态pod，会自动根据metadata.name及实例数量生产xxx-0,xxx-1的pod
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: projectName
spec:
  serviceName: mysql-service
  replicas: 1 #由于无k8s开发环境及本地性能原因 暂时无法研究高可用读写分离集群
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: 192.168.60.151:5000/mysql:1.0.0 # 配置镜像路径，也就是我们刚才push好的镜像
          imagePullPolicy: Always
          ports:
            - containerPort: 3306
          volumeMounts:
            - mountPath: /media
              name: mysql-data
          env: # 环境变量设置
            - name: MY_POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name # 取值metadata.name，这里注意，statefulset类型的对象取到的是有-0,-1序号后缀的
            - name: MYSQL_PORT
              value: "3306"
            - name: MYSQL_INSTANCE_HOSTNAME #环境变量，这个变量在我们项目的配置文件中有配置，作用是指定注册到eureka集群中的hostname
              value: ${MY_POD_NAME}.mysql-service
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
              value: mysql-service
            - name: MYSQL_REPLICAS
              value: "2"
      volumes:
        - name: mysql-data
          hostPath:
            path: /data
            type: Directory
  podManagementPolicy: "Parallel" # 以并行方式创建pod，默认是串行的
---
# headless service 无头服务
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
  namespace: projectName
  labels:
    app: mysql
spec:
  clusterIP: None
  ports:
    - port: 3306
      name: mysql-svc-port
      targetPort: 3306
  selector:
    app: mysql
```

这里因为Mysql高可用集群是有主从之分的，启停顺序也需要进行控制。所以我们采用的是StatefulSet+Headless service的方式来实现，这样便于我们后面高可用、读写分离Mysql的集群扩展。

```shell
[mysqld]
basedir = /opt/mysql
bind-address = 192.168.60.144
datadir = /data01/mysql-data/workdbs
tmpdir = /data01/mysql-data/tmp/
port = 3306
socket =/opt/mysql/lib/mysql.sock
lower_case_table_names=1
character-set-server = utf8
max_allowed_packet = 1000M
sql_mode=NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES,STRICT_ALL_TABLES
log-error=/data01/mysql-data/log/mysql_3306.log
max_connections=1000
event_scheduler=ON
[mysql]
default-character-set = utf8
socket =/opt/mysql/lib/mysql.sock

```



```shell
[root@localhost etc]# \cp -rf /opt/mysql/my.cnf /etc/my.cnf

```



 

c. 修改系统配置文件profile。 

1. 1. 1. 编辑etc目录下的“profile”文件。 

**# vi /etc/profile**

输入**i**进入编辑模式，在文件末尾添加如下内容：

export PATH=$PATH:/opt/mysql/bin

export PATH=$PATH:/etc/init.d

添加完成后按**Esc**退出编辑模式，执行**:wq!**保存并退出。

1. 1. 1. 重新加载etc目录下的profile文件。 

**# source /etc/profile**

d. 将mysql.server复制到/etc/init.d/ 。 

**# cd /opt/mysql**

**# cp -a ./support-files/mysql.server /etc/init.d/mysql.server**

e. 初始化mysql 

**# cd /opt/mysql**

**# ./bin/mysqld --initialize --user=mysql --basedir=/opt/mysql/ --datadir=/data01/mysql-data/workdbs**

命令执行后，如正确，则不会有显示信息。

f. 查看日志文件，获取临时密码。

**# cat /data01/mysql-data/log/mysql_3306.log**



```shell
[root@localhost mysql]# ./bin/mysqld --initialize --user=mysql --basedir=/opt/mysql/ --datadir=/data01/mysql-data/workdbs
[root@localhost mysql]# cat /data01/mysql-data/log/mysql_3306.log
2021-07-08T06:37:30.261037Z 0 [Warning] TIMESTAMP with implicit DEFAULT value is deprecated. Please use --explicit_defaults_for_timestamp server option (see documentation for more details).
2021-07-08T06:37:30.261108Z 0 [Warning] 'NO_ZERO_DATE', 'NO_ZERO_IN_DATE' and 'ERROR_FOR_DIVISION_BY_ZERO' sql modes should be used with strict mode. They will be merged with strict mode in a future release.
2021-07-08T06:37:30.261124Z 0 [Warning] 'NO_AUTO_CREATE_USER' sql mode was not set.
2021-07-08T06:37:31.376072Z 0 [Warning] InnoDB: New log files created, LSN=45790
2021-07-08T06:37:31.610332Z 0 [Warning] InnoDB: Creating foreign key constraint system tables.
2021-07-08T06:37:31.685245Z 0 [Warning] No existing UUID has been found, so we assume that this is the first time that this server has been started. Generating a new UUID: f8d794a3-dfb6-11eb-9de4-000c29da9da8.
2021-07-08T06:37:31.687340Z 0 [Warning] Gtid table is not ready to be used. Table 'mysql.gtid_executed' cannot be opened.
2021-07-08T06:37:31.690134Z 1 [Note] A temporary password is generated for root@localhost: yH0IbBLiq5=v

```



获取临时密码，如：“**yH0IbBLiq5=v**”。

g. 创建软连接。 

1. 1. 1. 将mysql的安装目录软连接到local下面。 

**# ln -s /opt/mysql /usr/local/mysql**

1. 1. 1. 将mysql.sock文件软连接到tmp下面 

**# ln -s /opt/mysql/lib/mysql.sock /tmp/mysql.sock**

1. 注册并设置mysql.server服务为开机自启动。 

**# systemctl enable mysql.server.service**

1. 启动并修改初始密码。     

1. 1. 在“/opt/mysql/support-files”目录下启动MySQL。      

**# cd /opt/mysql/support-files**

**# mysql.server start**

1. 1. 查看MySQL状态。 

**# mysql.server status**

系统显示如下类似信息表示MySQL状态正常：

![](https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/clip_image004.png)

 

1. 1. 在“opt/mysql/bin”目录下执行以下命令登录MySQL。      

**# cd /opt/mysql/bin**

**# mysql -u root -p**

![](https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/clip_image005.png)按照提示信息输入记录的临时密码。

 

![](https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/clip_image006.png)登录成功后系统显示如下类似信息：

 

1. 1. 修改root用户密码。 

mysql> **set password=password('***Password***');**

其中，单引号中的*Password*由用户自定义。

1. 1. 赋予任何主机访问数据的权限。      

mysql> **grant all privileges on \*.\* to 'root'@'%' identified by '***Password***' with grant option;**

其中，单引号中的*Password*由用户自定义。

1. 1. 使修改生效并使用数据库。      

mysql> **flush privileges;**

mysql> **use mysql;**

1. 1. 查看当前用户。      

mysql> **select host,user from user;**

![](https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/clip_image007.png)系统显示如下类似信息，表示数据库已正常安装和运行。

 

1. 1. 退出MySQL数据库。 

mysql> **exit**

\6. 将/opt/mysql/bin/目录下的可执行程序**mysql**拷贝到/usr/bin目录下，方便后续执行这个命令。 

**# cp /opt/mysql/bin/mysql /usr/bin**

 

 

## 2   Redis

### 2.1    软件获取

https://onebox.huawei.com/p/db2887087836d491d6b5aa88bcb372b6

### 2.2    操作步骤

以下以redis-4.0.9.tar.gz这个版本为例

​    tar -zxvf redis-4.0.9.tar.gz

```shell
#redis是C语言开发，运行需要c语言环境，安装gcc，如有跳过此步骤
yum install gcc
```

```shell
[root@storage01 redis-4.0.9]# make MALLOC=libc

```

   3.编译成功后，进入src文件夹，执行make install进行Redis安装。

​    如下图示安装完成，界面如下：
​    ![](https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/clip_image017.jpg)

 ![](https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/clip_image019.jpg)



```shell
[root@storage01 redis-4.0.9]# mkdir -p etc
[root@storage01 redis-4.0.9]# mkdir -p bin
[root@storage01 redis-4.0.9]# mv redis.conf etc/
[root@storage01 src]# mv mkreleasehdr.sh redis-benchmark redis-check-aof redis-check-rdb redis-cli redis-server /usr/local/redis-4.0.9/bin/

[root@storage01 bin]# ./redis-server
37934:C 16 Jul 14:41:46.457 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
37934:C 16 Jul 14:41:46.457 # Redis version=4.0.9, bits=64, commit=00000000, modified=0, pid=37934, just started
37934:C 16 Jul 14:41:46.457 # Warning: no config file specified, using the default config. In order to specify a config file use ./redis-server /path/to/redis.conf
37934:M 16 Jul 14:41:46.458 * Increased maximum number of open files to 10032 (it was originally set to 1024).
                _._
           _.-``__ ''-._
      _.-``    `.  `_.  ''-._           Redis 4.0.9 (00000000/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
 |    `-._   `._    /     _.-'    |     PID: 37934
  `-._    `-._  `-./  _.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |           http://redis.io
  `-._    `-._`-.__.-'_.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

37934:M 16 Jul 14:41:46.475 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
37934:M 16 Jul 14:41:46.476 # Server initialized
37934:M 16 Jul 14:41:46.477 # WARNING overcommit_memory is set to 0! Background save may fail under low memory condition. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect.
37934:M 16 Jul 14:41:46.481 # WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' as root, and add it to your /etc/rc.local in order to retain the setting after a reboot. Redis must be restarted after THP is disabled.
37934:M 16 Jul 14:41:46.481 * Ready to accept connections


[root@storage01 bin]# ./redis-server /usr/local/redis-4.0.9/etc/redis.conf

#把 redis.conf配置文件中的 bind 127.0.0.1 这一行给注释掉，这里的bind指的是只有指定的网段才能远程访问这个redis，注释掉后，就没有这个限制了。
sed -n 's/bind 127.0.0.1/#bind 127.0.0.1/p' 1.txt
#protected-mode 设置成no（默认是设置成yes的， 防止了远程访问，在redis3.2.3版本后）
sed -n 's/protected-mode yes/protected-mode no/p' 1.txt
#修改Redis默认密码 (默认密码为空)
sed -n 's/# requirepass foobared/requirepass Huawei@123/p' 1.txt

sed -n 's/bind 127.0.0.1/#bind 127.0.0.1/p' 1.txt

sed -n 's/bind 127.0.0.1/#bind 127.0.0.1/p' 1.txt
```

 

```shell
CREATE DATABASE `data_waiter` CHARACTER SET 'utf8' COLLATE 'utf8_general_ci';
```

