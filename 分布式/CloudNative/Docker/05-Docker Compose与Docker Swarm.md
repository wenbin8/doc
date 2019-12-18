# Docker Compose

> `官网`：<https://docs.docker.com/compose/>

## 业务背景

## Docker传统方式实现

### 写Python代码&build image

> (1)创建文件夹

```shell
mkdir -p /tmp/composetest
cd /tmp/composetest
```

> (2)创建app.py文件，写业务内容

```python
import time

import redis
from flask import Flask

app = Flask(__name__)
cache = redis.Redis(host='redis', port=6379)

def get_hit_count():
    retries = 5
    while True:
        try:
            return cache.incr('hits')
        except redis.exceptions.ConnectionError as exc:
            if retries == 0:
                raise exc
            retries -= 1
            time.sleep(0.5)

@app.route('/')
def hello():
    count = get_hit_count()
    return 'Hello World! I have been seen {} times.\n'.format(count)
```

> (3)新建requirements.txt文件

```
flask
redis
```

> (4)编写Dockerfile

```dockerfile
FROM python:3.7-alpine
WORKDIR /code
ENV FLASK_APP app.py
ENV FLASK_RUN_HOST 0.0.0.0
RUN apk add --no-cache gcc musl-dev linux-headers
COPY requirements.txt requirements.txt
RUN pip install -r requirements.txt
COPY . .
CMD ["flask", "run"]
```

> (5)根据Dockerfile生成image

```
docker build -t python-app-image .
```

> (6)查看images：docker images

```
python-app-image latest 7e1d81f366b7 3 minutes ago  213MB
```

### 获取Redis的image

```
docker pull redis:alpine
```

### 创建两个container

> (1)创建网络

```
docker network ls
docker network create --subnet=172.20.0.0/24 app-net 
```

> (1)创建python程序的container，并指定网段和端口

```
docker run -d --name web -p 5000:5000 --network app-net python-app-image
```

> (2)创建redis的container，并指定网段

```
docker run -d --name redis --network app-net redis:alpine
```

### 访问测试

ip[centos]:5000

## 简介和安装

### 简介

> `官网`：https://docs.docker.com/compose/
>
> ```
> Compose is a tool for defining and running multi-container Docker applications. With Compose, you use a YAML file to configure your application’s services. Then, with a single command, you create and start all the services from your configuration.
> ```

### 安装

> Linux环境中需要单独安装
>
> `官网`：https://docs.docker.com/compose/install/

```
sudo curl -L "https://github.com/docker/compose/releases/download/1.24.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```

```
sudo chmod +x /usr/local/bin/docker-compose
```

## docker compose实现

> `reference`：<https://docs.docker.com/compose/gettingstarted/>

### 同样的前期准备

新建目录，比如composetest

进入目录，编写app.py代码

创建requirements.txt文件

编写Dockerfile

### 编写docker-compose.yaml文件

默认名称，当然也可以指定，`docker-compose.yaml`

```yaml
version: '3'
services:
  web:
    build: .
    ports:
      - "5000:5000"
    networks:
      - app-net

  redis:
    image: "redis:alpine"
    networks:
      - app-net

networks:
  app-net:
    driver: bridge
```

> (1)通过docker compose创建容器

```
docker-compose up -d
```

> (2)访问测试

![image-20191218153551479](assets/image-20191218153551479.png)

## 详解docker-compose.yml文件

> (1)version: '3'

```
表示docker-compose的版本
```

> (2)services

```
一个service表示一个container
```

> (3)networks

```
相当于docker network create app-net
```

> (4)volumes

```
相当于-v v1:/var/lib/mysql
```

> (5)image

```
表示使用哪个镜像，本地build则用build，远端则用image
```

> (6)ports

```
相当于-p 8080:8080
```

> (7)environment

```
相当于-e 
```

## docker-compose常见操作

(1)查看版本

​	docker-compose version

(2)根据yml创建service

​	docker-compose up

​	指定yaml：docker-compose  up -f xxx.yaml

​	后台运行：docker-compose up

(3)查看启动成功的service

​	docker-compose ps

​	也可以使用docker ps

(4)查看images

​	docker-compose images

(5)停止/启动service

​	docker-compose stop/start 

(6)删除service[同时会删除掉network和volume]

​	docker-compose down

(7)进入到某个service

​	docker-compose exec redis sh

## scale扩缩容

> (1)修改docker-compose.yaml文件，主要是把web的ports去掉，不然会报错

