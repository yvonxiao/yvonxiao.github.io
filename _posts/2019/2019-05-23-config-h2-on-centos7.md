---
layout: post
title: CentOS 7基于Nginx配置HTTP2
categories: Development
description: config HTTP2 on CentOS 7
keywords: CentOS 7, nginx, HTTP2
---

记录阿里云CentOS 7机器上使用Let's Encrypt泛域名证书配置HTTP2

## 阿里云服务器基于Nginx配置HTTP2

### 安装Let's Encrypt泛域名证书

#### 填坑
[Let's Encrypt](https://letsencrypt.org/getting-started)的官网上介绍的[certbot](https://certbot.eff.org)工具，对于实际上服务器复杂的python环境及工具并不是处理的很好，至少在我的机器上报错了，我也懒的去改造，直接基于最稳妥的手工方式安装好了，反正第一次装好，后面都是自动更新。顺便说一句，现在的Let's Encrypt早就支持泛域名了，非常的省事。
#### 安装
```Bash
su
yum -y install python-tools python-pip
cd /usr/local/src
git clone https://github.com/letsencrypt/letsencrypt
cd letsencrypt
./certbot-auto certonly --manual --preferred-challenges=dns --email yvonxiao@gmail.com --server https://acme-v02.api.letsencrypt.org/directory --agree-tos -d *.yvonxiao.site -d yvonxiao.site
```
一路根据提示操作即可，中途需要根据给出的记录值去自己的网站dns解析里加两条指定的txt记录。

### 配置Nginx
#### 配置自己的静态网页目录
```Bash
cd /etc/nginx
mkdir ssl
mkdir vhosts
openssl dhparam -out /etc/nginx/ssl/dhparam.pem 2048
```
`nginx.conf`
```Nginx
user  nginx;
worker_processes  1;

# Linux上默认是epoll
events {
    worker_connections  8190; # 配置注意点1
    multi_accept on;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    tcp_nopush     on;
    tcp_nodelay on;
    keepalive_requests 1000; # 配置注意点2
    keepalive_timeout 60s 60s;
    types_hash_max_size 2048;
    server_tokens off;
    server_name_in_redirect off;
    server_names_hash_bucket_size 128;
    client_header_buffer_size 32k;
    large_client_header_buffers 4 32k;
    client_max_body_size 50m;

    gzip  on;
    gzip_disable "msie6";
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 4;
    gzip_buffers 16 8k;
    gzip_types text/plain application/javascript text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript;

    proxy_read_timeout 600;
    proxy_buffer_size 16k;
    proxy_buffers 4 64k;
    proxy_busy_buffers_size 128k;

    include /etc/nginx/vhosts/*.conf;

    server {
        listen       80;
        server_name  localhost;

        location / {
            root   html;
            index  index.html index.htm;
        }
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }
}

```
`www.yvonxiao.site.conf`
```Nginx
    server {
        listen 80;
        server_name www.yvonxiao.site;
        charset utf-8;
        root /home/www/yvonxiao-site;

        rewrite  ^/(.*)$  http://yvonxiao.site/$1  permanent; 

     }

    server {
        listen 443 ssl http2;
        server_name www.yvonxiao.site;
        charset utf-8;
        root /home/www/yvonxiao-site;

        #ssl on;
        ssl_certificate      /etc/letsencrypt/live/yvonxiao.site/fullchain.pem;
        ssl_certificate_key  /etc/letsencrypt/live/yvonxiao.site/privkey.pem;
        ssl_session_cache    shared:SSL:20m;
        ssl_session_timeout  60m;
        ssl_session_tickets  on;
        ssl_dhparam ssl/dhparam.pem;
        ssl_protocols   TLSv1 TLSv1.1 TLSv1.2;
        ssl_ciphers "EECDH+ECDSA+AESGCM EECDH+aRSA+AESGCM EECDH+ECDSA+SHA384 EECDH+ECDSA+SHA256 EECDH+aRSA+SHA384 EECDH+aRSA+SHA256 EECDH+aRSA+RC4 EECDH EDH+aRSA !aNULL !eNULL !LOW !3DES !MD5 !EXP !PSK !SRP !DSS !RC4";
        ssl_prefer_server_ciphers  on;
        ssl_stapling on;
        ssl_stapling_verify on;

        resolver 100.100.2.136 100.100.2.138 valid=300s;
        resolver_timeout 10s;

        #add_header Strict-Transport-Security max-age=15768000;

        rewrite  ^/(.*)$  https://yvonxiao.site/$1  permanent;

     }
```
`yvonxiao.site.conf`
```Nginx
    server {
        listen 80;
        server_name yvonxiao.site 47.101.139.34;
        charset utf-8;
        root /home/www/yvonxiao-site;

        location ~ /(css|font|img|js|lib|ico)/ {
            root /home/www/yvonxiao-site;
            access_log off;
            expires 7d;
        }

        error_page 500 502 503 504 /50x.html;
        location = /50x.html {
            root html;
        }
    }

    server {
        listen 443 ssl http2;
        server_name yvonxiao.site 47.101.139.34;
        charset utf-8;
        root /home/www/yvonxiao-site;

        #ssl on; # 新版配置已不建议这样开启SSL
        ssl_certificate      /etc/letsencrypt/live/yvonxiao.site/fullchain.pem;
        ssl_certificate_key  /etc/letsencrypt/live/yvonxiao.site/privkey.pem;
        ssl_session_cache    shared:SSL:20m;
        ssl_session_timeout  60m;
        ssl_session_tickets  on;
        ssl_dhparam ssl/dhparam.pem;
        ssl_protocols   TLSv1 TLSv1.1 TLSv1.2;
        ssl_ciphers "EECDH+ECDSA+AESGCM EECDH+aRSA+AESGCM EECDH+ECDSA+SHA384 EECDH+ECDSA+SHA256 EECDH+aRSA+SHA384 EECDH+aRSA+SHA256 EECDH+aRSA+RC4 EECDH EDH+aRSA !aNULL !eNULL !LOW !3DES !MD5 !EXP !PSK !SRP !DSS !RC4";
        ssl_prefer_server_ciphers  on;
        ssl_stapling on;
        ssl_stapling_verify on;

        resolver 100.100.2.136 100.100.2.138 valid=300s; # 配置注意点3
        resolver_timeout 10s;

        add_header Strict-Transport-Security max-age=15768000;

        location ~ /(css|font|img|js|lib|ico)/ {
            root /home/www/yvonxiao-site;
            access_log off;
            expires 7d;
        }

        error_page 500 502 503 504 /50x.html;
        location = /50x.html {
            root html;
        }
    }
```
#### 配置要点
> worker_connections  8190; # 配置注意点1

这个需要根据自己机器内核数量(还要考虑机器其他进程的内核使用率)和最大连接数来共同确定。根据默认浏览器客户端发送的并发连接数是1或者2来统一计算，这里就都当2来处理，那么HTTP服务器`max_clients = worker_processes * worker_connections/2`，反向代理服务器`max_clients = worker_processes * worker_connections/4`。考虑到后期会加入反向代理，且连接不是全分配给nginx，粗略的再除以2。这里我随意的把总连接数设置为65520(比自己机器上的65535小)，那么就是`65520/4/2=8190`

> keepalive_requests 1000; # 配置注意点2

`keepalive_requests`表示一个keep-alive连接上可以服务的请求的最大数量，具体解释可找底部`nginx优化——包括https、keepalive等`一文，要比较精确的计算的话，需要知道服务的QPS，我这里刚刚开始搭建服务，所以先简单的设置成一个1000

> resolver 100.100.2.136 100.100.2.138 valid=300s; # 配置注意点3

这里的100.100.2.136 100.100.2.138服务器来自阿里云的内网dns服务器，假如不是阿里云的机器就需要自行找最快的dns服务器。

我这里先确定服务器在华东2（上海），然后在[阿里云各地域【Region】内网DNS地址](https://help.aliyun.com/knowledge_detail/38094.html)里找对应的DNS服务器，但是最新的文档不知道为什么移动到其他地方去了，当时查询到的上海对应的DNS地址就是`100.100.2.136 100.100.2.138`，如果是其他地区，可以发起阿里云技术支持工单查询。我这里的是一份几年前的，用之前先测试一下是否有效吧。
```Text
青岛:
10.202.72.116
10.202.72.118
杭州:
10.143.22.116
10.143.22.118
上海：
100.100.2.136
100.100.2.138
香港:
10.143.22.116
10.143.22.118
美国：
10.143.22.116
10.143.22.118
北京:
10.202.72.116
10.202.72.118
深圳:
100.100.2.138
100.100.2.136
新加坡
100.100.2.136
100.100.2.138
```

## 参考
* [Generate Wildcard SSL certificate using Let’s Encrypt/Certbot](https://medium.com/@saurabh6790/generate-wildcard-ssl-certificate-using-lets-encrypt-certbot-273e432794d7)
* [nginx优化——包括https、keepalive等](https://lanjingling.github.io/2016/06/11/nginx-https-keepalived-youhua)
* [nginx 并发数问题思考：worker_connections,worker_processes与 max clients](https://blog.51cto.com/liuqunying/1420556)

