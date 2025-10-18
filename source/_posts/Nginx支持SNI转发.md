---
title: Nginx支持SNI转发
link: nginx-sni
date: 2020-12-05 15:20:31
tags:
  - Nginx
---

SNI是一个TLS的扩展字段，经常用于访问域名跳转到不同的后端地址。

配置方式如下：打开nginx.conf文件，以ttbb/nginx:nake镜像为例/usr/local/openresty/nginx/conf/nginx.conf

如下为默认的nginx.conf配置

```

#user  nobody;
worker_processes  1;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;


events {
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
    keepalive_timeout  65;

    #gzip  on;

    server {
        listen       80;
        server_name  localhost;

        #charset koi8-r;

        #access_log  logs/host.access.log  main;

        location / {
            root   html;
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

在最后面添加上

```
stream {

map $ssl_preread_server_name $name {
    backend.example.com      backend;
    default                  backend2;
}

upstream backend {
    server 192.168.0.3:12345;
    server 192.168.0.4:12345;
}

upstream backend2 {
    server 127.0.0.1:8071;
}

server {
    listen      12346;
    proxy_pass  $name;
    ssl_preread on;
}
}
```
这个时候，我们已经开启了SNI转发的功能，如果你使用backend.example.com的域名访问服务器，就会转发到backend，如果使用其他域名，就会转发到backend2

测试的时候，让我们在/etc/hosts里进行设置，添加

```
127.0.0.1 backend.example.com
```

然后进行请求

```bash
curl https://backend.example.com:12346
```

这里注意请求要使用https，http协议或者是tcp可没有SNI的说法

![nginx-sni-backend](Images/nginx-sni-backend.png)

发现请求的确实是backend

然后测试请求127.0.0.1:12346

```bash
curl https://127.0.0.1:12346
```

![nginx-sni-127](Images/nginx-sni-127.png)
