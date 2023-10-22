# MYSQL 部署

## 配置路径

```shell
mkdir -p /usr/local/mysql/{script,data}
```

```shell
cat > /usr/local/mysql/script/init.sql <<'EOF'
CREATE DATABASE IF NOT EXISTS `camunda` DEFAULT CHARACTER SET utf8mb4 DEFAULT COLLATE utf8mb4_general_ci;
EOF
```

## COMPOSE 文件

```shell
cat > /usr/local/mysql/docker-compose.yml <<'EOF'
version: '3.6'
services:
  mysql:
    image: mysql:8.0.20
    restart: always
    container_name: mysql
    environment:
      MYSQL_USER: root
      MYSQL_PASSWORD: 123456
      MYSQL_ROOT_PASSWORD: 123456
    command:
      --default-authentication-plugin=mysql_native_password
      --character-set-server=utf8mb4
      --collation-server=utf8mb4_general_ci
      --explicit_defaults_for_timestamp=true
      --lower_case_table_names=1
    ports:
      - 3306:3306
    volumes:
      - /usr/local/mysql/data:/var/lib/mysql
      - /etc/localtime:/etc/localtime:ro
      - /usr/local/mysql/script/init.sh:/docker-entrypoint-initdb.d/init.sh
EOF
```

## 启动

```shell
cd /usr/local/mysql/ && docker-compose up -d
```

## 配置

进入容器

```shell
docker exec -it mysql /bin/bash
```

登录

```shell
mysql -u root -p
```

开启权限

```sql
use mysql;
ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY '123456';
flush privileges;
```

退出登录

```sql
\q
```

退出容器

```shell
exit
```
