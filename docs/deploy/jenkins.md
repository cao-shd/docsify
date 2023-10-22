# JENKINS 部署

## 创建路径

```shell
mkdir -p /usr/local/jenkins/data/ && chmod -R +w /usr/local/jenkins/data/
```

## COMPOSE 文件

```shell
# 编辑配置文件
cat > /usr/local/jenkins/docker-compose.yml <<'EOF'
version: '3.6'
services:
  jenkins:
    restart: always
    user: root
    image: jenkins/jenkins:lts-jdk11
    container_name: jenkins
    ports:
      - 8082:8080
      - 50000:50000
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

## 镜像

```shell
sed -i 's#<url>https://updates.jenkins.io/update-center.json</url>#<url>https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json</url>#' \
       /usr/local/jenkins/data/hudson.model.UpdateCenter.xml
```

- 查看密码登录Jenkins，并登录下载插件

```shell
docker exec -it jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

## 插件

```shell
Git Parameter
Publish Over SSH
SonarQube Scanner
```
