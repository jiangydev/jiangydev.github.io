---
layout:     post
title:      "[笔记]Docker 基础知识"
subtitle:   ""
date:       2017-05-25
author:     "jiangydev"
header-img: "img/post-bg-docker.jpg"
header-mask: 0.5
catalog:    true
tags:
    - Docker
---

[TOC]

## 1 Docker 基本概念和架构

### 1.1 简介

一种虚拟化方案，`操作系统级别的虚拟化`；

决定其只能运行相同或相似内核的操作系统，依赖于Linux内核特性：Namespace和Cgroups(Control Group)。

将应用程序自动部署到容器。

鼓励使用面向服务的架构、快速高效的开发周期

### 1.2 Docker 的基本组成

- Docker 客户端/守护进程

- Docker Image 镜像：容器的基石、层叠的只读文件系统、联合加载（union mount）

- Docker Container 容器：通过镜像启动，可写层，写时复制

- Docker Registry 仓库：共有、私有

### 1.3 Docker 相关技术

#### 1 Namespaces 命名空间

编程语言：封装 -> 代码隔离
操作系统：系统资源的隔离
PID (Process ID): 进程隔离
NET (Network): 管理网络接口
IPC (InterProcess Communication): 管理跨进程通信的访问
MNT (Mount): 管理挂载点
UTS (Unix Timesharing System): 隔离内核和版本标识

#### 2 Control groups(cgroups) 控制组

用来分配资源；资源限制、优先级设定、资源计量、资源控制

#### 3 Docker 容器的能力

文件系统隔离：每个容器都有自己的root文件系统
进程隔离：每个容器都运行在自己的进程中
网络隔离：容器间的虚拟网络接口和IP地址都是分开的
资源隔离和分组：使用cgroups将CPU和内存之类的资源独立分配给每个Docker容器

### 1.4 Docker 与 OpenStack

| 类别     | Docker                | OpenStack                           |
| -------- | --------------------- | ----------------------------------- |
| 部署难度 | 非常简单              | 组件多，部署复杂                    |
| 启动速度 | 秒级                  | 分钟级                              |
| 执行性能 | 和物理系统几乎一致    | VM会占用一些资源                    |
| 镜像体积 | 镜像是MB级别          | 虚拟机是GB级别                      |
| 管理效率 | 管理简单              | 组件相互依赖，管理复杂              |
| 隔离性   | 隔离性高              | 彻底隔离                            |
| 可管理性 | 单进程、不建议启动SSH | 完整的系统管理                      |
| 网络连接 | 比较弱                | 借助Neutron可以灵活组件各类网络架构 |



## 2 Docker 安装和部署

### 2.1 Ubuntu

#### 1 安装前的检查：

```shell
# 查看内核版本，若低于3.8，需要升级内核
$ uname -a
# 检查Device Mapper
$ ls -l /sys/class/misc/device-mapper
```

#### 2 安装Docker

安装Docker维护的版本:

- 详细安装

  检查APT的HTTPS支持 查看/usr/lib/apt/methods/https文件是否存在,若不存在，运行安装命令

  ```shell
  $ apt-get update
  $ apt-get install -y apt-transport-https
  ```

  添加Docker的APT仓库

  ```shell
  $ echo deb https://get.docker.com/ubuntu docker main > /etc/apt/sources.list.d/docker.list
  ```

  添加仓库的key

  ```shell
  $ apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 36A1D7869245C8950F966E92D8576A8BA88D21E9
  ```

  安装

  ```shell
  $ apt-get update
  $ apt-get install -y lxc-docker
  ```

- 简易安装

  ```shell
  $ sudo apt-get install -y curl
  $ curl -sSL https://get.docker.com/ | sudo sh
  ```

  安装Ubuntu维护的版本

  ```shell
  $ sudo apt-get install docker.io
  $ source /etc/bash_completion.d/docker.io
  ```

#### 3 非root用户使用Docker

```shell
$ sudo groupadd docker
$ sudo gpasswd -a ${USER} docker
$ sudo service docker restart
```

