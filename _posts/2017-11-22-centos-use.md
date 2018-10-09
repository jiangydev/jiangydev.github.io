---
layout:     post
title:      "CentOS7 相关配置及使用"
subtitle:   ""
date:       2017-11-22
author:     "jiangydev"
header-img: "img/post-bg-unix-linux.jpg"
header-mask: 0.3
catalog:    true
tags:
    - CentOS
    - Linux
---

[TOC]

# CentOS 相关配置及使用

## 静态网络配置

1. 配置文件：`/etc/sysconfig-scripts/ifcfg-xxx`

2. 修改已有内容
```
# 设置静态
BOOTPROTO=static
# 系统启动时加载该配置
ONBOOT=yes
```

3. 添加静态配置
```
IPADDR=192.168.XX.XX
NETMASK=255.255.255.0
GATEWAY=192.168.XX.XX
DNS1=192.168.XX.XX
```

4. 重启下网络服务

   ```shell
   $ service network restart
   ```


## 更换YUM源

1. 首先备份`/etc/yum.repos.d/CentOS-Base.repo`

   ```shell
   $ mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
   ```

2. 下载对应版本的repo文件

   ```shell
   $ wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.163.com/.help/CentOS7-Base-163.repo http://mirrors.aliyun.com/repo/Centos-7.repo
   ```

3. 运行以下命令生成缓存

   ```shell
   $ yum clean all
   $ yum makecache
   ```
