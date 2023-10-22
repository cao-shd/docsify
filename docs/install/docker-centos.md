# DOCKER 安装

## 环境重置

```shell
yum remove docker \
    docker-client \
    docker-client-latest \
    docker-common \
    docker-latest \
    docker-latest-logrotate \
    docker-logrotate \
    docker-engine
```

## 环境准备

```shell
yum install -y yum-utils
```

## 配置镜像

```shell
# 使用阿里提供的镜像
yum-config-manager \
    --add-repo \
    http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

yum makecache fast
```

## 安装

```shell
yum install -y docker-ce docker-ce-cli containerd.io
```

## 验证

```shell
docker version
```

## 开机自启

```shell
systemctl enable docker --now
```

## 镜像加速

配置目录

```shell
mkdir -p /etc/docker
```

配置文件

```shell
tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://68k4zzfk.mirror.aliyuncs.com"],
  "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF

```

重载配置

```shell
systemctl daemon-reload

```

重启服务

```shell
systemctl restart docker
```

## 卸载

卸载依赖

```shell
yum remove docker-ce docker-ce-cli containerd.io
```

移除资源

```shell
rm -rf /var/lib/docker
```
