---
layout:     post
title:      "MySQL 慢SQL监控平台搭建和使用"
subtitle:   "使用 percona 工具和 Anemometer 搭建慢查平台"
date:       2019-11-22
author:     "jiangydev"
header-img: "img/post-bg-cloud.jpg"
header-mask: 0.5
catalog:    true
tags:
    - MySQL
    - MySQL 监控
---

[TOC]

## 1 架构图

![慢查平台架构图](/img/in-post/mysql/mysql-anemometer/慢查平台架构图.png)

## 2 环境搭建

### 2.1 数据库慢查

数据库生产环境搭建：

```shell
$ docker run -d -it --name test-mysql-prod -e MYSQL_ROOT_PASSWORD=jiangydev mysql:5.7
```

开启慢查日志：

```sql
-- 开启慢查
> set global slow_query_log=ON;
-- 设置慢查记录阈值：1s
> set global long_query_time=1;

-- 确认数据库慢查配置
> show variables where Variable_name in ('log_output','log_slow_admin_statements','long_query_time','slow_query_log','slow_query_log_file');
+---------------------------+--------------------------------------+
| Variable_name             | Value                                |
+---------------------------+--------------------------------------+
| log_output                | FILE                                 |
| log_slow_admin_statements | OFF                                  |
| long_query_time           | 1.000000                             |
| slow_query_log            | ON                                   |
| slow_query_log_file       | /var/lib/mysql/c4ad334eace1-slow.log |
+---------------------------+--------------------------------------+
```

> 小提示：
>
> 1. slow_query_log_file 是慢查日志生成的路径配置，在后面解析慢查日志时需要使用到。
> 2. 上面的数据库配置方式非永久性，重启后失效；若要持久化，需进入 docker 容器，修改数据库的配置文件。

### 2.2 搭建慢查记录的数据库环境

数据库慢查记录环境搭建：

```shell
$ docker run -d -it --name test-mysql-record -e MYSQL_ROOT_PASSWORD=jiangydev mysql:5.7
```

### 2.3 日志收集

CentOS 环境中安装解析工具 `pt-query-digest`，执行解析的命令后，可查看记录慢查的数据库中是否有生成的解析结果。

```shell
# 启动 CentOS 环境，同时关联记录慢查的数据库 test-mysql-record
$ docker run -dit --name test-centos --link test-mysql-record:test-mysql-record centos:7 /bin/bash
# 进入 CentOS 环境
$ docker exec -it test-centos /bin/bash

# 安装 pt-query-digest 所需依赖包和工具
$ yum -y install perl-DBI perl-DBD-MySQL perl-Time-HiRes perl-IO-Socket-SSL perl-Digest-MD5.x86_64
$ wget percona.com/get/pt-query-digest
# 给予执行权限
$ chmod u+x pt-query-digest

# 解析 c4ad334eace1-slow.log 慢查日志文件，并将分析结果保存到 test-mysql-record 数据库
$ ./pt-query-digest --host test-mysql-record --user=root --password=jiangydev \
--create-review-table \
--create-history-table \
--review h=test-mysql-record,D=slow_query_log,t=global_query_review \
--history h=test-mysql-record,D=slow_query_log,t=global_query_review_history \
--no-report --limit=0% \
--filter=" \$event->{Bytes} = length(\$event->{arg}) and \$event->{hostname}=\"$HOSTNAME\"" \
./c4ad334eace1-slow.log
```

![数据库中的分析结果记录](/img/in-post/mysql/mysql-anemometer/数据库中的分析结果记录.png)

该脚本会自动创建 global_query_review 和 global_query_review_history 表，每执行一次，会在 review 表更新记录，并插入 history 表。



### 2.4 Anemometer 部署

