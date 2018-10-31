---
layout:     post
title:      "Docker 三剑客"
subtitle:   "docker-machine, docker-compose, docker-swarm"
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

用 Docker Machine 可以在不同环境下批量安装和配置 Docker Host，包括：

- 常规 Linux 操作系统
- 虚拟化平台 - VirtualBox、VMWare、Hyper-V
- OpenStack
- 公有云 - Amazon Web Services、Microsoft Azure、Google Compute Engine、Digital Ocean等

> 参考文献
>
> [https://docs.docker.com/machine/](https://docs.docker.com/machine/)



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
    $ docker-machine create --driver generic --generic-ip-address 1.2.3.4 --generic-ssh-key="~/.ssh/id_rsa" tencent-test
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
> [https://docs.docker.com/compose/](https://docs.docker.com/compose/)

Compose 是一个定义和运行多Docker容器应用的工具。

该部分可以参考我的另一篇博客：[搭建一个技术博客 #3.2 Docker 及 docker-compose 安装][9]

案例中构建了 Jekyll + Jenkins + Nginx 的多容器环境。



## 3 Docker Swarm

> 参考文献
>
> [https://docs.docker.com/engine/swarm/](https://docs.docker.com/engine/swarm/)





[1]: https://github.com/docker/machine/releases/
[2]: https://docs.docker.com/docker-for-mac/
[3]: https://docs.docker.com/docker-for-windows/
[4]: https://www.docker.com/docker-toolbox
[5]: https://github.com/denverdino/docker-machine-driver-aliyunecs
[6]: https://github.com/AliyunContainerService/docker-machine-driver-aliyunecs#installation
[7]: https://usercenter.console.aliyun.com/
[8]: https://ram.console.aliyun.com/
[9]: https://jiangydev.github.io/2018/10/17/build-a-blog/#32-docker-%E5%8F%8A-docker-compose-%E5%AE%89%E8%A3%85