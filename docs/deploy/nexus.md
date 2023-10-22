# NEXUS 安装

## 创建路径

```shell
mkdir /usr/local/nexus && chown -R 200 /usr/local/nexus
```

## COMPOSE 文件

```shell
cat > /usr/local/nexus/docker-compose.yml <<'EOF'
version: '3.6'
services:
  nexus3:
    restart: always
    container_name: nexus
    image: sonatype/nexus3
    environment:
      - INSTALL4J_ADD_VM_PARAMS=-Xms2g -Xmx2g -XX:MaxDirectMemorySize=2g -Duser.timezone=Asia/Shanghai
    ports:
      - 8081:8081
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /etc/timezone:/etc/timezone:ro
      - /var/run/docker.sock:/var/run/docker.sock
      - /usr/local/nexus:/nexus-data
    healthcheck:
      test: ["CMD-SHELL", "curl -f localhost:8081 || exit 1"]
    logging:
        driver: "json-file"
        options:
          max-size: "10m"
EOF
```

## 启动

```shell
cd /usr/local/nexus/ && docker-compose up -d
```
