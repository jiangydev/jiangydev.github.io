---
layout:     post
title:      "Docker 安装生产环境所需软件"
subtitle:   "Tomcat, Nginx, MySQL, Oracle, vsFTP"
date:       2017-07-08
author:     "jiangydev"
header-img: "img/post-bg-docker.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Docker
---

[TOC]

## Tomcat8 + JDK8

需要挂载 /opt/tomcat/war/:/usr/local/tomcat/webapps/ 以便部署；

Tomcat 默认日志的输出路径 /usr/local/tomcat/logs/

挂载 /opt/tomcat/logs/:/usr/local/tomcat/logs/

```sh
docker run -d -it --name test_tomcat --link test_mysql:mysql -p 80:8080 -v /opt/tomcat/war/:/usr/local/tomcat/webapps/ -v /opt/tomcat/logs/:/usr/local/tomcat/logs/ jiangydev/tomcat:8.5-jre8
docker stop test_tomcat
docker rm test_tomcat
```


## Nginx

```sh
docker run -d -it --name test_nginx --link test_tomcat:tomcat -p 80:80 --volumes-from test_vsftpd -v /opt/nginx/conf/:/etc/nginx/conf.d/ -v /opt/nginx/www/:/usr/share/nginx/html/ -v /opt/nginx/logs/:/var/log/nginx/ jiangydev/nginx

docker stop test_nginx
docker rm test_nginx

docker exec -it test_nginx /bin/bash
```

nginx -s reload

nginx.conf 默认配置
user  nginx;
worker_processes  1;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;

## MySQL

```sh
docker run -d -it --name test_mysql -p 3306:3306 -v /opt/mysql/conf/:/etc/mysql/ -v /opt/mysql/data/:/var/lib/mysql/ -v /opt/mysql/logs/:/var/log/mysql/ -e MYSQL_ROOT_PASSWORD=19951218 mysql:5.7
```

docker run -d -it --name zlsg_mysql -p 3306:3306 -v /opt/server/mysql/conf/:/etc/mysql/ -v /opt/server/mysql/data/:/var/lib/mysql/ -v /opt/server/mysql/logs/:/var/log/mysql/ -e MYSQL_ROOT_PASSWORD=root1234 mysql:5.6

1. The MySQL  Client configuration file: /etc/mysql/conf.d/mysql.cnf
2. The MySQL  Server configuration file: /etc/mysql/mysql.conf.d/mysqld.cnf
3. /usr/my.cnf

查看mysql加载配置文件的顺序

```
# 找出带有'Default options'的行，输出中除显示该行外，还显示之后的一行(After 1)
[root@jiangydev /]# mysql --verbose --help | grep -A 1 'Default options'
/etc/my.cnf /etc/mysql/my.cnf ~/.my.cnf
```

-p 3306:3306：将容器的3306端口映射到主机的3306端口
-v /opt/mysql/conf/:/etc/mysql/：将主机目录下的conf/挂载到容器的/etc/mysql/
-v /opt/mysql/logs/:/var/log/mysql/：将主机目录下的logs目录挂载到容器的/logs
-v /opt/mysql/data/:/var/lib/mysql/：将主机目录下的data目录挂载到容器的/mysql_data
-e MYSQL_ROOT_PASSWORD=123456：初始化root用户的密码

测试配置文件挂载到容器中是否生效：在宿主机的/opt/mysql/conf/目录下新建一个`my.cnf`文件，并添加如下内容。

```
[mysqld]  
character-set-server=utf8
```

重启docker容器。
```
docker restart test_mysql
```

连上数据库，并使用SQL语句检查配置文件是否生效（默认情况下，character-set-server 的值为latin）。
```sql
SHOW VARIABLES LIKE '%char%';
```

port = 3306
character-set-server=utf8
collation-server=utf8_general_ci
default-storage-engine=INNODB

## oracle

#### 维护者
https://hub.docker.com/r/sath89/oracle-12c/

#### Run with data on host and reuse it:

-e ORACLE_ALLOW_REMOTE=true 远程连接
-e DEFAULT_SYS_PASS=

docker run -dit --name test_oracle -p 8080:8080 -p 1521:1521 -v /opt/oracle/data:/u01/app/oracle sath89/oracle-12c

自动导入 sh sql and dmp files：
-v /my/oracle/init/sh_sql_dmp_files:/docker-entrypoint-initdb.d

#### Run with Custom DBCA_TOTAL_MEMORY (in Mb):

docker run -d -p 8080:8080 -p 1521:1521 -v /my/oracle/data:/u01/app/oracle -e DBCA_TOTAL_MEMORY=1024 sath89/oracle-12c

#### 连接数据库的设置
port: 1521
sid: xe
service name: xe
username: system
password: oracle

#### 修改Oracle密码

```sql
-- 查看用户的proifle是哪个，一般是default：
SELECT username,PROFILE FROM dba_users;

-- 查看指定概要文件（如default）的密码有效期设置：
SELECT * FROM dba_profiles s WHERE s.profile='DEFAULT' AND resource_name='PASSWORD_LIFE_TIME';

-- 将密码有效期由默认的180天修改成“无限制”：修改之后不需要重启动数据库，会立即生效。
ALTER PROFILE DEFAULT LIMIT PASSWORD_LIFE_TIME UNLIMITED;

-- 修改用户SYSTEM 密码
alter user SYSTEM identified by "****password****";

-- 解锁方法
alter user SYSTEM account unlock;
```

## vsftpd

```sh
$ docker run --privileged -dit --name test_vsftpd -p 5020:5020 -p 5021:5021 -p 50000-50209:50000-50209 -v /opt/nginx/www/:/home/ jiangydev/vsftpd /usr/sbin/init

$ docker run --privileged -dit --name test_vsftpd -p 5020:5020 -p 5021:5021 -p 50000-50209:50000-50209 -v /opt/vsftpd/ftp/:/var/ftp/ -v /opt/vsftpd/home/:/home/ -v /opt/vsftpd/log/:/var/log/ -v /opt/vsftpd/conf/:/etc/vsftpd/ jiangydev/vsftpd /usr/sbin/init

$ docker stop test_vsftpd
$ docker rm test_vsftpd
$ docker exec -it test_vsftpd /bin/bash
```

```sh
$ systemctl restart vsftpd

$ useradd jiangydev -s /sbin/nologin
$ passwd jiangydev 123456
echo jiangydev >> /etc/vsftpd/user_list
# 编辑user_list文件，允许jiangydev用户访问FTP

# 建立根目录，并设置访问权限
$ mkdir /var/ftp
$ chown -R jiangydev /var/ftp
$ chmod -R 755 /var/ftp
# 开启vsftpd服务
$ systemctl start vsftpd
# 默认开启vsftp服务
$ chkconfig vsftpd on
```
