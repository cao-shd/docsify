# KUBERNETES 部署

## 主机规划

网段

```shell
192.168.127.0/24
```

主机 IP

```shell
192.168.127.05 bastion

192.168.127.10 ha vip

192.168.127.11 ha1
192.168.127.12 ha2

192.168.127.13 master01
192.168.127.14 master02
192.168.127.15 master03

192.168.127.21 worker01
192.168.127.22 worker02
```

## 通用节点环境配置

IP 地址

```shell
vi /etc/sysconfig/network-scripts/ifcfg-ens33
```

```shell
BOOTPROTO = static
ONBOOT=yes
NM_CONTROLLED=no
NETMASK=255.255.255.0
GATEWAY=192.168.127.1
DNS1=192.168.127.1
IPADDR=192.168.127.xx
UUID=<UUID>
```

关闭网络管理服务

```
systemctl stop NetworkManager
systemctl disable NetworkManager
```

重启服务

```shell
systemctl restart network
```

YUM 源

```shell
curl -o /etc/yum.repos.d/CentOS-Base.repo https://repo.huaweicloud.com/repository/conf/CentOS-7-reg.repo

yum clean all
yum makecache

yum update -y
```

常用工具
```shell
yum install wget jq psmisc vim net-tools telnet yum-utils device-mapper-persistent-data lvm2 git lrzsz -y
```

升级系统内核

```shell
yum -y install perl
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
yum -y install https://www.elrepo.org/elrepo-release-7.0-4.el7.elrepo.noarch.rpm
yum --enablerepo=elrepo-kernel -y install kernel-ml.x86_64
grub2-set-default 0
grub2-mkconfig -o /boot/grub2/grub.cfg
reboot
```

查看内核版本
```shell
cat /proc/version
```

关闭防火墙

```shell
systemctl stop firewalld
systemctl disable firewalld
systemctl status firewalld
```

关闭 selinux
```shell
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
getenforce
```

关闭 swap 分区
```shell
swapoff -a  
sed -ri 's/.*swap.*/#&/' /etc/fstab
```

时区
```shell
timedatectl set-timezone Asia/Shanghai
```

日期同步服务
```shell
yum install ntpdate -y
```

定时同步时间
```shell
crontab -e
* * * * * /usr/sbin/ntpdate time1.aliyun.com
```

重启定时任务
```shell
systemctl restart crond
systemctl enable crond
```

查看当前时间
```
date
```

主机系统优化
```shell

ulimit -SHn 65535

cat >> /etc/security/limits.conf <<'EOF'
* soft nofile 655360
* hard nofile 131072
* soft nproc 655350
* hard nproc 655350
* soft memlock unlimited
* hard memlock unlimited
EOF
```

主机名解析

```shell
cat >> /etc/hosts <<'EOF'
192.168.127.5 bastion
192.168.127.11 ha01
192.168.127.12 ha02
192.168.127.13 master01
192.168.127.14 master02
192.168.127.15 master03
192.168.127.21 worker01
192.168.127.22 worker02
192.168.127.23 worker03
EOF
```

免密登录

生成秘钥
```shell
ssh-keygen
```

分发秘钥
```shell
ssh-copy-id root@bastion
ssh-copy-id root@ha01
ssh-copy-id root@ha02
ssh-copy-id root@master01
ssh-copy-id root@master02
ssh-copy-id root@master03
ssh-copy-id root@worker01
ssh-copy-id root@worker02
ssh-copy-id root@worker03
```

验证连接
```shell
ssh root@bastion
ssh root@ha01
ssh root@ha02
ssh root@master01
ssh root@master02
ssh root@master03
ssh root@worker01
ssh root@worker02
ssh root@worker03
```


设置主机名

```shell
hostnamectl set-hostname  xxxx
```


环境配置检查

检查 NetworkManager 状态
```shell
systemctl status NetworkManager
```

检查 selinux 状态
```shell
getenforce
```

检查 swap 分区状态
```shell
free -m
```

检查时间同步状态
```shell
date
```

## HA节点环境配置

安装 haproxy 与 keepalived

```shell
yum -y install haproxy keepalived
```

HAProxy 配置

```shell
cat > /etc/haproxy/haproxy.cfg <<'EOF'
global
 maxconn 2000
 ulimit-n 16384
 log 127.0.0.1 local0 err
 stats timeout 30s

defaults
 log global
 mode http
 option httplog
 timeout connect 5000
 timeout client 50000
 timeout server 50000
 timeout http-request 15s
 timeout http-keep-alive 15s

frontend monitor-in
 bind *:33305
 mode http
 option httplog
 monitor-uri /monitor

frontend master
 bind 0.0.0.0:6443
 bind 127.0.0.1:6443
 mode tcp
 option tcplog
 tcp-request inspect-delay 5s
 default_backend master

backend master
 mode tcp
 option tcplog
 option tcp-check
 balance roundrobin
 default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
 server master01 192.168.127.13:6443 check
 server master02 192.168.127.14:6443 check
 server master03 192.168.127.15:6443 check
EOF
```

KeepAlived

健康检查脚本

```shell
cat > /etc/keepalived/check_apiserver.sh <<'EOF'
#!/bin/bash
err=0
for k in $(seq 1 3)
do
   check_code=$(pgrep haproxy)
   if [[ $check_code == "" ]]; then
       err=$(expr $err + 1)
       sleep 1
       continue
   else
       err=0
       break
   fi
done

if [[ $err != "0" ]]; then
   echo "systemctl stop keepalived"
   /usr/bin/systemctl stop keepalived
   exit 1
else
   exit 0
fi
EOF
```

```shell
chmod +x /etc/keepalived/check_apiserver.sh
```

> 主从配置不一致，需要注意。

主节点配置
```shell

cat >/etc/keepalived/keepalived.conf<<"EOF"
! Configuration File for keepalived
global_defs {
  router_id LVS_DEVEL
  script_user root
  enable_script_security
}
vrrp_script chk_apiserver {
  script "/etc/keepalived/check_apiserver.sh"
  interval 5
  weight -5
  fall 2 
  rise 1
}
vrrp_instance VI_1 {
  state MASTER
  interface ens33
  mcast_src_ip 192.168.127.11
  virtual_router_id 11
  priority 100
  advert_int 2
  authentication {
    auth_type PASS
    auth_pass K8SHA_KA_AUTH
  }
  virtual_ipaddress {
    192.168.127.10
  }
  track_script {
    chk_apiserver
  }
}
EOF
```

备节点配置
```shell
cat >/etc/keepalived/keepalived.conf<<"EOF"
! Configuration File for keepalived
! Configuration File for keepalived
global_defs {
  router_id LVS_DEVEL
  script_user root
  enable_script_security
}
vrrp_script chk_apiserver {
  script "/etc/keepalived/check_apiserver.sh"
  interval 5
  weight -5
  fall 2 
  rise 1
}
vrrp_instance VI_1 {
  state BACKUP
  interface ens33
  mcast_src_ip 192.168.127.12
  virtual_router_id 11
  priority 99
  advert_int 2
  authentication {
    auth_type PASS
    auth_pass K8SHA_KA_AUTH
  }
  virtual_ipaddress {
    192.168.127.10
  }
  track_script {
    chk_apiserver
  }
}
EOF
```



启动服务并验证

