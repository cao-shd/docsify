# DOCKER-COMPOSE 安装

## 下载

```shell
curl -L "https://github.com/docker/compose/releases/download/v2.11.2/docker-compose-linux-x86_64" \
     -o /usr/local/bin/docker-compose
```

## 添加权限

```shell
chmod +x /usr/local/bin/docker-compose
```

## 添加软连接

```shell
ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
```

## 验证

```shell
docker-compose --version
```