```yaml
version: '3'
services:
  web:
    build: .
    networks:
      - app-net

  redis:
    image: "redis:alpine"
    networks:
      - app-net

networks:
  app-net:
    driver: bridge
```

> (2)创建service

```
docker-compose up -d
```

> (3)若要对python容器进行扩缩容

```
docker-compose up --scale web=5 -d
docker-compose ps
docker-compose logs web
```

# Docker Swarm

> `官网`：<https://docs.docker.com/swarm/>

## Install Swarm

### 环境准备

> (1)根据Vagrantfile创建3台centos机器
>
> [**大家可以根据自己实际的情况准备3台centos机器，不一定要使用vagrant+virtualbox**]
>
> 新建swarm-docker-centos7文件夹，创建Vagrantfile

```ruby
boxes = [
    {
        :name => "manager-node",
        :eth1 => "192.168.0.11",
        :mem => "1024",
        :cpu => "1"
    },
    {
        :name => "worker01-node",
        :eth1 => "192.168.0.12",
        :mem => "1024",
        :cpu => "1"
    },
    {
        :name => "worker02-node",
        :eth1 => "192.168.0.13",
        :mem => "1024",
        :cpu => "1"
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

> (2)进入到对应的centos里面，使得root账户能够登陆，从而使用XShell登陆

```
vagrant ssh manager-node/worker01-node/worker02-node
sudo -i
vi /etc/ssh/sshd_config
修改PasswordAuthentication yes
passwd    修改密码
systemctl restart sshd
```

> (3)在win上ping一下各个主机，看是否能ping通

```
ping 192.168.0.11/12/13
```

> (4)在每台机器上安装docker engine
>
> `小技巧`：要想让每个shell窗口一起执行同样的命令"查看-->撰写-->撰写窗口-->全部会话"

### 搭建Swarm集群

> (1)进入manager
>
> `提示`：manager node也可以作为worker node提供服务

```
docker swarm init --advertise-addr=192.168.0.11
```

`注意观察日志，拿到worker node加入manager node的信息`

```
docker swarm join --token SWMTKN-1-0a5ph4nehwdm9wzcmlbj2ckqqso38pkd238rprzwcoawabxtdq-arcpra6yzltedpafk3qyvv0y3 192.168.0.11:2377
```

> (2)进入两个worker

```
docker swarm join --token SWMTKN-1-0a5ph4nehwdm9wzcmlbj2ckqqso38pkd238rprzwcoawabxtdq-arcpra6yzltedpafk3qyvv0y3 192.168.0.11:2377
```

`日志打印`

```
This node joined a swarm as a worker.
```

> (3)进入到manager node查看集群状态

```
docker node ls
```

> (4)node类型的转换
>
> 可以将worker提升成manager，从而保证manager的高可用

```
docker node promote worker01-node
docker node promote worker02-node

#降级可以用demote
docker node demote worker01-node
```

### 在线的

> http://labs.play-with-docker.com

## Swarm基本操作

### Service

> (1)创建一个tomcat的service

```
docker service create --name my-tomcat tomcat
```

> (2)查看当前swarm的service

```
docker service ls
```

> (3)查看service的启动日志

```
docker service logs my-tomcat
```

> (4)查看service的详情

```
docker service inspect my-tomcat
```

> (5)查看my-tomcat运行在哪个node上

```
docker service ps my-tomcat
```

`日志`

```
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE           ERROR               PORTS
u6o4mz4tj396        my-tomcat.1         tomcat:latest       worker01-node       Running             Running 3 minutes ago  
```

> (6)水平扩展service

```
docker service scale my-tomcat=3
docker service ls
docker service ps my-tomcat
```

`日志`：可以发现，其他node上都运行了一个my-tomcat的service

```
[root@manager-node ~]# docker service ps my-tomcat
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
u6o4mz4tj396        my-tomcat.1         tomcat:latest       worker01-node       Running             Running 8 minutes ago                        
v505wdu3fxqo        my-tomcat.2         tomcat:latest       manager-node        Running             Running 46 seconds ago                       
wpbsilp62sc0        my-tomcat.3         tomcat:latest       worker02-node       Running             Running 49 seconds ago  
```

此时到worker01-node上：docker ps，可以发现container的name和service名称不一样，这点要知道

```
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
bc4b9bb097b8        tomcat:latest       "catalina.sh run"   10 minutes ago      Up 10 minutes       8080/tcp            my-tomcat.1.u6o4mz4tj3969a1p3mquagxok
```

> (7)如果某个node上的my-tomcat挂掉了，这时候会自动扩展

```
[worker01-node]
docker rm -f containerid