```shell
systemctl daemon-reload
systemctl enable haproxy
systemctl restart haproxy
systemctl enable keepalived
systemctl restart keepalived
```

```shell
ip address show
```

## 集群节点环境配置

创建路径

```shell 
mkdir -p /etc/kubernetes/ssl
mkdir -p /var/log/kubernetes
```

安装 IPVS 模块

```shell
yum -y install ipvsadm ipset sysstat conntrack libseccomp
```

开启 IPVS 模块

```shell
cat >/etc/modules-load.d/ipvs.conf <<EOF 
ip_vs 
ip_vs_lc 
ip_vs_wlc 
ip_vs_rr 
ip_vs_wrr 
ip_vs_lblc 
ip_vs_lblcr 
ip_vs_dh 
ip_vs_sh 
ip_vs_fo 
ip_vs_nq 
ip_vs_sed 
ip_vs_ftp 
ip_vs_sh 
nf_conntrack 
ip_tables 
ip_set 
xt_set 
ipt_set 
ipt_rpfilter 
ipt_REJECT 
ipip 
EOF
```

开启 containerd 相关模块

```shell
cat > /etc/modules-load.d/containerd.conf <<'EOF'
overlay
br_netfilter
EOF
```

开机自启动
```shell
systemctl daemon-reload
systemctl enable systemd-modules-load
systemctl restart systemd-modules-load
```

内核优化
```shell
cat > /etc/sysctl.d/k8s.conf <<'EOF'
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
fs.may_detach_mounts = 1
vm.overcommit_memory=1
vm.panic_on_oom=0
fs.inotify.max_user_watches=89100
fs.file-max=52706963
fs.nr_open=52706963
net.netfilter.nf_conntrack_max=2310720

net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_probes = 3
net.ipv4.tcp_keepalive_intvl =15
net.ipv4.tcp_max_tw_buckets = 36000
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_max_orphans = 327680
net.ipv4.tcp_orphan_retries = 3
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.ip_conntrack_max = 131072
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.tcp_timestamps = 0
net.core.somaxconn = 16384
EOF
```

应用配置
```shell
sysctl --system
```

重启
```shell
reboot
```

检查模块
```shell
lsmod | grep -e ip_vs -e nf_conntrack_ipv4
```

## 软件下载
> 在 bastion 上操作

切换工作路径
```shell
cd /usr/local/
```

下载

```shell
wget https://dl.k8s.io/v1.24.16/kubernetes-server-linux-amd64.tar.gz
```

解压

```shell
tar -xvf kubernetes-server-linux-amd64.tar.gz
```

切换工作路径
```shell
cd /usr/local/kubernetes/server/bin/
```

kubectl 安装
```shell
cp kubectl /usr/local/bin/
```

> 控制节点分发
```shell

for i in master{01,02,03}; \
do \
scp kube-apiserver kubectl kube-controller-manager kube-scheduler $i:/usr/local/bin/ ;\
done
```

> 工作节点分发
```shell
for i in worker{01,02,03}; \
do \
scp kubelet kube-proxy $i:/usr/local/bin/ ;\
done
```

## 配置证书

> 在 bastion 上操作

切换工作路径

```shell
cd /usr/local/kubernetes/
mkdir master{01,02,03}
mkdir worker{01,02,03}
```

CFSSL 证书工具

```shell
# cert cli
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
# cert json output
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
# cert info view
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
```

```shell
chmod +x cfssl*
```

```shell
mv /usr/local/kubernetes/cfssl_linux-amd64 /usr/local/bin/cfssl
mv /usr/local/kubernetes/cfssljson_linux-amd64 /usr/local/bin/cfssljson
mv /usr/local/kubernetes/cfssl-certinfo_linux-amd64 /usr/local/bin/cfssl-certinfo
```

```shell
cfssl version
```

CA 证书配置

```shell
cat > ca-csr.json <<'EOF'
{
  "CN": "kubernetes",
  "key": {
      "algo": "rsa",
      "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Beijing",
      "L": "Beijing",
      "O": "kubemsb",
      "OU": "CN"
    }
  ],
  "ca": {
    "expiry": "87600h"
  }
}
EOF
```

CA 证书

```shell
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

CA 证书策略

```shell
cat > /usr/local/kubernetes/ca-config.json <<'EOF'
{
  "signing": {
      "default": {
          "expiry": "87600h"
        },
      "profiles": {
          "kubernetes": {
              "usages": [
                  "signing",
                  "key encipherment",
                  "server auth",
                  "client auth"
              ],
              "expiry": "87600h"
          }
      }
  }
}
EOF
```

## 配置 ETCD

> 在 控制节点 上操作

```shell
mkdir -p /etc/etcd/ssl
mkdir -p /var/lib/etcd/default.etcd
mkdir -p /root/.kube
```

> 在 bastion 上操作

ETCD 证书配置

```shell
cat > etcd-csr.json <<'EOF'
{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "192.168.127.13",
    "192.168.127.14",
    "192.168.127.15"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [{
    "C": "CN",
    "ST": "Beijing",
    "L": "Beijing",
    "O": "kubemsb",
    "OU": "CN"
  }]
}
EOF
```

ECTD 证书

```shell
cfssl gencert \
  -ca=ca.pem -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  etcd-csr.json | \
cfssljson \
  -bare 
  etcd
```

安装包下载

```shell
wget https://github.com/etcd-io/etcd/releases/download/v3.5.2/etcd-v3.5.2-linux-amd64.tar.gz
```

安装包解压
```shell
tar -zxvf etcd-v3.5.2-linux-amd64.tar.gz
```

控制节点分发
```
for i in master{01,02,03}; \
do \
scp etcd-v3.5.2-linux-amd64/etcd* $i:/usr/local/bin/ ;\
done
```

ETCD 配置模板

```shell
cat > etcd.conf.tmpl <<'EOF'
#[Member]
ETCD_NAME="$ETCDNAME"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://$HOSTNAME:2380"
ETCD_LISTEN_CLIENT_URLS="https://$HOSTNAME:2379,http://127.0.0.1:2379"

#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://$HOSTNAME:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://$HOSTNAME:2379"
ETCD_INITIAL_CLUSTER="etcd01=https://$HOSTNAME1:2380,etcd02=https://$HOSTNAME2:2380,etcd03=https://$HOSTNAME3:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
EOF
```

ETCD 配置

```shell
sed -e "s|\$HOSTNAME1|192.168.127.13|g" \
    -e "s|\$HOSTNAME2|192.168.127.14|g" \
    -e "s|\$HOSTNAME3|192.168.127.15|g" \
    -e "s|\$ETCDNAME|etcd01|g" \
    -e "s|\$HOSTNAME|192.168.127.13|g" \
    etcd.conf.tmpl > master01/etcd.conf

sed -e "s|\$HOSTNAME1|192.168.127.13|g" \
    -e "s|\$HOSTNAME2|192.168.127.14|g" \
    -e "s|\$HOSTNAME3|192.168.127.15|g" \
    -e "s|\$ETCDNAME|etcd02|g" \
    -e "s|\$HOSTNAME|192.168.127.14|g" \
    etcd.conf.tmpl > master02/etcd.conf

    sed -e "s|\$HOSTNAME1|192.168.127.13|g" \
    -e "s|\$HOSTNAME2|192.168.127.14|g" \
    -e "s|\$HOSTNAME3|192.168.127.15|g" \
    -e "s|\$ETCDNAME|etcd03|g" \
    -e "s|\$HOSTNAME|192.168.127.15|g" \
    etcd.conf.tmpl > master03/etcd.conf

