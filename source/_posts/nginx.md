---
title: nginx的那些事儿
date: 2021-07-22 18:35:44
excerpt: nginx的主要配置，原理和优化
categories:
- nginx
tags: nginx
---

### nginx常用命令

+ nginx主要命令
  - `./nginx` 启动nginx
  - `./nginx -s stop` 终止nginx(当然也可以找到nginx进程号，然后使用kill -9 杀掉nginx进程)
  - `./nginx -s reload` (重新加载nginx.conf配置文件)
    
### nginx核心配置

nginx的配置分为*全局块*、*events块*、*http块*

#### 全局块
从配置文件开始到events块之间的内容，此处的配置影响nginx服务器整体的运行，比如worker进 程的数量、错误日志的位置等

#### events块
events块主要影响nginx服务器与用户的网络连接，比如worker_connections为1024，表示每个
worker process 支持的最大连接数为1024

#### http块
http块是配置最频繁的部分，如虚拟主机的配置，监听端口的配置，请求转发、反向代理、负载均衡等

```text

#user  nobody;
# worker进程数量，即工作进程，通常设置=cpu数量
worker_processes  1;
# 全局错误日志和pid文件位置
#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;

# events模块，影响nginx服务器和用户的网络连接
events {
    # 单个worker连接的最大并发连接数
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    # 连接超时时间
    keepalive_timeout  65;

    # 压缩
    #gzip  on;

    server {
        # 监听的端口
        listen       80;
        # 虚拟主机 www.xxx.com
        server_name  localhost;

        #charset koi8-r;

        #access_log  logs/host.access.log  main;

        # tomcat context
        location / {
            root   html; # 网站的根目录
            index  index.html index.htm;
        }

        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }

        # proxy the PHP scripts to Apache listening on 127.0.0.1:80
        #
        #location ~ \.php$ {
        #    proxy_pass   http://127.0.0.1;
        #}

        # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
        #
        #location ~ \.php$ {
        #    root           html;
        #    fastcgi_pass   127.0.0.1:9000;
        #    fastcgi_index  index.php;
        #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
        #    include        fastcgi_params;
        #}

        # deny access to .htaccess files, if Apache's document root
        # concurs with nginx's one
        #
        #location ~ /\.ht {
        #    deny  all;
        #}
    }


    # another virtual host using mix of IP-, name-, and port-based configuration
    #
    #server {
    #    listen       8000;
    #    listen       somename:8080;
    #    server_name  somename  alias  another.alias;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}


    # HTTPS server
    #
    #server {
    #    listen       443 ssl;
    #    server_name  localhost;

    #    ssl_certificate      cert.pem;
    #    ssl_certificate_key  cert.key;

    #    ssl_session_cache    shared:SSL:1m;
    #    ssl_session_timeout  5m;

    #    ssl_ciphers  HIGH:!aNULL:!MD5;
    #    ssl_prefer_server_ciphers  on;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}
}
```
