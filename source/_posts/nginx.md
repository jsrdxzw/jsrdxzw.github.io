---
title: nginx的那些事儿
date: 2021-07-22 18:35:44
excerpt: nginx的主要配置，原理和优化
categories:
- nginx
tags: nginx
---

Nginx是一个高并发，低内存占用的web服务器，单机可以达到5wQPS以上，并且内存占用非常低

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
    # 总处理量 = worker_processes * worker_connections
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

        # 默认请求
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

### 反向代理
比如 xxx.com/abc -> tomcat 8080, xxx.com/def -> tomcat 8081
```text
server {
        listen       80;
        server_name  localhost;
        #charset koi8-r;
        #access_log  logs/host.access.log  main;
 
        location /abc {
            proxy_pass   http://127.0.0.1:8080;
        }
        
        location /def {
            http://127.0.0.1:8081;
        }
}
```

### 负载均衡
相同请求会分发到不同的服务器上，降低单个节点的请求压力
```text
# 配置集群的信息, 默认是轮询策略, weight值越大越有可能被分配到
upstream xxxServer {
    server 127.0.0.1:8080 max_fails=3 fail_timeout=3s weight=2;
    server 127.0.0.1:8082;
}
server {
        listen       80;
        server_name  localhost;
        #charset koi8-r;
        #access_log  logs/host.access.log  main;
 
        location /abc {
            proxy_pass   http://xxxServer;
        }
        
        location /def {
            proxy_pass http://127.0.0.1:8081;
        }
}
```
#### 负载均衡策略
+ 轮询
  默认策略，每个请求按时间顺序逐一分配到不同的服务器，如果某一个服务器下线，能自动剔除
  
```text
upstream xxxServer{
    server 111.229.248.243:8080; 
    server 111.229.248.243:8082;
}
location /abc {
    proxy_pass http://xxxServer;
}
```

+ weight
  weight代表权重，默认每一个负载的服务器都为1，权重越高那么被分配的请求越多(用于服务器性能不均衡的场景)
  
```text  
upstream xxxServer{
    server 111.229.248.243:8080 weight=1; 
    server 111.229.248.243:8082 weight=2;
}
```

+ ip_hash 
  每个请求按照ip的hash结果分配，每一个客户端的请求会固定分配到同一个目标服务器处理，可以解决session问题
  
```text
upstream xxxServer{ 
    ip_hash;
    server 111.229.248.243:8080;
    server 111.229.248.243:8082; 
}
```

#### 动静分离
Nginx在静态资源请求上性能较好，业务处理交给Tomcat，实现动静分离。
Nginx实现静态资源配置也很容易，只需要将静态资源文件放到*Nginx服务器*上，在配置文件上修改即可。

```text
location /static/ {
    # nginx服务器目录加载，需要自己创建
    root staticDir;
}
```

#### Nginx底层进程机制

Nginx启动后，以daemon多进程方式在后台运行，包括一个Master进程和多个Worker进程，Master
进程负责管理和监控Worker进程，Worker进程负责处理具体的请求。
+ master进程 
  
  主要是管理worker进程，比如:
    - 接收外界信号向各worker进程发送信号(./nginx -s reload)
    - 监控worker进程的运行状态，当worker进程异常退出后Master进程会自动重新启动新的worker进程
  
+ worker进程
  
  worker进程具体处理网络请求。多个worker进程之间是对等的，他们同等竞争来自客户端的请求，各进程互相之间是独立的。
  nginx使用互斥锁来保证只有一个worker进程能够处理请求
  
多进程机制的好处：
+ 一个进程挂了，其他进程可以照样提供服务
+ 为热部署提供支撑，主要就是reload配置文件的时候，不会影响到旧请求的处理


#### nginx信号处理

以 `./nginx -s reload` 来说明nginx信号处理这部分 
1. master进程对配置文件进行语法检查 
2. 尝试配置(比如修改了监听端口，那就尝试分配新的监听端口) 
3. 尝试成功则使用新的配置，**新建worker进程** 
4. 新建成功，给旧的worker进程发送关闭消息 
5. 旧的worker进程收到信号会**继续服务**，直到把当前进程接收到的请求处理完毕后关闭 所以reload之后worker进程pid是发生了变化的

#### 零拷贝技术和异步io
传输文件的时候，要根据文件的大小来使用不同的方式：
1. 传输大文件的时候，使用「异步 I/O + 直接 I/O」
2. 传输小文件的时候，则使用「零拷贝技术」
```text
location /video/ { 
    sendfile on; 
    aio on; 
    directio 1024m; 
}
```
当文件大小大于 `directio` 值后，使用「异步 I/O + 直接 I/O」，否则使用「零拷贝技术」。

**补充**

零拷贝技术可以直接从内存缓冲区拷贝数据到socket，但是需要使用pageCache，而大文件往往会占用pageCache的内存空间，
限制了它的使用场景，所以大文件传输一般使用异步io+直接io的方式

![异步io模型](/img/nginx-1.png)