```

ETCD 服务配置

```shell
cat > etcd.service <<'EOF'
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
EnvironmentFile=-/etc/etcd/etcd.conf
WorkingDirectory=/var/lib/etcd/
ExecStart=/usr/local/bin/etcd \
  --cert-file=/etc/etcd/ssl/etcd.pem \
  --key-file=/etc/etcd/ssl/etcd-key.pem \
  --trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --peer-cert-file=/etc/etcd/ssl/etcd.pem \
  --peer-key-file=/etc/etcd/ssl/etcd-key.pem \
  --peer-trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --peer-client-cert-auth \
  --client-cert-auth
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

控制节点分发

```shell
for i in master{01,02,03}; \
do \
scp ca*.pem etcd*.pem $i:/etc/etcd/ssl/ ;\
scp etcd.service $i:/etc/systemd/system/ ;\
scp $i/etcd.conf $i:/etc/etcd/ ;\
done
```

## 部署 ETCD

> 在 控制节点 上操作

启动集群

```shell
systemctl daemon-reload
systemctl enable etcd
systemctl restart etcd
systemctl status etcd
```

验证状态

```shell
ETCDCTL_API=3 /usr/local/bin/etcdctl --write-out=table --cacert=/etc/etcd/ssl/ca.pem --cert=/etc/etcd/ssl/etcd.pem --key=/etc/etcd/ssl/etcd-key.pem --endpoints=https://192.168.127.13:2379,https://192.168.127.14:2379,https://192.168.127.15:2379 endpoint health
```

检查性能
```shell
ETCDCTL_API=3 /usr/local/bin/etcdctl --write-out=table --cacert=/etc/etcd/ssl/ca.pem --cert=/etc/etcd/ssl/etcd.pem --key=/etc/etcd/ssl/etcd-key.pem --endpoints=https://192.168.127.13:2379,https://192.168.127.14:2379,https://192.168.127.15:2379 check perf
```

成员列表
```shell
ETCDCTL_API=3 /usr/local/bin/etcdctl --write-out=table --cacert=/etc/etcd/ssl/ca.pem --cert=/etc/etcd/ssl/etcd.pem --key=/etc/etcd/ssl/etcd-key.pem --endpoints=https://192.168.127.13:2379,https://192.168.127.14:2379,https://192.168.127.15:2379 member list
```

集群状态
```shell
ETCDCTL_API=3 /usr/local/bin/etcdctl --write-out=table --cacert=/etc/etcd/ssl/ca.pem --cert=/etc/etcd/ssl/etcd.pem --key=/etc/etcd/ssl/etcd-key.pem --endpoints=https://192.168.127.13:2379,https://192.168.127.14:2379,https://192.168.127.15:2379 endpoint status
```

## 配置 API-SERVER

> 在 bastion 上操作

证书配置

```shell
cd /usr/local/kubernetes
cat > /usr/local/kubernetes/kube-apiserver-csr.json << 'EOF'
{
"CN": "kubernetes",
  "hosts": [
    "127.0.0.1",
    "192.168.127.10",
    "192.168.127.11",
    "192.168.127.12",
    "192.168.127.13",
    "192.168.127.14",
    "192.168.127.15",
    "192.168.127.16",
    "192.168.127.17",
    "192.168.127.18",
    "192.168.127.19",
    "192.168.127.20",
    "192.168.127.22",
    "192.168.127.23",
    "192.168.127.24",
    "192.168.127.25",
    "192.168.127.26",
    "192.168.127.27",
    "192.168.127.28",
    "192.168.127.29",
    "10.96.0.1",
    "kubernetes",
    "kubernetes.default",
    "kubernetes.default.svc",
    "kubernetes.default.svc.cluster",
    "kubernetes.default.svc.cluster.local"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Beijing",
      "L": "Beijing",
      "O": "kubemsb",
      "OU": "CN"
    }
  ]
}
EOF
```

API-SERVER 证书

```shell
cfssl gencert -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-apiserver-csr.json | \
cfssljson \
  -bare \
  kube-apiserver
```

API-SERVER TOKEN

```shell
cat > token.csv << EOF
$(head -c 16 /dev/urandom | od -An -t x | tr -d ' '),kubelet-bootstrap,10001,"system:kubelet-bootstrap"
EOF
```

API-SERVER 配置模板

```shell
cat > /usr/local/kubernetes/kube-apiserver.conf.tmpl <<'EOF'
KUBE_APISERVER_OPTS="--enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \
  --anonymous-auth=false \
  --bind-address=$HOSTNAME \
  --secure-port=6443 \
  --advertise-address=192.168.127.10 \
  --authorization-mode=Node,RBAC \
  --runtime-config=api/all=true \
  --enable-bootstrap-token-auth \
  --service-cluster-ip-range=10.96.0.0/16 \
  --token-auth-file=/etc/kubernetes/token.csv \
  --service-node-port-range=30000-32767 \
  --tls-cert-file=/etc/kubernetes/ssl/kube-apiserver.pem  \
  --tls-private-key-file=/etc/kubernetes/ssl/kube-apiserver-key.pem \
  --client-ca-file=/etc/kubernetes/ssl/ca.pem \
  --kubelet-client-certificate=/etc/kubernetes/ssl/kube-apiserver.pem \
  --kubelet-client-key=/etc/kubernetes/ssl/kube-apiserver-key.pem \
  --service-account-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --service-account-signing-key-file=/etc/kubernetes/ssl/ca-key.pem  \
  --service-account-issuer=api \
  --etcd-cafile=/etc/etcd/ssl/ca.pem \
  --etcd-certfile=/etc/etcd/ssl/etcd.pem \
  --etcd-keyfile=/etc/etcd/ssl/etcd-key.pem \
  --etcd-servers=https://$HOSTNAME1:2379,https://$HOSTNAME2:2379,https://$HOSTNAME3:2379 \
  --allow-privileged=true \
  --apiserver-count=3 \
  --audit-log-maxage=30 \
  --audit-log-maxbackup=3 \
  --audit-log-maxsize=100 \
  --audit-log-path=/var/log/kube-apiserver-audit.log \
  --event-ttl=1h \
  --alsologtostderr=true \
  --logtostderr=false \
  --log-dir=/var/log/kubernetes \
  --v=4"
EOF
```

API-SERVER 配置

```shell
sed -e "s|\$HOSTNAME1|192.168.127.13|g" \
    -e "s|\$HOSTNAME2|192.168.127.14|g" \
    -e "s|\$HOSTNAME3|192.168.127.15|g" \
    -e "s|\$HOSTNAME|192.168.127.13|g" \
    kube-apiserver.conf.tmpl > master01/kube-apiserver.conf

sed -e "s|\$HOSTNAME1|192.168.127.13|g" \
    -e "s|\$HOSTNAME2|192.168.127.14|g" \
    -e "s|\$HOSTNAME3|192.168.127.15|g" \
    -e "s|\$HOSTNAME|192.168.127.14|g" \
    kube-apiserver.conf.tmpl > master02/kube-apiserver.conf

sed -e "s|\$HOSTNAME1|192.168.127.13|g" \
    -e "s|\$HOSTNAME2|192.168.127.14|g" \
    -e "s|\$HOSTNAME3|192.168.127.15|g" \
    -e "s|\$HOSTNAME|192.168.127.15|g" \
    kube-apiserver.conf.tmpl > master03/kube-apiserver.conf
```

