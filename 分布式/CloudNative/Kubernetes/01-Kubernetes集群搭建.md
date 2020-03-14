# 01 搭建K8s集群[无需科学上网]

> `官网`：<https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl>
>
> `GitHub`：https://github.com/kubernetes/kubeadm
>
> `课程中`：使用kubeadm搭建一个3台机器组成的k8s集群，1台master节点，2台worker节点
>
> **如果大家机器配置不够，也可以使用在线的，或者minikube的方式或者1个master和1个worker**
>
> `配置要求`：
>
> - One or more machines running one of:
>   - Ubuntu 16.04+
>   - Debian 9+
>   - CentOS 7【课程中使用】
>   - Red Hat Enterprise Linux (RHEL) 7
>   - Fedora 25+
>   - HypriotOS v1.0.1+
>   - Container Linux (tested with 1800.6.0)
> - 2 GB or more of RAM per machine (any less will leave little room for your apps)
> - 2 CPUs or more
> - Full network connectivity between all machines in the cluster (public or private network is fine)
> - Unique hostname, MAC address, and product_uuid for every node. See here for more details.
> - Certain ports are open on your machines. See here for more details.
> - Swap disabled. You **MUST** disable swap in order for the kubelet to work properly.

## 1.1 版本统一

```
Docker       18.09.0
---
kubeadm-1.14.0-0 
kubelet-1.14.0-0 
kubectl-1.14.0-0
---
k8s.gcr.io/kube-apiserver:v1.14.0
k8s.gcr.io/kube-controller-manager:v1.14.0
k8s.gcr.io/kube-scheduler:v1.14.0
k8s.gcr.io/kube-proxy:v1.14.0
k8s.gcr.io/pause:3.1
k8s.gcr.io/etcd:3.3.10
k8s.gcr.io/coredns:1.3.1
---
calico:v3.9
```

## 1.2 准备3台centos

大家根据自己的情况来准备centos7的虚拟机。

要保证彼此之间能够ping通，也就是处于同一个网络中，虚拟机的配置要求上面也描述咯。

Vagrantfile：

```yaml
boxes = [
    {
        :name => "master-kubeadm-k8s",
        :eth1 => "192.168.1.51",
        :mem => "2048",
        :cpu => "2"
    },
    {
        :name => "worker01-kubeadm-k8s",
        :eth1 => "192.168.1.52",
        :mem => "2048",
        :cpu => "2"
    },
    {
        :name => "worker02-kubeadm-k8s",
        :eth1 => "192.168.1.53",
        :mem => "2048",
        :cpu => "2"
    }
]

Vagrant.configure(2) do |config|

  config.vm.box = "centos/7"
  
   boxes.each do |opts|
      config.vm.define opts[:name] do |config|
        config.vm.hostname = opts[:name]
        config.vm.provider "vmware_fusion" do |v|
          v.vmx["memsize"] = opts[:mem]
          v.vmx["numvcpus"] = opts[:cpu]
        end

        config.vm.provider "virtualbox" do |v|
          v.customize ["modifyvm", :id, "--memory", opts[:mem]]
		  v.customize ["modifyvm", :id, "--cpus", opts[:cpu]]
		  v.customize ["modifyvm", :id, "--name", opts[:name]]
        end

        config.vm.network :public_network, ip: opts[:eth1]
      end
  end

end
```

### 安装centos7

```
01 创建centos7文件夹，并进入其中[目录全路径不要有中文字符]

02 在此目录下打开cmd，运行vagrant init centos/7
   此时会在当前目录下生成Vagrantfile，同时指定使用的镜像为centos/7，关键是这个镜像在哪里，我已经提前准备好了，名称是virtualbox.box文件
   链接:https://pan.baidu.com/s/1kkl7EKuGodjfKFwjgS3IsA  密码:sgzo
   
03 将virtualbox.box文件添加到vagrant管理的镜像中
	(1)下载网盘中的virtualbox.box文件
    (2)保存到磁盘的某个目录，比如D:\virtualbox.box
    (3)添加镜像并起名叫centos/7：vagrant box add centos/7 D:\virtualbox.box
    (4)vagrant box list  查看本地的box[这时候可以看到centos/7]
    
04 centos/7镜像有了，根据Vagrantfile文件启动创建虚拟机
	来到centos7文件夹，在此目录打开cmd窗口，执行vagrant up[打开virtual box观察，可以发现centos7创建成功]
	
05 以后大家操作虚拟机，还是要在centos文件夹打开cmd窗口操作
	vagrant halt   优雅关闭
	vagrant up     正常启动
	
06 vagrant常用命令
	(1)vagrant ssh    
    	进入刚才创建的centos7中
    (2)vagrant status
    	查看centos7的状态
    (3)vagrant halt
    	停止/关闭centos7
    (4)vagrant destroy
    	删除centos7
    (5)vagrant status
    	查看当前vagrant创建的虚拟机
    (6)Vagrantfile中也可以写脚本命令，使得centos7更加丰富
    	但是要注意，修改了Vagrantfile，要想使正常运行的centos7生效，必须使用vagrant reload
```