### 2.2 Windows 安装Docker

Boot2Docker -> Virtual Box

### 2.3 OS X中安装

```sh
$ mkdir -p ~/.boot2docker
$ if [!-f ~/.boot2docker/boot2docker.iso];then cp /usr/local/share/boot2docker/boot2docker.iso ~/.boot2docker/ ;fi
$ boot2docker init
$ boot2docker up
$ boot2docker shellinit
$ docker version
# 进入Docker
$ boot2docker ssh
```

## 3 Docker 容器

### 3.1 容器的基本操作

- 启动容器

  ```shell
  $ docker run IMAGE [COMMAND] [ARG...]
  $ docker run -i -t --name name IMAGE /bin/bash	 # 该语句退出后生命周期结束
  $ docker run -d --name <name> nginx  			 # 后台运行
  ```

- 重新启动停止的容器

  ```shell
  $ docker start [-i] <CONTAINER ID|NAME>
  ```

- 删除容器

  ```shell
  $ docker rm <CONTAINER ID|NAME>
  ```

- 查看容器

  ```shell
  $ docker ps -a		# 查看所有容器
  $ docker ps -l  	# 查看最新创建的容器
  $ ip ad li    		# 在docker中查看网卡信息
  ```

- 进入容器

  - docker exec

  - docker attach

  - nsenter

    如果没有，那么需要安装 util-linux 软件包

    ```shell
    # 获取容器的Pid
    $ docker inspect --format "{{.State.Pid}}" <CONTAINER ID|name>
    $ nsenter --target <Pid> --mount --uts --ipc --net --pid  -> 进入容器
    ```

### 3.2 容器的导出和导入

容器导出为镜像命令：

```sh
$ docker export 容器名 > filename.tar
```

导入镜像：

```sh
$ cat filename.tar | docker import - 镜像名
```



### 3.3 守护式容器

(1) Ctrl + P   Ctrl + Q   -d

(2) 查看容器日志

```sh
$ docker logs [-f] [-t] [--tail] 容器名
    -f --follows=true | false 默认false
    -t --timestamps=true | false 默认false
    --tail ="all"
```

(3) 在运行中的容器内启动新进程

```sh
$ docker exec [-d] [-i] [-t] 容器名 [COMMAND] [ARG...]
```

(4) 容器的停止

```sh
$ docker stop <CONTAINER ID|NAME>   发送信号等待停止
$ docker kill <CONTAINER ID|NAME>   直接停止
```

### 3.4 限制容器资源

查看容器的状态：

```shell
$ docker stats 容器名
```

(1) 内存

测试工具：memload

参数：`--memory 1024m`

(2) CPU(亲和性)

CPU 亲和性（affinity）就是进程要在某个给定的 CPU 上尽量长时间地运行而不被迁移到其他处理器的倾向性。

```shell
# 查看 cpu 属性
$ lscpu
# 查看进程使用的 cpu
$ ps mo pid,comm,psr `pgrep cat`
# 指定进程使用的 cpu
$ taskset -c 0 ps
```

参数

- 使用的cpu数量: `--cpus 0.5`;
- 使用的cpu: `--cpuset-cpu 0`



### 3.5 容器监控工具(cAdvisor)

监控工具：`cAdvisor 容器`

通过挂载宿主机中的资源文件(如 /sys 等)，读取资源的状态，以达到分析的目的。

```shell
$ docker pull google/cadvisor
$ docker run \
-v /var/run:/var/run \
-v /sys:/sys:ro \
-v /var/lib/docker:/var/lib/docker:ro \
-d -p 8080:8080 google/cadvisor
```



## 4 Docker 镜像与仓库

### 4.1 查看与删除镜像

```sh
# 查看镜像
$ docker images [OPTIONS] [REPOSITORY]
    -a, --all=false
    -f, --filter=[]
    --no-trunc=false : 不截断镜像ID
    -q, --quiet=false

# 查看镜像的详细信息
$ docker inspect [OPTIONS] CONTAINER|IMAGE [CONTAINER|IMAGE...]
    -f, --format=""
# 删除镜像
$ docker rmi [OPTIONS] IMAGE [IMAGE...]
    -f, --force=false : Force removal of the image
    --no-prune=false : Do not delete untagged parents
# 删除所有ubuntu镜像
$ docker rmi $(docker images -q ubuntu)
```

