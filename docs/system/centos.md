# CENTOS 安装

## IP地址修改

```shell

# VMnet8
GATEWAY IP 192.168.127.1
Subnet IP      192.168.127.0
Subnet NETMASK 255.255.255.0
DHCP START     192.168.127.3 
DHCP END       192.168.127.254

# HOST
HOST IP         192.168.127.2
```

```shell
# 切换为根用户
su root

# 编辑指定配置文件
vi /etc/sysconfig/network-scripts/ifcfg-ens # 按下tab键自动补全配置文件名

# 修改配置文件内容
BOOTPROTO = static
ONBOOT=yes
NETMASK=255.255.255.0
GATEWAY=192.168.127.1
DNS1=192.168.127.1
IPADDR=192.168.127.10



# 重启服务
systemctl restart network
```

## 关闭图形界面

```shell
# 默认启动命令行页面
systemctl set-default multi-user.target

# 默认启动图形界面
systemctl set-default graphical.target
```

## 防火墙

### 防火墙设置

关闭/禁用防火墙

```shell
# 关闭防火墙
systemctl stop firewalld
# 禁用防火墙 
systemctl disable firewalld
```

启动/重启防火墙

```shell
# 启动防火墙
systemctl start firewalld
# 重启防火墙
systemctl restart firewalld
```

查看防火墙状态

```shell
systemctl status firewalld
```

查看防火墙是否开启

```shell
firewall-cmd --state
```

防火墙详细配置

```shell
# 查询端口是否开放
firewall-cmd --query-port=8080/tcp

# 开放ssh以及http服务
firewall-cmd --add-service=ssh --permanent
firewall-cmd --add-service=http --permanent

# 开放22和80端口
firewall-cmd --permanent --add-port=22/tcp
firewall-cmd --permanent --add-port=80/tcp

# 移除端口
firewall-cmd --permanent --remove-port=8080/tcp

# 重启防火墙(修改配置后要重启防火墙)
firewall-cmd --reload

# 参数解释
# 1、firwall-cmd: linux提供的操作 firewall 的工具
# 2、--permanent: 设置为持久
# 3、--add-port: 添加端口
```

## 配置YUM源

```shell
# 获取华为中文镜像源
curl -o /etc/yum.repos.d/CentOS-Base.repo https://repo.huaweicloud.com/repository/conf/CentOS-7-reg.repo

# 清除以前使用yum的缓存
yum clean all

# 建立一个缓存，以后方便在缓存中搜索
yum makecache
```

## 主机名

### 修改主机名

```shell
# 设置主机名
hostnamectl set-hostname <hostname>
# 重新登录后生效
exit
```

### 主机名绑定IP地址

```shell
# 编辑指定配置文件
vim etc/hosts

# 绑定 ip 和主机
192.168.1.111 <hostname>

# 保存退出后 验证
ping <hostname>
```

## 用户操作

### 新增用户

```shell
useradd <username> [-m] [-g] <groupname> <username>
    # -m 创建用户同时自动创建家目录
    # -g <group> 设置用户所属组  不加默认 以用户名作为所属组
```

### 修改密码

普通用户密码修改

```shell
su - root # 获取root权限
passwd <username> # 如 passwd user1 
# 输入两遍新密码
```

超级用密码修改

```shell
passwd 超级用户名 # 如 passwd root
# 输入两遍新密码
```

### 修改用户

```shell
/var/run/utmp
# 修改指定用户用户名 (重启后执行)
usermod -l <username> -d /home/<username> -m <username-old>
```

### 修改用户组

```shell
# 修改用户主组信息, 即 /etc/passwd 文件中的用户组信息(不常用) 
usermod -g <groupname> <username>

# 修改用户附属组信息, 即 /etc/group 文件中的用户组信息(常用)
usermod -G <groupname> <username>       
# 为张三添加 sudo 权限
# zhangsan 需要重新登录, 才会有 sudo 权限 
usermod -G sudo zhangsan
```

### 删除用户

```shell
/var/run/utmp

# 删除指定用户 (重启后执行)
usrdel [-r] <username> # -r 同时删除家目录  

# 如果没有使用 -r, 可以进入`/home`, 手动删除家目录
rm -rf /home/<username>
```

## IP转发配置

修改配置

```shell
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
```

重载配置

```shell
sysctl -p
```

## 关闭 selinux

查看状态

```shell
getenforce
```

> [!info]
> Permissive 关闭 <br>
> Enforcing 开启

临时关闭

```shell
setenforce 0
```

永久关闭

```shell
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

> [!info] 设置完需要重启操作系统

## 关闭 swap 分区

```shell
swapoff -a  
sed -ri 's/.*swap.*/#&/' /etc/fstab
```

## ping 命令

```shell
# 编辑指定配置文件
vim /etc/sysctl.conf

# 添加使用如下配置
net.ipv4.icmp_echo_ignore_all=0 

# 关闭使用如下配置
net.ipv4.icmp_echo_ignore_all=1

# 使其生效
sysctl -p
```

## window 文件字符集处理

windows平台上编辑后的脚本文件放到linux中, 需要使用dos2unix工具处理一下

```shell
dos2unix filename
```
