# Gitlab 部署

## Gitlab 部署

>
> 前提 : docker 和docker-compose 都已准备完成。

- 创建配置文件

```shell
# 创建配置文件存放路径
mkdir -p /usr/local/gitlab
cd /usr/local/gitlab

# 编辑配置文件
vi docker-compose.yml

# 添加如下内容
version: '3.6'
services:
  gitlab:
    image: 'gitlab/gitlab-ce:latest'
    container_name: gitlab-server
    restart: always
    hostname: '192.168.131.50'
    environment:
      TZ: "Asia/Shanghai"
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'http://192.168.131.50:8888'
        gitlab_rails['gitlab_shell_ssh_port'] = 2222
        gitlab_rails['time_zone'] = 'Asia/Shanghai'
    ports:
      - '8888:8888'
      - '2222:22'
    volumes:
      - './config:/etc/gitlab'
      - './logs:/var/log/gitlab'
      - './data:/var/opt/gitlab'     
```

- 创建gitlab-server

```shell
docker-compose up -d
```

- 获取启动密码

```shell
docker exec -it $(docker ps | grep gitlab-server | awk '{print $1}') grep 'Password:' /etc/gitlab/initial_root_password

用户名： root
密码： nNtcb86aTTHBAq4joW9OMJvxA8D0D/9qvjvlYNhztcM=
```

- 配置邮件服务

```shell
# 修改如下配置文件

# 修改如下内容
gitlab_rails['smtp_enable'] = true
gitlab_rails['smtp_address'] = "smtp-mail.outlook.com"
gitlab_rails['smtp_port'] = 587
gitlab_rails['smtp_user_name'] = "<username>@outlook.com"
gitlab_rails['smtp_password'] = "<password>"
gitlab_rails['smtp_domain'] = "outlook.com"
gitlab_rails['smtp_authentication'] = "login"
gitlab_rails['smtp_enable_starttls_auto'] = true
gitlab_rails['smtp_tls'] = false
gitlab_rails['smtp_openssl_verify_mode'] = 'none'
gitlab_rails['gitlab_email_enabled'] = true
gitlab_rails['gitlab_email_from'] = '<username>@outlook.com'
```

- 生成 ssh-key

```shell
# 使用 git bash
ssh-keygen -t rsa -C 'cao-shd@outlook.com'
# 连按三次回车
```

- 获取公钥

```shell
# 查看公钥内容
cat ~/.ssh/id_rsa.pub

# 复制公钥内容
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDTQoHD4STdAvjYuRFxSuhWJMuQvFCC5HAwF1FIqBvXLtmtycVJ/mgJO1SHoD+rFwbWt+pdD8jqScgq0yv7+BFt/S6an2MiF3ZBku/cEpWYJCrNFTRIXyM4XW4vTU4z5hCx1DDAl5t/iGtyukvnGZfj3pUvKZceTH1AdWKHs3HceexDmoPQHUGEdUMAvzqCQFuYmSC8zmxi8GAQGJrjP/OG3dsgVBjt/50JfvStFVcvcV5mPtYwoUbMY4b5edpUoRGo++ppr7AlMrOl6fkWM7nx8FGRfJEmJzQF9ktOacPViZQAEHFJotBSeOUPui8l1rkixY2ZKwwRB2M2jW1KNswrEqCo5hEL/WHcam8kuggNR5Wi1UtIgCqX6z386ZoIpv30VLgQAII3kJBOiucZmhx2PMWYdJ6poWG+D1Ito04oylPUxONRWKNerZKbNkOXxvN1eQbGbMQBA7grD0T7mJiuuKTqpfA7d6ZrSeWvflIkcSt7pwe4oGxCImZQTJIzs1E= cao-shd@outlook.com
```

- gitlab 页面追加公钥信息

```shell
User Settings -> SSH Keys -> Add SSH Key
```

## Gitlab Runner 安装

- 安装

```shell
export GITLAB_RUNNER_HOME=/usr/local/gitlab-runner
docker run  --name gitlab-runner \
            --restart always \
            -v /usr/local/gitlab-runner/config:/etc/gitlab-runner \
            -v /var/run/docker.sock:/var/run/docker.sock \
            -d gitlab/gitlab-runner:latest
```

- 注册

```shell
# 进入容器
docker exec -it gitlab-runner bash
# 注册
gitlab-runner register \
  --non-interactive \
  --url "http://52.184.19.229" \
  --registration-token "hp4J4D4FYdnVLJ1g4wV3" \
  --executor "docker" \
  --docker-image alpine:latest \
  --description "docker-runner" \
  --tag-list "docker,default" \
  --run-untagged="true" \
  --locked="false" \
  --access-level="not_protected"
```
