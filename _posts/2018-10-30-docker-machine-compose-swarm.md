---
layout:     post
title:      "Docker 三剑客"
subtitle:   "docker-machine, docker-compose, docker swarm"
date:       2018-10-30
author:     "jiangydev"
header-img: "img/post-bg-docker.jpg"
header-mask: 0.5
catalog:    true
tags:
    - Docker
---

[TOC]

## 1 Docker Machine

> 参考文献
>
> [https://docs.docker.com/machine/](https://docs.docker.com/machine/)

用 Docker Machine 可以在不同环境下批量安装和配置 Docker Host，包括：

- 常规 Linux 操作系统
- 虚拟化平台 - VirtualBox、VMWare、Hyper-V
- OpenStack
- 公有云 - Amazon Web Services、Microsoft Azure、Google Compute Engine、Digital Ocean等



### 1.1 docker-machine 安装

下载地址：[https://github.com/docker/machine/releases/][1]

> 小提示：
>
> 如果你使用的 Mac 或 Windows，可以下载安装最新版  [Docker for Mac][2], [Docker for Windows][3] 或 [Docker Toolbox][2]（安装包含了最新的 Docker Engine, Compose 和 Kitematic）。

OS X 平台

```shell
$ curl -L https://github.com/docker/machine/releases/download/v0.15.0/docker-machine-$(uname -s)-$(uname -m) >/usr/local/bin/docker-machine 
    && chmod +x /usr/local/bin/docker-machine
```

Linux 平台

```shell
$ curl -L https://github.com/docker/machine/releases/download/v0.15.0/docker-machine-$(uname -s)-$(uname -m) >/tmp/docker-machine 
    && chmod +x /tmp/docker-machine 
    && sudo cp /tmp/docker-machine /usr/local/bin/docker-machine
```

Windows 平台（git-bash 环境，可安装 [Git for Windows](https://git-for-windows.github.io/)）

```shell
$ mkdir -p "$HOME/bin" 
    && curl -L https://github.com/docker/machine/releases/download/v0.15.0/docker-machine-Windows-x86_64.exe > "$HOME/bin/docker-machine.exe" 
    && chmod +x "$HOME/bin/docker-machine.exe"
```

安装后测试是否安装成功

```shell
$ docker-machine -v
# 出现如下信息，则安装成功
docker-machine version 0.15.0, build b48dc28d
```



### 1.2 使用 docker-machine 创建虚拟机

#### 1.2.1 使用本地 VM

如果是 Windows 10 专业版（只要支持 Hyper-V 的版本就行），自带 VM Hyper-V。

```shell
# -d 表示 driver，default 是创建的 VM 名称
$ docker-machine create -d hyperv default
```

不支持 Hyper-V 的 Windows OS，可以安装 VirtualBox 或 VMWare 等 Docker Machine 支持的虚拟机。

Mac OSX 也可以安装支持的虚拟机。

以 VirtualBox 为例。

- 创建 docker machine 环境

    ```powershell
    $ docker-machine create -d virtualbox default
    ```

    > 小提示：
    >
    > 创建 Docker Machine 的过程中，可能需要更新 Boot2Docker ISO，若更新失败，可以手动下载最新的`boot2docker.iso`并替换过时的 ISO。
    >
    > 下载最新 ISO 和替换的目录可以看 docker-machine create 的输出信息，例如：
    >
    > ```
    > Downloading C:\Users\JY\.docker\machine\cache\boot2docker.iso from https://github.com/boot2docker/boot2docker/releases/download/v18.06.1-ce/boot2docker.iso...
    > ```
    >
    > 那么可以从 `https://github.com/boot2docker/boot2docker/releases/`下载后替换`C:\Users\JY\.docker\machine\cache\`目录下的`boot2docker.iso`。

- docker-machine 命令

    ```powershell
    # 列出 VM
    $ docker-machine ls
    # 获得主机 IP
    $ docker-machine ip default
    # 查看 default 的环境变量
    $ docker-machine env default
    SET DOCKER_TLS_VERIFY=1
    SET DOCKER_HOST=tcp://192.168.99.100:2376
    SET DOCKER_CERT_PATH=C:\Users\HS\.docker\machine\machines\default
    SET DOCKER_MACHINE_NAME=default
    SET COMPOSE_CONVERT_WINDOWS_PATHS=true
    REM 以下输出信息，用于提示设置上面的环境变量到系统中
    REM Run this command to configure your shell:
    REM     @FOR /f "tokens=*" %i IN ('docker-machine env default') DO @%i
    ```

    > 注意：
    >
    > 通过`docker-machine env default`返回的命令设置环境变量，可以简化 VM 的登录操作。
    >
    > 之所以能免密登录 VM，查看`~/.docker/machine/machines/<VM 名称>/`目录下的密钥就明白了。

- 进入 default VM

    ```shell
    $ docker-machine ssh default
    ```

- 停止和启动 VM

    ```shell
    $ docker-machine stop default
    $ docker-machine start default
    ```



#### 1.2.2 使用云平台 VM

以阿里云为例（目的是走一套流程），若是腾讯云等无 Docker 驱动支持的云平台，可以不装驱动，直接以 `-d generic` 驱动创建主机环境。

阿里云Docker Machine驱动项目地址：[https://github.com/denverdino/docker-machine-driver-aliyunecs][5]

##### 1 安装云平台驱动

驱动下载地址：[docker-machine-driver-aliyunecs][6]

下载后解压，并修改文件名为`docker-machine-driver-aliyunecs.exe`(Windows 下修改后缀为 `.exe`)，放在 docker 的 path 目录中。

> 注意：
>
> 若 Windows 下出现如下信息，除了确保该驱动已放在 PATH 下，请检查是否修改插件的后缀为 `.exe`，并重新打开 CMD 或 bash。
>
> ```shell
> $ docker-machine create -d aliyunecs --help
> Driver "aliyunecs" not found. Do you have the plugin binary "docker-machine-driver-aliyunecs" accessible in your PATH?
> ```



##### 2 在云平台创建主机

目的是创建一个远程的安装好 Docker Engine 的 ECS。

- 阿里云平台

  创建主机必须要`ECS_ACCESS_KEY_ID` 和 `ECS_ACCESS_KEY_SECRET`，更重要的是要有余额！

  阿里云的账户分为主账户和子账户，为了安全性，这里我使用子账户（要赋予管理云服务器ECS的权限）。

  主账户管理地址：[https://usercenter.console.aliyun.com/][7]

  子账户管理地址：[https://ram.console.aliyun.com/][8]

  ```shell
  $ docker-machine create -d aliyunecs \
      --aliyunecs-access-key-id=<Your access key ID for the Aliyun ECS API> \
      --aliyunecs-access-key-secret=<Your secret access key for the Aliyun ECS API> \
      --aliyunecs-region=<Region> \
      --aliyunecs-image-id centos_7_04_64_20G_alibase_201701015.vhd \
      --aliyunecs-instance-type ecs.n4.small \
      --aliyunecs-disk-size 40 \
      --aliyunecs-ssh-password='1218' \
      <machine-name>
      
  # 查看详细参数
  $ docker-machine create -d aliyunecs --help
  ```

  更多参数及说明（默认值）可以查看[项目地址][5]。

  像 `--aliyunecs-region`，`--aliyunecs-instance-type`等参数可以在 ECS 的购买页找到，如图。 

  ![docker-aliyunecs](/img/in-post/docker/docker-aliyunecs.png)

  > 小提示：
  >
  > 如果创建过程中出错，无法通过 `docker-machine rm` 删除，可以直接删除 `~/.docker/machine/machines/<需要删除的 VM 名称>`。
  >
  > ```shell
  > $ rm -rf ~/.docker/machine/machines/aliyunecs-test
  > ```

- 云平台的通用方法

  - Docker Machine create 要求免密登录远程主机，使用如下命令将 SSH key 拷贝到目标主机。

    ```shell
    $ ssh-copy-id -i ~/.ssh/id_rsa root@1.2.3.4
    ```

  - 创建远程主机环境

    该过程会在`~/.docker/machine/machines/`下创建 tencent-test 目录，并复制 `~/.ssh/id_rsa`和`~/.ssh/id_rsa.pub`到该目录下。

    ```shell
    $ docker-machine create --driver generic --generic-ip-address 1.2.3.4 --generic-ssh-key ~/.ssh/id_rsa tencent-test
    ```

    > 注意：
    >
    > 1. 该步骤会多次通过 SSH 方式向远程终端发送命令（只会影响远程终端的 Docker Engine），密钥设置了 passphrase，则在创建远程主机的过程中需要反复输入！
    > 2. 如果是 Windows 用户，请使用 git-bash 环境。
    > 3. 检查云平台安全规则是否放行 2376，这是默认的连接端口。



##### 3 连接远程主机

只能连接 Docker Machine 管理的主机。

```shell
$ docker-machine ssh tencent-test
$ docker-machine ssh default
```



## 2 Docker Compose

> 参考文献
>
> [https://docs.docker.com/compose/install/](https://docs.docker.com/compose/install/)

Compose 是一个定义和运行多Docker容器应用的工具。可以访问官网查看最新的安装方式。

该部分实战可以参考我的另一篇博客：[搭建一个技术博客 #3.2 Docker 及 docker-compose 安装][9]

案例中构建了 Jekyll + Jenkins + Nginx 的多容器环境。



## 3 Docker Swarm

> 参考文献
>
> [https://docs.docker.com/engine/swarm/](https://docs.docker.com/engine/swarm/)

要求 Docker Engine 1.12 或更新的版本。

### 3.1 swarm 特性

- **集群管理与Docker Engine集成：**使用Docker Engine CLI创建一大堆Docker引擎，可以在其中部署应用程序服务，无需其他编排软件即可创建或管理群组。
- **分散式设计：** Docker Engine不是在部署时处理节点角色之间的区别，而是在运行时处理。您可以使用Docker Engine部署两种节点，管理器和工作器。这意味着您可以从单个磁盘映像构建整个swarm。
- **声明性服务模型：** Docker Engine使用声明性方法来定义应用程序堆栈中各种服务的所需状态。例如，可以描述由具有消息队列服务的Web前端服务和数据库后端组成的应用程序。
- **缩放：**对于每个服务，可以声明要运行的任务数。当向上或向下扩展时，群集管理器会通过添加或删除任务来自动调整以维持所需的状态。
- **期望的状态协调：**群集管理器节点持续监视群集状态，并协调实际状态与表达的所需状态之间的任何差异。例如，如果设置服务以运行容器的10个副本，并且托管其中两个副本的工作计算机崩溃，则管理器将创建两个新副本以替换崩溃的副本。群集管理器将新副本分配给正在运行且可用的工作线程。
- **多主机网络：**可以为服务指定覆盖网络。群集管理器在初始化或更新应用程序时自动为覆盖网络上的容器分配地址。
- **服务发现：** Swarm管理器节点为swarm中的每个服务分配一个唯一的DNS名称，并负载平衡正在运行的容器。可以通过嵌入在swarm中的DNS服务器查询swarm中运行的每个容器。
- **负载平衡：**可以将服务端口公开给外部负载平衡器。在内部，swarm允许您指定如何在节点之间分发服务容器。
- **默认安全：** swarm中的每个节点都强制执行TLS相互身份验证和加密，以保护自身与所有其他节点之间的通信。可以选择使用自签名根证书或自定义根CA的证书。
- **滚动更新：**在启动时，可以逐步将服务更新应用于节点。通过swarm管理器，可以控制服务部署到不同节点集之间的延迟。如果出现任何问题，可以将任务回滚到以前版本的服务。

### 3.2 创建 swarm 集群

#### 3.2.1 集群情况说明

- manager：一台远程主机（tencent-test）
- worker：两台本地 VirtualBox VM（swm0-worker-local-vb-0，swm0-worker-local-vb-1）
- 放行端口
  - 集群管理端口：TCP 2377
  - 节点间的通讯端口：TCP和UDP 7946
  - 网络间流量端口：TCP 4789

#### 3.2.2 初始化管理节点

```shell
# 查看 manager 的 IP
$ docker-machine env tencent-test
$ docker-machine ssh tencent-test

> docker swarm init --advertise-addr 123.206.190.189
Swarm initialized: current node (nsv3bw1fciqhgu6r9s6pmfve0) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-4jjyfmqjkxwvh6pros81wy9z7hmd8wwhjrtinq1si24k1mqj4e-1wdc7768nuforapprt3wl2dex \
    123.206.190.189:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

### 3.3 向 swarm 增加节点

> 小提示：
>
> 记得要放行2377，7946，4789端口。

- 创建主机并进入，然后以 worker 加入 swarm。

  ```shell
  $ docker-machine create -d virtualbox swm0-worker-local-vb-0
  $ docker-machine ssh swm0-worker-local-vb-0
  > docker swarm join --token SWMTKN-1-4jjyfmqjkxwvh6pros81wy9z7hmd8wwhjrtinq1si24k1mqj4e-1wdc7768nuforapprt3wl2dex 123.206.190.189:2377
  This node joined a swarm as a worker.
  ```

- 如果忘记了加入命令，可以在管理节点运行如下命令。

  ```shell
  $ docker swarm join-token worker
  ```

- 在管理节点上查看和修改集群节点状态。

  ```shell
  # 列出 swarm 中的节点
  $ docker node ls
  ID                           HOSTNAME                STATUS  AVAILABILITY  MANAGER STATUS
  59a1igkgktdrlif3r3vrinyxp    swm0-worker-local-vb-0  Ready   Active
  nsv3bw1fciqhgu6r9s6pmfve0 *  tencent-test            Ready   Active        Leader
  qcrbd1n93gqxmqinv711hfr7p    swm0-worker-local-vb-1  Ready    Active
  ```

  > 小提示：
  >
  > - `docker node ls` 只能在 swarm 管理节点上使用。
  > - node ID 旁边的 * 号表示当前连接到的节点。

### 3.4 向 swarm 部署一个服务

- 进入 manager 节点，并部署一个服务。

  ```shell
  $ docker-machine ssh tencent-test
  > docker service create --replicas 1 --name helloworld alpine ping docker.com
  k6qyjdr8nvir8g1iwep7lze6h
  overall progress: 1 out of 1 tasks
  1/1: running   [==================================================>]
  verify: Service converged
  ```

  - 命令 `docker service create` 创建了服务。
  -  `--name` 命名了服务 `helloworld`.
  -  `--replicas` 指定了期望 1 运行实例.
  - 参数  `alpine ping docker.com` 定义了服务以 Alpine Linux 容器执行命令 `ping docker.com`.


- 查看服务列表。

  ```shell
  > docker service ls
  ID            NAME        MODE        REPLICAS  IMAGE
  k6qyjdr8nvir  helloworld  replicated  1/1       alpine:latest
  ```


### 3.5 检查 swarm 上的服务

- 进入 manager 节点，并查看服务的详细信息。

  ```shell
  $ docker-machine ssh tencent-ssh
  # 如果不加 “--pretty”，则以 json 格式显示
  > docker service inspect --pretty helloworld
  ```

- 查看运行服务的节点。

  ```shell
  > docker service ps helloworld
  ID           NAME          IMAGE          NODE         DESIRED STATE  CURRENT STATE     ERROR        PORTS
  pa5nivr0vt4m helloworld.1  alpine:latest  tencent-test Running        Running 18 minutes ago
  ```

  在上面的案例中，helloworld 服务运行在 tencent-test 节点上，可以看到服务的状态、错误和端口。

### 3.6 服务的缩放

运行服务的容器称为`tasks`，可以缩放服务的容器数量。

- 进入 manager 节点，并改变服务期望的状态

  语法

  ```shell
  $ docker service scale <SERVICE-ID>=<NUMBER-OF-TASKS>
  ```

  例子

  ```shell
  $ docker-machine ssh tencent-test
  > docker service scale helloworld=5
  ```

- 查看运行服务的节点。

  ```shell
  > docker service ps helloworld
  ```

### 3.7 服务的删除

- 进入 manager 节点，并删除服务

  ```shell
  $ docker-machine ssh tencent-test
  > docker service rm helloworld
  ```

### 3.8 服务的滚动更新

该部分将部署一个基于 Redis 3.0.6 容器镜像的服务，然后使用滚动更新升级 Redis 容器镜像到 3.0.7。

- 进入 manager 节点，并部署  Redis 3.0.6

  ```shell
  $ docker-machine ssh tencent-test
  > docker service create \
    --replicas 3 \
    --name redis \
    --update-delay 10s \
    redis:3.0.6
  ```

  在部署时配置滚动更新策略。

  `--update-delay` 配置更新的延迟时间，可以使用`s`，`m`，`h` 为时间单位（支持组合，如`10m30s`）。

  默认情况下，调度程序一次更新1个任务。您可以传递该 `--update-parallelism`标志以配置调度程序同时更新的最大服务任务数。

  默认情况下，当对单个任务的更新返回状态时 `RUNNING`，调度程序会安排另一个任务进行更新，直到所有任务都更新为止。如果在更新任务期间的任何时间返回`FAILED`，则调度程序会暂停更新。您可以使用或`--update-failure-action`标志来控制行为 。

- 检查服务的详细信息

  ```shell
  > docker service inspect --pretty redis
  ID:             ga4ngquo6x1p0f3ejlh64yfxc
  Name:           redis
  Service Mode:   Replicated
   Replicas:      3
  Placement:
  UpdateConfig:
   Parallelism:   1
   Delay:         10s
   On failure:    pause
   Monitoring Period: 5s
   Max failure ratio: 0
   Update order:      stop-first
  RollbackConfig:
   Parallelism:   1
   On failure:    pause
   Monitoring Period: 5s
   Max failure ratio: 0
   Rollback order:    stop-first
  ContainerSpec:
   Image:         redis:3.0.6
   Init:          false
  Resources:
  Endpoint Mode:  vip
  ```

- 更新容器镜像

  swarm 会根据 `UpdateConfig` 中的策略将更新应用到节点。

  ```shell
  >  docker service update --image redis:3.0.7 redis
  ```

  调度程序默认情况下的滚动更新原则，如下所示：

  - 停止第一个任务。
  - 安排已停止任务的更新。
  - 启动容器以获取更新的任务。
  - 如果对任务的更新返回`RUNNING`，等待指定的延迟时间，然后开始下一个任务。
  - 如果在更新期间的任何时间返回任务`FAILED`，暂停更新。

- 查看节点滚动更新

  ```shell
  > docker service ps redis
  ID           NAME       IMAGE         NODE                    DESIRED STATE       CURRENT STATE          ERROR    PORTS
  ky9xtn2t7f9k redis.1    redis:3.0.7   swm0-worker-local-vb-0  Running             Running 5 minutes ago
  allh9nviqxke redis.2    redis:3.0.7   tencent-test            Running             Running 4 minutes ago
  qqtw4oo6nqo4 redis.3    redis:3.0.7   tencent-test            Running             Running 4 minutes ago
  ```

- 重新启动暂停的更新

  ```shell
  > docker service update redis
  ```

### 3.9 在 swarm 下线一个节点

有时需要下线一个节点，例如计划的维护。`DRAIN` 可以防止一个节点从 swarm manager 收到新的 tasks，也意味着 manager 停止节点上的 tasks（容器） 并在 `ACTIVE` 的节点上启动重复的 tasks。

> 注意点：
>
> 把节点设置为 `DRAIN`，不是从节点上移除容器。`DRAIN` 是一个节点的状态，仅仅影响计划的 swarm 服务的负载。

- 进入 manager 节点，并验证所有节点的 active 状态

  ```shell
  $ docker-machine ssh tencent-test
  > docker node ls
  ```

- 重新创建 redis 服务

  ```shell
  > docker service create --replicas 3 --name redis --update-delay 10s redis:3.0.6
  ```

- 查看 manager 指派到不同节点的 tasks

  ```shell
  > docker service ps redis
  ID            NAME     IMAGE        NODE                    DESIRED STATE       CURRENT STATE           ERROR    PORTS
  xlwppubi4rgr  redis.1  redis:3.0.6  tencent-test            Running             Running 24 seconds ago
  rjp003vkd5x7  redis.2  redis:3.0.6  swm0-worker-local-vb-0  Running             Running 20 seconds ago
  axxmyjxr583z  redis.3  redis:3.0.6  swm0-worker-local-vb-0  Running             Running 19 seconds ago
  ```

  在案例中，manager 节点运行了一个 task，1 个 worker 节点运行了两个 tasks。

- 下线一个 worker 节点

  ```shell
  > docker node update --availability drain swm0-worker-local-vb-0
  ```

- 重新上线一个 worker 节点

  ```shell
  > docker node update --availability active swm0-worker-local-vb-0
  ```

### 3.10 使用 swarm 模式 routing mesh

Docker Engine swarm 模式可以轻松地发布服务端口，使其可用于 swarm 之外的资源。所有的节点都参与 ingress routing mesh。Routing mesh 允许集群中的每个节点接受已发布端口上的连接（即使节点上没有任何在运行的 tasks）。Routing mesh 将所有传入的请求路由到可用节点上的已发布端口到活动容器中。

要在 swarm 中使用 ingress 网络，需要在启用 swarm 模式之前在节点之间打开以下端口：

- TCP / UDP 端口`7946`用于容器网络发现。
- `4789`容器 ingress 网络的UDP端口。

#### 3.10.1 发布服务端口

语法

```shell
$ docker service create \
  --name <SERVICE-NAME> \
  --publish published=<PUBLISHED-PORT>,target=<CONTAINER-PORT> \
  <IMAGE>
```

- `--publish`标志在创建服务时发布端口。
- `target` 用于指定容器内的端口。

> 注意点：
>
> 1. 该语法的旧形式为冒号分隔，如`-p 80:8080`，该新语法是首选，具有可读性和灵活性。
> 2. `<PUBLISHED-PORT>` 是必需的，如果不指定，则指派一个随机高编号端口。

实例，将 nginx 容器中的 80 端口发布到群集中任何节点的 8080 端口：

```shell
$ docker service create \
  --name my-web \
  --publish published=8080,target=80 \
  --replicas 2 \
  nginx
```

此时，在任意节点访问 8080 端口，Docker 会将你的请求路由到一个 active 容器，Routing mesh 知道如何路由流量并防止任何端口冲突。

![ingress-routing-mesh](/img/in-post/docker/ingress-routing-mesh.png)

可以通过以下命令增加（删除）服务发布的端口。

```shell
$ docker service update \
  --publish-add published=<PUBLISHED-PORT>,target=<CONTAINER-PORT> \
  <SERVICE>
# --publish-rm # 删除发布的端口
```

查看服务发布的端口信息。

```shell
$ docker service inspect --format="{{json .Endpoint.Spec.Ports}}" my-web
```

#### 3.10.2 发布仅 TCP 或 仅 UDP 端口

默认情况下是 TCP，可以指定为 UDP。

**长语法:**

```shell
$ docker service create --name dns-cache \
  --publish published=53,target=53 \
  --publish published=53,target=53,protocol=udp \
  dns-cache
```

**短语法:**

```shell
$ docker service create --name dns-cache \
  -p 53:53 \
  -p 53:53/udp \
  dns-cache
```

#### 3.10.3 绕过 routing mesh

你可以绕过 routing mesh，以便访问节点上绑定端口时，始终访问该节点上运行的服务实例，这被称为 `host` 模式。

需要记住，如果一个节点上有多个服务实例 tasks，此时无法指定静态目标端口。

```shell
$ docker service create --name dns-cache \
  --publish published=53,target=53,protocol=udp,mode=host \
  --mode global \
  dns-cache
```

#### 3.10.4 配置额外的负载均衡器

可以与 routing mesh 组合使用，也可以不适用 routing mesh。

- 组合 routing mesh 使用

  例如，可以配置 [HAProxy][10] 以平衡发布到端口 8080 的 Nginx 服务的请求。

  ![ingress-lb](/img/in-post/docker/ingress-lb.png)

  可以配置负载平衡器以平衡群中每个节点之间的请求，即使节点上没有安排 tasks 也是如此。例如，可以在以下位置进行以下HAProxy配置`/etc/haproxy/haproxy.cfg`：

  ```
  global
          log /dev/log    local0
          log /dev/log    local1 notice
  ...snip...
  
  # Configure HAProxy to listen on port 80
  frontend http_front
     bind *:80
     stats uri /haproxy?stats
     default_backend http_back
  
  # Configure HAProxy to route requests to swarm nodes on port 8080
  backend http_back
     balance roundrobin
     server node1 192.168.99.100:8080 check
     server node2 192.168.99.101:8080 check
     server node3 192.168.99.102:8080 check
  ```

  可以配置任何类型的负载均衡器以将请求路由到群集节点。要了解有关HAProxy的更多信息，请参阅[HAProxy文档](https://cbonte.github.io/haproxy-dconv/)。

- 不使用 routing mesh

  要使用不带路由网格的外部负载平衡器，设置`--endpoint-mode` 为`dnsrr (DNS Round Robin)`，而不是默认值`vip`。在这种情况下，没有虚拟IP。相反，Docker为服务设置DNS条目，以便服务名称的DNS查询返回IP地址列表，客户端直接连接到其中一个。

  需要向负载均衡器提供IP地址和端口列表。可参考 [配置服务发现][11]。





[1]: https://github.com/docker/machine/releases/
[2]: https://docs.docker.com/docker-for-mac/
[3]: https://docs.docker.com/docker-for-windows/
[4]: https://www.docker.com/docker-toolbox
[5]: https://github.com/denverdino/docker-machine-driver-aliyunecs
[6]: https://github.com/AliyunContainerService/docker-machine-driver-aliyunecs#installation
[7]: https://usercenter.console.aliyun.com/
[8]: https://ram.console.aliyun.com/
[9]: https://jiangydev.github.io/2018/10/17/build-a-blog/#32-docker-%E5%8F%8A-docker-compose-%E5%AE%89%E8%A3%85
[10]: http://www.haproxy.org/
[11]: https://docs.docker.com/engine/swarm/networking/#configure-service-discovery

