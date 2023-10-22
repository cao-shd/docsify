## 安装kubectl
```shell
# 切换镜像源
cat <<EOF | tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
   http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

# 官网查看最新版本
https://github.com/kubernetes/kubernetes/releases/

# 安装
yum install -y kubectl-1.27.3 --disableexcludes=kubernetes
```
## 安装 minikube
```shell

# 获取 minikube
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 \
&& chmod +x minikube \
&& sudo mv minikube /usr/local/bin/
```
## 使用 minikube 用户启动dkcker
```bash
# 添加一个用户
adduser minikube
# 将该用户添加到docker组 --> gpasswd -a 用户名 用户组
gpasswd -a minikube docker
# 切换到该用户
su minikube -
# 将当前用户切换到docker组
newgrp - docker

# 启动docker
systemctl start docker
```
```bash
minikube start --driver=docker
```

