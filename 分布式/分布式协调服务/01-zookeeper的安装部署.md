# zookeeper的安装部署

## 安装

zookeeper有两种运行模式：集群模式和单机模式。

下载zookeeper安装包：http://apache.fayea.com/zookeeper/

下载完成，通过 tar -zxvf 解压。

### 单机环境安装

一般情况下，在开发测试环境，没有这么多资源的情况，而且也不需要特别好的稳定性的前提下，我们可以使用单机部署。

初次使用zookeeper，需要将conf目录下的zoo_sample.cfg文件copy一份重命名为zoo.cfg修改dataDir目录，dataDir表示日志文件存放的路径。

### 集群环境安装

在zookeeper集群中，各个节点总共有三种角色，分别是：leader\follower\observer集群模式，采用模拟3台机器来搭建zookeeper集群。分别复制安装包到三台机器上并解压，同时copy一份zoo.cfg。

1. 修改配置文件

   server.1=IP1:2888:3888【2888:访问zookeeper的端口，3888:重新选举leader的端口】

   server.2=IP2:2888:3888

   server.3=IP3:2888:3888

   其格式为：server.A=B:C:D

   A:是一个数字，表示这个是第几号服务器

   B:是这个服务器的IP地址

   C:表示的是这个服务器于集群中的Leader服务器交换信息的端口

   D:表示的是万一集群中的Leader服务器挂了，需要一个端口来重新进行选举，选出一个新Leader。而这个端口就是用来执行选举时服务器相互通信的端口。如果是伪集群的配置方式，由于B都是一样，所以不同的Zookeeper实例通信端口号不能一样，所以要给他们分配不同的端口号。

   在集群模式下，集群中每台机器都需要感知到整个集群是由哪几台机器组成的，在配置文件中，按照格式server.id=host:prot:port，每一行代表一个机器配置。id：指的是server ID，用来标识机器在集群中的机器序号。

2. 新建dataDir目录，设置myid

   在每台zookeeper机器上，都需要在数据目录（dataDir）下创建一个myid文件，该文件只有一行内容，对应每台机器的ServerID数字。比如：server.1的myid文件内容就是1.【必须确保每个服务器的myid文件中的数字不同，并且和自己所在机器的zoo.cfg中server.id的id值一致，id的范围是1-255】

3. 启动zookeeper。

### 常用命令

启动ZK服务: 

bin/zkServer.sh start 

查看 ZK 服务状态: 

bin/zkServer.sh status 

停止 ZK 服务: 

bin/zkServer.sh stop 

重启 ZK 服务: 

bin/zkServer.sh restart 

连接服务器 

zkCli.sh -timeout 0 -r -server ip:port 