## 1.3 更新并安装依赖

> 3台机器都需要执行

```shell
yum -y update
yum install -y conntrack ipvsadm ipset jq sysstat curl iptables libseccomp
```

## 1.4 安装Docker

> 根据之前学习的Docker方式[Docker第一节课的笔记中也有这块的说明]
>
> 在每一台机器上都安装好Docker，版本为18.09.0
>
> ```shell
> 01 安装必要的依赖
> 	sudo yum install -y yum-utils \
>     device-mapper-persistent-data \
>     lvm2
>     
>     
> 02 设置docker仓库
> 	sudo yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
> 	
> 【设置要设置一下阿里云镜像加速器】
> sudo mkdir -p /etc/docker
> sudo tee /etc/docker/daemon.json <<-'EOF'
> {
>   "registry-mirrors": ["这边替换成自己的实际地址"]
> }
> EOF
> sudo systemctl daemon-reload
> 
> 
> 03 安装docker
> 
>   yum install -y docker-ce-18.09.0 docker-ce-cli-18.09.0 containerd.io
> 
> 
> 04 启动docker
> 	sudo systemctl start docker && sudo systemctl enable docker
> ```

## 1.5 修改hosts文件

> (1)master

```shell
# 设置master的hostname，并且修改hosts文件
sudo hostnamectl set-hostname m

vi /etc/hosts
192.168.1.51 m
192.168.1.52 w1
192.168.1.53 w2
```

> (2)两个worker

```shell
# 设置worker01/02的hostname，并且修改hosts文件
sudo hostnamectl set-hostname w1
sudo hostnamectl set-hostname w2

vi /etc/hosts
192.168.1.51 m
192.168.1.52 w1
192.168.1.53 w2
```

> (3)使用ping测试一下

## 1.6 系统基础前提配置

```shell
# (1)关闭防火墙
systemctl stop firewalld && systemctl disable firewalld

# (2)关闭selinux
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

# (3)关闭swap
swapoff -a
sed -i '/swap/s/^\(.*\)$/#\1/g' /etc/fstab

# (4)配置iptables的ACCEPT规则
iptables -F && iptables -X && iptables -F -t nat && iptables -X -t nat && iptables -P FORWARD ACCEPT

# (5)设置系统参数
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sysctl --system
```

## 1.7 Installing kubeadm, kubelet and kubectl

> (1)配置yum源

```shell
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
       http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

> (2)安装kubeadm&kubelet&kubectl

```shell
yum install -y kubeadm-1.14.0-0 kubelet-1.14.0-0 kubectl-1.14.0-0
```

> (3)docker和k8s设置同一个cgroup

```shell
# docker
vi /etc/docker/daemon.json
    "exec-opts": ["native.cgroupdriver=systemd"],
    
systemctl restart docker
    
# kubelet，这边如果发现输出directory not exist，也说明是没问题的，大家继续往下进行即可
sed -i "s/cgroup-driver=systemd/cgroup-driver=cgroupfs/g" /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
	
systemctl enable kubelet && systemctl start kubelet
```

## 1.8 proxy/pause/scheduler等国内镜像

> (1)查看kubeadm使用的镜像
>
> kubeadm config images list
>
> 可以发现这里都是国外的镜像

```
k8s.gcr.io/kube-apiserver:v1.14.0
k8s.gcr.io/kube-controller-manager:v1.14.0
k8s.gcr.io/kube-scheduler:v1.14.0
k8s.gcr.io/kube-proxy:v1.14.0
k8s.gcr.io/pause:3.1
k8s.gcr.io/etcd:3.3.10
k8s.gcr.io/coredns:1.3.1
```

> (2)解决国外镜像不能访问的问题

- 创建kubeadm.sh脚本，用于拉取镜像/打tag/删除原有镜像

```shell
#!/bin/bash

set -e