API-SERVER 服务配置

```shell
cat > /usr/local/kubernetes/kube-apiserver.service <<'EOF'
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
After=etcd.service
Wants=etcd.service

[Service]
EnvironmentFile=-/etc/kubernetes/kube-apiserver.conf
ExecStart=/usr/local/bin/kube-apiserver $KUBE_APISERVER_OPTS
Restart=on-failure
RestartSec=5
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

控制节点分发

```shell
for i in master{01,02,03}; \
do \
scp ca*.pem kube-apiserver*.pem $i:/etc/kubernetes/ssl/ ;\
scp token.csv $i/kube-apiserver.conf $i:/etc/kubernetes/ ;\
scp kube-apiserver.service $i:/etc/systemd/system/kube-apiserver.service; \
done
```

## 部署 API-SERVER 

> 在控制节点上操作

```shell
systemctl daemon-reload
systemctl enable kube-apiserver
systemctl restart kube-apiserver
systemctl status kube-apiserver
```

测试

```shell
curl --insecure https://192.168.127.13:6443/
curl --insecure https://192.168.127.14:6443/
curl --insecure https://192.168.127.15:6443/
curl --insecure https://192.168.127.10:6443/
```

> 报 401 是正常的

## 配置 KUBECTL

> 在 bastion 上操作

KUBECTL 证书配置

```shell
cd /usr/local/kubernetes
cat > admin-csr.json << "EOF"
{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Beijing",
      "L": "Beijing",
      "O": "system:masters",             
      "OU": "system"
    }
  ]
}
EOF
```

KUBECTL 证书

```shell
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | \
cfssljson \
  -bare \
  admin
```

KUBECTL 配置文件

```shell
kubectl config set-cluster kubernetes \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://192.168.127.10:6443 \
  --kubeconfig=kube.config

kubectl config set-credentials admin \
  --client-certificate=admin.pem \
  --client-key=admin-key.pem \
  --embed-certs=true \
  --kubeconfig=kube.config

kubectl config set-context kubernetes \
  --cluster=kubernetes \
  --user=admin \
  --kubeconfig=kube.config

kubectl config use-context kubernetes \
  --kubeconfig=kube.config
```

控制节点分发

```shell
for i in master{01,02,03}; \
do \
scp admin*.pem $i:/etc/kubernetes/ssl/ ;\
scp kube.config $i:/root/.kube/config ;\
done
```

## 部署 KUBECTL

> 在 控制节点 上操作

修改环境变量

```shell
cat >> $HOME/.bash_profile <<'EOF'
export KUBECONFIG=$HOME/.kube/config
EOF

source $HOME/.bash_profile
```

集群角色绑定
```shell
kubectl create clusterrolebinding kube-apiserver:kubelet-apis \
  --clusterrole=system:kubelet-api-admin \
  --user kubernetes \
  --kubeconfig=/root/.kube/config
```

> 只有一个会成功 其他两个已存在

查看集群信息

```shell
kubectl cluster-info
```

查看组件状态

```shell
kubectl get componentstatuses
```

查看资源对象

```shell
kubectl get all --all-namespaces
```

命令补全

```shell
yum install -y bash-completion
source /usr/share/bash-completion/bash_completion
source <(kubectl completion bash)
kubectl completion bash > ~/.kube/completion.bash.inc
source '/root/.kube/completion.bash.inc'  
source $HOME/.bash_profile
```

## 配置 KUBE-CONTROLLER-MANAGER

> 在 bastion 节点操作

KUBE-CONTROLLER-MANAGER 证书配置

```shell
cd /usr/local/kubernetes
cat > kube-controller-manager-csr.json << "EOF"
{
    "CN": "system:kube-controller-manager",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "hosts": [
      "127.0.0.1",
      "192.168.127.13",
      "192.168.127.14",
      "192.168.127.15"
    ],
    "names": [
      {
        "C": "CN",
        "ST": "Beijing",
        "L": "Beijing",
        "O": "system:kube-controller-manager",
        "OU": "system"
      }
    ]
}
EOF
```

KUBE-CONTROLLER-MANAGER 证书

```shell
cfssl gencert \
  -ca=ca.pem -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-controller-manager-csr.json | \
cfssljson \
  -bare \
  kube-controller-manager
```

KUBE-CONTROLLER-MANAGER 配置

```shell
kubectl config set-cluster kubernetes \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://192.168.127.10:6443 \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-credentials system:kube-controller-manager \
  --client-certificate=kube-controller-manager.pem \
  --client-key=kube-controller-manager-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-context system:kube-controller-manager \
  --cluster=kubernetes \
  --user=system:kube-controller-manager \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config use-context system:kube-controller-manager \
  --kubeconfig=kube-controller-manager.kubeconfig
```

```shell
cat > kube-controller-manager.conf << "EOF"
KUBE_CONTROLLER_MANAGER_OPTS="\
  --secure-port=10257 \
  --bind-address=127.0.0.1 \
  --kubeconfig=/etc/kubernetes/kube-controller-manager.kubeconfig \
  --service-cluster-ip-range=10.96.0.0/16 \
  --cluster-name=kubernetes \
  --cluster-signing-cert-file=/etc/kubernetes/ssl/ca.pem \
  --cluster-signing-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --allocate-node-cidrs=true \
  --cluster-cidr=10.244.0.0/16 \
  --experimental-cluster-signing-duration=87600h \
  --root-ca-file=/etc/kubernetes/ssl/ca.pem \
  --service-account-private-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --leader-elect=true \
  --feature-gates=RotateKubeletServerCertificate=true \
  --controllers=*,bootstrapsigner,tokencleaner \
  --horizontal-pod-autoscaler-sync-period=10s \
  --tls-cert-file=/etc/kubernetes/ssl/kube-controller-manager.pem \
  --tls-private-key-file=/etc/kubernetes/ssl/kube-controller-manager-key.pem \
  --use-service-account-credentials=true \
  --alsologtostderr=true \
  --logtostderr=false \
  --log-dir=/var/log/kubernetes \
  --v=4"
EOF
```

KUBE-CONTROLLER-MANAGER 服务配置

```shell
cat > kube-controller-manager.service << "EOF"
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=-/etc/kubernetes/kube-controller-manager.conf
ExecStart=/usr/local/bin/kube-controller-manager $KUBE_CONTROLLER_MANAGER_OPTS
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

控制节点分发

```shell
for i in master{01,02,03}; \
do \
scp kube-controller-manager*.pem $i:/etc/kubernetes/ssl/; \
scp kube-controller-manager.service $i:/usr/lib/systemd/system/; \
scp kube-controller-manager.kubeconfig kube-controller-manager.conf $i:/etc/kubernetes/; \
done
```

## 部署 KUBE-CONTROLLER-MANAGER

> 在 控制节点 上操作

启动服务