[manager-node]
docker service ls
docker service ps my-tomcat
```

> (8)删除service

```
docker service rm my-tomcat
```

### 多机通信overlay网络[3.7的延续]

> `业务场景`：workpress+mysql实现个人博客搭建
>
> > <https://hub.docker.com/_/wordpress?tab=description>

#### 传统手动方式实现

##### 一台centos上，分别创建容器

```
01-创建mysql容器[创建完成等待一会，注意mysql的版本]
	docker run -d --name mysql -v v1:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=examplepass -e MYSQL_DATABASE=db_wordpress mysql:5.6
	
02-创建wordpress容器[将wordpress的80端口映射到centos的8080端口]
	docker run -d --name wordpress --link mysql -e WORDPRESS_DB_HOST=mysql:3306 -e WORDPRESS_DB_USER=root -e WORDPRESS_DB_PASSWORD=examplepass -e WORDPRESS_DB_NAME=db_wordpress -p 8080:80 wordpress
	
03-查看默认bridge的网络，可以发现两个容器都在其中
	docker network inspect bridge
	
04-访问测试
	win浏览器中输入：ip[centos]:8080，一直下一步
```

##### 使用docker compose创建

> docker-compose的方式还是在一台机器中，网络这块很清晰

```
01-创建wordpress-mysql文件夹
	mkdir -p /tmp/wordpress-mysql
	cd /tmp/wordpress-mysql
	
02-创建docker-compose.yml文件
```

`文件内容`

```yml
version: '3.1'

services:

  wordpress:
    image: wordpress
    restart: always
    ports:
      - 8080:80
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: exampleuser
      WORDPRESS_DB_PASSWORD: examplepass
      WORDPRESS_DB_NAME: exampledb
    volumes:
      - wordpress:/var/www/html

  db:
    image: mysql:5.7
    restart: always
    environment:
      MYSQL_DATABASE: exampledb
      MYSQL_USER: exampleuser
      MYSQL_PASSWORD: examplepass
      MYSQL_RANDOM_ROOT_PASSWORD: '1'
    volumes:
      - db:/var/lib/mysql

volumes:
  wordpress:
  db:
```

```
03-根据docker-compose.yml文件创建service
	docker-compose up -d
	
04-访问测试
	win10浏览器ip[centos]:8080，一直下一步
	
05-值得关注的点是网络
	docker network ls
	docker network inspect wordpress-mysql_default
```

#### Swarm中实现

> 还是wordpress+mysql的案例，在docker swarm集群中怎么玩呢？

> (1)创建一个overlay网络，用于docker swarm中多机通信

```
【manager-node】
docker network create -d overlay my-overlay-net

docker network ls[此时worker node查看不到]
```

> (2)创建mysql的service

```
【manager-node】
01-创建service
docker service create --name mysql --mount type=volume,source=v1,destination=/var/lib/mysql --env MYSQL_ROOT_PASSWORD=examplepass --env MYSQL_DATABASE=db_wordpress --network my-overlay-net mysql:5.6

02-查看service
	docker service ls
	docker service ps mysql
```

> (3)创建wordpress的service

```
01-创建service  [注意之所以下面可以通过mysql名字访问，也是因为有DNS解析]
docker service create --name wordpress --env WORDPRESS_DB_USER=root --env WORDPRESS_DB_PASSWORD=examplepass --env WORDPRESS_DB_HOST=mysql:3306 --env WORDPRESS_DB_NAME=db_wordpress -p 8080:80 --network my-overlay-net wordpress

02-查看service
	docker service ls
	docker service ps mysql
	
03-此时mysql和wordpress的service运行在哪个node上，这时候就能看到my-overlay-net的网络
```

> (4)测试

```
win浏览器访问ip[manager/worker01/worker02]:8080都能访问成功
```

> (5)查看my-overlay-net

```
docker network inspect my-overlay-net
```

> (6)为什么没有用etcd？docker swarm中有自己的分布式存储机制

## Routing Mesh

### Ingress

> 通过前面的案例我们发现，部署一个wordpress的service，映射到主机的8080端口，这时候通过swarm集群中的任意主机ip:8080都能成功访问，这是因为什么？
>
> `把问题简化`：docker service create --name tomcat  -p 8080:8080 --network my-overlay-net tomcat

> (1)记得使用一个自定义的overlay类型的网络

```
--network my-overlay-net
```

> (2)查看service情况

```
docker service ls
docker service ps tomcat
```

> (3)访问3台机器的ip:8080测试

```
发现都能够访问到tomcat的欢迎页
```

### Internal

> 之前在实战wordpress+mysql的时候，发现wordpress中可以直接通过mysql名称访问
>
> 这样可以说明两点，第一是其中一定有dns解析，第二是两个service的ip是能够ping通的
>
> `思考`：不妨再创建一个service，也同样使用上述tomcat的overlay网络，然后来实验
>
> docker service create --name whoami -p 8000:8000 --network my-overlay-net -d  jwilder/whoami

> (1)查看whoami的情况

```
docker service ps whoami
```

> (2)在各自容器中互相ping一下彼此，也就是容器间的通信

```
#tomcat容器中ping whoami
docker exec -it 9d7d4c2b1b80 ping whoami
64 bytes from bogon (10.0.0.8): icmp_seq=1 ttl=64 time=0.050 ms
64 bytes from bogon (10.0.0.8): icmp_seq=2 ttl=64 time=0.080 ms