KUBE_VERSION=v1.14.0
KUBE_PAUSE_VERSION=3.1
ETCD_VERSION=3.3.10
CORE_DNS_VERSION=1.3.1

GCR_URL=k8s.gcr.io
ALIYUN_URL=registry.cn-hangzhou.aliyuncs.com/google_containers

images=(kube-proxy:${KUBE_VERSION}
kube-scheduler:${KUBE_VERSION}
kube-controller-manager:${KUBE_VERSION}
kube-apiserver:${KUBE_VERSION}
pause:${KUBE_PAUSE_VERSION}
etcd:${ETCD_VERSION}
coredns:${CORE_DNS_VERSION})

for imageName in ${images[@]} ; do
  docker pull $ALIYUN_URL/$imageName
  docker tag  $ALIYUN_URL/$imageName $GCR_URL/$imageName
  docker rmi $ALIYUN_URL/$imageName
done
```

- 运行脚本和查看镜像

```
# 运行脚本
sh ./kubeadm.sh

# 查看镜像
docker images
```

- 将这些镜像推送到自己的阿里云仓库【可选，根据自己实际的情况】

```shell
# 登录自己的阿里云仓库
docker login --username=xxx registry.cn-hangzhou.aliyuncs.com
```

```shell
#!/bin/bash

set -e

KUBE_VERSION=v1.14.0
KUBE_PAUSE_VERSION=3.1
ETCD_VERSION=3.3.10
CORE_DNS_VERSION=1.3.1

GCR_URL=k8s.gcr.io
ALIYUN_URL=registry.cn-beijing.aliyuncs.com/dwb_k8s

images=(kube-proxy:${KUBE_VERSION}
kube-scheduler:${KUBE_VERSION}
kube-controller-manager:${KUBE_VERSION}
kube-apiserver:${KUBE_VERSION}
pause:${KUBE_PAUSE_VERSION}
etcd:${ETCD_VERSION}
coredns:${CORE_DNS_VERSION})

for imageName in ${images[@]} ; do
  docker tag $GCR_URL/$imageName $ALIYUN_URL/$imageName
  docker push $ALIYUN_URL/$imageName
  docker rmi $ALIYUN_URL/$imageName
done
```

> 运行脚本 sh ./kubeadm-push-aliyun.sh

## 1.9 kube init初始化master

> (1)kube init流程

```
01-进行一系列检查，以确定这台机器可以部署kubernetes

