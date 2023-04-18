---
title: Linux 安装 Nginx
date: {{ date }}
categories:
- Linux
---

## Nginx 安装

安装依赖包

```sh
yum -y install gcc zlib zlib-devel pcre-devel openssl openssl-devel
```

下载并解压nginx

```sh
wget http://nginx.org/download/nginx-1.18.0.tar.gz
tar -xvf nginx-1.18.0.tar.gz
```

进入nginx目录安装ssl模块

```sh
./configure --with-http_stub_status_module --with-http_ssl_module
```

执行编译

```sh
make && make install
```

nginx默认会在`/usr/local/`下创建一个`nginx`目录

启动脚本会在 `/usr/local/nginx/sbin` 目录下

启动脚本

```sh
./nginx
```

软连接启动脚本

```sh
ln -s /usr/local/nginx/sbin/nginx /usr/sbin
```

`nginx` 命令

```sh
# 启动nginx
nginx
# 关闭nginx
nginx -s stop
# 重启nginx
nginx -s reload
```

可能出现的报错

```sh
nginx: [alert] could not open error log file: open() "/usr/local/nginx/logs/error.log" failed (2: No such file or directory)
```

解决方案：在nginx目录下创建 logs 文件夹并授权

```sh
mkdir logs
chmod 700 logs
```

## Nginx 配置

配置文件位置 `/usr/local/nginx/conf/nginx.conf`

配置文件样例

```json
worker_processes  1;

events {
    worker_connections  1024;
}

http {
    include             mime.types;
    default_type        application/octet-stream;
    sendfile            on;
    keepalive_timeout   65;

    # 配置SSL
    ssl_certificate      /usr/local/nginx/ssl/7995548__51nftcard.com.pem;        # 配置证书
    ssl_certificate_key  /usr/local/nginx/ssl/7995548__51nftcard.com.key;        # 配置证书私钥
    ssl_protocols        TLSv1 TLSv1.1 TLSv1.2;                                  # 配置SSL协议版本 
    ssl_ciphers          ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE; # 配置SSL加密算法
    ssl_prefer_server_ciphers   on;                 # 优先采取服务器算法
    ssl_session_cache           shared:SSL:10m;     # 配置共享会话缓存大小
    ssl_session_timeout         10m;                # 配置会话超时时间

    # 配置跨域
    add_header Access-Control-Allow-Origin '*';
    add_header Access-Control-Allow-Methods 'GET, POST, OPTIONS';
    add_header Access-Control-Allow-Headers 'DNT, X-Mx-ReqToken, Keep-Alive, User-Agent, X-Requested-With, If-Modified-Since, Cache-Control, Content-Type, Authorization';
        
    # 配置转发请求头，让应用可以受到真实的请求
    proxy_set_header Host $proxy_host; 
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

    server {
        listen       443 ssl;			# 配置端口
        server_name  item.demo.com;		# 配置域名
        
        location / {
            proxy_pass  http://127.0.0.1:8081;
            if ($request_method = 'OPTIONS') {
                return 204;
            }
        }

        location /job/ {
            proxy_pass  http://127.0.0.1:9999/;
            if ($request_method = 'OPTIONS') {
                return 204;
            }
        }
    }
}
```

