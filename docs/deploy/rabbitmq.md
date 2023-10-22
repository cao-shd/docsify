# RABBITMQ 部署

## 创建路径

```shell
mkdir -p /usr/local/rabbitmq/
```

## COMPOSE 文件

```shell
# 编辑配置文件
cat > /usr/local/rabbitmq/docker-compose.yml <<'EOF'
version: '3.6'
services:
  rabbitmq:
    restart: always
    image: rabbitmq:management
    container_name: rabbitmq
    environment:
      RABBITMQ_DEFAULT_USER: rabbit
      RABBITMQ_DEFAULT_PASS: 123456
    ports:
      - 5672:5672
      - 15672:15672
    volumes:
      - /usr/local/jenkins/data/:/var/jenkins_home/
      - /usr/bin/docker:/usr/bin/docker
      - /etc/docker/daemon.json:/etc/docker/daemon.json
      - /var/run/docker.sock:/var/run/docker.sock
EOF
```

## 启动

```shell
cd /usr/local/jenkins/ && docker-compose up -d
```

## 插件

下载： [rabbitmq_delayed_message_exchang](https://github.com/rabbitmq/rabbitmq-delayed-message-exchange/releases)
复制到容器

```shell
docker cp rabbitmq_delayed_message_exchange* rabbitmq:/plugins
```

进入容器

```shell
docker exec -it rabbitmq /bin/bash
```

启用插件

```shell
cd /plugins
rabbitmq-plugins enable rabbitmq_delayed_message_exchange
```
