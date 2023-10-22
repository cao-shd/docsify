
# REDIS 安装

## 配置文件

创建路径

```shell
mkdir -p /usr/local/redis/{data,conf,logs}
```

官网获取 [redis.conf](http://download.redis.io/redis-stable/redis.conf)

```sehll
curl -L http://download.redis.io/redis-stable/redis.conf\
     -o /usr/local/redis/conf/redis.conf
```

修改配置

```shell
# 创建配置文件
sed -i 's/^bind 127.0.0.1/#bind 127.0.0.1/' /usr/local/redis/conf/redis.conf
sed -i 's/^appendonly no/appendonly yes/' /usr/local/redis/conf/redis.conf
sed -i 's/^no-appendfsync-on-rewrite no/no-appendfsync-on-rewrite yes/' /usr/local/redis/conf/redis.conf
```

## COMPOSE 文件

```shell
cat > /usr/local/redis/docker-compose.yml <<'EOF'
version: '3.6'
services:
  redis:
    restart: always
    image: redis
    container_name: redis
    ports:
      - 6379:6379
    volumes:
      - /usr/local/redis/data:/data
      - /usr/local/redis/conf/redis.conf:/usr/local/etc/redis/redis.conf
      - /usr/local/redis/logs:/logs
EOF
```

## 启动

```shell
cd /usr/local/redis/ && docker-compose up -d
```

## CLI

登录

```shell
docker exec -it redis redis-cli
```

KEY 操作

```sql
set name redis
get name
```

退出

```shell
quit
```
