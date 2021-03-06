# 基于Vagrant的CoreOS集群部署Kubernetes

Deploy Kubernetes on 3-node-cluster of CoreOS, which created by Vagrant 

## Kubernetes简介

[Kubernetes](http://kubernetes.io)是一个开源的系统，用于自动部署，弹性扩容以及管理容器化的应用，是[Docker](https://www.docker.com/)生态圈中重要一员。Kubernetes来源于Google的[Borg系统](https://research.google.com/pubs/pub43438.html)，经过了十多年的实践经验已经被证实是可以用于大数据量、高并发的实际生产环境中。其主要功能包括：

1. 使用Docker对应用程序包装(package)、实例化(instantiate)、运行(run)。
2. 以集群的方式运行、管理跨机器的容器。
3. 解决Docker跨机器容器之间的通讯问题。
4. Kubernetes的自我修复机制使得容器集群总是运行在用户期望的状态。

接下来将简单介绍以下几个概念，后续文章中会逐个解释，并最终完成运行一个深度学习的应用为目标，理解并实践Kubernetes。

### kubectl

与Kubernetes集群最简单的交互方式就是通过命令行kubectl命令，可以实现查看集群、服务、Pods状态，创建服务并部署新的应用等功能。

### Pod

Pod是Kubernetes的基本单元，也是由Kubernetes创建和管理的最小可部署单元。一个Pod由相关的一个或多个Container(s)构成，通常Pod里的容器运行相同的应用。Pod共享相同的volumes、*namespaces*、IP地址和port空间。

### Volumes

Volume是一个能够被Container访问的目录，用于解决：

1. 当Container崩溃时，Kubernetes会重启一个新的Container，但之上的的文件却无法被找回的问题；
2. 在Containers的运行过程中，会存在共享一些必要文件的需求。

### Service

Services也是Kubernetes的基本操作单元，是真实应用服务的抽象。因为Pod虽然拥有自己的IP，但是随着时间可能变化并不稳定。因此需要Service来维护Pod之间的联系，通过Label Selector决定服务请求传递给后端提供服务的Pod。Service对外表现为一个单一访问接口，外部不需要了解后端如何运行，这给扩展或维护后端带来很大的好处。

### Label

Labels是用于区分Pod、Service等对象的key/value键值对，Pod、Service可以有多个label，但是每个label的key只能对应一个value。Labels是Service和Replication Controller运行的基础，为了将访问Service的请求转发给后端提供服务的多个容器，正是通过标识容器的labels来选择正确的容器。同样，Replication Controller也使用labels来管理通过pod 模板创建的一组容器，这样Replication Controller可以更加容易，方便地管理多个容器，无论有多少容器。

## 部署Kubernetes

在上一篇文章中提到的已经建立的三台coreos机器集群安装并配置Kubernetes，其中1个结点作为master（172.17.8.101 core-01），并不参与schedule的任务调度，另外2个结点作为worker(172.17.8.102 core02, 172.17.8.103 core-03)，来创建并运行Pod。

如果是在物理机上装虚拟机搭建Kubernetes集群，一般需要在物理机上安装`kubectl`与其进行交互。而`kubectl`只有Linux和Mac OS X两种版本，因此对于在Windows系统环境下搭建的CoreOS集群的比较方便的做法是利用Vagrant再建立一台Linux的虚拟机，采用CentOS或者Ubuntu系统，作为client机器管理Kubernetes集群。如果是Linux和Mac OS X系统则可直接在宿主机上进行client机器上的操作。

### 创建client机器

用Vagrant创建一台client机器，本机使用`ubuntu/trusty64`镜像，步骤如下所示：

	$ mkdir client-ubuntu
	$ cd client-ubuntu
	$ vagrant init ubuntu/trusty64

修改`Vagrantfile`中，指定私有的网址，以便与集群之间数据交换。同时还可以指定默认的ssh登录账户，因为有些操作可能涉及到root用户的授权，`config.ssh`选项请参考[此处](https://www.vagrantup.com/docs/vagrantfile/ssh_settings.html)。

	Vagrant.configure("2") do |config|
	  config.vm.network "private_network", ip: "172.17.8.88"
	  # optional
	  #config.ssh.username = "root"
	  #config.ssh.password = "your password"
	  #config.ssh.insert_key = true
	end

然后启动该虚拟机

	vagrant up

进入虚拟机后运行新建一个目录，运行`kubernetes-generate-cas.sh`用OpenSSL生成Kubernetes运行安装所需要的[认证证书](https://coreos.com/kubernetes/docs/latest/openssl.html)，如

	$ mkdir certs
	$ sh kubernetes-generate-cas.sh

配置`/etc/hosts`与集群之间方便通信（可选）

	172.17.8.101 core-01
	172.17.8.102 core-02	172.17.8.103 core-03

### 配置master结点

首先进入主结点，在本例中，以core-01机器为主结点。前提是etcd集群能正常工作。因为在user-data文件中已经修改过discovery的token，所以默认情况下已经启动了3结点的etcd集群，可使用如下命令查看etcd集群各结点：

	$ etcdctl member list

待确认etcd服务正常后，为core用户修改密码，目的是方便与client机器进行数据传输（使用`scp`命令拷贝文件）。

	$ vagrant ssh core-01 -- -A
	$ sudo passwd core

配置`/etc/hosts`在集群之间方便通信（可选）

	172.17.8.101 core-01
	172.17.8.102 core-02	172.17.8.103 core-03

将`ca.pem`，`apiserver.pem`，`apiserver-key.pem`上传至用户目录，在该目录下运行`kubernetes-master-deploy.sh`

	$ sudo sh kubernetes-master-deploy.sh

等待后台进程下载，过程可能会需要几分钟到几个小时，然后确认`apiserver`进程是否启动

	$ curl http://127.0.0.1:8080/version

如果成功会返回如下形式的响应：

	{
	  "major": "1",
	  "minor": "1",
	  "gitVersion": "v1.1.7_coreos.2",
	  "gitCommit": "388061f00f0d9e4d641f9ed4971c775e1654579d",
	  "gitTreeState": "clean"
	}

现在我们可以创建`kube-system`命令空间：

	$ curl -H "Content-Type: application/json" -XPOST -d'{"apiVersion":"v1","kind":"Namespace","metadata":{"name":"kube-system"}}' "http://127.0.0.1:8080/api/v1/namespaces"

### 配置worker结点

首先进入每个worker结点，为core用户修改密码，才能scp文件

	$ vagrant ssh core-02 -- -A
	$ sudo passwd core

配置/etc/hosts在集群之间方便通信（可选）

	172.17.8.101 core-01
	172.17.8.102 core-02	172.17.8.103 core-03

将`ca.pem`，`${WORKER_FQDN}-worker.pem`，`${WORKER_FQDN}-worker-key.pem`上传至用户目录，**修改**并运行`kubernetes-worker-deploy.sh`。其中，`${WORKER_FQDN}`是该worker结点在集群中唯一的名称，在本例中`172.17.8.102`为`kube-worker-1`，`172.17.8.103`为`kube-worker-2`。

修改`${WORKER_FQDN}`以及`${ADVERTISE_IP}`，分别为结点在Kubernetes集群中的唯一名称和其对应的可访问的IP地址。

	$ sudo sh kubernetes-worker-deploy.sh

### 配置kubectl并验证

下载`kubectl`，授权并加入到`path`路径中。

	$ curl -O https://storage.googleapis.com/kubernetes-release/release/v1.2.4/bin/linux/amd64/kubectl
	$ chmod +x kubectl
	$ mv kubectl /usr/local/bin/kubectl

用下列命令配置kubectl以连接目标集群，根据说明替换其中关键变量：

替换`${MASTER_HOST}`为主结点的IP地址或者Host名称。

	$ kubectl config set-cluster default-cluster --server=https://${MASTER_HOST} --certificate-authority=ca.pem
	$ kubectl config set-credentials default-admin --certificate-authority=ca.pem --client-key=admin-key.pem --client-certificate=admin.pem
	$ kubectl config set-context default-system --cluster=default-cluster --user=default-admin
	$ kubectl config use-context default-system

在client机器上键入命令，可以查看Kubernetes结点状态

	$ kubectl get nodes

## 附录

### 调试技巧

可以在CoreOS系统中使用`journalctl`命令查看日志，根据日志的提示与报错再针对性搜索相关解决方案，该命令一般常用选项有：

	$ journalctl -xe

从未行开始浏览日志，`f`向前翻页，`b`向后翻页，`q`键退出。

	$ journalctl -u docker.service

查看特定单元(`unit`)，如`docker.service`产生的日志。

	$ journalctl -f

跟踪日志，按`ctrl` + `c`键退出。

### GFW issue

In China, you know. We can't pull images from `gci.io` and `qury.io`. So I pull the substitute from `docker.io` manually, and rename it using `docker tag` command. As shown in `kubernetes-master-deploy.sh` and `kubernetes-worker-deploy.sh`.

### 参考：

1. [Kubernetes - Volumes](http://kubernetes.io/docs/user-guide/volumes/)
2. [GOOGLE KUBERNETES设计文档之VOLUMES](http://www.sel.zju.edu.cn/?p=394&cpage=1)
3. [Kubernetes - Services](http://kubernetes.io/docs/user-guide/services/)
4. [CoreOS + Kubernetes Step By Step](https://coreos.com/kubernetes/docs/latest/getting-started.html)