#whoami容器中ping tomcat
docker exec -it 5c4fe39e7f60 ping tomcat
64 bytes from bogon (10.0.0.18): icmp_seq=1 ttl=64 time=0.050 ms
64 bytes from bogon (10.0.0.18): icmp_seq=2 ttl=64 time=0.080 ms
```

> (3)将whoami进行扩容

```
docker service scale whoami=3
docker service ps whoami     #manager,worker01,worker02
```

> (4)此时再ping whoami service，并且访问whoami服务

```
#ping
docker exec -it 9d7d4c2b1b80 ping whoami
64 bytes from bogon (10.0.0.8): icmp_seq=1 ttl=64 time=0.055 ms
64 bytes from bogon (10.0.0.8): icmp_seq=2 ttl=64 time=0.084 ms

#访问
docker exec -it 9d7d4c2b1b80 curl whoami:8000  [多访问几次]
I'm 09f4158c81ae
I'm aebc574dc990
I'm 7755bc7da921
```

`小结`：通过上述的实验可以发现什么？whoami服务对其他服务暴露的ip是不变的，但是通过whoami名称访问8000端口，确实访问到的是不同的service，就说明访问其实是像下面这张图。

也就是说whoami service对其他服务提供了一个统一的VIP入口，别的服务访问时会做负载均衡。

## Stack

> docker stack deploy：https://docs.docker.com/engine/reference/commandline/stack_deploy/
>
> compose-file：https://docs.docker.com/compose/compose-file/
>
> 有没有发现上述部署service很麻烦？要是能够类似于docker-compose.yml文件那种方式一起管理该多少？这就要涉及到docker swarm中的Stack，我们直接通过前面的wordpress+mysql案例看看怎么使用咯。

> (1)新建service.yml文件

```yml
version: '3'

services:

  wordpress:
    image: wordpress
    ports:
      - 8080:80
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: exampleuser
      WORDPRESS_DB_PASSWORD: examplepass
      WORDPRESS_DB_NAME: exampledb
    networks:
      - ol-net
    volumes:
      - wordpress:/var/www/html
    deploy:
      mode: replicated
      replicas: 3
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
      update_config:
        parallelism: 1
        delay: 10s

  db:
    image: mysql:5.7
    environment:
      MYSQL_DATABASE: exampledb
      MYSQL_USER: exampleuser
      MYSQL_PASSWORD: examplepass
      MYSQL_RANDOM_ROOT_PASSWORD: '1'
    volumes:
      - db:/var/lib/mysql
    networks:
      - ol-net
    deploy:
      mode: global
      placement:
        constraints:
          - node.role == manager

volumes:
  wordpress:
  db:

networks:
  ol-net:
    driver: overlay
```

> (2)根据service.yml创建service

```
docker statck deploy -c service.yml my-service
```

> (3)常见操作

```
01-查看stack具体信息
	docker stack ls
	NAME                SERVICES            ORCHESTRATOR
	my-service          2                   Swarm
	
02-查看具体的service
	docker stack services my-service
	
ID                  NAME                   MODE                REPLICAS            IMAGE               PORTS
icraimlesu61        my-service_db          global              1/1                 mysql:5.7           
iud2g140za5c        my-service_wordpress   replicated          3/3                 wordpress:latest    *:8080->80/tcp

03-查看某个service
	docker service inspect my-service-db
	
"Endpoint": {
            "Spec": {
                "Mode": "vip"
            },
            "VirtualIPs": [
                {
                    "NetworkID": "kz1reu3yxxpwp1lvnrraw0uq6",
                    "Addr": "10.0.1.5/24"
                }
            ]
        }

```

> (4)访问测试

win浏览器ip[manager,worker01,worker02]:8080