```shell
systemctl daemon-reload
systemctl enable kube-controller-manager
systemctl restart kube-controller-manager
systemctl status kube-controller-manager
```

查看组件状态

```shell
kubectl get componentstatuses
```

## 配置 KUBE-SCHEDULER

KUBE-SCHEDULER 证书配置

```shell
cd /usr/local/kubernetes
cat > kube-scheduler-csr.json << "EOF"
{
    "CN": "system:kube-scheduler",
    "hosts": [
      "127.0.0.1",
      "192.168.127.13",
      "192.168.127.14",
      "192.168.127.15"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
      {
        "C": "CN",
        "ST": "Beijing",
        "L": "Beijing",
        "O": "system:kube-scheduler",
        "OU": "system"
      }
    ]
}
EOF
```

KUBE-SCHEDULER 证书

```shell
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-scheduler-csr.json | \
cfssljson \
  -bare \
  kube-scheduler
```

KUBE-SCHEDULER 配置

```shell
cat > kube-scheduler.conf << "EOF"
KUBE_SCHEDULER_OPTS="--bind-address=127.0.0.1 \
--kubeconfig=/etc/kubernetes/kube-scheduler.kubeconfig \
--leader-elect=true \
--alsologtostderr=true \
--logtostderr=false \
--log-dir=/var/log/kubernetes \
--v=4"
EOF
```

```shell
kubectl config set-cluster kubernetes \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://192.168.127.10:6443 \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-credentials system:kube-scheduler \
  --client-certificate=kube-scheduler.pem \
  --client-key=kube-scheduler-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-context system:kube-scheduler \
  --cluster=kubernetes \
  --user=system:kube-scheduler \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config use-context system:kube-scheduler \
  --kubeconfig=kube-scheduler.kubeconfig
```


KUBE-SCHEDULER 服务配置

```shell
cat > kube-scheduler.service << "EOF"
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=-/etc/kubernetes/kube-scheduler.conf
ExecStart=/usr/local/bin/kube-scheduler $KUBE_SCHEDULER_OPTS
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

控制节点分发

```shell
for i in master{01,02,03}; \
do \
scp kube-scheduler*.pem $i:/etc/kubernetes/ssl/; \
scp kube-scheduler.service $i:/usr/lib/systemd/system/; \
scp kube-scheduler.kubeconfig kube-scheduler.conf $i:/etc/kubernetes/; \
done
```

## 启动 KUBE-SCHEDULER

> 在 控制节点 上操作

启动服务

```shell
systemctl daemon-reload
systemctl enable kube-scheduler
systemctl restart kube-scheduler
systemctl status kube-scheduler
```

查看组件状态
```shell
kubectl get componentstatuses
```

## 配置 CONTAINERD

> 在 工作节点 上操作

创建路径
```shell
mkdir -p /etc/containerd/
```

下载
```shell
wget https://github.com/containerd/containerd/releases/download/v1.7.5/cri-containerd-cni-1.7.5-linux-amd64.tar.gz
```

解压
```shell
tar -xf cri-containerd-cni-1.7.5-linux-amd64.tar.gz -C /
```

配置
```shell
cat >/etc/containerd/config.toml<<EOF
root = "/var/lib/containerd"
state = "/run/containerd"
oom_score = -999

[grpc]
  address = "/run/containerd/containerd.sock"
  uid = 0
  gid = 0
  max_recv_message_size = 16777216
  max_send_message_size = 16777216

[debug]
  address = ""
  uid = 0
  gid = 0
  level = ""

[metrics]
  address = ""
  grpc_histogram = false

[cgroup]
  path = ""

[plugins]
  [plugins.cgroups]
    no_prometheus = false
  [plugins.cri]
    stream_server_address = "127.0.0.1"
    stream_server_port = "0"
    enable_selinux = false
    sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.6"
    stats_collect_period = 10
    systemd_cgroup = true
    enable_tls_streaming = false
    max_container_log_line_size = 16384
    [plugins.cri.containerd]
      snapshotter = "overlayfs"
      no_pivot = false
      [plugins.cri.containerd.default_runtime]
        runtime_type = "io.containerd.runtime.v1.linux"
        runtime_engine = ""
        runtime_root = ""
      [plugins.cri.containerd.untrusted_workload_runtime]
        runtime_type = ""
        runtime_engine = ""
        runtime_root = ""
    [plugins.cri.cni]
      bin_dir = "/opt/cni/bin"
      conf_dir = "/etc/cni/net.d"
      conf_template = "/etc/cni/net.d/10-default.conf"
    [plugins.cri.registry]
      [plugins.cri.registry.mirrors]
        [plugins.cri.registry.mirrors."docker.io"]
          endpoint = [
            "https://68k4zzfk.mirror.aliyuncs.com",
            "https://docker.mirrors.ustc.edu.cn",
            "https://hub-mirror.c.163.com"
          ]
        [plugins.cri.registry.mirrors."gcr.io"]
          endpoint = [
            "https://registry.aliyuncs.com"
          ]
        [plugins.cri.registry.mirrors."k8s.gcr.io"]
          endpoint = [
            "https://registry.aliyuncs.com/google_containers"
          ]
    [plugins.cri.x509_key_pair_streaming]
      tls_cert_file = ""
      tls_key_file = ""
  [plugins.diff-service]
    default = ["walking"]
  [plugins.linux]
    shim = "containerd-shim"
    runtime = "runc"
    runtime_root = ""
    no_shim = false
    shim_debug = false
  [plugins.opt]
    path = "/opt/containerd"
  [plugins.restart]
    interval = "10s"
  [plugins.scheduler]
    pause_threshold = 0.02
    deletion_threshold = 0
    mutation_threshold = 100
    schedule_delay = "0s"
    startup_delay = "100ms"
EOF
```

runc

下载
```shell
wget https://github.com/opencontainers/runc/releases/download/v1.1.9/runc.amd64
```

安装
```shell
chmod +x runc.amd64
mv runc.amd64 /usr/local/sbin/runc
```

查看
```shell
runc -v
```

## 启动 CONTAINERD

启动服务
```shell
systemctl daemon-reload
systemctl enable containerd
systemctl restart containerd
systemctl status containerd
```


## 配置 KUBERLET

> 在 bastion 上操作

KUBERLET 模板

```shell
cat > kubelet.json.tmpl << "EOF"
{
  "kind": "KubeletConfiguration",
  "apiVersion": "kubelet.config.k8s.io/v1beta1",
  "authentication": {
    "x509": {
      "clientCAFile": "/etc/kubernetes/ssl/ca.pem"
    },
    "webhook": {
      "enabled": true,
      "cacheTTL": "2m0s"
    },
    "anonymous": {
      "enabled": false
    }
  },
  "authorization": {
    "mode": "Webhook",
    "webhook": {
      "cacheAuthorizedTTL": "5m0s",
      "cacheUnauthorizedTTL": "30s"
    }
  },
  "address": "$HOSTNAME",
  "port": 10250,
  "readOnlyPort": 10255,
  "cgroupDriver": "systemd",                    
  "hairpinMode": "promiscuous-bridge",
  "serializeImagePulls": false,
  "clusterDomain": "cluster.local.",
  "clusterDNS": ["10.96.0.2"]
}
EOF
```

KUBERLET 配置

```shell
sed -e "s|\$HOSTNAME|192.168.127.21|g" \
  kubelet.json.tmpl > worker01/kubelet.json

