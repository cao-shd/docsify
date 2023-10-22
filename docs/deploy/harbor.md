# HARBOR 部署

## 注意

* 使用单独的机器安装
* 脚本安装 nginx/redis 镜像  
* 使用 80/443 端口
* 需配置 HTTPS

## 创建路径

```shell
mkdir -p /usr/local/harbor
```

## 下载

[](https://github.com/goharbor/harbor/releases)

```shell
curl -L https://github.com/goharbor/harbor/releases/download/v2.8.4/harbor-offline-installer-v2.8.4.tgz \
     -o /usr/local/harbor/harbor-installer.tgz
```

## 解压

```shell
cd /usr/local && tar zxvf /usr/local/harbor/harbor-installer.tgz

```

## 配置

复制配置

```shell
cd /usr/local/harbor && cp harbor.yml.tmpl harbor.yml
```

修改配置

```shell
sed -i 's@^https:@#https:' /usr/local/harbor/harbor.yml
sed -i 's@^  port: 443@#  port: 443@' /usr/local/harbor/harbor.yml
sed -i 's@^  certificate: /your/certificate/path@#  certificate: /your/certificate/path@' /usr/local/harbor/harbor.yml
sed -i 's@^  private_key: /your/private/key/path@#  private_key: /your/private/key/path@' /usr/local/harbor/harbor.yml
sed -i 's@^harbor_admin_password: Harbor12345@harbor_admin_password: 123456@' /usr/local/harbor/harbor.yml
sed -i 's@^hostname: reg\.mydomain\.com@hostname: 192.168.127.10@' /usr/local/harbor/harbor.yml

```

## 安装启动

```shell
cd /usr/local/harbor && ./install.sh
```

## 创建服务

服务脚本

```shell
cat > /etc/systemd/system/harbor.service <<'EOF'
[Unit]
Description=Harbor
After=docker.service systemd-networkd.service systemd-resolved.service
Requires=docker.service
Documentation=http://github.com/vmware/harbor

[Service]
Type=simple
Restart=on-failure
RestartSec=5
ExecStart=/usr/local/bin/docker-compose -f /usr/local/harbor/docker-compose.yml up
ExecStop=/usr/local/bin/docker-compose -f /usr/local/harbor/docker-compose.yml down

[Install]
WantedBy=multi-user.target
EOF
```

开机自启

```shell
systemctl enable harbor
```

## 其他机器连接

修改配置

```shell
vi /etc/docker/deamon.json

{
  ...
  "insecure-registries": [ "192.168.127.10:80" ]
  ...
}

```

登录

```shell
docker login -u admin -p "123456" 192.168.127.10:80
```

镜像打标签

```shell
docker pull nginx
docker tag $(docker images | awk '/^nginx/ {print $3}') 192.168.127.10:80/library/nginx:latest
```

推送镜像

```shell
docker push 192.168.127.10:80/library/nginx:latest
```
