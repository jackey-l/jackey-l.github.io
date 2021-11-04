---
title: (二)容器化实战--K8s探索
comments: true
index_img: 'https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/docker_k8s.png'
banner_img: >-
  https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/background/vilige.jpg
tags: [容器化,Kubernates]
date: 2021-06-04 16:18:57
updated: 2021-11-04 16:18:57
categories: 容器化技术
---

## 1.	什么是Kubernates

​	K8s是一个基于Docker容器的PAAS平台。源于Google十多年的容器化实践--Borg。谷歌每周要启动销毁数十亿容器。其合理的抽象、API化的架构等功能广受好评。是目前业界公认的容器化最佳方案。

我们本次探索的出发点主要2点

1. 需求驱动，客户要求我们进行容器化部署，那么我们的项目在从传统的部署方式向容器化演进当中到底有哪些挑战，我们又该如何解决呢？
2. 经验积累，客户要求的是基于PAAS平台的容器化能力，那么基于业界主流解决方案docker和k8s的原生使用的能力能让我们将来更好地适配任何容器化平台。

我们本次的目标：

- 基于docker、k8s原生能力，打造可插拔，松耦合，与开发分支代码0冲突的程序容器化能力，完美兼容原来的部署方式。完善的部署脚本，支持一键部署k8s环境、一键发布应用、推送镜像、页面式运维。



## 2.	如何搭建一个本地K8s学习测试环境

### 一、	IAAS准备工作

> **准备搭建1个master节点 2个node节点的环境。**

#### 1.1准备IAAS资源

