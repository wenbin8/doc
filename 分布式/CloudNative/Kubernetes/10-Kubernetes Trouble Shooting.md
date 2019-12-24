# Kubernetes Trouble Shooting

## Master

master上的组件共同组成了控制平面

```
01 若apiserver出问题了 会导致整个K8s集群不可以使用，因为apiserver是K8s集群的大脑
02 若etcd出问题了 apiserver和etcd则无法通信，kubelet也无法更新所在node上的状态
03 当scheduler或者controller manager出现问题时 会导致deploy，pod，service等无法正常运行
```

解决方案 :出现问题时，监听到自动重启或者搭建高可用的master集群

## Worker

worker节点挂掉或者上面的kubelet服务出现问题时，w上的pod则无法正常运行。

## Addons

dns和网络插件比如calico发生问题时，集群内的网络无法正常通信，并且无法根据服务名称进行解析。

## 系统问题排查

- 查看Node的状态

  ```
  kubectl get nodes
  kubectl describe node-name
  ```

- 查看集群master和worker组件的日志

  ```
  journalctl -u apiserver 
  journalctl -u scheduler 
  journalctl -u kubelet 
  journalctl -u kube-proxy 
  ...
  ```

## Pod的问题排查

K8s中最小的操作单元是Pod，最重要的操作也是Pod，其他资源的排查可以参照Pod问题的排查

### 查看Pod运行情况

```
kubectl get pods -n namespace
```

### 查看Pod的具体描述，定位问题

```
kubectl describe pod pod-name -n namespace
```

### 检查Pod对应的yaml是否有误

```
kubectl get pod pod-name -o yaml
```

### 查看Pod日志

```
kubectl logs ...
```

### Pod可能会出现哪些问题及解决方案

1. **处于Pending状态** 

   说明Pod还没有被调度到某个node上，可以describe一下详情。可能因为资源不足，端口被占用等。

2. 处于Waiting/ContainerCreating状态

   可能因为镜像拉取失败，或者是网络插件的问题，比如calico，或者是容器本身的问题，可以检查一下容器的yaml文件内容和Dockerfile的书写。

3. 处于ImagePullBackOff状态

   镜像拉取失败，可能是镜像不存在，或者没有权限拉取。

4. 处于CrashLoopBackOff状态

   Pod之前启动成功过，但是又失败了，不断在重启。

5. 处于Error状态

   有些内容不存在，比如ConfigMap，PV，没有权限等，需要创建一下。

6.  处于Terminating状态

   说明Pod正在停止

7. 处于Unknown状态

   说明K8s已经失去对Pod的管理监听。