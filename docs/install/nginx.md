# Nginx 安装

## 下载安装

官网下载

```shell
curl -O  https://nginx.org/download/nginx-1.24.0.tar.gz
```

解压安装

```shell
tar -zxvf nginx-1.24.0.tar.gz
```

安装依赖

```shell
yum install -y \
  gcc \
  pcre pcre-devel \
  zlib zlib-devel \
  openssl openssl-devel
```

编译配置

```shell
cd nginx-1.24.0 && ./configure --prefix=/usr/local/nginx --with-http_stub_status_module --with-http_ssl_module
```

编译安装

```shell
make && make install
```

启动服务

```shell
cd /usr/local/nginx/sbin/ && ./nginx
```

查看状态

```shell
ps -ef | grep nginx
```

访问服务

```shell
curl http://192.168.127.10/
```

重载配置

```shell
cd /usr/local/nginx/sbin/ && ./nginx -s reload
```

停止服务

```shell
cd /usr/local/nginx/sbin/ && ./nginx -s quit
```

## 服务脚本

创建服务

```shell
tee /usr/lib/systemd/system/nginx.service <<-'EOF'
[Unit]
Description=nginx - web server
After=network.target remote-fs.target nss-lookup.target
[Service]
Type=forking
PIDFile=/usr/local/nginx/logs/nginx.pid
ExecStartPre=/usr/local/nginx/sbin/nginx -t -c /usr/local/nginx/conf/nginx.conf
ExecStart=/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
ExecReload=/usr/local/nginx/sbin/nginx -s reload
ExecStop=/usr/local/nginx/sbin/nginx -s stop
ExecQuit=/usr/local/nginx/sbin/nginx -s quit
PrivateTmp=true
[Install]
WantedBy=multi-user.target
EOF
```

加载服务

```shell
systemctl daemon-reload
```

启动服务

```shell
systemctl start nginx
```

查看服务

```shell
systemctl status nginx
```

开机自启

```shell
systemctl enable nginx
```

## 虚拟主机

创建网站

```shell
mkdir -p  /www/{aaa,bbb,ccc}

echo "hello aaa" > /www/aaa/index.html
echo "hello bbb" > /www/bbb/index.html
echo "hello ccc" > /www/ccc/index.html
```

修改 HOSTS

```shell
echo "127.0.0.1 www.aaa.com www.bbb.com www.ccc.com" >> /etc/hosts
```

修改配置

```shell
cat > /usr/local/nginx/conf/nginx.conf <<'EOF'
worker_processes 1;

events {
  worker_connections 1024;
}

http {
  include mime.types;
  default_type application/octet-stream;
  sendfile on;
  keepalive_timeout 65;

  server {
    listen 80;
    # 修改主机名
    server_name www.aaa.com;
    location / {
      # 修改访问站点路径
      root /www/aaa;
      index index.html;
    }
  }
  server {
    listen 80;
    # 修改主机名
    server_name www.bbb.com;
    location / {
      # 修改访问站点路径
      root /www/bbb;
      index index.html;
    }
  }
  server {
    listen 80;
    # 修改主机名
    server_name www.ccc.com;
    location / {
      # 修改访问站点路径
      root /www/ccc;
      index index.html;
    }
  }

}
EOF
```

重载配置

```shell
systemctl reload nginx
```

访问地址

```shell
curl http://www.aaa.com
curl http://www.bbb.com
curl http://www.ccc.com
```

## 反向代理

修改配置

```shell
cat > /usr/local/nginx/conf/nginx.conf <<'EOF'
worker_processes 1;

events {
  worker_connections 1024;
}

http {
  include mime.types;
  default_type application/octet-stream;
  sendfile on;
  keepalive_timeout 65;

  server {
    listen 80;
    server_name www.aaa.com;
    location / {
      # 反向代理 http://<HOST>/<IP>:<PORT>
      proxy_pass http://www.bbb.com;
    }
  }
  
  server {
    listen 80;
    # VHOST
    server_name www.bbb.com;
    location / {
      root /www/bbb;
      index index.html;
    }
  }

}
EOF
```

重载配置

```shell
systemctl reload nginx
```

访问地址

```shell
curl http://www.aaa.com
```

## 负载均衡

创建网站

```shell
mkdir -p  /www/aaa{1,2}
echo "hello aaa1" > /www/aaa1/index.html
echo "hello aaa2" > /www/aaa2/index.html
```

修改 HOSTS

```shell
echo "127.0.0.1 www.aaa1.com www.aaa2.com" >> /etc/hosts
```

修改配置

```shell
cat > /usr/local/nginx/conf/nginx.conf <<'EOF'
worker_processes 1;

events {
  worker_connections 1024;
}

http {
  include mime.types;
  default_type application/octet-stream;
  sendfile on;
  keepalive_timeout 65;

  upstream aaa {
    server www.aaa1.com:81 weight=8;
    server www.aaa2.com:82 weight=2;
  }
    
  server {
    listen 80;
    server_name www.aaa.com;
    location / {
      # 反向代理 http://<upstream_name>
      proxy_pass http://aaa;
    }
  }

  server {
    listen 81;
    server_name www.aaa1.com;
    location / {
      root /www/aaa1;
      index index.html;
    }
  }
    
  server {
    listen 82;
    server_name www.aaa2.com;
    location / {
      root /www/aaa2;
      index index.html;
    }
  }

}
EOF
```