Anemometer 项目地址：[https://github.com/box/Anemometer](https://github.com/box/Anemometer)

#### 2.4.1 搭建过程

##### 1 编写 Anemometer 配置文件 

文件名：`config.inc.php`

> 小提示：
>
> 此处只给出关键的配置，其他的配置和解释可以在项目中找到：`conf/sample.config.inc.php`。

```php
<?php
$conf['datasources']['localhost'] = array(
	'host'	=> 'localhost',
	'port'	=> 3306,
	'db'	=> 'slow_query_log',
	'user'	=> 'root',
	'password' => 'password',
	'tables' => array(
		'global_query_review' => 'fact',
		'global_query_review_history' => 'dimension'
	),
	'source_type' => 'slow_query_log'
);

/**
 * 如果有其他的数据源，可以继续添加
 *
$conf['datasources']['mysql56'] = array(
	'host'	=> 'localhost',
	'port'	=> 3306,
	'db'	=> 'performance_schema',
	'user'	=> 'root',
	'password' => '',
	'tables' => array(
		'global_query_review' => 'fact',
		'global_query_review_history' => 'dimension'
	),
	'source_type' => 'slow_query_log'
);
*/

$conf['plugins'] = array(
	'visual_explain' => '/usr/bin/pt-visual-explain',
#	percona toolkit has removed query advisor
#	'query_advisor'	=> '/usr/bin/pt-query-advisor',
    # 此处省略若干配置
    'explain'	=>	function ($sample) {
		# 此处省略若干配置
		$conn['user'] = 'root';
		$conn['password'] = 'password';
		return $conn;
	},
);

# 此处省略若干配置
?>
```

##### 2 编写 Dockerfile

> 小提示：
>
> 1. Anemometer 要求 php 版本高于 5.3。

此处使用 docker 提供的 centos:7 基础镜像，使用 Dockerfile 制作 Docker 镜像：

```dockerfile
FROM centos:7
WORKDIR /var/www/html/
RUN rpm -Uvh https://mirror.webtatic.com/yum/el7/epel-release.rpm \
    && rpm -Uvh https://mirror.webtatic.com/yum/el7/webtatic-release.rpm \
    && yum -y install httpd httpd-devel \
    && sed -i '/#ServerName/a\ServerName localhost:80' /etc/httpd/conf/httpd.conf \
    && yum -y install php70w \
    && yum -y install php70w-mysql php70w-common php70w-gd php70w-odbc php70w-pear php70w-xml php70w-bcmath php70w-mbstring \
    && yum -y install wget perl-DBI perl-DBD-MySQL perl-Time-HiRes perl-IO-Socket-SSL perl-Digest-MD5.x86_64 \
    && wget -P /usr/bin/ percona.com/get/pt-visual-explain && chmod a+x /usr/bin/pt-visual-explain \
    && yum -y install git \
    && git clone -b master https://gitee.com/jiangydev/Anemometer.git
COPY ./config.inc.php Anemometer/conf/config.inc.php
EXPOSE 80
ENTRYPOINT /usr/sbin/httpd -k start && tail -f /etc/httpd/logs/error_log
```



##### 3 构建镜像及启动

此时的目录下，有两个文件：`config.inc.php`、`dockerfile`。

```shell
# 构建镜像
$ docker build -t jiangydev/anemometer .
# 启动容器
$ docker run -dit --name test-anemometer -p 80:80 jiangydev/anemometer
```



#### 2.4.2 可能遇到的问题

1 `SELECT list is not in GROUP BY clause`

问题描述：

```
Expression #2 of SELECT list is not in GROUP BY clause and contains nonaggregated column 'slow_query_log.dimension.sample' which is not functionally dependent on columns in GROUP BY clause; this is incompatible with sql_mode=only_full_group_by (1055)
```

问题分析：

MySQL 5.7.5及以上功能依赖检测功能。如果启用了ONLY_FULL_GROUP_BY SQL模式（默认情况下），MySQL将拒绝选择列表，HAVING条件或ORDER BY列表的查询引用在GROUP BY子句中既未命名的非集合列，也不在功能上依赖于它们。

5.7.5之前，MySQL没有检测到功能依赖关系，默认情况下不启用ONLY_FULL_GROUP_BY。

解决方案：

重新设置 SQL 模式。

方式一：

```sql
> select @@global.sql_mode;
+-------------------------------------------------------------------------------------------------------------------------------------------+
| @@global.sql_mode                                                                                                                         |
+-------------------------------------------------------------------------------------------------------------------------------------------+
| ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION |
+-------------------------------------------------------------------------------------------------------------------------------------------+

mysql> set @@global.sql_mode = 'STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION';
```

方式二：

修改配置文件，并重启 mysql 服务。

```ini
[mysqld]
sql_mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION
```