02-生成kubernetes对外提供服务所需要的各种证书可对应目录
/etc/kubernetes/pki/*

03-为其他组件生成访问kube-ApiServer所需的配置文件
    ls /etc/kubernetes/
    admin.conf  controller-manager.conf  kubelet.conf  scheduler.conf
    
04-为 Master组件生成Pod配置文件。
    ls /etc/kubernetes/manifests/*.yaml
    kube-apiserver.yaml 
    kube-controller-manager.yaml
    kube-scheduler.yaml
    
05-生成etcd的Pod YAML文件。
    ls /etc/kubernetes/manifests/*.yaml
    kube-apiserver.yaml 
    kube-controller-manager.yaml
    kube-scheduler.yaml
	etcd.yaml
	
06-一旦这些 YAML 文件出现在被 kubelet 监视的/etc/kubernetes/manifests/目录下，kubelet就会自动创建这些yaml文件定义的pod，即master组件的容器。master容器启动后，kubeadm会通过检查localhost：6443/healthz这个master组件的健康状态检查URL，等待master组件完全运行起来

07-为集群生成一个bootstrap token

08-将ca.crt等 Master节点的重要信息，通过ConfigMap的方式保存在etcd中，工后续部署node节点使用

09-最后一步是安装默认插件，kubernetes默认kube-proxy和DNS两个插件是必须安装的
```

> (2)初始化master节点
>
> 官网：<https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm/>
>
> `注意`：**此操作是在主节点上进行**

```
# 本地有镜像
kubeadm init --kubernetes-version=1.14.0 --apiserver-advertise-address=192.168.1.51 --pod-network-cidr=10.244.0.0/16
【若要重新初始化集群状态：kubeadm reset，然后再进行上述操作】
```

**记得保存好最后kubeadm join的信息**

> (3)根据日志提示

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

**此时kubectl cluster-info查看一下是否成功**

> (4)查看pod验证一下
>
> 等待一会儿，同时可以发现像etc，controller，scheduler等组件都以pod的方式安装成功了
>
> `注意`：coredns没有启动，需要安装网络插件

```
kubectl get pods -n kube-system
```

> (5)健康检查

```
curl -k https://localhost:6443/healthz
```

## 1.10 部署calico网络插件

> 选择网络插件：<https://kubernetes.io/docs/concepts/cluster-administration/addons/>
>
> calico网络插件：<https://docs.projectcalico.org/v3.9/getting-started/kubernetes/>

> `calico，同样在master节点上操作`

```
# 在k8s中安装calico
kubectl apply -f https://docs.projectcalico.org/v3.9/manifests/calico.yaml

# 确认一下calico是否安装成功
kubectl get pods --all-namespaces -w
```

## 1.11 kube join

> **记得保存初始化master节点的最后打印信息【注意这边大家要自己的，下面我的只是一个参考】**

```
kubeadm join 192.168.1.51:6443 --token nygr6b.jm5imbmycc4upkf8 \
    --discovery-token-ca-cert-hash sha256:161fad728e919c240ba4f6c80ed6e90363b5dba1ff0d404d259c9d608c9d48be
```

> (1)在woker01和worker02上执行上述命令

> (2)在master节点上检查集群信息

```
kubectl get nodes

NAME                   STATUS   ROLES    AGE     VERSION
master-kubeadm-k8s     Ready    master   19m     v1.14.0
worker01-kubeadm-k8s   Ready    <none>   3m6s    v1.14.0
worker02-kubeadm-k8s   Ready    <none>   2m41s   v1.14.0
```

## 1.12 再次体验Pod

> (1)定义pod.yml文件，比如pod_nginx_rs.yaml

```yml
cat > pod_nginx_rs.yaml <<EOF
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx
  labels:
    tier: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      name: nginx
      labels:
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
EOF
```

> (2)根据pod_nginx_rs.yml文件创建pod

```
kubectl apply -f pod_nginx_rs.yaml
```

> (3)查看pod

```
kubectl get pods
kubectl get pods -o wide
kubectl describe pod nginx
```

> (4)感受通过rs将pod扩容

```
kubectl scale rs nginx --replicas=5
kubectl get pods -o wide
```

> (5)删除pod

```
kubectl delete -f pod_nginx_rs.yaml
```

# 02 Basic

## 2.1 yaml文件

### 2.1.1 简介

YAML（IPA: /ˈjæməl/）是一个可读性高的语言，参考了XML、C、Python等。

理解：Yet Another Markup Language

后缀：可以是.yml或者是.yaml，更加推荐.yaml，其实用任意后缀都可以，只是阅读性不强

### 2.1.2 基础

- 区分大小写
- 缩进表示层级关系，相同层级的元素左对齐
- 缩进只能使用空格，不能使用TAB
- "#"表示当前行的注释
- 是JSON文件的超级，两个可以转换
- ---表示分隔符，可以在一个文件中定义多个结构
- 使用key: value，其中":"和value之间要有一个英文空格

### 2.1.3 Maps

#### 2.1.3.1 简单

```yaml
apiVersion: v1
kind: Pod
```

> ---表示分隔符，可选。要定义多个结构一定要分隔
>
> apiVersion表示key，v1表示value，英文":"后面要有一个空格
>
> kind表示key，Pod表示value
>
> 也可以这样写apiVersion: "v1"
>
> `转换为JSON格式`
>
> ```json
> {
> "apiVersion": "v1",
> "kind": "Pod"
> }
> ```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
```

#### 2.1.2.2 复杂

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
```

> metadata表示key，下面的内容表示value，该value中包含两个直接的key：name和labels
>
> name表示key，nginx-deployment表示value
>
> labels表示key，下面的表示value，这个值又是一个map
>
> app表示key，nginx表示value
>
> 相同层级的记得使用空间缩进，左对齐
>
> `转换为JSON格式`
>
> ```json
> {
> "apiVersion": "apps/v1",
> "kind": "Deployment",
> "metadata": {
>             "name": "nginx-deployment",
>             "labels": {
>                        "app": "nginx"
>                       }
>            }
> }
> ```

### 2.1.4 Lists

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container01
    image: busybox:1.28
  - name: myapp-container02
    image: busybox:1.28
```

> containers表示key，下面的表示value，其中value是一个数组
>
> 数组中有两个元素，每个元素里面包含name和image
>
> image表示key，myapp-container表示value
>
> `转换成JSON格式`
>
> ```json
> {
> "apiVersion": "v1",
> "kind": "Pod",
> "metadata": {
>               "name": "myapp",
>               "labels": {
>                           "app": "myapp"
>                         }
>             },
>  "spec": {
>     "containers": [{
>                     "name": "myapp-container01",
>                     "image": "busybox:1.28",
>                    }, 
>                    {
>                     "name": "myapp-container02",
>                     "image": "busybox:1.28",
>                    }]
>          }
> }
> ```

### 2.1.5 找个k8s的yaml文件

> `官网`：<https://kubernetes.io/docs/reference/>

```yaml
# yaml格式对于Pod的定义：
apiVersion: v1          #必写，版本号，比如v1
kind: Pod               #必写，类型，比如Pod
metadata:               #必写，元数据
  name: nginx           #必写，表示pod名称
  namespace: default    #表示pod名称属于的命名空间
  labels:
    app: nginx                  #自定义标签名字
spec:                           #必写，pod中容器的详细定义
  containers:                   #必写，pod中容器列表
  - name: nginx                 #必写，容器名称
    image: nginx                #必写，容器的镜像名称
    ports:
    - containerPort: 80         #表示容器的端口
```

## 2.2 Container

> `官网`：<https://kubernetes.io/docs/concepts/containers/>

### 2.2.1 Docker世界中

可以通过docker run运行一个容器

或者定义一个yml文件，本机使用docker-compose，多机通过docker swarm创建

### 2.2.2 K8S世界中

同样以一个yaml文件维护，container运行在pod中

## 2.3 Pod

> `官网`：<https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/>

### 2.3.1 What is Pod

```
A Pod is the basic execution unit of a Kubernetes application
A Pod encapsulates an application’s container (or, in some cases, multiple containers), storage resources, a unique network IP, and options that govern how the container(s) should run
```

### 2.3.2 Pod初体验

> (1)创建一个pod的yaml文件，名称为nginx_pod.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx-container
    image: nginx
    ports:
    - containerPort: 80
```

> (2)根据该nginx_pod.yaml文件创建pod

```
kubectl apply -f nginx_pod.yaml
```

> (3)查看pod

01 kubectl get pods

```
NAME        READY   STATUS    RESTARTS   AGE
nginx-pod   1/1     Running   0          29s
```

02 kubectl get pods -o wide

```
NAME       READY     STATUS   RESTARTS   AGE             IP             NODE   
nginx-pod   1/1     Running      0       40m       192.168.80.194        w2 
```

03 kubectl describe pod nginx-pod

```
Name:               nginx-pod
Namespace:          default
Priority:           0
PriorityClassName:  <none>
Node:               w2/192.168.0.62
Start Time:         Sun, 06 Oct 2019 20:45:35 +0000
Labels:             app=nginx
Annotations:        cni.projectcalico.org/podIP: 192.168.80.194/32
                    kubectl.kubernetes.io/last-applied-configuration:
                      {"apiVersion":"v1","kind":"Pod","metadata":{"annotations":{},"labels":{"app":"nginx"},"name":"nginx-pod","namespace":"default"},"spec":{"c...
Status:             Running
IP:                 192.168.80.194
Containers:
  nginx-container:
    Container ID:   docker://eb2fd0b2906f53e9892e22a6fd791c9ac68fb8e5efce3bbf94ec12bae96e1984
    Image:          nginx
    Image ID:       docker-pullable:/
```

> (4)可以发现该pod运行在worker02节点上

于是来到worker02节点，docker ps一下

```
CONTAINER ID  IMAGE  COMMAND                    CREATED        STATUS   PORTS   NAMES
eb2fd0b2906f  nginx  "nginx -g 'daemon of…"   6 minutes ago       Up 6 minutes           k8s_nginx-container_nginx-pod_default_3ee0706d-e87a-11e9-a904-5254008afee6_0
```

不妨进入该容器试试[可以发现只有在worker02上有该容器，因为pod运行在worker02上]：

docker exec -it k8s_nginx-container_nginx-pod_default_3ee0706d-e87a-11e9-a904-5254008afee6_0 bash

```
root@nginx-pod:/#
```

> (5)访问nginx容器

```
curl 192.168.80.194    OK，并且在任何一个集群中的Node上访问都成功
```

> (6)删除Pod

```
kubectl delete -f nginx_pod.yaml
kubectl get pods
```

### 2.3.3 Storage and Networking

> `官网`：https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/#networking

- Networking

```
Each Pod is assigned a unique IP address. Every container in a Pod shares the network namespace, including the IP address and network ports. 
```

> `官网`：https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/#storage

- Storage

```
A Pod can specify a set of shared storage Volumes. All containers in the Pod can access the shared volumes, allowing those containers to share data. 
```