### 4.2 获取和推送镜像

(1) 查找镜像

```sh
$ docker search [OPTIONS] TERM		# 最多返回25个结果
```

(2) 拉取镜像

```sh
$ docker pull [OPTIONS] NAME [:TAG]
    -a, --all-tags=false  Download all tagged images in the repository
# 镜像加速 (DAOCLOUNG | 阿里云)
$ sudo vim /etc/default/docker
# 添加：`DOCKER_OPTS= "--registry-mirror=http://MIRROR-ADDR"`
$ ps -ef | grep docker
```

(3) 推送镜像

```sh
$ docker push NAME[:TAG]
```

此时可能会报错：不安全的...，需要修改docker配置文件，加入--insecure-registry

```shell
$ vim /etc/sysconfig/docker
other_args="--insecure-registry 192.168.199.220:5001"
$ /etc/init.d/docker restart
```

(4) 标记镜像

```shell
$ docker tag NAME[:TAG] custom_name[:TAG]
```

TAG 本质为`硬链接`，镜像的 ID 都是相同的（类似于文件的硬链接，inode 相同）。 

(5) 修改默认的镜像仓库地址

修改的文件：`/etc/sysconfig/docker`，增加的参数如下：

```shell
ADD_REGISTRY='--add-registry 192.168.1.1:5000'
```

### 4.3 镜像备份和导入

镜像备份命令：

```sh
$ docker save 镜像名 > filename.tar
```

备份镜像导入：

```sh
$ docker load -i filename.tar
# 或者
$ docker load < filename.tar
```



### 4.4 构建镜像

#### 1 构建方式

(1) 通过容器构建

运行容器后，在容器内进行操作，如添加、修改或删除文件等，之后 commit 可写层即产生镜像。

```sh
$ docker commit [OPTIONS] CONTAINER [REPOSITORY[:TAG]]
  -a, --author=""  Author
  -m, --message="" Commit message
  -p, --pause=true  Pause container during commit
```

(2) 通过Dockerfile构建

```sh
$ docker build [OPTIONS] PATH | URL | -
    --force-rm=false
    --no-cache=false
    --pull=false
    -q, --quiet=false
    -t, --tag=""
```

#### 2 构建过程

>  构建完成后可以使用`docker history <image>`查看)

①从基础镜像运行一个容器；
②执行一条指令，对容器做出修改；
③再基于刚提交的镜像运行一个新容器；
④执行第②步，否则结束。
那么过程就允许中间层镜像的调试。
可以使用 `--no-cache` 不使用缓存，或者在Dockerfile中使用`ENV REFRESH_DATE`

### 4.5 Dockerfile 指令

编写 dockerfile 文件，如下：

```dockerfile
# 已经存在的镜像、基础镜像、必须是第一条非注释指令
FROM <image>[:<tag>]

# 指定镜像的作者信息，包含镜像的所有者和联系信息
MAINTAINER <name>

# 指定构建镜像时运行的命令
RUN <command>  (shell 模式)
RUN ["executable", "param1", "param2"]  (exec 模式)

# 声明指定端口，实际不会打开端口，需要在运行容器时用-p
EXPOSE <PORT> [<PORT>...]

# 运行容器时的指令
CMD ["executable", "param1", "param2"]  (exec 模式)
CMD command param1 param2  (shell 模式)
CMD ["param1", "param2"]  (作为ENTRYPOINT指令的默认参数)
*若运行容器时，指定运行指令如`/bin/bash`，则CMD命令被覆盖*

# 与CMD不同，ENTRYPOINT不会被覆盖
ENTRYPOINT ["executable", "param1", "param2"]  (exec 模式)
ENTRYPOINT command param1 param2  (shell 模式)
*可以使用 `docker run --entrypoint` 覆盖*
ENTRYPOINT 和 CMD 组合使用，CMD 提供参数。

# ADD & COPY
ADD <src>...<dest>
ADD ["<src>"..."<dest>"]  (适用于文件路径中有空格的情况)
COPY <src>...<dest>
COPY ["<src>"..."<dest>"]  (适用于文件路径中有空格的情况)
*ADD 包含类似tar的解压功能，如果单纯复制文件，使用COPY*

# 添加卷
VOLUME ["/data"]

# 为后续的操作设置工作目录
WORKDIR /path
*通常使用绝对路径，如果使用相对路径，会传递*

# 环境变量
ENV <key><value>
ENV <key>=<value>

# 运行指令的用户，默认为root
USER nginx:group

# 镜像触发器，当一个镜像被其他镜像作为基础镜像时执行，会在构建过程中插入指令
ONBUILD [INSTRUCTION]
```

