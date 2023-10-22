# Kubernetes

## Pod

### 查看

查看指定命名空间的 pod

```shell
kubectl get pods -n default
kubectl get pods -n kube-system
```

查看所有命名空间的 pod

```shell
kubectl get pods -A
```

查看 pod 详细信息

```shell
kubectl get pods -o wide -A
```

实时监控 pod 详细信息

```shell
kubectl get pods -o wide -A -w
```

描述 pod 详细信息

```shell
kubectl describe pod xxxxx
```


查看 pod 命名帮助信息

```shell
kubectl get pods --help
```

### 创建

命令行创建

```shell
kubectl run nginx --image=nginx
```

创建路径

```shell
mkdir -p /usr/local/kubernetes/config/ && cd /usr/local/kubernetes/config/
```

配置文件创建

```shell

cat > nginx-pod.yml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - name: nginx
      image: nginx
      ports: 
        - containerPort: 80
EOF
```