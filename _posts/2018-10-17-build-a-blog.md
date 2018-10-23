---
layout:     post
title:      "搭建一个属于自己的博客"
subtitle:   "博客本地环境，Github托管，自动化部署等"
date:       2018-10-17
author:     "jiangydev"
header-img: "img/post-bg-cloud.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Blog
    - Docker
---

[TOC]

# 搭建一个技术博客

在朋友圈看见 SpringCloud 中国社区大佬 [方志朋][2] 推送了微文 [如何搭建自己的个人博客][3] ，才想起来自己也应该写一篇文章。

另一位大佬站点：[CodeSheep][13]。

金九银十，在9月中旬，我搭建了自己的博客，主要是之前的`学习笔记`和项目执行过程中遇到的 `BUG 解决方案`。

## 1 使用GitHub托管博客

在GitHub上搭建博客很简单，主要步骤如下。

> 小提示
>
> 为了学习，我自己创建了 repo，分别 clone 自己的 repo 和 huxpro 的主题到本地，复制修改后 push 到 github 上。
>
> 如果觉得繁琐，你可以直接 fork huxpro 的主题，rename repo 后 clone 到本地修改博文内容后 push。

### 1.1 创建 repo

用 GitHub 账号创建一个`<github-username>.github.io`的 repository 即可（GitHub Pages 会自动构建），例如我创建的 [jiangydev.github.io][1]，（新构建的repo）此时访问该站点如下。

![github-pages-404](/img/in-post/blog/github-pages-404.png)

### 1.2 选择合适的 Jekyll 主题

这里我选择的是 [huxpro][4]开源的主题。

clone 项目至本地。

```shell
$ git clone git@github.com:Huxpro/huxpro.github.io.git
$ git clone git@github.com:jiangydev/jiangydev.github.io.git
```

复制 huxpro.github.io 内的文件到自己的项目中。

比较重要的文件（或目录）作用如下表：

| 文件/目录名 | 作用                                                         |
| ----------- | ------------------------------------------------------------ |
| _includes   | 页面需要包含的公共模块，含"About"页面                        |
| _posts      | 博文，文件格式为 Markdown，文件名格式如"2018-10-17-your-artical-name.md" |
| _config.yml | 博客的一些属性配置，如博客名，评论模块等                     |
| CNAME       | 自己的域名。填写后，访问 `<github-username>.github.io`会跳转到该域名 |

### 1.3 创建你的第一篇博文

新建一个 Markdown 文本，文件名`2018-10-17-build-a-blog.md`，内容如下。

```markdown
---
layout:     post
title:      "搭建一个属于自己的博客"
subtitle:   "博客本地环境，Github托管，自动化部署等"
date:       2018-10-17
author:     "jiangydev"
header-img: "img/post-bg-cloud.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Blog
---

# 搭建一个技术博客
## 使用GitHub托管博客
```

然后放置在 `_posts` 目录下即可。

### 1.4 访问自己的博客

先将本地自己的 repo push 到 github。

```shell
$ cd jiangydev.github.io
$ git add .
$ git commit - m "提交第一篇博文"
$ git push -u origin master
```