构建镜像

```shell
$ docker build -t my-image:v1 .
```



## 5 Docker 客户端与守护进程

### 5.1 Docker 的 C/S 模式

Docker CLI客户端 <--> Docker 守护进程
Remote API (RESTful风格API)

连接方式：
>unix:///var/run/docker.sock (默认)
>tcp://host:port
>fd://socketfd

使用Linux的NetCat命令：

> NetCat 介绍：
>
> **主要用于调试领域、传输领域甚至黑客攻击领域**。利用该工具，可以将网络中一端的数据完整的发送至另一台主机终端显示或存储，**常见的应用为文件传输、与好友即时通信、传输流媒体或者作为用来验证服务器的独立的客户端。**

```sh
$ nc -U /var/run/docker.sock
"GET /info HTTP/1.1"
```

### 5.2 配置和操作

(1) 查看docker进程：

```sh
$ ps -ef / status docker
```

(2) 操作docker：

```sh
$ service docker stop|start|restart
```

(3) Docker 命令

```sh
$ docker -d [OPTIONS]
    -运行相关：
      -D, --debug=false
      -e, --exec-driver="native"
      -g, --graph="/var/lib/docker"
      --icc=true             # 默认true，允许inter-container来通信
      --label=[]
      -p, --pidfile="/var/run/docker.pid"
    -与服务器连接相关：
      -G, --group="docker"
      -H, --host=[]
      --tls=false
      --tlscacert="/home/sven/.docker/ca.pem"
      --tlscert="/home/sven/.docker/cert.pem"
      --tlskey="/home/sven/.docker/key.pem"
      --tlsverity=false
    -RemoteAPI相关：
      --api-enable-cors=false
    -存储相关：
      -s, --storage-driver="" # 默认是空，这是docker运行使用的一个存储驱动器
      --selinux-enabled=false
      --storage-opt=[]
    -Registry相关：
      --insecure-registry=[]
      --registry-mirror=[]
    -网络设置相关：
      --ip=0.0.0.0            # 默认"0.0.0.0"：绑定容器端口的默认Ip地址
      --ip-forward=true
      --ip-masp=true
      --iptables=true
      --ipv6=false
      --mtu=0
```

(4) 配置文件

```sh
$ vi /etc/default/docker
  DOCKER_OPTS="--label name=docker_server_1"
$ service docker restart
```

### 5.3 远程访问

客户端与服务器端的访问。

(1) 需要第二台装有Docker的服务器

修改Docker守护进程启动选项，区别服务器
保证Client API与Server API版本一致

(2) 修改服务器端配置

守护进程启动选项：

```sh
-H  tcp://host:port
    unix:///path/to/socket
    fd://* or fd://socketfd
```

默认配置：

```shell
-H  unix:///var/run/docker.sock
```

(3) 修改方式

方式一 `可以指定多个-H`

> 文件 /etc/default/docker 生效的方式请看 #6.1

```shell
$ vi /etc/default/docker
# 添加
DOCKER_OPTS="... -H tcp://0.0.0.0:2375"
```

方式二

```shell
$ export DOCKER_HOST="tcp://10.10.10.10:2375"
```



