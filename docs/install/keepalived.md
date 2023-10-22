# KEEPALIVED 安装

## 准备机器

```shell
# 主机1 IP
192.168.127.11
# 主机2 IP
192.168.127.12
```

## 安装

```shell
# 安装
yum install -y keepalived
```

## 配置主机 1

```shell
# 编辑配置文件
cat > /etc/keepalived/keepalived.conf <<'EOF'
global_defs {
  router_id lb_keepalived_11
}

vrrp_instance lb_keepalived {
  state MASTER
  interface ens33
  virtual_router_id 51
  priority 100
  advert_int 1
  authentication {
    auth_type PASS
    auth_pass 1111
  }
  virtual_ipaddress {
    192.168.127.10
  }
}
EOF
```

## 配置主机 2

```shell
cat > /etc/keepalived/keepalived.conf <<'EOF'
global_defs {
  router_id lb_keepalived_12
}

vrrp_instance lb_keepalived {
  state BACKUP
  interface ens33
  virtual_router_id 51
  priority 100
  advert_int 1
  authentication {
    auth_type PASS
    auth_pass 1111
  }
  virtual_ipaddress {
    192.168.127.10
  }
}
EOF
```

## 服务管理

启动服务

```shell
systemctl start keepalived
```

关闭服务

```shell
systemctl stop keepalived
```

开机启动

```shell
systemctl enable keepalived
```

## 查看状态

运行状态

```shell
systemctl status keepalived

# 显示如下内容代表启动成功
# Active: active (running)
```

查看 IP

```shell
# 分别查看两台机器 IP
# 当前主节点会多一个虚拟 IP 地址
# 虚拟 IP : 192.168.127.10
ip a
```

访问地址

```shell
# 其他机器访问虚拟 IP
ping 192.168.127.10 -t
```

## 检查

主节点关机

```shell
init 0
```

访问地址

```shell
ping 192.168.127.10 -t
```

备份节点 虚拟IP

```shell
ip a
```

## 扩展

Keepalived 通过检测进程来判断是否存活, 可通过编写 shell 来检测任何需要检测的东西。

> 本机 nginx 心跳检测

创建目录

```shell
mkdir -p /usr/local/keepalived/logs
```

创建脚本

```shell
tee /usr/local/keepalived/monitor.sh <<-'EOF'
#!/bin/sh
curl -I 127.0.0.1 >/dev/null 2>&1
if [ $? -ne 0 ]; then
  echo "stop keepalived" at "$(date +"%Y-%m-%d %H:%M:%S")"
  systemctl stop keepalived
fi
EOF
```

添加权限

```shell
chmod +x /usr/local/keepalived/monitor.sh
```

关闭邮件

```shell
sed -i s/MAILTO=root/MAILTO=/ /etc/crontab
```

添加定时任务

```shell
crontab -e
* * * * * /usr/local/keepalived/monitor.sh >> /usr/local/keepalived/logs/monitor.log
```

重启定时任务

```shell
systemctl restart crond
```

查看定时任务

```shell
crontab -l
```

查看监控日志

```shell
tail -f -n 10 /usr/local/keepalived/logs/monitor.log
```
