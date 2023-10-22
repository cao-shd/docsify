
# SONARQUBE 安装

## 环境准备

查看配置

```shell
cat /etc/sysctl.conf
```

追加配置

```shell
echo "net.ipv4.ip_forward=1" > /etc/sysctl.conf
echo "vm.max_map_count=262144" > /etc/sysctl.conf
```

重载配置

```shell
sysctl -p
```

## 创建路径

```shell
mkdir -p /usr/local/sonar/
```

## COMPOSE 文件

```shell
cat > /usr/local/sonar/docker-compose.yml <<'EOF'
version: '3.6'
services:
  sonardb:
    restart: always
    image: postgres
    container_name: sonar_database
    ports:
      - 5432:5432
    networks:
      - network
    environment:
      POSTGRES_USER: sonar
      POSTGRES_PASSWORD: sonar@123456
      TZ: Asia/Shanghai
  sonarqube:
    restart: always
    image: sonarqube:community
    container_name: sonarqube
    depends_on:
      - sonardb
    ports:
      - 9000:9000
    networks:
      - network
    environment:
      SONAR_JDBC_URL: jdbc:postgresql://sonardb:5432/sonar
      SONAR_JDBC_USERNAME: sonar
      SONAR_JDBC_PASSWORD: sonar@123456
networks:
  network:
    driver: bridge
EOF
```

## 启动

```shell
cd /usr/local/sonar/ && docker-compose up -d
```

## 登录

```shell
username: admin
password: admin
```
