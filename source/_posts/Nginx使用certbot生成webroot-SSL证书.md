---
title: Nginx使用certbot生成webroot SSL证书
date: 2023-11-11 13:42:00
tags: [nginx, certbot, ssl]
categories: devops
---
环境：
    centos7.9
    nginx 1.22.0

### 安装certbot

``` bash
$ yum install certbot-nginx -y
```

### 生成证书

``` bash
$ certbot certonly --email YOUR@EMAIL.COM -n --agree-tos --webroot -w /usr/share/nginx/html/ -d YOUR DOMAIN HERE
```

### 配置nginx

``` bash
$ vim /etc/nginx/nginx.conf

# 增加以下内容
# ...
        server {         
            listen       80;
            root         /usr/share/nginx/html;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

            
            location /path { 
                proxy_pass http://ip:port/path;
            }
        
            error_page 500 502 503 504 /50x.html;
                location = /50x.html {
            }
        }
        server {
            listen       443 ssl;
            root         /usr/share/nginx/html;
            ssl_certificate /etc/letsencrypt/live/YOUR DOMAIN/fullchain.pem;
            ssl_certificate_key /etc/letsencrypt/live/YOUR DOMAIN/privkey.pem;
            # Load configuration files for the default server block.
            #include /etc/nginx/default.d/*.conf;
            ssl_session_timeout 5m;
            #ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
            ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
            ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
            ssl_prefer_server_ciphers on;

            # 配置转发
            location / {
                proxy_pass http://127.0.0.1:80/;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            }
        }

# ...
```