## 6 Docker 容器的网络连接

### 6.1 容器的网络基础

```shell
# 查看路由信息
$ ip ro li
# 查看网卡信息，可以看到 docker0(虚拟网桥)
$ ifconfig
# 管理以太网桥
$ apt-get install bridge-utils
```

可以自定义虚拟网桥

(1) 添加虚拟网桥

```sh
$ brctl addbr br0
$ ifconfig br0 192.168.100.1 netmask 255.255.255.0
```

(2) 配置 `/etc/default/docker` 生效

```shell
# 修改配置文件
$ vi /usr/lib/systemd/system/docker.service
EnvironmentFile=-/etc/default/docker # -代表ignore error
# ExecStart 需要追加一个值 $DOCKER_OPTS
ExecStart=$DOCKER_OPTS

# 重载
$ systemctl daemon-reload
# 重启docker服务
$ service docker restart
```

(3) 更改启动配置

```shell
$ echo 'DOCKER_OPTS="-b=br0"' >> /etc/default/docker
```

### 6.2 容器的互联

(1) 默认情况下，允许所有容器互联。 icc=true

为了解决容器间的互联不会因为IP的变化收影响，使用link的方式：

```sh
$ docker run --link=[CONTAINER NAME]:[ALIAS] [IMAGE] [COMMAND]
```

(2) 拒绝所有容器互联

```
icc=false
```

(3) 允许特定容器的连接

```
icc=false --iptables=true
--link
```

### 6.3 容器与外部网络的连接

(1) ip-forward 流量转发

```sh
--ip-forward=true
$ sysctl net.ipv4.conf.all.forwarding  # 查看流量转发是否开启，1为开启
```

(2) iptables

iptables是与Linux内核集成的包过滤防火墙系统，几乎所有的linux发行版都会包含iptables的功能。
包含表、链、规则，ACCEPT, REJECT, DROP
filter表中包含的链：INOUT, FORWARD, OUTPUT

```sh
# 默认为filter
$ iptables -t filter -L -n
# 在物理机设置过滤规则
$ iptables -I DOCKER -s 10.211.55.3 -d 172.17.0.7 -p TCP --dport 80 -j DROP
```



## 7 Docker 容器的数据管理

### 7.1 容器的数据卷 Data Volume

数据卷是经过特殊设计的目录，可以绕过联合文件系统（UFS），为一个或多个容器提供访问。

**数据卷设计的目的**

在于数据的永久化，它完全独立于容器的生存周期，因此，Docker不会在容器删除其挂载的数据卷，也不会存在类似的垃圾收集机制，对容器引用的数据卷进行处理。

**特点**
(1)数据卷可以是`目录或文件`。数据卷在容器启动时初始化，如果容器使用的镜像在挂载点包含了数据，这些数据会拷贝到新初始化的数据卷中。
(2)数据卷可以在容器之间共享和重用。
(3)可以对数据卷里的内容直接进行修改。
(4)数据卷的变化不会影响镜像的更新。
(5)卷会一直存在，即使挂载数据卷的容器已经被删除。

```sh
$ docker run -it -v ~/data:/data ubuntu /bin/bash
# 为数据卷添加访问权限
$ docker run -it -v ~/data:/data:ro ubuntu /bin/bash
```

Dockerfile 中:
> 初始化两个数据卷，此时不能实现挂载和共享
VOLUME [/data1, /data2]

### 7.2 数据卷容器

挂载数据卷容器，即使容器已经退出，仍然可用。

```sh
$ docker run --volumes-from [CONTAINER NAME]
```

### 7.3 数据卷的备份和还原

(1) 数据备份方法

```sh
$ docker run --volumes-from [CONTAINER NAME] -v $(pwd):/backup ubuntu tar cvf /backup/backup.tar [CONTAINER DATA VOLUME]
```

(2) 数据还原即解压缩

## 8 Docker 容器跨主机网络访问

### 8.1 使用网桥实现

(1) 修改 /etc/network/interfaces 文件：

