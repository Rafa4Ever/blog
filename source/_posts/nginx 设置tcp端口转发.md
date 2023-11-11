---
title: Nginx 配置tcp端口转发
date: 2023/11/11 09:11:03
categories: devops
tags: [nginx, tcp, geoip]
---
环境：
    centos7.9
    nginx 1.22.0

## Quick Start

### 安装编译环境

``` bash
$ yum -y install zlib zlib-devel openssl openssl-devel pcre pcre-devel GeoIP GeoIP-devel GeoIP-data gcc gcc-c++ autoconf automake make -y
```

### 安装 MaxMind GeoIP2
利用该模块做请求源ip的路由区域转发
``` bash
$ cd /usr/local/src/
$ wget https://github.com/maxmind/libmaxminddb/releases/download/1.7.1/libmaxminddb-1.7.1.tar.gz
$ tar -xvzf libmaxminddb-1.7.1.tar.gz
$ cd libmaxminddb-1.7.1
$ ./configure
$ make && make install
#克隆仓库ngx_http_geoip2_module.git到/usr/lib/nginx/modules/ngx_http_geoip2_module:
$ git clone https://github.com/leev/ngx_http_geoip2_module.git /usr/lib/nginx/modules/ngx_http_geoip2_module
$ ln -s /usr/local/lib/libmaxminddb.so.0.0.7 /lib64/libmaxminddb.so.0
$ rm -rf /usr/local/src/libmaxminddb-1.7.1.tar.gz
```


### 编译安装 nginx

``` bash
$ cd /usr/local/src/
$ wget http://nginx.org/download/nginx-1.22.0.tar.gz
$ tar -xvzf nginx-1.22.0.tar.gz
$ mv nginx-1.22.0 nginx &&  cd nginx-1.22.0	
$ ./configure --prefix=/usr/share/nginx --conf-path=/etc/nginx/nginx.conf --http-log-path=/var/log/nginx/access.log --error-log-path=/var/log/nginx/error.log  --lock-path=/var/lock/nginx.lock --pid-path=/run/nginx.pid  --modules-path=/usr/lib/nginx/modules  --http-client-body-temp-path=/var/lib/nginx/body  --http-fastcgi-temp-path=/var/lib/nginx/fastcgi  --http-proxy-temp-path=/var/lib/nginx/proxy  --http-scgi-temp-path=/var/lib/nginx/scgi  --http-uwsgi-temp-path=/var/lib/nginx/uwsgi  --with-compat  --with-debug  --with-pcre-jit  --with-http_ssl_module  --with-http_stub_status_module  --with-http_realip_module  --with-http_auth_request_module  --with-http_v2_module  --with-http_dav_module  --with-http_slice_module  --with-threads   --with-stream_ssl_module --with-http_addition_module  --with-http_gunzip_module  --with-http_gzip_static_module  --with-http_sub_module  --with-stream  --with-stream_realip_module --add-dynamic-module=/usr/lib/nginx/modules/ngx_http_geoip2_module
$ make && make install
$ ln -s /usr/share/nginx/sbin/nginx /usr/bin/nginx 
$ rm -rf /usr/local/src/nginx-1.22.0.tar.gz
```


### 配置nginx tcp转发规则

``` bash
#user  nobody;
worker_processes  1;

error_log  /var/log/nginx/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;
load_module /usr/lib/nginx/modules/ngx_http_geoip2_module.so;
load_module /usr/lib/nginx/modules/ngx_stream_geoip2_module.so;


events {
    worker_connections  1024;
}


stream {
    log_format proxy '$remote_addr [$time_local] '
                 '$protocol $status $bytes_sent $bytes_received '
                 '$session_time "$upstream_addr" '
                 '"$upstream_bytes_sent" "$upstream_bytes_received" "$upstream_connect_time"';
    access_log /var/log/nginx/access.log proxy;
    #gzip  on;
    #include /etc/nginx/conf.d/*.conf;

    geoip2 /usr/share/nginx/geoip/gc1.mmdb{
        $geoip2_data_country_code country iso_code;
    }

    map $geoip2_data_country_code $nearest_server {
        default      as;
        us           na;
    }

    upstream as {
          server 1.2.3.4:4901;
    }

    upstream na {
          server 2.3.4.5:4901;
    }
    # ...
    server {
        listen       4900;
        proxy_pass   $nearest_server;
        #proxy_protocol on;
    }
}
```

参考文档: [docs-nginx](https://docs.nginx.com/nginx/admin-guide/security-controls/controlling-access-by-geoip/#example)
