# Redis安装与启动

## **CentOS7安装Redis单实例**

### 下载,解压

下载地址在：redis.io，比如把Redis安装到/usr/local/soft/

```shell
cd /usr/local/soft/
wget http://download.redis.io/releases/redis-5.0.5.tar.gz

tar -zxvf redis-5.0.5.tar.gz
```

### 安装gcc依赖

Redis是C语言编写的，编译需要

```shell
yum install gcc
```

### 编译安装

```shell
cd redis-5.0.5
make MALLOC=libc
```

将/usr/local/soft/redis-5.0.5/src目录下二进制文件安装到/usr/local/bin

```shell
cd src
make install
```

### 修改配置文件

默认的配置文件是/usr/local/soft/redis-5.0.5/redis.conf

后台启动

```
daemonize no
```

改成

```
daemonize yes
```

下面一行必须改成 bind 0.0.0.0 或注释，否则只能在本机访问

```
bind 127.0.0.1 
```

如果需要密码访问，取消requirepass的注释

```
requirepass yourpassword
```

### 启动

使用指定配置文件启动Redis（这个命令建议配置alias）

```shell
/usr/local/soft/redis-5.0.5/src/redis-server /usr/local/soft/redis-5.0.5/redis.conf
```

进入客户端（这个命令建议配置alias）

```
/usr/local/soft/redis-5.0.5/src/redis-cli
```

### 停止

```
redis> shutdown
```

或

```
ps -aux | grep redis
kill -9 xxxx
```

## **单机安装Redis Cluster（3主3从）**

首先，本篇要基于单实例的安装，也就是上面的单实例安装完成。

为了节省机器，我们直接把6个Redis实例安装在同一台机器上（3主3从），只是使用不同的端口号。
机器IP 192.168.8.207

更新：新版的cluster已经不需要通过ruby脚本创建，删掉了ruby相关依赖的安装

```
cd /usr/local/soft/redis-5.0.5
mkdir redis-cluster
cd redis-cluster
mkdir 7291 7292 7293 7294 7295 7296
```

复制redis配置文件到7291目录

```
cp /usr/local/soft/redis-5.0.5/redis.conf /usr/local/soft/redis-5.0.5/redis-cluster/7291
```

修改7291的redis.conf配置文件，内容：

```
port 7291
dir /usr/local/soft/redis-5.0.5/redis-cluster/7291/
cluster-enabled yes
cluster-config-file nodes-7291.conf
cluster-node-timeout 5000
appendonly yes
pidfile /var/run/redis_7291.pid
```

把7291下的redis.conf复制到其他5个目录。

```
cd /usr/local/soft/redis-5.0.5/redis-cluster/7291
cp redis.conf ../7292
cp redis.conf ../7293
cp redis.conf ../7294
cp redis.conf ../7295
cp redis.conf ../7296
```

批量替换内容

```
cd /usr/local/soft/redis-5.0.5/redis-cluster
sed -i 's/7291/7292/g' 7292/redis.conf
sed -i 's/7291/7293/g' 7293/redis.conf
sed -i 's/7291/7294/g' 7294/redis.conf
sed -i 's/7291/7295/g' 7295/redis.conf
sed -i 's/7291/7296/g' 7296/redis.conf
```

启动6个Redis节点

```
cd /usr/local/soft/redis-5.0.5/
./src/redis-server redis-cluster/7291/redis.conf
./src/redis-server redis-cluster/7292/redis.conf
./src/redis-server redis-cluster/7293/redis.conf
./src/redis-server redis-cluster/7294/redis.conf
./src/redis-server redis-cluster/7295/redis.conf
./src/redis-server redis-cluster/7296/redis.conf
```

是否启动了6个进程

```
ps -ef|grep redis
```

创建集群
旧版本中的redis-trib.rb已经废弃了，直接用–cluster命令
注意用绝对IP，不要用127.0.0.1

```
cd /usr/local/soft/redis-5.0.5/src/
redis-cli --cluster create 192.168.8.207:7291 192.168.8.207:7292 192.168.8.207:7293 192.168.8.207:7294 192.168.8.207:7295 192.168.8.207:7296 --cluster-replicas 1
```

Redis会给出一个预计的方案，对6个节点分配3主3从，如果认为没有问题，输入yes确认

```
>>> Performing hash slots allocation on 6 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
Adding replica 127.0.0.1:7295 to 127.0.0.1:7291
Adding replica 127.0.0.1:7296 to 127.0.0.1:7292
Adding replica 127.0.0.1:7294 to 127.0.0.1:7293
>>> Trying to optimize slaves allocation for anti-affinity
[WARNING] Some slaves are in the same host as their master
M: dfdc9c0589219f727e4fd0ad8dafaf7e0cfb4f1c 127.0.0.1:7291
   slots:[0-5460] (5461 slots) master
M: 8c878b45905bba3d7366c89ec51bd0cd7ce959f8 127.0.0.1:7292
   slots:[5461-10922] (5462 slots) master
M: aeeb7d7076d9b25a7805ac6f508497b43887e599 127.0.0.1:7293
   slots:[10923-16383] (5461 slots) master
S: ebc479e609ff8f6ca9283947530919c559a08f80 127.0.0.1:7294
   replicates aeeb7d7076d9b25a7805ac6f508497b43887e599
S: 49385ed6e58469ef900ec48e5912e5f7b7505f6e 127.0.0.1:7295
   replicates dfdc9c0589219f727e4fd0ad8dafaf7e0cfb4f1c
S: 8d6227aefc4830065624ff6c1dd795d2d5ad094a 127.0.0.1:7296
   replicates 8c878b45905bba3d7366c89ec51bd0cd7ce959f8
Can I set the above configuration? (type 'yes' to accept): 
```

