---
title: openresty编译安装
authors: [HappyLadySauce]  #作者
date: 2024-9-22   #时间
comments: false
hide:
  - footer
  - feedback
categories:  #分类
  - 运维
---

# openresty编译安装

# 前置

```text
sudo apt-get install -y libssl-dev libpcre3 libpcre3-dev
```

## openresty Makefile生成脚本: make.sh


```shell
#!/bin/bash
./configure \
--prefix=/etc/openresty \
--add-module=/opt/ngx_http_proxy_connect_module \
--with-luajit \
--with-pcre-jit \
--with-stream \
--with-stream_ssl_module \
--with-http_ssl_module \
--with-http_auth_request_module \
--with-http_v2_module \
--with-http_v3_module \
--without-mail_pop3_module \
--without-mail_imap_module \
--without-mail_smtp_module \
--with-http_stub_status_module \
--with-http_realip_module \
--with-http_addition_module \
--with-http_auth_request_module \
--with-http_secure_link_module \
--with-http_random_index_module \
--with-http_gzip_static_module \
--with-http_sub_module \
--with-http_dav_module \
--with-http_flv_module \
--with-http_mp4_module \
--with-http_gunzip_module \
--with-threads \
--with-file-aio \
--without-http_echo_module \
-j2
```

## ngx_http_proxy_connect_module补丁添加

```shell
patch -d build/nginx-1.25.3/ -p 1 < /opt/ngx_http_proxy_connect_module/patch/proxy_connect_rewrite_102101.patch
```

# 系统优化

## 文件打开数量

```shell
vim /etc/security/limits.conf

#设置文件最大打开数量
*   -   nofile  65535
*   soft    nofile  65535
*   hard    nofile  65535
root    soft    nofile  65535
root    hard    nofile  65535
```

## 内核参数优化

```shell
#允许TIME_WAIT套接字数量的最大值
net.ipv4.tcp_max_tw_buckets = 6000
#启用timewait快速回收
net.ipv4.tcp_tw_recycle = 1
#允许将TIME-WAIT sockets重新用于新的TCP连接
net.ipv4.tcp_tw_reuse = 1
#保持在FIN-WAIT-2状态的时间
net.ipv4.tcp_fin_timeout = 60
#TCP发送keeplive消息的频度,若将其设置的小一些,可以更快地清理无效的连接
net.ipv4.tcp_keepalive_time = 30
#监听队列长度
net.core.somaxconn = 262144
#允许送到队列的数据包的最大数目
net.core.netdev_max_backlog = 262144
#TCP三次握手建立阶段接受SYN请求队列的最大长度
net.ipv4.tcp_max_syn_backlog = 262144
#TCP接受缓存(用于TCP接受滑动窗口)的最小值、默认值、最大值
net.ipv4.tcp_rmem = 10240 87380 12582912
#TCP发送缓存（用于TCP发送滑动窗口）的最小值、默认值、最大值
net.ipv4.tcp_wmem = 10240 87380 12582912
#解决TCP的SYN攻击
net.ipv4.tcp_syncookies = 1
#内核缓冲区最大和默认大小
net.core.wmem_default = 8388608
net.core.rmem_default = 8388608
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216

#开启路由转发功能
net.ipv4.ip_forward = 0
```

# NGINX主配置文件

## nginx.conf

```shell
worker_cpu_affinity auto;
worker_processes  auto;
worker_rlimit_nofile 65535;
#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;


events {
    use epoll;
    worker_connections  2048;
    multi_accept on;
}


http {
    include       mime.types;
    default_type  application/octet-stream;
    charset utf-8;
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;

	#开启高效文件传输模式
    sendfile        on;
    #增加包大小 减少网络堵塞
    tcp_nopush     on;
	
	
	#fastcgi缓存目录
	fastcgi_cache_path /webdata/nginx/fastcgi_cache levels=1:2 keys_zone=cache_fastcgi:10m inactive=5m;
	#fastcgi缓存临时目录
	fastcgi_temp_path /webdata/nginx/fastcgi_tmp;
	#fastcgi_cache
	fastcgi_cache cache_fastcgi;
	#FastCGI的超时时间
	fastcgi_connect_timeout 600;
	#FastCGI传送请求的超时时间
	fastcgi_send_timeout 600;
	#FastCGI接收请求的超时时间
	fastcgi_read_timeout 600;
	#FastCGI应答第一部分需要用多大的缓冲区
	fastcgi_buffer_size 16k;
	#FastCGI的缓冲区
	fastcgi_buffers 16 16k;
	#FastCGI繁忙时候的缓冲区
	fastcgi_busy_buffers_size 32k;
	#在写入fastcgi_temp_path时将用多大的数据块
	fastcgi_temp_file_write_size 32k;

	#fastcgi缓存设置
	fastcgi_cache_valid 200 302 1h;
    fastcgi_cache_valid 301 1d;
    fastcgi_cache_valid any 1m;
    fastcgi_cache_min_uses 1;
	
	#连接超时时长
    keepalive_timeout  60;
    #增加包大小 减少网络堵塞
    tcp_nodelay on;
    #客户端请求头缓冲区大小
    client_header_buffer_size 4k;
    #打开文件缓存
    open_file_cache max=65535 inactive=20s;
    #打开文件缓存最少使用数量
    open_file_cache_min_uses 1;
    #请求头超时时间
    client_header_timeout 15;
    #请求体超时时间
    client_body_timeout 15;
    #清理未响应连接
    reset_timedout_connection on;
    #响应客户端超时时间
    send_timeout 15;
    #客户端最大上传文件大小
	client_max_body_size 10m;
	
	variables_hash_max_size 2048;
	variables_hash_bucket_size 64;
    include /webdata/nginx/server.conf;
    
    #关闭nginx错误版本显示
    #server_tokens off;
    
    #openresty 服务器信息替换
    more_set_headers "Server: Happlelaoganma";
}
```

