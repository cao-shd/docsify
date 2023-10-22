# NGINX 部署

## 配置文件

启动容器

```shell
docker run -d --name nginx nginx
```

创建路径

```shell
mkdir -p /usr/local/nginx/conf
```

拷贝配置

```shell
docker cp nginx:/etc/nginx/nginx.conf /usr/local/nginx/conf/nginx.conf
```

删除容器

```shell
docker rm -f nginx
```

## COMPOSE 文件

```shell
cat > /usr/local/nginx/docker-compose.yml <<'EOF'
version: '3.6'
services:
  nginx:
    restart: always
    image: nginx
    container_name: nginx
    ports:
      - 80:80
    volumes:
      - /usr/local/nginx/conf/nginx.conf:/etc/nginx/nginx.conf
      - /usr/local/nginx/html:/usr/share/nginx/html
      - /usr/local/nginx/logs:/var/log/nginx
EOF
```

## 启动

```shell
cd /usr/local/nginx/ && docker-compose up -d
```