```config
auto br0
iface br0 inet static
address 10.211.55.3
netmask 255.255.255.0
gateway 10.211.55.1
bridge_ports eth0
```

(2) 修改 /etc/default/docker 文件

```sh
-b 指定使用自定义网桥 `-b=br0`
--fixed-cidr 限制ip地址分配范围
  HOST1: 10.211.55.64/26
  HOST2: 10.211.55.128/26
```

*配置简单，不依赖第三方软件；与主机同网段，需要有网段控制权，不易实现和管理，兼容性不佳*

### 8.2 Open vSwitch 实现

Open vSwitch：高质量的、多层虚拟交换机，开源；使得大规模网络自动化可以通过编程扩展，同时仍然支持标准的管理接口和协议。

GRE （通用路由协议封装）

隧道技术：是一种通过使用互联网络的基础设施在网络间传递数据的方式。使用隧道传递的数据（或负载）可以是不同协议的数据帧或包。隧道协议将其他协议的数据帧或包重新封装后通过隧道发送。新的帧头提供路由信息，以便通过互联网传递被封装的负载数据。

```sh
$ apt-get install openvswitch-switch
$ apt-get install bridge-utils
```

**步骤**

* 建立ovs网桥 

    ```shell
    $ ovs-vsctl add-br obr0
    ```

* 添加gre连接

    ```sh
    # 在obr0网桥下添加两个Port
    $ ovs-vsctl add-port obr0 gre0
    $ ovs-vsctl set interface gre0 type=gre options:remote_ip=192.168.59.104
    
    $ brctl addbr br0
    $ ifconfig br0 192.168.1.1 netmask 255.255.255.0
    $ brctl addif br0 obr0
    ```

* 配置docker容器虚拟网桥

* 为虚拟网桥添加ovs接口

* 添加不同docker容器网段路由
    ```sh
    $ ip route add 192.168.2.0/24 via 192.168.59.104 dev eth0
    ```

### 8.3 weave 实现

weave: 编织，建立一个虚拟的网络，用于将运行在不同主机的Docker容器连接起来。

```sh
$ wget -O /usr/bin/weave https://raw.githubusercontent.com/zettio/weave/master/weave
$ chmod a+x /usr/bin/weave
$ weave launch (IP地址)

$ c2=$(weave run 192.168.1.2/24 -it ubuntu /bin/bash)  # 启动容器中包含一个ethwe
```



## 9 Docker 资源隔离和限制

### 9.1 资源隔离 LXC

Kernel namespace

pid: 进程隔离，可以嵌套
net: 网络隔离
ipc: 进程间交互隔离
mnt: 进程挂载在目录
uts: 每一个容器有hostname，domain
user: 用户及组

### 9.2 资源限制 Cgroup

即使个别容器崩溃，不影响。

CPU 内存 （磁盘暂时不行）

### 9.3 压力测试

Docker 每个容器默认会有1024 shares（相对拥有物理100% CPU），当有多个容器，配额会按比例分配。

查看当前CPU信息

```shell
$ cat /proc/cpuinfo
```

压力测试工具：`stress`

```shell
# 可以使用 stress 镜像测试
$ docker pull progrium/stress
# 测试样例
$ docker run --rm -it progrium/stress --cpu 2 --io 1 --vm 2 --vm-bytes 128M --timeout 10s
```



## 10 Docker Dashboard

目前 Docker Web  页面管理的平台：

- [Portainer](https://github.com/portainer/portainer)
- [Rancher](https://github.com/rancher/rancher)
- [Docker UI](https://github.com/kevana/ui-for-docker) 项目不再维护

Shipyard 项目已经从 GitHub 中删除，这里不考虑。

### 10.1 Portainer

> 参考文献：
>
> [官方文档：Portainer Docs](https://portainer.readthedocs.io/)

一个开源的轻量级的 Docker 管理工具。

### 10.2 Rancher

> 参考文献：
>
> [官方文档：Rancher 1.6 Docs](https://rancher.com/docs/rancher/latest/zh/)

官方文档有中文版，只建议不要用配置过低的主机做 rancher server，会无响应。