push 完成后，再次访问网址  [https://jiangydev.github.io][1]。



## 2 搭建本地博客环境

如果想在 push 前先查看一下将要发布的博文样式是否有问题，搭建一个本地环境还是有必要的。

这里使用 Jekyll，一个简单、可扩展的静态网站生成器。GitHub Pages 也是由 Jekyll 提供技术支持的，所以可以免费的快速部署。

### 2.1 安装 Ruby 和 Jekyll 

Ruby 下载地址：[https://www.ruby-lang.org/en/downloads/][5] （Jekyll 2 需要 v1.9.3 及以上版本，Jekyll 3 需要 v2 及以上版本）

安装 Ruby 开发环境后，请检查环境变量是否有效，便于在任意位置使用 Ruby 命令。

```shell
$ gem install jekyll bundler
# 博客还需用到 jekyll-paginate，因此遇到无法启动
```

### 2.2 本地访问博客

```shell
# 切换到博客的目录下
$ cd jiangydev.github.io
# 开启 Jekyll 本地服务
$ jekyll serve --watch
```

开启成功后，访问 [http://localhost:4000](http://localhost:4000/) 就能看到自己的博客主页啦！

> 小提示
>
> 如果在 Jekyll 启动过程中报错，若是缺少组件，使用 `gem install <组件名称>` 安装。



## 3 使用自己的服务器部署博客（Docker+自动化部署）

首先需要一个云服务器 ECS，阿里云（[点这里领代金券，购买服务器][6]）或腾讯云（[点这里购买服务器][7]）都可以，购买后可以通过 SecureCRT、[PUTTY][8]、[Xshell][9] 等工具 SSH 连接上云服务器。

为了免去安装各种基本环境的苦恼，以下采用 Docker 部署，使用 docker-compose 做服务编排。

### 3.1 域名解析

添加两条 A 记录，'www' 和 '@'，`www`将解析域名 [www.jiangjiangy.xyz][14]，`@` 将解析域名 jiangjiangy.xyz

![aliyun-dns](/img/in-post/blog/aliyun-dns.png)

### 3.2 Docker 及 docker-compose 安装

Docker 环境安装可参考我的另一篇文章：[在 CentOS7 上安装 Docker][10]

docker-compose 安装

```shell
$ yum install -y docker-compose
```

### 3.3 准备工作

文件结构如下

```
compose
├── jekyll         # 博客静态资源网站目录，所有者 1000！(权限用于自动化部署)
├── jenkins        # 自动化部署，所有者 1000！
├── nginx          # 负载均衡，反向代理等
    ├── conf       # 重要！nginx.conf 配置文件
    ├── logs       # nginx 日志
    └── www        # 静态文件
```

#### 3.2.1 Nginx 配置文件内容`（请认真看注释）`

文件名及所在目录：`/opt/compose/nginx/conf/nginx.conf`

```nginx
server {  
    listen 80;
    # 若没有域名，就不用写域名（以空格分隔）
    server_name jiangjiangy.xyz www.jiangjiangy.xyz 106.14.141.162;
    
    charset uft-8;
    access_log on;
    access_log  /var/log/nginx/host.access.log;
    error_log /var/log/nginx/error.log;

    location / {
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        # 这里的 jekyll 是 Jekyll 容器的域名
        proxy_pass http://jekyll:4000;
    }
    error_page 500 502 503 504  /50x.html;  
    location = /50x.html {  
        root html;
    }
}
# 以下是 Jenkins 配置，用于配置自动化部署
# 若没有域名，请修改 listen 端口，将 Jenkins 容器的 8080 端口暴露出来
server {
    # 访问方式 http://jenkins.jiangjiangy.xyz./
    listen 80;
    # listen 8080; # 没有域名，访问方式: http://<ip>:8080
    server_name jenkins.jiangjiangy.xyz;

    location / {
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        # 这里的 jenkins 是 Jenkins 容器的域名
        proxy_pass http://jenkins:8080;
    }
    error_page 500 502 503 504  /50x.html;
    location = /50x.html {
        root html;
    }
}
```

#### 3.2.2 docker-compose 配置文件内容

文件名及所在目录：`/opt/compose/docker-compose.yml`

```yaml
version: "2"
services:
  jekyll:
    command: jekyll serve
    image: "jekyll/jekyll"
    volumes:
      - $PWD/jekyll/:/srv/jekyll/
      - $PWD/jekyll/vendor/bundle:/usr/local/bundle
    environment:
      TZ: Asia/Shanghai
  jenkins:
    image: "jenkinsci/jenkins:2.147"
    volumes:
      - $PWD/jenkins/:/var/jenkins_home/
      - $PWD/jekyll/:/jekyll/
    environment:
      TZ: Asia/Shanghai
  web:
    image: "nginx:1.14.0"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - $PWD/nginx/conf/nginx.conf:/etc/nginx/conf.d/nginx.conf
      - $PWD/nginx/www/:/usr/share/nginx/html/
      - $PWD/nginx/logs/:/var/log/nginx/
    environment:
      TZ: Asia/Shanghai
    links:
      - jekyll:jekyll
      - jenkins:jenkins
```

### 3.4 启动服务

服务器端`/opt/compose/`是我们操作的目录，所有操作在该目录下完成！

```shell
# 切换到 compose 操作目录
$ cd /opt/compose
$ mkdir jekyll
# 从 GitHub 中拉取自己的博客
$ wget https://github.com/jiangydev/jiangydev.github.io/archive/master.zip
$ unzip master.zip && cp -f master/* jekyll/
# 下面这行命令，为了解决 Jenkins 容器中的用户权限问题
$ mkdir jenkins && chown -R 1000 jenkins/ && chown -R 1000 jekyll/
# 启动编排的服务
$ docker-compose up -d
```

等待服务启动成功后即可访问自己的博客了！

访问地址：[http://www.jiangjiangy.xyz./][14]

> 小提示
>
> 1. `未备案`的域名，请在访问时在域名尾部加一个英文的点`.`。
> 2. 从 GitHub 拉取博客项目可能会因为网络问题中断，拉取不了的话，就本地下载后，使用 scp 等方式上传到服务器的 /opt/compose 目录下。
> 3. 解决 Jenkins 容器挂载目录权限的问题，主要有两种方法，可参考 Stackoverflow [what-is-the-best-way-to-manage-permissions-for-docker-shared-volumes][11]

### 3.5 配置自动化部署

其实到上面一步已经能正常访问了，但是当博客有新的 push 后，从本地传输文件过于复杂，不如配置自动化部署，如下。

#### 3.5.1 添加 Webhooks

设置github仓库的webhook，在github仓库的项目界面，点击Setting->Webhooks->Add Webhook，添加Webhooks的配置信息，我的配置信息如下：

- Payload URL: http://jenkins.jiangjiangy.xyz./github-webhook/
- Content type: application/json

#### 3.5.2 解决从 GitHub 拉取项目失败的问题

修改域名解析文件 /etc/hosts。

```shell
# 将 github.com 直接解析到 IP
$ vi /etc/hosts
151.101.229.194   github.global.ssl.fastly.net
192.30.255.113    github.com
192.30.255.113    www.github.com
192.30.253.118    gist.github.com
# 刷新 DNS
$ sudo systemctl restart NetworkManager
# 若上面的命令无效，尝试这个：sudo rcnscd restart
```

#### 3.5.3 配置 Jenkins

该部分配置可参考我的另一篇博客 [Jenkins + Docker 持续集成方案及案例][12]

#### 3.5.4 重新启动服务

```shell
# 切换到 compose 操作目录
$ cd /opt/compose
# 启动编排的服务
$ docker-compose up -d
```



## 4 结束

如有疑问，欢迎联系我！



[1]: https://jiangydev.github.io
[2]: https://www.fangzhipeng.com
[3]: https://mp.weixin.qq.com/s/zli48oPHIgeMTwi4ItxHwA
[4]: https://github.com/Huxpro/huxpro.github.io
[5]: https://www.ruby-lang.org/en/downloads/
[6]: https://promotion.aliyun.com/ntms/yunparter/invite.html?userCode=gixemo0z
[7]: https://cloud.tencent.com/redirect.php?redirect=1025&cps_key=35362d4b60aa7d269b4a2bc0b05f6b27&from=console
[8]: https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html
[9]: https://www.netsarang.com/products/xsh_overview.html
[10]: https://jiangydev.github.io/2017/07/08/docker-centos7-install/
[11]: http://stackoverflow.com/questions/23544282/what-is-the-best-way-to-manage-permissions-for-docker-shared-volumes
[12]: https://jiangydev.github.io/2018/07/29/ci-jenkins/#6-Jenkins--gitgithub--静态文件-实现自动化部署
[13]: https://www.codesheep.cn/

[14]: http://www.jiangjiangy.xyz./