重载配置

```shell
systemctl reload nginx
```

访问地址

```shell
curl http://www.aaa.com (*4)
```

## 动静分离

创建资源

```shell
mkdir -p /www/aaa/static/js
echo '<body><script src="js/index.js"></script></body>' > /www/bbb/index.html
echo 'document.body.innerText="js write";' > /www/aaa/static/js/index.js
```

修改配置

```shell
cat > /usr/local/nginx/conf/nginx.conf <<'EOF'
worker_processes 1;

events {
  worker_connections 1024;
}

http {
  include mime.types;
  default_type application/octet-stream;
  sendfile on;
  keepalive_timeout 65;

  server {
    listen 80;
    server_name www.aaa.com;
    location / {
      # 动态资源访问后端
      proxy_pass http://www.bbb.com;
    }
    # 正则表达式匹配路径
    location ~*/(css|js) {
      # 静态资源访问本地
      root /www/aaa/static;
    }
  }
  server {
    listen 80;
    server_name www.bbb.com;
    location / {
      root /www/bbb;
      index index.html;
    }
  }

}
EOF
```

重载配置

```shell
systemctl reload nginx
```

访问地址

```shell
curl http://www.aaa.com
```

## 重定向

修改配置

```shell
cat > /usr/local/nginx/conf/nginx.conf <<'EOF'
worker_processes 1;

events {
  worker_connections 1024;
}

http {
  include mime.types;
  default_type application/octet-stream;
  sendfile on;
  keepalive_timeout 65;

  server {
    listen 80;
    server_name www.aaa.com;
    location / {
      # rewrite <regex> <replacement> [flag];
      rewrite ^/index/([0-9]+).html$ /index.html?pageNum=$1 break;
      index index.html;
    }
  }
}
EOF
```

重载配置

```shell
systemctl reload nginx
```

访问地址

``` shell
curl http://www.aaa.com/index/1.html
```

## 防盗链

创建资源

```shell
mkdir -p /www/aaa/static/js
echo '<body><script src="js/index.js"></script></body>' > /www/aaa/index.html
echo 'document.body.innerText="js write";' > /www/aaa/static/js/index.js
```

修改配置

```shell
cat > /usr/local/nginx/conf/nginx.conf <<'EOF'
worker_processes 1;

events {
  worker_connections 1024;
}

http {
  include mime.types;
  default_type application/octet-stream;
  sendfile on;
  keepalive_timeout 65;

  server {
    listen 80;
    server_name www.aaa.com;
    location / {
      root /www/aaa;
      index index.html;
    }
    location ~*/(css|js) {
      # 设置允许访问的域名
      # 设置 none 表示没有 refer 也可以访问
      valid_referers www.aaa.com;
      if ($invalid_referer) {
        # 注意只能是 css|js|img 资源
        # rewrite ^/ /img/error.png break;
        return 403;
      }
      root /www/aaa/static;
    }
  }

}
EOF
```

重载配置

```shell
systemctl reload nginx
```

访问地址

``` shell
curl http://www.aaa.com/js/index.js
curl -H "Referer:http://www.aaa.com/" http://www.aaa.com/js/index.js
```

## HTTPS

准备证书

```shell
pwd 

# /usr/local/nginx/conf/

ls *.key *.pem

# www.aaa.com.key
# www.aaa.com.pem
```

修改配置

```shell
cat > /usr/local/nginx/conf/nginx.conf <<'EOF'
worker_processes 1;

events {
  worker_connections 1024;
}

http {
  include mime.types;
  default_type application/octet-stream;
  sendfile on;
  keepalive_timeout 65;

  server {
    listen 80;
    server_name www.aaa.com;
    # 自动跳转 https 请求
    return 301 https://$server_name$request_uri;
  }
  server {
    listen 443 ssl;
    server_name www.aaa.com;
    ssl_certificate www.aaa.com.key;
    ssl_certificate_key www.aaa.com.key;
    location / {
      root /www/aaa/;
      index index.html;
    }
  }

}
```

重载配置

```shell
systemctl reload nginx
```

访问地址

``` shell
curl http://www.aaa.com
curl https://www.aaa.com
```

## 高可用

> 使用 [KEEPALIVED](docs/install/keepalived.md) 实现

## 插件安装

查看

```shell
/usr/local/nginx/sbin/nginx -V 
```

显示

```shell
nginx version: nginx/1.24.0
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-44) (GCC)
built with OpenSSL 1.0.2k-fips  26 Jan 2017
TLS SNI support enabled
configure arguments: --prefix=/usr/local/nginx --with-http_stub_status_module --with-http_ssl_module
```

配置

```shell
./configure --prefix=/usr/local/nginx \
  --with-http_stub_status_module \
  --with-http_ssl_module \
  --add-module=<module_path>
```

编译

```shell
make
```

备份

```shell
mv /usr/local/nginx/sbin/nginx /usr/local/nginx/sbin/nginx.bak
```

更新

```shell
cp -r objs/nginx /usr/local/nginx/sbin/nginx
```