# module文件夹：

## cache.conf

```shell
location ~ \.*(ico|png|gif|jpe?g)$ {
    expires 30d;
    access_log off;
}

location ~* \.(js|css)$ {
	expires 7d;
	log_not_found off;
	
	access_log off;
}
```

## refere.conf

```
#防盗链
location ~*^.+\.(jpg|gif|png|swf|flv|wma|wmv|asf|mp3|mmf|zip|rar)$ {
	valid_referers none blocked www.benet.com benet.com;
	if($invalid_referer) {
	  return 404;
	  break;
	}
	access_log off;
}
```

## deny.conf

```shell
#禁止访问的文件或目录
location ~ ^/(\.user.ini|\.htaccess|\.git|\.env|\.svn|\.project|LICENSE|README.md)
{
    return 404;
}

#禁止在证书验证目录放入敏感文件
if ( $uri ~ "^/\.well-known/.*\.(php|jsp|py|js|css|lua|ts|go|zip|tar\.gz|rar|7z|sql|bak)$" ) {
    return 403;
}

#禁止访问日志
location ~ .*\.(gif|jpg|jpeg|bmp|swf)$
{
    expires      30d;
    error_log /dev/null;
    access_log /dev/null;
}

location ~ .*\.(js|css)?$
{
    expires      12h;
    error_log /dev/null;
    access_log /dev/null;
}
```

## error.conf

```shell
#error page
error_page 500 502 503 504 /50x.html;
location = /50x.html {
	root html;
}
```

## gzip.conf

```shell
location ~ \.*(txt|xml|html|js|css)$ {
    gzip on;
    #最小压缩原文件大小
    gzip_min_length 1k;
    #压缩缓冲区大小
    gzip_buffers 4 32k;
    gzip_types text/plain text/css text/javascriptapplication/json application/javascript application/x-javascriptapplication/xml;
    gzip_comp_level 6;
    gzip_http_version 1.1;
}
```

## ssl.conf

```shell
ssl_certificate      /ssl/happlelaoganma.cn_bundle.crt;
ssl_certificate_key  /ssl/happlelaoganma.cn.key;

ssl_session_cache    shared:SSL:1m;
ssl_session_timeout  5m;

ssl_prefer_server_ciphers  on;
ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
```



# openresty by Systemd

```shell
vim /etc/systemd/system/openresty.service
```

```shell
[Unit]
Description=The OpenResty Application
After=syslog.target network-online.target remote-fs.target nss-lookup.target
Wants=network-online.target

[Service]
Type=forking
PIDFile=/etc/openresty/nginx/logs/nginx.pid
ExecStartPre=/etc/openresty/nginx/sbin/nginx -t -q -g 'daemon on; master_process on;'
ExecStart=/etc/openresty/nginx/sbin/nginx -g 'daemon on; master_process on;'
ExecReload=/etc/openresty/nginx/sbin/nginx -g 'daemon on; master_process on;' -s reload
ExecStop=-/sbin/start-stop-daemon --quiet --stop --retry QUIT/5 --pidfile /etc/openresty/nginx/logs/nginx.pid
TimeoutStopSec=5
KillMode=mixed

[Install]
WantedBy=multi-user.target
```



# 代理服务器

```shell
server {
    listen 80; # 监听端口，也可以是其他端口
    listen 443 ssl http2;
    server_name www.happlelaoganma.cn; # 你的域名或服务器地址

    location /download/ {
        rewrite ^/download(/.*)$ $1 break;
        proxy_pass http://game.happlelaoganma.cn:18080;
    }

#php反向代理
    location / {
        proxy_pass https://10.7.7.11:22443; # 内网 PHP 服务器地址
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Connection "close";
        proxy_redirect off;
    }

    #证书 ssl优化
    include /webdata/nginx/module/ssl.conf;
    #禁止文件
    #include /webdata/nginx/module/deny.conf;
    #错误页面
    #include /webdata/nginx/module/error.conf;
    #缓存设置
    #include /webdata/nginx/module/cache.conf;
    #压缩文件
    #include /webdata/nginx/module/gzip.conf;
}
```

# 皮肤站文件: server.conf文件

```shell
#MC皮肤站
server {
    listen 2280;
    listen 22443 ssl;

    #server_name 192.168.14.243;
    access_log  logs/web_access.log;
    root /webdata/skin/public;

    index index.php;

    #skin伪静态
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    #代理访问php
    location ~ \.php$ {
    try_files $uri =404;
    fastcgi_keep_conn on;
    fastcgi_connect_timeout 60s;
        fastcgi_pass   127.0.0.1:9000;
        fastcgi_index  index.php;
    fastcgi_split_path_info ^(.+\.php)(/.+)$;
        include        fastcgi.conf;
    }

    #证书 ssl优化
    include /webdata/nginx/module/ssl.conf;
    #禁止文件
    include /webdata/nginx/module/deny.conf;
    #错误页面
    include /webdata/nginx/module/error.conf;
    #缓存设置
    include /webdata/nginx/module/cache.conf;
    #压缩文件
    include /webdata/nginx/module/gzip.conf;
}

#正向代理服务器
server {
    listen 1180;
    server_name 192.168.14.243;
    resolver 8.8.8.8 valid=60s ipv6=off;
    resolver_timeout 30s;
    proxy_connect;
    proxy_connect_allow            80 443 563;
    proxy_connect_connect_timeout  20s;
    proxy_connect_read_timeout     20s;
    proxy_connect_send_timeout     20s;
    location / {
        proxy_pass $scheme://$http_host$request_uri;
        proxy_set_header Host $host;
    }
}
```