sed -e "s|\$HOSTNAME|192.168.127.22|g" \
  kubelet.json.tmpl > worker02/kubelet.json

sed -e "s|\$HOSTNAME|192.168.127.23|g" \
  kubelet.json.tmpl > worker03/kubelet.json
```

```shell
BOOTSTRAP_TOKEN=$(awk -F "," '{print $1}' token.csv)
echo $BOOTSTRAP_TOKEN

kubectl config set-cluster kubernetes \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://192.168.127.10:6443 \
  --kubeconfig=kubelet-bootstrap.kubeconfig

kubectl config set-credentials kubelet-bootstrap \
  --token=${BOOTSTRAP_TOKEN} \
  --kubeconfig=kubelet-bootstrap.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes \
  --user=kubelet-bootstrap \
  --kubeconfig=kubelet-bootstrap.kubeconfig

kubectl config use-context default \
  --kubeconfig=kubelet-bootstrap.kubeconfig

kubectl create clusterrolebinding cluster-system-anonymous \
  --clusterrole=cluster-admin \
  --user=kubelet-bootstrap \
  --kubeconfig=kube.config

kubectl create clusterrolebinding kubelet-bootstrap \
  --clusterrole=system:node-bootstrapper \
  --user=kubelet-bootstrap \
  --kubeconfig=kubelet-bootstrap.kubeconfig
```

KUBERLET 服务配置

```shell
cat > kubelet.service << "EOF"
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=containerd.service
Requires=containerd.service

[Service]
WorkingDirectory=/var/lib/kubelet
ExecStart=/usr/local/bin/kubelet \
  --bootstrap-kubeconfig=/etc/kubernetes/kubelet-bootstrap.kubeconfig \
  --cert-dir=/etc/kubernetes/ssl \
  --kubeconfig=/etc/kubernetes/kubelet.kubeconfig \
  --config=/etc/kubernetes/kubelet.json \
  --container-runtime=remote \
  --container-runtime-endpoint=unix:///run/containerd/containerd.sock \
  --rotate-certificates \
  --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.2 \
  --root-dir=/etc/cni/net.d \
  --alsologtostderr=true \
  --logtostderr=false \
  --log-dir=/var/log/kubernetes \
  --v=4
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

同步到工作节点

```shell
for i in worker{01,02,03}; \
do \
scp ca.pem $i:/etc/kubernetes/ssl/; \
scp $i/kubelet.json $i:/etc/kubernetes/; \
scp kubelet.service $i:/usr/lib/systemd/system/; \
scp kubelet-bootstrap.kubeconfig $i:/etc/kubernetes/; \
done
```

## 启动 KUBERLET

> 所有工作节点

创建目录及启动服务

```shell
mkdir -p /var/lib/kubelet
```

```shell
systemctl daemon-reload
systemctl enable kubelet
systemctl restart kubelet
systemctl status kubelet
```

> 所有控制节点

```shell
kubectl describe clusterrolebinding cluster-system-anonymous
kubectl describe clusterrolebinding kubelet-bootstrap
```

```shell
kubectl get nodes
```

```shell
kubectl get csr
```


## 配置 KUBE-PROXY

> 在 bastion 上操作

```shell
cd /usr/local/kubernetes
```

KUBE-PROXY 证书配置

```shell
cat > kube-proxy-csr.json << "EOF"
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Beijing",
      "L": "Beijing",
      "O": "kubemsb",
      "OU": "CN"
    }
  ]
}
EOF
```

KUBE-PROXY 证书

```shell
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-proxy-csr.json | \
cfssljson \
  -bare \
  kube-proxy
```

KUBE-PROXY 服务配置

```shell
cat >  kube-proxy.service << "EOF"
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
WorkingDirectory=/var/lib/kube-proxy
ExecStart=/usr/local/bin/kube-proxy \
  --config=/etc/kubernetes/kube-proxy.yaml \
  --alsologtostderr=true \
  --logtostderr=false \
  --log-dir=/var/log/kubernetes \
  --v=4
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

KUBE-PROXY 模板

```shell
cat > kube-proxy.yaml.tmpl << "EOF"
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: $HOSTNAME
clientConnection:
  kubeconfig: /etc/kubernetes/kube-proxy.kubeconfig
clusterCIDR: 10.244.0.0/16
healthzBindAddress: $HOSTNAME:10256
kind: KubeProxyConfiguration
metricsBindAddress: $HOSTNAME:10249
mode: "ipvs"
EOF
```

KUBE-PROXY 配置

```shell
sed -e "s|\$HOSTNAME|192.168.127.21|g" \
  kube-proxy.yaml.tmpl > worker01/kube-proxy.yaml

sed -e "s|\$HOSTNAME|192.168.127.22|g" \
  kube-proxy.yaml.tmpl > worker02/kube-proxy.yaml

sed -e "s|\$HOSTNAME|192.168.127.23|g" \
  kube-proxy.yaml.tmpl > worker03/kube-proxy.yaml
```

```shell
kubectl config set-cluster kubernetes \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://192.168.127.10:6443 \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-credentials kube-proxy \
  --client-certificate=kube-proxy.pem \
  --client-key=kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config use-context default \
  --kubeconfig=kube-proxy.kubeconfig
```

同步到工作节点

```shell
for i in worker{01,02,03}; \
do \
scp $i/kube-proxy.yaml $i:/etc/kubernetes/; \
scp kube-proxy.service $i:/usr/lib/systemd/system/; \
scp kube-proxy.kubeconfig $i:/etc/kubernetes/; \
done
```

## 启动 KUBE-PROXY

服务启动

```shell
mkdir -p /var/lib/kube-proxy
```

```shell
systemctl daemon-reload
systemctl enable kube-proxy
systemctl restart kube-proxy
systemctl status kube-proxy
```

## Calico 插件

> 在随便一台控制节点上操作  

下载
```shell
wget https://docs.projectcalico.org/v3.19/manifests/calico.yaml
```

修改

```shell
vi calico.yaml
```

```
3683 gg
- name: CALICO_IPV4POOL_CIDR
  value: "10.244.0.0/16"
