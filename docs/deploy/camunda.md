# CAMUNDA 部署

## 配置文件

创建路径

```shell
mkdir -p /usr/local/camunda/data
```

> [!attention] 数据库环境要提前准备

```sql
CREATE DATABASE IF NOT EXISTS `camunda` DEFAULT CHARSET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

## COMPOSE 文件

```shell
cat > /usr/local/camunda/docker-compose.yml <<'EOF'
version: '3.6'
services:
  camunda:
    restart: always
    image: camunda/camunda-bpm-platform
    container_name: camunda
    ports:
      - 8080:8080
    environment:
      DB_DRIVER: com.mysql.cj.jdbc.Driver
      DB_URL: jdbc:mysql://192.168.127.10:3306/camunda
      DB_USERNAME: root
      DB_PASSWORD: 123456
      WAIT_FOR: 192.168.127.10:3306
      WAIT_FOR_TIMEOUT: 60
      TZ: Asia/Shanghai
EOF
```

## 启动

```shell
cd /usr/local/camunda/ && docker-compose up -d
```

## 访问

```shell
curl http://<hostname>:8080/camunda
```

> [!tip]
> username: demo <br>
> password: demo

## 流程编辑器

```shell
curl https://github.com/camunda/camunda-modeler/releases/download/v5.14.0/camunda-modeler-5.14.0-linux-x64.tar.gz
```
