---
layout:     post
title:      "Nginx 日志切割"
subtitle:   "Nginx 日志配置相关"
date:       2018-06-05
author:     "jiangydev"
header-img: "img/post-bg-cloud.jpg"
tags:
    - Nginx
---

# Nginx 日志配置切割

## 日志配置分析

### 1 log_format

语法: `log_format name string ...`;
默认值: `log_format combined "..."`;
配置段: `http`

> 小提示：
>
> name表示格式名称，string表示等义的格式。

log_format有一个默认的无需设置的combined日志格式，相当于apache的combined日志格式，如下所示：

```nginx
log_format combined '$remote_addr - $remote_user  [$time_local] '
					' "$request"  $status  $body_bytes_sent  '
					' "$http_referer"  "$http_user_agent" ';
```

如果nginx位于负载均衡器，squid，nginx反向代理之后，web服务器无法直接获取到客户端真实的IP地址了。 $remote_addr获取反向代理的IP地址。反向代理服务器在转发请求的http头信息中，可以增加X-Forwarded-For信息，用来记录客户端IP地址和客户端请求的服务器地址。如下所示：

```nginx
log_format porxy '$http_x_forwarded_for - $remote_user  [$time_local] '
				' "$request"  $status $body_bytes_sent '
				' "$http_referer"  "$http_user_agent" '; 
```

日志格式允许包含的变量注释如下：

| 变量名                                 | 注释                                                         |
| -------------------------------------- | ------------------------------------------------------------ |
| \$remote\_addr, \$http_x_forwarded_for | 记录客户端IP地址                                             |
| \$remote_user                          | 记录客户端用户名称$request 记录请求的URL和HTTP协议           |
| \$status                               | 记录请求状态                                                 |
| \$body_bytes_sent                      | 发送给客户端的字节数，不包括响应头的大小；该变量与Apache模块mod_log_config里的“%B”参数兼容。 |
| \$bytes_sent                           | 发送给客户端的总字节数。                                     |
| \$connection                           | 连接的序列号。                                               |
| \$connection_requests                  | 当前通过一个连接获得的请求数量。                             |
| \$msec                                 | 日志写入时间。单位为秒，精度是毫秒。                         |
| \$pipe                                 | 如果请求是通过HTTP流水线(pipelined)发送，pipe值为“p”，否则为“.”。 |
| \$http_referer                         | 记录从哪个页面链接访问过来的。                               |
| \$http_user_agent                      | 记录客户端浏览器相关信息                                     |
| \$request_length                       | 请求的长度（包括请求行，请求头和请求正文）。                 |
| \$request_time                         | 请求处理时间，单位为秒，精度毫秒； 从读入客户端的第一个字节开始，直到把最后一个字符发送给客户端后进行日志写入为止。 |
| \$time_iso8601                         | ISO8601标准格式下的本地时间。                                |
| \$time_local                           | 通用日志格式下的本地时间。                                   |

<warning>发送给客户端的响应头拥有“sent_http_”前缀。 比如$sent_http_content_range。</warning>

#### 实例

```nginx
http {
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
        '"$status" $body_bytes_sent "$http_referer" '
        '"$http_user_agent" "$http_x_forwarded_for" '
        '"$gzip_ratio" $request_time $bytes_sent $request_length';
    log_format srcache_log '$remote_addr - $remote_user [$time_local] "$request" '
        '"$status" $body_bytes_sent $request_time $bytes_sent $request_length '
        '[$upstream_response_time][$srcache_fetch_status] [$srcache_store_status] [$srcache_expire]';
    open_log_file_cache max=1000 inactive=60s;
    
    server {
        server_name ~^(www\.)?(.+)$;
        access_log logs/$2-access.log main;
        error_log logs/$2-error.log;
        location /srcache {
            access_log logs/access-srcache.log srcache_log;
        }
    }
} 
```



```nginx
# 这是一个获取当前的日期时间示例
if ($time_iso8601 ~ "(\d{4})-(\d{2})-(\d{2})")
{
    set $time $1$2$3;
}
access_log /www/wwwlogs/access-blue-${time}.log;
# 上面有效，下面的始终不生效
# error_log /www/wwwlogs/error-blue-${time}.log;
```



### 2 access_log

语法: 

```nginx
access_log path [format [buffer=size [flush=time]]];
access_log path format gzip=level [flush=time];
access_log syslog:server=address,parameter=value;
access_log off; 
```

默认值: `access_log logs/access.log combined;` 

配置段: `http, server, location, if in location, limit_except`

不记录日志：`access_log off;` 

使用默认 combined 格式记录日志：`access_log logs/access.log` 或 `access_log logs/access.log combined;` 



## Nginx 日志切割

### 1 日志切割脚本

```shell
#!/bin/bash
# 此脚本用于自动分割Nginx的日志，包括access.log和error.log
# 每天00:00执行此脚本
# 将前一天的access.log重命名为access-xxxx-xx-xx.log格式，并重新打开日志文件
# Nginx日志文件所在目录
LOG_PATH=/opt/nginx/log/
# 获取昨天的日期
YESTERDAY=$(date -d "yesterday" +%Y-%m-%d)

#获取pid文件路径
PID=/var/run/nginx/nginx.pid
#分割日志
mv ${LOG_PATH}access.log ${LOG_PATH}access-${YESTERDAY}.log
mv ${LOG_PATH}error.log ${LOG_PATH}error-${YESTERDAY}.log
# 向Nginx主进程发送USR1信号，重新打开日志文件
# [ -f xxx] 是 测试 文件 是否 存在
# USR1亦通常被用来告知应用程序重载配置文件；例如，向Apache HTTP服务器发送一个USR1信号将导致以下步骤的发生：停止接受新的连接，等待当前连接停止，重新载入配置文件，重新打开日志文件，重启服务器，从而实现相对平滑的不关机的更改。
[ -f PID ] && kill -USR1 `cat ${PID}`
```