```

应用文件

```shell
kubectl apply -f calico.yaml
```

验证应用结果

```shell
kubectl get pods -A
```

```shell
kubectl get nodes
```


## CoreDNS 安装

```shell
cat >  coredns.yaml << "EOF"
apiVersion: v1
kind: ServiceAccount
metadata:
  name: coredns
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:coredns
rules:
  - apiGroups:
    - ""
    resources:
    - endpoints
    - services
    - pods
    - namespaces
    verbs:
    - list
    - watch
  - apiGroups:
    - discovery.k8s.io
    resources:
    - endpointslices
    verbs:
    - list
    - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:coredns
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:coredns
subjects:
- kind: ServiceAccount
  name: coredns
  namespace: kube-system
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
          lameduck 5s
        }
        ready
        kubernetes cluster.local  in-addr.arpa ip6.arpa {
          fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . /etc/resolv.conf {
          max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: coredns
  namespace: kube-system
  labels:
    k8s-app: kube-dns
    kubernetes.io/name: "CoreDNS"
spec:
  # replicas: not specified here:
  # 1. Default is 1.
  # 2. Will be tuned in real time if DNS horizontal auto-scaling is turned on.
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  selector:
    matchLabels:
      k8s-app: kube-dns
  template:
    metadata:
      labels:
        k8s-app: kube-dns
    spec:
      priorityClassName: system-cluster-critical
      serviceAccountName: coredns
      tolerations:
        - key: "CriticalAddonsOnly"
          operator: "Exists"
      nodeSelector:
        kubernetes.io/os: linux
      affinity:
         podAntiAffinity:
           preferredDuringSchedulingIgnoredDuringExecution:
           - weight: 100
             podAffinityTerm:
               labelSelector:
                 matchExpressions:
                   - key: k8s-app
                     operator: In
                     values: ["kube-dns"]
               topologyKey: kubernetes.io/hostname
      containers:
      - name: coredns
        image: coredns/coredns:1.8.4
        imagePullPolicy: IfNotPresent
        resources:
          limits:
            memory: 170Mi
          requests:
            cpu: 100m
            memory: 70Mi
        args: [ "-conf", "/etc/coredns/Corefile" ]
        volumeMounts:
        - name: config-volume
          mountPath: /etc/coredns
          readOnly: true
        ports:
        - containerPort: 53
          name: dns
          protocol: UDP
        - containerPort: 53
          name: dns-tcp
          protocol: TCP
        - containerPort: 9153
          name: metrics
          protocol: TCP
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            add:
            - NET_BIND_SERVICE
            drop:
            - all
          readOnlyRootFilesystem: true
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 60
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 5
        readinessProbe:
          httpGet:
            path: /ready
            port: 8181
            scheme: HTTP
      dnsPolicy: Default
      volumes:
        - name: config-volume
          configMap:
            name: coredns
            items:
            - key: Corefile
              path: Corefile
---
apiVersion: v1
kind: Service
metadata:
  name: kube-dns
  namespace: kube-system
  annotations:
    prometheus.io/port: "9153"
    prometheus.io/scrape: "true"
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: "CoreDNS"
spec:
  selector:
    k8s-app: kube-dns
  clusterIP: 10.96.0.2
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
    protocol: TCP
  - name: metrics
    port: 9153
    protocol: TCP
 
EOF
```

```shell
kubectl apply -f coredns.yaml
```

```shell
kubectl get endpoints
kubectl get nodes
kubectl get pods -A
kubectl get cs
kubectl get --raw='/readyz?verbose'
```

部署应用验证

```shell
cat >  nginx.yaml  << "EOF"
---
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx-web
spec:
  replicas: 2
  selector:
    name: nginx
  template:
    metadata:
      labels:
        name: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.19.6
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service-nodeport
spec:
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30001
      protocol: TCP
  type: NodePort
  selector:
    name: nginx
EOF
```

```shell
kubectl apply -f nginx.yaml
```

```shell
kubectl get pods -o wide

curl 192.168.127.21:30001
curl 192.168.127.22:30001
curl 192.168.127.23:30001
```





































# 切换镜像源
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
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




# 安装
sudo yum install -y kubelet-1.20.9 kubeadm-1.20.9 kubectl-1.20.9 --disableexcludes=kubernetes

# 配置 kubelet 的 cgroup
vi /etc/sysconfig/kubelet
KUBELET_EXTRA_ARGS="--cgroup-driver=systemd"
KUBE_PROXY_MODE="ipvs"

# 设置开机自启
sudo systemctl enable kubelet --now
集群搭建

kube-apiserver:v1.20.9
kube-proxy:v1.20.9
kube-controller-manager:v1.20.9
kube-scheduler:v1.20.9
coredns:1.7.0
etcd:3.4.13-0
pause:3.2


registry.k8s.io/kube-apiserver:v1.27.2
registry.k8s.io/kube-controller-manager:v1.27.2
registry.k8s.io/kube-scheduler:v1.27.2
registry.k8s.io/kube-proxy:v1.27.2
registry.k8s.io/pause:3.9
registry.k8s.io/etcd:3.5.7-0
registry.k8s.io/coredns/coredns:v1.10.1


# 查看需要的镜像
kubeadm config images list

# 设置集群安装脚本
sudo tee ./install_k8s_imgs.sh <<-'EOF'
#!/bin/bash
images=(
kube-apiserver:v1.27.2
kube-controller-manager:v1.27.2
kube-scheduler:v1.27.2
kube-proxy:v1.27.2
pause:3.9
etcd:3.5.7-0
coredns/coredns:v1.10.1
)
for imageName in ${images[@]} ; do
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
docker tag  registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName k8s.gcr.io/$imageName
docker rmi  registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
done
EOF

# 执行安装镜像
chmod +x ./install_k8s_imgs.sh && ./install_k8s_imgs.sh
● 初始化集群(Master节点)
kubeadm init \
  --kubernetes-version v1.20.9 \
  --service-cidr=10.96.0.0/16 \
  --pod-network-cidr=10.244.0.0/16 \
  --control-plane-endpoint=cluster-endpoint \
  --apiserver-advertise-address=192.168.127.31

● 使用集群(Master节点)
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
● 加入集群(Node节点)
# node 目标服务器 创建目录
kubeadm join cluster-endpoint:6443 --token d6g0w0.txtmdqqij59x8id8 \
    --discovery-token-ca-cert-hash sha256:64e6f8455fce99f81fadcfdb87a37cbb5634c3cedc609ceac74293b419cb6e8d
● flannel 配置文件(Master节点)
vi kube-flannel.yml

---
kind: Namespace
apiVersion: v1
metadata:
  name: kube-flannel
  labels:
    pod-security.kubernetes.io/enforce: privileged
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: flannel
rules:
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - get
- apiGroups:
  - ""
  resources:
  - nodes
  verbs:
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - nodes/status
  verbs:
  - patch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: flannel
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: flannel
subjects:
- kind: ServiceAccount
  name: flannel
  namespace: kube-flannel
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: flannel
  namespace: kube-flannel
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: kube-flannel-cfg
  namespace: kube-flannel
  labels:
    tier: node
    app: flannel
data:
  cni-conf.json: |
    {
      "name": "cbr0",
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "flannel",
          "delegate": {
            "hairpinMode": true,
            "isDefaultGateway": true
          }
        },
        {
          "type": "portmap",
          "capabilities": {
            "portMappings": true
          }
        }
      ]
    }
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-flannel-ds
  namespace: kube-flannel
  labels:
    tier: node
    app: flannel
spec:
  selector:
    matchLabels:
      app: flannel
  template:
    metadata:
      labels:
        tier: node
        app: flannel
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/os
                operator: In
                values:
                - linux
      hostNetwork: true
      priorityClassName: system-node-critical
      tolerations:
      - operator: Exists
        effect: NoSchedule
      serviceAccountName: flannel
      initContainers:
      - name: install-cni-plugin
       #image: flannelcni/flannel-cni-plugin:v1.1.0 for ppc64le and mips64le (dockerhub limitations may apply)
        image: docker.io/rancher/mirrored-flannelcni-flannel-cni-plugin:v1.1.0
        command:
        - cp
        args:
        - -f
        - /flannel
        - /opt/cni/bin/flannel
        volumeMounts:
        - name: cni-plugin
          mountPath: /opt/cni/bin
      - name: install-cni
       #image: flannelcni/flannel:v0.19.2 for ppc64le and mips64le (dockerhub limitations may apply)
        image: docker.io/rancher/mirrored-flannelcni-flannel:v0.19.2
        command:
        - cp
        args:
        - -f
        - /etc/kube-flannel/cni-conf.json
        - /etc/cni/net.d/10-flannel.conflist
        volumeMounts:
        - name: cni
          mountPath: /etc/cni/net.d
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      containers:
      - name: kube-flannel
       #image: flannelcni/flannel:v0.19.2 for ppc64le and mips64le (dockerhub limitations may apply)
        image: docker.io/rancher/mirrored-flannelcni-flannel:v0.19.2
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
          limits:
            cpu: "100m"
            memory: "50Mi"
        securityContext:
          privileged: false
          capabilities:
            add: ["NET_ADMIN", "NET_RAW"]
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: EVENT_QUEUE_DEPTH
          value: "5000"
        volumeMounts:
        - name: run
          mountPath: /run/flannel
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
        - name: xtables-lock
          mountPath: /run/xtables.lock
      volumes:
      - name: run
        hostPath:
          path: /run/flannel
      - name: cni-plugin
        hostPath:
          path: /opt/cni/bin
      - name: cni
        hostPath:
          path: /etc/cni/net.d
      - name: flannel-cfg
        configMap:
          name: kube-flannel-cfg
      - name: xtables-lock
        hostPath:
          path: /run/xtables.lock
          type: FileOrCreate


● 安装
kubectl apply -f kube-flannel.yml
● 查看节点状态
kubectl get nodes
● 查看 pod 状态
kubectl get pods -A
kubectl describe pod <pod_name> [-n <namespace>]
部署 dashboard
● 获取配置文件
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.3.1/aio/deploy/recommended.yaml
● dashboard 配置文件
vi kubernetes-dashboard.yml

# Copyright 2017 The Kubernetes Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

apiVersion: v1
kind: Namespace
metadata:
  name: kubernetes-dashboard

---

apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard

---

kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  ports:
    - port: 443
      targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard

---

apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-certs
  namespace: kubernetes-dashboard
type: Opaque

---

apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-csrf
  namespace: kubernetes-dashboard
type: Opaque
data:
  csrf: ""

---

apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-key-holder
  namespace: kubernetes-dashboard
type: Opaque

---

kind: ConfigMap
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-settings
  namespace: kubernetes-dashboard

---

kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
rules:
  # Allow Dashboard to get, update and delete Dashboard exclusive secrets.
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["kubernetes-dashboard-key-holder", "kubernetes-dashboard-certs", "kubernetes-dashboard-csrf"]
    verbs: ["get", "update", "delete"]
    # Allow Dashboard to get and update 'kubernetes-dashboard-settings' config map.
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["kubernetes-dashboard-settings"]
    verbs: ["get", "update"]
    # Allow Dashboard to get metrics.
  - apiGroups: [""]
    resources: ["services"]
    resourceNames: ["heapster", "dashboard-metrics-scraper"]
    verbs: ["proxy"]
  - apiGroups: [""]
    resources: ["services/proxy"]
    resourceNames: ["heapster", "http:heapster:", "https:heapster:", "dashboard-metrics-scraper", "http:dashboard-metrics-scraper"]
    verbs: ["get"]

---

kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
rules:
  # Allow Metrics Scraper to get metrics from the Metrics server
  - apiGroups: ["metrics.k8s.io"]
    resources: ["pods", "nodes"]
    verbs: ["get", "list", "watch"]

---

apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: kubernetes-dashboard
subjects:
  - kind: ServiceAccount
    name: kubernetes-dashboard
    namespace: kubernetes-dashboard

---

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kubernetes-dashboard
subjects:
  - kind: ServiceAccount
    name: kubernetes-dashboard
    namespace: kubernetes-dashboard

---

kind: Deployment
apiVersion: apps/v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: kubernetes-dashboard
  template:
    metadata:
      labels:
        k8s-app: kubernetes-dashboard
    spec:
      containers:
        - name: kubernetes-dashboard
          image: kubernetesui/dashboard:v2.3.1
          imagePullPolicy: Always
          ports:
            - containerPort: 8443
              protocol: TCP
          args:
            - --auto-generate-certificates
            - --namespace=kubernetes-dashboard
            # Uncomment the following line to manually specify Kubernetes API server Host
            # If not specified, Dashboard will attempt to auto discover the API server and connect
            # to it. Uncomment only if the default does not work.
            # - --apiserver-host=http://my-address:port
          volumeMounts:
            - name: kubernetes-dashboard-certs
              mountPath: /certs
              # Create on-disk volume to store exec logs
            - mountPath: /tmp
              name: tmp-volume
          livenessProbe:
            httpGet:
              scheme: HTTPS
              path: /
              port: 8443
            initialDelaySeconds: 30
            timeoutSeconds: 30
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            runAsUser: 1001
            runAsGroup: 2001
      volumes:
        - name: kubernetes-dashboard-certs
          secret:
            secretName: kubernetes-dashboard-certs
        - name: tmp-volume
          emptyDir: {}
      serviceAccountName: kubernetes-dashboard
      nodeSelector:
        "kubernetes.io/os": linux
      # Comment the following tolerations if Dashboard must not be deployed on master
      tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule

---

kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: dashboard-metrics-scraper
  name: dashboard-metrics-scraper
  namespace: kubernetes-dashboard
spec:
  ports:
    - port: 8000
      targetPort: 8000
  selector:
    k8s-app: dashboard-metrics-scraper

---

kind: Deployment
apiVersion: apps/v1
metadata:
  labels:
    k8s-app: dashboard-metrics-scraper
  name: dashboard-metrics-scraper
  namespace: kubernetes-dashboard
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: dashboard-metrics-scraper
  template:
    metadata:
      labels:
        k8s-app: dashboard-metrics-scraper
      annotations:
        seccomp.security.alpha.kubernetes.io/pod: 'runtime/default'
    spec:
      containers:
        - name: dashboard-metrics-scraper
          image: kubernetesui/metrics-scraper:v1.0.6
          ports:
            - containerPort: 8000
              protocol: TCP
          livenessProbe:
            httpGet:
              scheme: HTTP
              path: /
              port: 8000
            initialDelaySeconds: 30
            timeoutSeconds: 30
          volumeMounts:
          - mountPath: /tmp
            name: tmp-volume
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            runAsUser: 1001
            runAsGroup: 2001
      serviceAccountName: kubernetes-dashboard
      nodeSelector:
        "kubernetes.io/os": linux
      # Comment the following tolerations if Dashboard must not be deployed on master
      tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
      volumes:
        - name: tmp-volume
          emptyDir: {}
● 安装
kubectl apply -f kubernetes-dashboard.yml
● 配置端口访问
kubectl edit svc kubernetes-dashboard -n kubernetes-dashboard

# 修改如下内容
type: ClusterIP -> type: NodePort
● 访问： 
https://集群任意IP:端口 
● 创建访问账号
vi dashboard-account.yml

apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
● 获取令牌
kubectl -n kubernetes-dashboard get secret $(kubectl -n kubernetes-dashboard get sa/admin-user -o jsonpath="{.secrets[0].name}") -o go-template="{{.data.token | base64decode}}"