- 这里使用VMware本地创建虚拟机。

  [Centos7.6操作系统安装及优化全纪录](https://blog.51cto.com/3241766/2398136)

- 准备4台Linux系统虚拟机或物理机，一台作为Master节点，2台作为node节点,1台作为私有仓库

  这里使用VMware本地创建4台虚拟机

  系统镜像为CentOS7

  修改网络设置，防止虚拟机IP经常变动

![](https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/8d233f592456ad0e8393cc5d8200e018_453x254.png@900-0-90-f.png)

![](https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/2ff1c142f7e34644e983669e52747f14_730x702.png@900-0-90-f.png)

![](https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/b684626c7075be9545f93189f02285dd_730x666.png@900-0-90-f.png)

![](https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/e1f7c035e07a17b3f882cf799304a61b_494x373.png@900-0-90-f.png)



|  主机名  |       IP       |
| :------: | :------------: |
| master01 | 192.168.60.128 |
|  node01  | 192.168.60.134 |
|  node02  | 192.168.60.135 |



#### 1.2	修改主机名

```shell
hostnamectl set-hostname master01   #分别修改master和node节点
[root@master01 /]# more /etc/hostname
master01
```

修改完后需要重启:	`reboot`

#### 1.3	修改hosts文件

```shell
[root@master01 /]# cat >> /etc/hosts << EOF
192.168.60.128    master01
192.168.60.134     node01
192.168.60.135     node02
EOF

[root@master01 /]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.60.128    master01
192.168.60.129     node01
192.168.60.130     node02

#后面由于资源不足  IAAS层已改为
[root@master01 /]# cat >> /etc/hosts << EOF
192.168.4.114    master01
192.168.4.115     node01
192.168.4.116     node02
192.168.4.117     node03
192.168.4.118     storage01
EOF
```

#### 1.4	验证mac地址与uuid

```shell
[root@master01 /]# cat /sys/class/net/ens33/address
00:0c:29:87:a4:9d
[root@master01 /]# cat /sys/class/dmi/id/product_uuid
9F064D56-2A45-962C-09AC-D4732487A49D
```

确保各节点mac和uuid唯一

#### 1.5	禁用swap

```shell
[root@master01 /]# swapoff -a	#s\临时禁用
[root@master01 /]# sed -i.bak '/swap/s/^/#/' /etc/fstab   #重启后也生效---》注释swap
[root@master01 /]# cat /etc/fstab

#
# /etc/fstab
# Created by anaconda on Fri Jun 18 07:00:09 2021
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/centos-root /                       xfs     defaults        0 0
UUID=1977c58c-bd28-4d57-aec2-3ae6329638d9 /boot                   xfs     defaults        0 0
#/dev/mapper/centos-swap swap                    swap    defaults        0 0

```

#### 1.6	br_netfilter模块加载

> centos7用户需要设置路由

**查看br_netfilter模块：**

```shell
[root@master01 ~]# lsmod |grep br_netfilter
```

如果系统没有br_netfilter模块则执行下面的新增命令，如有则忽略。

**临时新增br_netfilter模块:**

```shell
[root@master01 ~]# modprobe br_netfilter
```

该方式重启后会失效

**永久新增br_netfilter模块：**

```shell
[root@master01 ~]# cat > /etc/rc.sysinit << EOF
#!/bin/bash
for file in /etc/sysconfig/modules/*.modules ; do
[ -x $file ] && $file
done
EOF
[root@master01 ~]# cat > /etc/sysconfig/modules/br_netfilter.modules << EOF
modprobe br_netfilter
EOF
[root@master01 ~]# chmod 755 /etc/sysconfig/modules/br_netfilter.modules
[root@master01 /]# lsmod |grep br_netfilter
br_netfilter           22256  0
bridge                151336  1 br_netfilter
```

#### 1.7	内核参数修改

**临时修改**

```shell
[root@master01 /]# sysctl net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-iptables = 1
[root@master01 /]# sysctl net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-iptables = 1
```

**永久修改**

```shell
[root@master01 /]# cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
[root@master01 /]# sysctl -p /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
```

#### 2.免密登录

配置master01到node01、node02免密登录，本步骤只在master01上执行。

##### 2.1	创建秘钥

```shell
[root@master01 ~]# ssh-keygen -t rsa

Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa):
Created directory '/root/.ssh'.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:judqqhhvCscqRnBFEwKTACUK7F8W3ZTIu+xnHEUK0aE root@master01
The key's randomart image is:
+---[RSA 2048]----+
|@+o.+.o.*oo      |
|++ ..o =.+ .     |
|o  .  .Eo o      |
|...  o . . .     |
|... o . S .      |
| o .   = .       |
|+ o   o + .      |
|oB.   .+ +       |
|*oo..o..+        |
+----[SHA256]-----+
```

##### 2 .2	将秘钥同步至node01、node02

```shell
[root@master01 ~]# ssh-copy-id -i /root/.ssh/id_rsa.pub root@192.168.60.134
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/root/.ssh/id_rsa.pub"
The authenticity of host '192.168.60.134 (192.168.60.134)' can't be established.
ECDSA key fingerprint is SHA256:gb1k/wlKIeRuRh79sGI26DxKRAzM+uu52mVRMOCVGRc.
ECDSA key fingerprint is MD5:5d:80:79:e6:fe:3b:8d:10:47:cc:85:5b:79:af:69:29.
Are you sure you want to continue connecting (yes/no)? yes
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
root@192.168.60.134's password:

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'root@192.168.60.134'"
and check to make sure that only the key(s) you wanted were added.


[root@master01 ~]# ssh-copy-id -i /root/.ssh/id_rsa.pub root@192.168.60.135
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/root/.ssh/id_rsa.pub"
The authenticity of host '192.168.60.135 (192.168.60.135)' can't be established.
ECDSA key fingerprint is SHA256:yWCykmYj1RPf5XMyQDwVPMYbMWwUJceME83vXjiIMII.
ECDSA key fingerprint is MD5:a9:7a:4a:f5:70:1c:b0:47:ad:a5:3f:30:62:ee:c6:d1.
Are you sure you want to continue connecting (yes/no)? yes
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
root@192.168.60.135's password:

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'root@192.168.60.135'"
and check to make sure that only the key(s) you wanted were added.

```

##### 2.3	免密登录测试

```shell
[root@master01 /]# ssh node01
The authenticity of host 'node01 (192.168.60.134)' can't be established.
ECDSA key fingerprint is SHA256:nHfATQ3qdPDhXIYUolQZ08ln0NHNJfr1ul7qwQM3DvU.
ECDSA key fingerprint is MD5:5a:18:ac:bd:40:c0:a6:ce:66:1b:0b:c3:64:a9:30:db.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added 'node01' (ECDSA) to the list of known hosts.
Last login: Fri Jun 18 01:11:31 2021 from 192.168.60.1
[root@node01 ~]# exit
logout
Connection to node01 closed.
```

master01可以直接登录node01、node02，不需要输入密码。



### 二、	搭建Kubernates集群环境

#### 1.	Docker安装

> control plane和work节点都执行本部分操作

#### 2.	K8s安装

##### 2.1	配置k8s源

> 这里为了速度，设置为阿里源，需要官方源可在官网查询 [链接](https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/install-kubeadm)

```shell
[root@master01 /]#  cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

#	更新缓存
[root@master01 ~]# yum clean all
[root@master01 ~]# yum -y makecache
```

##### 安装kubelet、kubeadm和kubectl

>你需要在每台机器上安装以下的软件包：
>
>- `kubeadm`：用来初始化集群的指令。
>- `kubelet`：在集群中的每个节点上用来启动 Pod 和容器等。
>- `kubectl`：用来与集群通信的命令行工具。
>
>kubeadm **不能** 帮你安装或者管理 `kubelet` 或 `kubectl`，所以你需要 确保它们与通过 kubeadm 安装的控制平面的版本相匹配。 如果不这样做，则存在发生版本偏差的风险，可能会导致一些预料之外的错误和问题。 然而，控制平面与 kubelet 间的相差一个次要版本不一致是支持的，但 kubelet 的版本不可以超过 API 服务器的版本。 例如，1.7.0 版本的 kubelet 可以完全兼容 1.8.0 版本的 API 服务器，反之则不可以。

```shell
# yum安装k8s工具
yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
# 或指定版本安装
yum install -y kubelet-1.19.2 kubeadm-1.19.2 kubectl-1.19.2  --disableexcludes=kubernetes
```

##### 2.2	启动kubelet并设置开启自启动

```shell
[root@master01 ~]# systemctl enable --now kubelet
```

##### 2.3	设置kubelet命令补全

```shell
[root@master01 ~]# yum install -y bash-completion 
source /usr/share/bash-completion/bash_completion
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

#### 3.	创建K8s集群

- #### Master端

  **拉取镜像**

```shell
#	默认为google镜像，这里采用速度快的阿里镜像
[root@master01 /]# vim image.sh 
#!/bin/bash
url=registry.aliyuncs.com/google_containers
version=v1.19.2
images=(`kubeadm config images list --kubernetes-version=$version|awk -F '/' '{print $2}'`)
for imagename in ${images[@]} ; do
  docker pull $url/$imagename
  docker tag $url/$imagename k8s.gcr.io/$imagename
  docker rmi -f $url/$imagename
done
#	运行脚本文件
[root@master01 /]# sh image.sh
[root@master01 /]# docker images
REPOSITORY                           TAG                 IMAGE ID            CREATED             SIZE
hello-world                          latest              d1165f221234        3 months ago        13.3kB
k8s.gcr.io/kube-proxy                v1.19.2             d373dd5a8593        9 months ago        118MB
k8s.gcr.io/kube-apiserver            v1.19.2             607331163122        9 months ago        119MB
k8s.gcr.io/kube-controller-manager   v1.19.2             8603821e1a7a        9 months ago        111MB
k8s.gcr.io/kube-scheduler            v1.19.2             2f32d66b884f        9 months ago        45.7MB
k8s.gcr.io/etcd                      3.4.13-0            0369cf4303ff        9 months ago        253MB
k8s.gcr.io/coredns                   1.7.0               bfe3a36ebd25        12 months ago       45.2MB
k8s.gcr.io/pause 
```

##### 3.1	初始化Master节点

```shell
[root@master01 /]# kubeadm init --apiserver-advertise-address=192.168.60.128 --kubernetes-version v1.19.2 --service-cidr=10.1.0.0/16 --pod-network-cidr=10.244.0.0/16

W0618 11:08:09.040687   46771 configset.go:348] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]
[init] Using Kubernetes version: v1.19.2
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local master01] and IPs [10.1.0.1 192.168.60.128]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [localhost master01] and IPs [192.168.60.128 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [localhost master01] and IPs [192.168.60.128 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 16.041657 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.19" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node master01 as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node master01 as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: qn63oc.jgql71fc6p7tu4bi
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:
#记住下面命令，用于纳管node节点
kubeadm join 192.168.60.128:6443 --token qn63oc.jgql71fc6p7tu4bi \
    --discovery-token-ca-cert-hash sha256:96a72722626fcc40bae0ab1b6bda78065d6a765cf77ce67e5f8bd07b2bd8b992
```

​	如果初始化失败，需要reset后再执行初始化

```shell
[root@master01 ~]# kubeadm reset
[root@master01 ~]# rm -rf $HOME/.kube/config
```

##### 3.2	加载环境变量

```shell
[root@master01 ~]# echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bash_profile
[root@master01 ~]# source .bash_profile
```

##### 3.3	安装flannel网络

>国内无法访问https://raw.githubusercontent.com 改为其他方式获得kube-flannel.yml

```shell
#	在本地生成kube-flannel.yml 在同级目录下执行
[root@master01 opt]# kubectl apply -f kube-flannel.yml
podsecuritypolicy.policy/psp.flannel.unprivileged created
Warning: rbac.authorization.k8s.io/v1beta1 ClusterRole is deprecated in v1.17+, unavailable in v1.22+; use rbac.authorization.k8s.io/v1 ClusterRole
clusterrole.rbac.authorization.k8s.io/flannel created
Warning: rbac.authorization.k8s.io/v1beta1 ClusterRoleBinding is deprecated in v1.17+, unavailable in v1.22+; use rbac.authorization.k8s.io/v1 ClusterRoleBinding
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.apps/kube-flannel-ds-amd64 created
daemonset.apps/kube-flannel-ds-arm64 created
daemonset.apps/kube-flannel-ds-arm created
daemonset.apps/kube-flannel-ds-ppc64le created
daemonset.apps/kube-flannel-ds-s390x created
```

##### 3.4	查看Master是否部署成功

```shell
[root@master01 opt]# kubectl get pods -n kube-system
NAME                               READY   STATUS    RESTARTS   AGE
coredns-f9fd979d6-kh2fc            1/1     Running   0          37m
coredns-f9fd979d6-lrc27            1/1     Running   0          37m
etcd-master01                      1/1     Running   0          37m
kube-apiserver-master01            1/1     Running   0          37m
kube-controller-manager-master01   1/1     Running   0          37m
kube-flannel-ds-amd64-ppp6k        1/1     Running   0          116s
kube-proxy-slfk9                   1/1     Running   0          37m
kube-scheduler-master01            1/1     Running   0          37m
[root@master01 opt]# kubectl get node
NAME       STATUS   ROLES    AGE   VERSION
master01   Ready    master   37m   v1.19.2
```

##### 3.5	纳管node节点

```shell
#	由于国内网络有墙，会导致部分镜像拉取失败，因此手动拉取镜像
#安装镜像
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.19.2
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.2

#修改镜像tag
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.19.2  k8s.gcr.io/kube-proxy:v1.19.2
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.2  k8s.gcr.io/pause:3.2

#删除旧的镜像
docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.19.2
docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.2
```



```shell
#	在需要纳管的节点上输入之前生成的join口令
kubeadm join 192.168.60.128:6443 --token qn63oc.jgql71fc6p7tu4bi \
    --discovery-token-ca-cert-hash sha256:96a72722626fcc40bae0ab1b6bda78065d6a765cf77ce67e5f8bd07b2bd8b992
```

##### 3.6	检查集群状态

```shell
检查下集群的状态：
[root@master01 manifests]# kubectl get cs
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS    MESSAGE             ERROR
etcd-0               Healthy   {"health":"true"}
controller-manager   Healthy   ok
scheduler            Healthy   ok
[root@master01 manifests]# kubectl get node
NAME       STATUS   ROLES    AGE   VERSION
master01   Ready    master   82m   v1.19.2
node01     Ready    <none>   30m   v1.19.2
node02     Ready    <none>   30m   v1.19.2
[root@master01 manifests]# kubectl get pod  --all-namespaces
NAMESPACE     NAME                               READY   STATUS    RESTARTS   AGE
kube-system   coredns-f9fd979d6-kh2fc            1/1     Running   0          82m
kube-system   coredns-f9fd979d6-lrc27            1/1     Running   0          82m
kube-system   etcd-master01                      1/1     Running   0          82m
kube-system   kube-apiserver-master01            1/1     Running   0          82m
kube-system   kube-controller-manager-master01   1/1     Running   0          4m55s
kube-system   kube-flannel-ds-amd64-ppp6k        1/1     Running   0          46m
kube-system   kube-flannel-ds-amd64-xq9jb        1/1     Running   1          31m
kube-system   kube-flannel-ds-amd64-zfdxg        1/1     Running   0          31m
kube-system   kube-proxy-5dg4c                   1/1     Running   0          31m
kube-system   kube-proxy-lzjzc                   1/1     Running   0          31m
kube-system   kube-proxy-slfk9                   1/1     Running   0          82m
kube-system   kube-scheduler-master01            1/1     Running   0          3m56s 
```

##### 3.7	问题处理

**问题1：**执行完kubeadm join后在master节点执行kubectl get nodes 发现node节点状态为NotReady

**解决方法：**在node节点查看日志，或者journalctl -f -u kubelet发现有报错日志如下

```shell
Container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized
#通过日志可以看出是网络组件出现的问题，然后在node节点上执行docker images，查看到flannel组件的镜像没有拉取到，如没有可用手动拉取
```

**问题2：**在我们正常安装kubernetes1.19.2之后，可能会出现以下错误

```shell
[root@k8s-master manifests]# kubectl get cs
NAME                 STATUS      MESSAGE                                                                                     ERROR
scheduler            Unhealthy   Get http://127.0.0.1:10251/healthz: dial tcp 127.0.0.1:10251: connect: connection refused
controller-manager   Unhealthy   Get http://127.0.0.1:10252/healthz: dial tcp 127.0.0.1:10252: connect: connection refused
etcd-0               Healthy     {"health":"true"}
```

**解决方法：**出现这种情况，是/etc/kubernetes/manifests下的kube-controller-manager.yaml和kube-scheduler.yaml设置的默认端口是0，在文件中注释掉就可以了。

kube-controller-manager.yaml文件修改：注释掉27行

```shell
apiVersion: v1
  2 kind: Pod
  3 metadata:
  4   creationTimestamp: null
  5   labels:
  6     component: kube-controller-manager
  7     tier: control-plane
  8   name: kube-controller-manager
  9   namespace: kube-system
 10 spec:
 11   containers:
 12   - command:
 13     - kube-controller-manager
 14     - --allocate-node-cidrs=true
 15     - --authentication-kubeconfig=/etc/kubernetes/controller-manager.conf
 16     - --authorization-kubeconfig=/etc/kubernetes/controller-manager.conf
 17     - --bind-address=127.0.0.1
 18     - --client-ca-file=/etc/kubernetes/pki/ca.crt
 19     - --cluster-cidr=10.244.0.0/16
 20     - --cluster-name=kubernetes
 21     - --cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt
 22     - --cluster-signing-key-file=/etc/kubernetes/pki/ca.key
 23     - --controllers=*,bootstrapsigner,tokencleaner
 24     - --kubeconfig=/etc/kubernetes/controller-manager.conf
 25     - --leader-elect=true
 26     - --node-cidr-mask-size=24
 27   #  - --port=0
 28     - --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
 29     - --root-ca-file=/etc/kubernetes/pki/ca.crt
 30     - --service-account-private-key-file=/etc/kubernetes/pki/sa.key
 31     - --service-cluster-ip-range=10.1.0.0/16
 32     - --use-service-account-credentials=true
```

kube-scheduler.yaml配置修改：注释掉19行

```shell
1 apiVersion: v1
  2 kind: Pod
  3 metadata:
  4   creationTimestamp: null
  5   labels:
  6     component: kube-scheduler
  7     tier: control-plane
  8   name: kube-scheduler
  9   namespace: kube-system
 10 spec:
 11   containers:
 12   - command:
 13     - kube-scheduler
 14     - --authentication-kubeconfig=/etc/kubernetes/scheduler.conf
 15     - --authorization-kubeconfig=/etc/kubernetes/scheduler.conf
 16     - --bind-address=127.0.0.1
 17     - --kubeconfig=/etc/kubernetes/scheduler.conf
 18     - --leader-elect=true
 19   #  - --port=0
```

然后三台机器均重启kubelet即可

```shell
[root@master01]# systemctl restart kubelet.service
```

##### 3.8	部署Dashboard

>如果国内无法访问raw.githubusercontent.com，需要在hosts里配置一下。

```shell
[root@master01 opt]# vim /etc/hosts
#在最后一行添加（添加前先ping下IP看是不是通的）
199.232.96.133     raw.githubusercontent.com
```

>注意版本配套关系，这里我们的K8s采用的是1.19.2版本，在https://github.com/kubernetes/dashboard/releases，上可查到配套关系为v2.0.4。 
>
>| Kubernetes version | 1.16 | 1.17 | 1.18 | 1.19 |
>| ------------------ | ---- | ---- | ---- | ---- |
>| Compatibility      | ?    | ?    | ?    | ✓    |

```shell
#下载recommended.yaml 配置文件
[root@master01]# wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.4/aio/deploy/recommended.yaml

#DashBoard默认为ClusterIP访问，为了方便，改为通过NodeIP浏览器访问。修改recommended.yaml中Service部分
-----------------------------------
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  type: NodePort
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 30443
  selector:
    k8s-app: kubernetes-dashboard
-----------------------------------
#启动Dashboard
[root@master01]# kubectl apply -f recommended.yaml

#这里通过token方式登录，我们先创建管理员账号
[root@master01 opt]# cat > dashboard-adminuser.yaml << EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
EOF
[root@master01 opt]# kubectl apply -f dashboard-adminuser.yaml
serviceaccount/admin-user created
clusterrolebinding.rbac.authorization.k8s.io/admin-user created
#获取登录token
[root@master01 opt]# kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
Name:         admin-user-token-w27xh
Namespace:    kubernetes-dashboard
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: admin-user
              kubernetes.io/service-account.uid: c1e55fd6-a07c-4920-9ca6-a359db8994de

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1066 bytes
namespace:  20 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6InFyX0QwcEQxMTVFVU4zaDk5c0xTYVoxZkJRQnBNUy1OSUxwR3kwNmFnaDgifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLXcyN3hoIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiJjMWU1NWZkNi1hMDdjLTQ5MjAtOWNhNi1hMzU5ZGI4OTk0ZGUiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZXJuZXRlcy1kYXNoYm9hcmQ6YWRtaW4tdXNlciJ9.nGEpnZmSQoUTqdLE9NWVIumVxqRh-AFkoLQPLUzWyHQwMPhc1XrTyW8CEDSSqWWgFWOihKEFmPY5WtNjew-p3RZheT2Gj2nBmj84Bzv7FD0JgYw8WLw_Uvjt_c90tq18qwqe9KAT-ameqthdIEVxEAcYIywuMujZrDeR346HLo2vvXSrZHe8ELlwJdyg_VoSlzRc9f15Qt5h8EvyBULdYx5ssSqeJXXWVDhysxl99IBNkrFmU_CqOA42fydVn_S5p1cJDWgAw9YcQ8ub7D3g-98uw6RR2hjDpV8Q-ciUiH321tI1xvuvJX2DC_nSvB85b52r03tzosydfNe5hTmWbQ
```

输入https://AnyNodeIp:30443 输入token登录即可

![](https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/37f6d6a87a27477236dd925641ee3130_1073x442.png@900-0-90-f.png)

### 三、	部署应用

### 四、	常用K8s命令

---

> 常用参数：
>
> -n	：指定命名空间，缺省值为default
>
> -A	：查看所有资源

#### 查看资源

##### 查看有哪些对象在命名空间里面

```shell
[root@master01 ~]# kubectl api-resources --namespaced=true
NAME                        SHORTNAMES   APIGROUP                    NAMESPACED   KIND
bindings                                                             true         Binding
configmaps                  cm                                       true         ConfigMap
endpoints                   ep                                       true         Endpoints
events                      ev                                       true         Event
limitranges                 limits                                   true         LimitRange
persistentvolumeclaims      pvc                                      true         PersistentVolumeClaim
pods                        po                                       true         Pod
podtemplates                                                         true         PodTemplate
replicationcontrollers      rc                                       true         ReplicationController
resourcequotas              quota                                    true         ResourceQuota
......
```

---

##### 查看命名空间下游哪些资源

```shell
kubectl api-resources --verbs=list --namespaced -o name | xargs -n 1 kubectl get --show-kind --ignore-not-found -n projectName
```



##### 查看有哪些命名空间（namespace）

```shell
[root@master01 ~]# kubectl get namespace
NAME                   STATUS   AGE
default                Active   15h
kube-node-lease        Active   15h
kube-public            Active   15h
kube-system            Active   15h
kubernetes-dashboard   Active   3h46m
```

##### 通过describe即查看所有细节

```shell
[root@master01 ~]# kubectl describe statefulset/registry -n projectName
Name:               registry
Namespace:          projectName
CreationTimestamp:  Mon, 28 Jun 2021 11:08:37 -0400
Selector:           app=registry
Labels:             <none>
Annotations:        <none>
Replicas:           3 desired | 3 total
Update Strategy:    RollingUpdate
  Partition:        0
Pods Status:        3 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  app=registry
  Containers:
   registry:
    Image:      192.168.60.128:5000/registry:1.0.0
    Port:       8022/TCP
    Host Port:  0/TCP
    Environment:
      MY_POD_NAME:                (v1:metadata.name)
      EUREKA_SERVER:             http://registry-0.registry-service:8022/eureka,http://eureka-1.eureka-headless:8761/eureka
      EUREKA_INSTANCE_HOSTNAME:  ${MY_POD_NAME}.eureka-headless
    Mounts:                      <none>
  Volumes:                       <none>
Volume Claims:                   <none>
Events:                          <none>
```

##### 追踪部署情况

```shell
[root@master01 ~]# kubectl rollout status statefulset/registry -n projectNamepartitioned roll out complete: 3 new pods have been updated...
```

##### 查看statefulset

```shell
[root@master01 ~]# kubectl get statefulsets -n projectNameNAME                 READY   AGEregistry   3/3     33h
```

##### 查看service

```shell
[root@master01 ~]# kubectl get svc -ANAMESPACE              NAME                        TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)                  AGEdefault                kubernetes                  ClusterIP   10.1.0.1      <none>        443/TCP                  11dkube-system            kube-dns                    ClusterIP   10.1.0.10     <none>        53/UDP,53/TCP,9153/TCP   11dkubernetes-dashboard   dashboard-metrics-scraper   ClusterIP   10.1.159.53   <none>        8000/TCP                 10dkubernetes-dashboard   kubernetes-dashboard        NodePort    10.1.241.91   <none>        443:30306/TCP            10dprojectName              registry-service            ClusterIP   None          <none>        8022/TCP                 33h
```

##### 查看所有POD信息

```shell
[root@master01 ~]# kubectl get pods -A -o wideNAMESPACE              NAME                                         READY   STATUS    RESTARTS   AGE   IP               NODE       NOMINATED NODE   READINESS GATESkube-system            coredns-f9fd979d6-kh2fc                      1/1     Running   2          11d   10.244.0.7       master01   <none>           <none>kube-system            coredns-f9fd979d6-lrc27                      1/1     Running   2          11d   10.244.0.6       master01   <none>           <none>kube-system            etcd-master01                                1/1     Running   4          11d   192.168.60.128   master01   <none>           <none>kube-system            kube-apiserver-master01                      1/1     Running   4          11d   192.168.60.128   master01   <none>           <none>kube-system            kube-controller-manager-master01             1/1     Running   19         10d   192.168.60.128   master01   <none>           <none>kube-system            kube-flannel-ds-amd64-ppp6k                  1/1     Running   3          10d   192.168.60.128   master01   <none>           <none>kube-system            kube-flannel-ds-amd64-xq9jb                  1/1     Running   2          10d   192.168.60.130   node02     <none>           <none>kube-system            kube-flannel-ds-amd64-zfdxg                  1/1     Running   1          10d   192.168.60.129   node01     <none>           <none>kube-system            kube-proxy-5dg4c                             1/1     Running   2          10d   192.168.60.130   node02     <none>           <none>kube-system            kube-proxy-lzjzc                             1/1     Running   2          10d   192.168.60.129   node01     <none>           <none>kube-system            kube-proxy-slfk9                             1/1     Running   3          11d   192.168.60.128   master01   <none>           <none>kube-system            kube-scheduler-master01                      1/1     Running   24         10d   192.168.60.128   master01   <none>           <none>kubernetes-dashboard   dashboard-metrics-scraper-7b59f7d4df-cslj5   1/1     Running   2          10d   10.244.2.4       node02     <none>           <none>kubernetes-dashboard   kubernetes-dashboard-665f4c5ff-wc2t6         1/1     Running   25         10d   10.244.1.3       node01     <none>           <none>projectName              registry-0                         1/1     Running   0          15h   10.244.1.12      node01     <none>           <none>projectName              registry-1                         1/1     Running   0          15h   10.244.1.11      node01     <none>           <none>projectName              registry-2                         1/1     Running   3          15h   10.244.2.7       node02     <none>           <none>
```

#### 操作资源

##### 部署service

服务的暴露需要Service，它是Pod的抽象代理

```
apiVersion: v1
kind: Service
metadata:
 name: nginx-service
spec:
 type: NodePort
 sessionAffinity: ClientIP
 selector:
 app: nginx
 ports:
 - port: 80
 nodePort: 30080
```

- kind：Service代表是一个服务
- type：NodePort k8s将会在每个Node上打开一个端口并且每个Node的端口都是一样的，通过<NodeIP>:NodePort的方式Kubernetes集群外部的程序可以访问Service。
- selector：哪个服务需要暴露
- port：service暴露的端口
- TargetPort：pod的端口
- nodePort：对外暴露的端口，不设置会默认分配，范围：30000－32767
- 转发逻辑是：
  <NodeIP>:<nodeport> => <ServiceVIP>:<port>=> <PodIP>:<targetport>

部署service服务：

```
[root@master yaml]# kubectl create -f nginx-service.yaml 
service "nginx-service" created
```

可看到启动了一个svc

```
NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
svc/kubernetes ClusterIP 10.96.0.1 <none> 443/TCP 1d
```

##### 进入容器内部

```shell
# -n 命名空间
[root@master01 ~]# kubectl exec -it registry-1 -n projectName /bin/bash
```

##### 进入crash容器进行Debug

在 [How to debug CrashLoopBackOff](https://snippets.aktagon.com/snippets/871-how-to-debug-crashloopbackoff) 找到了解决方法：修改 pod 部署配置文件，将容器启动入口命令修改为 sleep 命令

```yml
spec:
    containers:
    - image: yyy/xxx:1.0.0
    name: xxx-service
    ...
    command:
        - "sh"
        - "-c"
        - "sleep 10000"
```

##### 更新Deployment

假设我们想把nginx从1.7.9更新到1.9.1

###### 方式1：直接set命令设置变更的部分

```shell
[root@master01 ~]# kubectl set image deployment/nginx-deployment nginx=nginx:1.9.1
deployment "nginx-deployment" image updated
```

以上命令会自动回滚更改Pods，即停止一定量的老的，新建新的，直到来的终止完，新的启动完

###### 方式2：通过直接修改线上的配置修改

```shell
[root@master01 ~]# kubectl edit deployment/nginx-deployment
```

会打开一个编辑器，修改指定的部分即可，这里是.spec.template.spec.containers[0].image

###### 方式3：修改yaml文件，通过apply重新部署

```shell
[root@master yaml]# kubectl apply -f nginx-deployment.yaml
```

##### 回滚Deployment

有时需要回滚的操作，比如更新错误，手误等一系列问题

比如上面的操作更新了一个不存在或者错误的版本1.0.1时

```
[root@master yaml]# kubectl set image statefulset/registry -n projectName app=registry:1.0.1statefulset "registry" image updated
```

追踪状态

```
[root@master01 ~]# kubectl rollout status statefulset/registry -n projectNameWaiting for 1 pods to be ready...
```

可见卡住不动了， Ctrl+C终止，查看rs如下

```
[root@master01 ~]# kubectl get statefulsets -ANAMESPACE   NAME                 READY   AGEprojectName   registry   2/3     33h
```

新的rs只启动了Pod但没有处于READY状态

查看Pods

```
[root@master01 ~]# kubectl get pods -ANAMESPACE              NAME                                         READY   STATUS             RESTARTS   AGE...projectName              registry-0                         1/1     Running            1          24hprojectName              registry-1                         1/1     Running            1          24hprojectName              registry-2                         0/1     ImagePullBackOff   0          49s
```

可发现ImagePullBackOff，实际就是镜像不存在

要修复这个，我们就需要rollback到前一个ok的版本

查看操作历史

```
[root@master01 ~]# kubectl rollout history statefulset/registry -n projectName
statefulset.apps/registry
REVISION
1
2
```

要查看每个版本的详细情况，指定--revision

```
[root@master yaml]# kubectl rollout history deployment/nginx-deployment --revision=2
deployments "nginx-deployment" with revision #2
Pod Template:
 Labels: app=nginx
 pod-template-hash=2710681425
 Annotations: kubernetes.io/change-cause=kubectl edit deployment/nginx-deployment
 Containers:
 nginx:
 Image: nginx:1.9.1
 Port: 80/TCP
 Environment: <none>
 Mounts: <none>
 Volumes: <none>
```

接下来进行回滚的操作

不指定版本，默认回滚到上一个版本

```
[root@master01 ~]# kubectl rollout undo statefulset/registry -n projectName
statefulset.apps/registry rolled back
```

指定版本，通过--to-revision指定

```
[root@master01 ~]# kubectl rollout undo statefulset/registry -n projectName --to-revision=2
statefulset.apps/registry rolled back
```

查看

```
kubectl describe deployment/nginx-deployment
....略
Events:
 FirstSeen LastSeen Count From SubobjectPath Type Reason Message
 --------- -------- ----- ---- ------------- -------- ------ -------

 30m 30m 1 {deployment-controller } Normal ScalingReplicaSet Scaled up replica set nginx-deployment-2035384211 to 3
 29m 29m 1 {deployment-controller } Normal ScalingReplicaSet Scaled up replica set nginx-deployment-1564180365 to 1
 29m 29m 1 {deployment-controller } Normal ScalingReplicaSet Scaled down replica set nginx-deployment-2035384211 to 2
 29m 29m 1 {deployment-controller } Normal ScalingReplicaSet Scaled up replica set nginx-deployment-1564180365 to 2
 29m 29m 1 {deployment-controller } Normal ScalingReplicaSet Scaled down replica set nginx-deployment-2035384211 to 0
 29m 29m 1 {deployment-controller } Normal ScalingReplicaSet Scaled up replica set nginx-deployment-3066724191 to 2
 29m 29m 1 {deployment-controller } Normal ScalingReplicaSet Scaled up replica set nginx-deployment-3066724191 to 1
 29m 29m 1 {deployment-controller } Normal ScalingReplicaSet Scaled down replica set nginx-deployment-1564180365 to 2
 2m 2m 1 {deployment-controller } Normal ScalingReplicaSet Scaled down replica set nginx-deployment-3066724191 to 0
 2m 2m 1 {deployment-controller } Normal DeploymentRollback Rolled back deployment "nginx-deployment" to revision 2
 29m 2m 2 {deployment-controller } Normal ScalingReplicaSet Scaled up replica set nginx-deployment-1564180365 to 3
```

可看到有DeploymentRollback Reason的事件

##### 删除Deployment(这里是Statefulset)

```shell
[root@master01 k8s-projectName]# kubectl delete -f registry-sts.yml
namespace "projectName" deleted
statefulset.apps "registry" deleted
service "registry-service" deleted
```



> 至此，一个完整的K8s本地学习测试环境就搭建好了，如果细心一点会发现，我们的集群还面临高可用性、性能调优、日志挂载等问题，这个我们留下一个疑问，在后面的博文中在深入探讨。下面我们将进行微服务的改造、部署。

## 3.	如何发布应用

### 3.1	构建镜像

容器化需要对我们的部署包进行改造，以前我们使用maven打成jar包 java -jar或 war包放在tomcat的webapps中来方式来部署我们的应用。现在我们需要把我们的应用层、所有的配置信息、日志的挂载卷统统打包成一个docker镜像，这个镜像包可以运行在任何一个docker运行时之上。其中具体的操作步骤就不在此细谈，我们放到另外一个帖子中。

### 3.2	配置文件

#### 	3.2.1	命名空间

k8s 的隔离机制是使用命名空间进行隔离，不同的命名空间网络是不通的。创建一个命名空间，把应用相关的所有资源放在一个命名空间下。

```yaml
#Namespace
apiVersion: v1
kind: Namespace
metadata:
   name: namespace
   labels:
     name: namespace
```

#### 	3.2.2	MicroService

**StatefulSet**

```yaml
# 有状态pod，会自动根据metadata.name及实例数量生产xxx-0,xxx-1的pod
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: registry
  namespace: projectName
spec:
  serviceName: eureka-headless
  replicas: 3
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
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8022
          env: # 环境变量设置
            - name: MY_POD_NAME 
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name # 取值metadata.name，这里注意，statefulset类型的对象取到的是有-0,-1序号后缀的
            - name: EUREKA_SERVER #环境变量，这个变量在我们项目的配置文件中有配置,作用是配置eureka集群的所有地址，注意相同namespace底下使用dns访问时，不需要配置全路径（全路径为podname.headless-name.namespace.cluster.svc.local),只要到podname.headless-name
              value: "http://eureka-0.eureka-headless:8761/eureka,http://eureka-1.eureka-headless:8761/eureka"
            - name: EUREKA_INSTANCE_HOSTNAME #环境变量，这个变量在我们项目的配置文件中有配置，作用是指定注册到eureka集群中的hostname
              value: ${MY_POD_NAME}.eureka-headless
  podManagementPolicy: "Parallel" # 以并行方式创建pod，默认是串行的
---
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

**进入容器内检查**

```shell
[root@master01 ~]# kubectl exec -it registry-0 -n projectName /bin/bash
[registry@registry-0 /]$ hostname
registry-0
[registry@registry-0 /]$ cat /etc/resolv.conf
nameserver 10.1.0.10
search projectName.svc.cluster.local svc.cluster.local cluster.local localdomain
options ndots:5
#检查环境变量是否生效
[registry@registry-0 /]$ printenv
...
INSTALL_HOME=/opt/docker_install
HOSTNAME=registry-0
EUREKA_SERVER=http://registry-0.registry-service:8022/eureka,http://eureka-1.eureka-headless:8761/eureka
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT=tcp://10.1.0.1:443
TERM=xterm
KUBERNETES_SERVICE_PORT=443
KUBERNETES_SERVICE_HOST=10.1.0.1
JRE_HOME=/opt/JRE/jre1.8.0_232
SCRIPT_NAME=install_projectName_registry.sh
EUREKA_INSTANCE_HOSTNAME=${MY_POD_NAME}.eureka-headless
TOMCATPATH=/data01/registry/tomcat
PATH=/opt/JRE/jre1.8.0_232/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
MY_POD_NAME=registry-0
PWD=/
JAVA_HOME=/opt/JRE/jre1.8.0_232
APP_HOME=/data01/registry
SHLVL=1
HOME=/data01/registry
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_SERVICE_PORT_HTTPS=443
CLASSPATH=:/opt/JRE/jre1.8.0_232/lib
REGISTRY_HOME=/data01/registry
KUBERNETES_PORT_443_TCP_ADDR=10.1.0.1
KUBERNETES_PORT_443_TCP=tcp://10.1.0.1:443
_=/usr/bin/printenv
...

```



​	**检查podDNS地址**

```shell
[root@master01 ~]# kubectl run -i --tty --image=busybox:1.27 -n projectName dns-test --restart=Never --rm
If you don't see a command prompt, try pressing enter.
/ # nslookup registry-service   #注意nslookup 后面跟的是服务名
Server:    10.1.0.10
Address 1: 10.1.0.10 kube-dns.kube-system.svc.cluster.local

Name:      registry-service
Address 1: 10.244.1.4 registry-1.registry-service.projectName.svc.cluster.local
Address 2: 10.244.2.6 registry-2.registry-service.projectName.svc.cluster.local
Address 3: 10.244.2.5 registry-0.registry-service.projectName.svc.cluster.local

#可以查看到各POD的DNS信息
```

### 3.3 模型组网设计

![](https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/image-20211104175811014.png)

## 4.模板文件

### 4.1 deployment.yaml文件详解（集群pod管理，自动伸缩相关）

```yaml
apiVersion: extensions/v1beta1   #接口版本   #查看api接口命令kubectl api-version
kind: Deployment                 #接口类型
metadata:
  name: cango-demo               #Deployment名称
  namespace: cango-prd           #命名空间
  labels:
    app: cango-demo              #标签
spec:
  replicas: 3
  strategy:
    rollingUpdate:  ##由于replicas为3,则整个升级,pod个数在2-4个之间
      maxSurge: 1      #滚动升级时会先启动1个pod
      maxUnavailable: 1 #滚动升级时允许的最大Unavailable的pod个数
  template:         
    metadata:
      labels:             # matchLabels一样匹配下面的pods标签
        app: cango-demo  #模板名称必填
    sepc: #定义容器模板，该模板可以包含多个容器
      containers:                                                                   
        - name: cango-demo                                                           #镜像名称
          image: swr.cn-east-2.myhuaweicloud.com/cango-prd/cango-demo:0.0.1-SNAPSHOT #镜像地址
          command: [ "/bin/sh","-c","cat /etc/config/path/to/special-key" ]    #启动命令
          args:                                                                #启动参数
            - '-storage.local.retention=$(STORAGE_RETENTION)'
            - '-storage.local.memory-chunks=$(STORAGE_MEMORY_CHUNKS)'
            - '-config.file=/etc/prometheus/prometheus.yml'
            - '-alertmanager.url=http://alertmanager:9093/alertmanager'
            - '-web.external-url=$(EXTERNAL_URL)'
    #如果command和args均没有写，那么用Docker默认的配置。
    #如果command写了，但args没有写，那么Docker默认的配置会被忽略而且仅仅执行.yaml文件的command（不带任何参数的）。
    #如果command没写，但args写了，那么Docker默认配置的ENTRYPOINT的命令行会被执行，但是调用的参数是.yaml中的args。
    #如果如果command和args都写了，那么Docker默认的配置被忽略，使用.yaml的配置。
          imagePullPolicy: IfNotPresent  #如果不存在则拉取
          livenessProbe:       #表示container是否处于live状态。如果LivenessProbe失败，LivenessProbe将会通知kubelet对应的container不健康了。随后kubelet将kill掉container，并根据RestarPolicy进行进一步的操作。默认情况下LivenessProbe在第一次检测之前初始化值为Success，如果container没有提供LivenessProbe，则也认为是Success；
            httpGet:
              path: /health #如果没有心跳检测接口就为/
              port: 8080
              scheme: HTTP
            initialDelaySeconds: 60 ##启动后延时多久开始运行检测
            timeoutSeconds: 5
            successThreshold: 1
            failureThreshold: 5
          readinessProbe:
            httpGet:
              path: /health #如果没有心跳检测接口就为/
              port: 8080
              scheme: HTTP
            initialDelaySeconds: 30 ##启动后延时多久开始运行检测
            timeoutSeconds: 5
            successThreshold: 1
            failureThreshold: 5
          resources:              ##CPU内存限制
            requests:             ##资源请求
              cpu: 2
              memory: 2048Mi
            limits:
              cpu: 2
              memory: 2048Mi
          env:                    ##通过环境变量的方式，直接传递pod=自定义Linux OS环境变量
            - name: LOCAL_KEY     #本地Key
              value: value
            - name: CONFIG_MAP_KEY  #局策略可使用configMap的配置Key，
              valueFrom:
                configMapKeyRef:
                  name: special-config   #configmap中找到name为special-config
                  key: special.type      #找到name为special-config里data下的key
          ports:
            - name: http
              containerPort: 8080 #对service暴露端口
          volumeMounts:     #挂载volumes中定义的磁盘
          - name: log-cache
            mount: /tmp/log
          - name: sdb       #普通用法，该卷跟随容器销毁，挂载一个目录
            mountPath: /data/media    
          - name: nfs-client-root    #直接挂载硬盘方法，如挂载下面的nfs目录到/mnt/nfs
            mountPath: /mnt/nfs
          - name: example-volume-config  #高级用法第1种，将ConfigMap的log-script,backup-script分别挂载到/etc/config目录下的一个相对路径path/to/...下，如果存在同名文件，直接覆盖。
            mountPath: /etc/config       
          - name: rbd-pvc                #高级用法第2中，挂载PVC(PresistentVolumeClaim)
 
#使用volume将ConfigMap作为文件或目录直接挂载，其中每一个key-value键值对都会生成一个文件，key为文件名，value为内容，
  volumes:  # 定义磁盘给上面volumeMounts挂载
  - name: log-cache
    emptyDir: {}
  - name: sdb  #挂载宿主机上面的目录
    hostPath:
      path: /any/path/it/will/be/replaced
  - name: example-volume-config  # 供ConfigMap文件内容到指定路径使用
    configMap:
      name: example-volume-config  #ConfigMap中名称
      items:
      - key: log-script           #ConfigMap中的Key
        path: path/to/log-script  #指定目录下的一个相对路径path/to/log-script
      - key: backup-script        #ConfigMap中的Key
        path: path/to/backup-script  #指定目录下的一个相对路径path/to/backup-script
  - name: nfs-client-root         #供挂载NFS存储类型
    nfs:
      server: 10.42.0.55          #NFS服务器地址
      path: /opt/public           #showmount -e 看一下路径
  - name: rbd-pvc                 #挂载PVC磁盘
    persistentVolumeClaim:
      claimName: rbd-pvc1         #挂载已经申请的pvc磁盘
      

```

### 4.2	Pod yaml文件详解（池子，容器服务运行在其中，资源隔离限制）

```yaml
# yaml格式的pod定义文件完整内容：
apiVersion: v1       #必选，版本号，例如v1
kind: Pod       #必选，Pod
metadata:       #必选，元数据
  name: string       #必选，Pod名称
  namespace: string    #必选，Pod所属的命名空间
  labels:      #自定义标签
    - name: string     #自定义标签名字
  annotations:       #自定义注释列表
    - name: string
spec:         #必选，Pod中容器的详细定义
  containers:      #必选，Pod中容器列表
  - name: string     #必选，容器名称
    image: string    #必选，容器的镜像名称
    imagePullPolicy: [Always | Never | IfNotPresent] #获取镜像的策略 Alawys表示下载镜像 IfnotPresent表示优先使用本地镜像，否则下载镜像，Nerver表示仅使用本地镜像
    command: [string]    #容器的启动命令列表，如不指定，使用打包时使用的启动命令
    args: [string]     #容器的启动命令参数列表
    workingDir: string     #容器的工作目录
    volumeMounts:    #挂载到容器内部的存储卷配置
    - name: string     #引用pod定义的共享存储卷的名称，需用volumes[]部分定义的的卷名
      mountPath: string    #存储卷在容器内mount的绝对路径，应少于512字符
      readOnly: boolean    #是否为只读模式
    ports:       #需要暴露的端口库号列表
    - name: string     #端口号名称
      containerPort: int   #容器需要监听的端口号
      hostPort: int    #容器所在主机需要监听的端口号，默认与Container相同
      protocol: string     #端口协议，支持TCP和UDP，默认TCP
    env:       #容器运行前需设置的环境变量列表
    - name: string     #环境变量名称
      value: string    #环境变量的值
    resources:       #资源限制和请求的设置
      limits:      #资源限制的设置
        cpu: string    #Cpu的限制，单位为core数，将用于docker run --cpu-shares参数
        memory: string     #内存限制，单位可以为Mib/Gib，将用于docker run --memory参数
      requests:      #资源请求的设置
        cpu: string    #Cpu请求，容器启动的初始可用数量
        memory: string     #内存清楚，容器启动的初始可用数量
    livenessProbe:     #对Pod内个容器健康检查的设置，当探测无响应几次后将自动重启该容器，检查方法有exec、httpGet和tcpSocket，对一个容器只需设置其中一种方法即可
      exec:      #对Pod容器内检查方式设置为exec方式
        command: [string]  #exec方式需要制定的命令或脚本
      httpGet:       #对Pod内个容器健康检查方法设置为HttpGet，需要制定Path、port
        path: string
        port: number
        host: string
        scheme: string
        HttpHeaders:
        - name: string
          value: string
      tcpSocket:     #对Pod内个容器健康检查方式设置为tcpSocket方式
         port: number
       initialDelaySeconds: 0  #容器启动完成后首次探测的时间，单位为秒
       timeoutSeconds: 0   #对容器健康检查探测等待响应的超时时间，单位秒，默认1秒
       periodSeconds: 0    #对容器监控检查的定期探测时间设置，单位秒，默认10秒一次
       successThreshold: 0
       failureThreshold: 0
       securityContext:
         privileged:false
    restartPolicy: [Always | Never | OnFailure]#Pod的重启策略，Always表示一旦不管以何种方式终止运行，kubelet都将重启，OnFailure表示只有Pod以非0退出码退出才重启，Nerver表示不再重启该Pod
    nodeSelector: obeject  #设置NodeSelector表示将该Pod调度到包含这个label的node上，以key：value的格式指定 #这一步需要先在node节点上先定义标签，最好唯一
    imagePullSecrets:    #Pull镜像时使用的secret名称，以key：secretkey格式指定
    - name: string
    hostNetwork:false      #是否使用主机网络模式，默认为false，如果设置为true，表示使用宿主机网络
    volumes:       #在该pod上定义共享存储卷列表
    - name: string     #共享存储卷名称 （volumes类型有很多种）
      emptyDir: {}     #类型为emtyDir的存储卷，与Pod同生命周期的一个临时目录。为空值
      hostPath: string     #类型为hostPath的存储卷，表示挂载Pod所在宿主机的目录
        path: string     #Pod所在宿主机的目录，将被用于同期中mount的目录
      secret:      #类型为secret的存储卷，挂载集群与定义的secre对象到容器内部
        scretname: string  
        items:     
        - key: string
          path: string
      configMap:     #类型为configMap的存储卷，挂载预定义的configMap对象到容器内部
        name: string
        items:
        - key: string
          path: string
```

### 4.3 Service yaml文件详解（真实运行在pod里的服务配置）

```yaml
apiVersion: v1
kind: Service
matadata:                                #元数据
  name: string                           #service的名称
  namespace: string                      #命名空间  
  labels:                                #自定义标签属性列表
    - name: string
  annotations:                           #自定义注解属性列表  
    - name: string
spec:                                    #详细描述
  selector: []                           #label selector配置，将选择具有label标签的Pod作为管理 
                                         #范围
  type: string                           #service的类型，指定service的访问方式，默认为 
                                         #clusterIp
  clusterIP: string                      #虚拟服务地址      
  sessionAffinity: string                #是否支持session
  ports:                                 #service需要暴露的端口列表
  - name: string                         #端口名称
    protocol: string                     #端口协议，支持TCP和UDP，默认TCP
    port: int                            #服务监听的端口号
    targetPort: int                      #需要转发到后端Pod的端口号
    nodePort: int                        #当type = NodePort时，指定映射到物理机的端口号
  status:                                #当spce.type=LoadBalancer时，设置外部负载均衡器的地址
    loadBalancer:                        #外部负载均衡器    
      ingress:                           #外部负载均衡器 
        ip: string                       #外部负载均衡器的Ip地址值
        hostname: string                 #外部负载均衡器的主机名
```



>引用资料如下
>
>[k8s使用篇](https://blog.csdn.net/weixin_34268753/article/details/89586977)
