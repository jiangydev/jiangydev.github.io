---
layout:     post
title:      "Docker 中的 gosu"
subtitle:   ""
date:       2017-02-18
author:     "jiangydev"
header-img: "img/post-bg-docker.jpg"
header-mask: 0.5
catalog:    true
tags:
    - Docker
---

[TOC]

# sudo or gosu

> 参考文献
>
> 1. [sudo or gosu](https://segmentfault.com/a/1190000004527476)

官方：Docker 官方文档的 Dockerfile 部分，有一节讲的是 `ENTRYPOINT`，提到了如果在启动脚本中需要指定运行命令的用户，建议用`gosu`代替`sudo`。

- gosu 避免了 strange and often annoying TTY and signal-forwarding behavior

- sudo 缺点(至少以下两点)

  - sudo 会作为被授权的命令的父进程一直存在，直到该命令退出。

  - sudo 模式下的 HOME 环境变量仍是用 sudo 者原来的值。

    ```
    ~ sudo ps -o pid,ppid,cmd
      PID  PPID CMD
    12599  4281 sudo ps -o pid,ppid,cmd
    12600 12599 ps -o pid,ppid,cmd
    ~ sudo env | grep HOME
    HOME=/home/lzx
    ```

- sudo and su
  sudo 跟 su 不同，在于 sudo 是对使用者鉴权，而 su 是对目标权限进行鉴权。假定你是 sudoer，运行 sudo时你要输入自己的密码，也即证明自己有扮演的权限；而运行 su 时，你要输入的是要扮演的用户的密码，也即证明你有扮演的那个用户的权限。所以 sudo 会认为，那你使用 sudo 只是想临时使用某一身份。