注意看slot的分布：

```
7291  [0-5460] (5461个槽) 
7292  [5461-10922] (5462个槽) 
7293  [10923-16383] (5461个槽)
```

集群创建完成

```
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join
....
>>> Performing Cluster Check (using node 127.0.0.1:7291)
M: dfdc9c0589219f727e4fd0ad8dafaf7e0cfb4f1c 127.0.0.1:7291
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
M: 8c878b45905bba3d7366c89ec51bd0cd7ce959f8 127.0.0.1:7292
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
M: aeeb7d7076d9b25a7805ac6f508497b43887e599 127.0.0.1:7293
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
S: 8d6227aefc4830065624ff6c1dd795d2d5ad094a 127.0.0.1:7296
   slots: (0 slots) slave
   replicates aeeb7d7076d9b25a7805ac6f508497b43887e599
S: ebc479e609ff8f6ca9283947530919c559a08f80 127.0.0.1:7294
   slots: (0 slots) slave
   replicates dfdc9c0589219f727e4fd0ad8dafaf7e0cfb4f1c
S: 49385ed6e58469ef900ec48e5912e5f7b7505f6e 127.0.0.1:7295
   slots: (0 slots) slave
   replicates 8c878b45905bba3d7366c89ec51bd0cd7ce959f8
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

重置集群的方式是在每个节点上个执行`cluster reset`，然后重新创建集群

连接到客户端

```
redis-cli -p 7291
redis-cli -p 7292
redis-cli -p 7293
```

批量写入值

```
cd /usr/local/soft/redis-5.0.5/redis-cluster/
vim setkey.sh
```

脚本内容

```
#!/bin/bash
for ((i=0;i<20000;i++))
do
echo -en "helloworld" | redis-cli -h 192.168.8.207 -p 7291 -c -x set name$i >>redis.log
done
chmod +x setkey.sh
./setkey.sh
```

每个节点分布的数据

```
127.0.0.1:7292> dbsize
(integer) 6683
127.0.0.1:7293> dbsize
(integer) 6665
127.0.0.1:7291> dbsize
(integer) 6652
```

其他命令，比如添加节点、删除节点，重新分布数据：

```
redis-cli --cluster help
Cluster Manager Commands:
  create         host1:port1 ... hostN:portN
                 --cluster-replicas <arg>
  check          host:port
                 --cluster-search-multiple-owners
  info           host:port
  fix            host:port
                 --cluster-search-multiple-owners
  reshard        host:port
                 --cluster-from <arg>
                 --cluster-to <arg>
                 --cluster-slots <arg>
                 --cluster-yes
                 --cluster-timeout <arg>
                 --cluster-pipeline <arg>
                 --cluster-replace
  rebalance      host:port
                 --cluster-weight <node1=w1...nodeN=wN>
                 --cluster-use-empty-masters
                 --cluster-timeout <arg>
                 --cluster-simulate
                 --cluster-pipeline <arg>
                 --cluster-threshold <arg>
                 --cluster-replace
  add-node       new_host:new_port existing_host:existing_port
                 --cluster-slave
                 --cluster-master-id <arg>
  del-node       host:port node_id
  call           host:port command arg arg .. arg
  set-timeout    host:port milliseconds
  import         host:port
                 --cluster-from <arg>
                 --cluster-copy
                 --cluster-replace
  help           

For check, fix, reshard, del-node, set-timeout you can specify the host and port of any working node in the cluster.
```

附录：

### 集群命令

cluster info ：打印集群的信息
cluster nodes ：列出集群当前已知的所有节点（node），以及这些节点的相关信息。
cluster meet ：将 ip 和 port 所指定的节点添加到集群当中，让它成为集群的一份子。
cluster forget <node_id> ：从集群中移除 node_id 指定的节点(保证空槽道)。
cluster replicate <node_id> ：将当前节点设置为 node_id 指定的节点的从节点。
cluster saveconfig ：将节点的配置文件保存到硬盘里面。

### 槽slot命令

cluster addslots [slot …] ：将一个或多个槽（slot）指派（assign）给当前节点。
cluster delslots [slot …] ：移除一个或多个槽对当前节点的指派。
cluster flushslots ：移除指派给当前节点的所有槽，让当前节点变成一个没有指派任何槽的节点。
cluster setslot node <node_id> ：将槽 slot 指派给 node_id 指定的节点，如果槽已经指派给另一个节点，那么先让另一个节点删除该槽>，然后再进行指派。
cluster setslot migrating <node_id> ：将本节点的槽 slot 迁移到 node_id 指定的节点中。
cluster setslot importing <node_id> ：从 node_id 指定的节点中导入槽 slot 到本节点。
cluster setslot stable ：取消对槽 slot 的导入（import）或者迁移（migrate）。

### 键命令

cluster keyslot ：计算键 key 应该被放置在哪个槽上。
cluster countkeysinslot ：返回槽 slot 目前包含的键值对数量。
cluster getkeysinslot ：返回 count 个 slot 槽中的键