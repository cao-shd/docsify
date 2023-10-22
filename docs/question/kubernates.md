## Namespace 
> 只隔离资源 不隔离网络

- 查看
```shell
kubectl get namespace/ns
```

- 创建
```shell
# 创建配置文件
vi hello-namespace.yml

apiVersion: v1
kind: Namespace
metadata:
  name: hello-namespace
```
```shell
kubectl apply -f hello-namespace.yml
```

- 删除
```shell
kubectl delete -f hello-namespace.yml
```
## Pod 
> 是多个容器的组合

- 查看
```shell
kubectl get pod [-n <namespace>/ -A] [-owide]
```

- 创建
```shell
vi hello-niginx.yml

apiVersion: v1
kind: Pod
metadata:
  labels:
    run: hello-nginx
  name: hello-nginx
#  namespace: default
spec:
  containers:
  - image: nginx
    name: hello-nginx
```
```shell
kubectl apply -f hello-niginx.yml
```

- 删除
```shell
kubectl delete -f hello-niginx.yml
```
## Deployment

- 查看
```shell
kubectl get deploys [-n <namespace>/ -A]
```

- 创建
```shell
vi hello-deploy.yml

apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: hello-deploy
  name: hello-deploy
spec:
  replicas: 3 # 创建三个副本
  selector:
    matchLabels:
      app: hello-deploy
  template:
    metadata:
      labels:
        app: hello-deploy
    spec:
      containers:
      - image: nginx
        name: hello-nginx
```
```shell
kubectl apply -f hello-deploy.yml
```

- 删除
```shell
kubectl delete -f hello-deploy.yml
```
