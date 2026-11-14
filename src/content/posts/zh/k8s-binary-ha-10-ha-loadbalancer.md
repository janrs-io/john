---
title: "【实战加详解】二进制部署k8s高可用集群教程系列十 - 部署高可用"
author: "John Yang"
lang: zh
pubDatetime: 2022-10-17T06:16:00Z
tags:
  - Kubernetes
  - HA
description: "master 中的 kube-apiserver , kube-controller-manager , kube-scheduler 这三大组件中。 kube-controller-manager 与 kube-scheduler 有自带的 leader 选举机制。 通过部署的时候设置参数：--lea..."
---

> [!TIP]
> 二进制部署 `k8s` - 部署 `HA` 高可用

***

> [!WARNING]
> 如果该环节在阿里云 `ecs` 部署，因为阿里云现在已经不支持 `vip` 了。
>   

> `HA` 部署环节可以在本地测试。或者线上不使用 `keepalived` 做高可用。
>   

> 直接用 `nginx` 的 `tcp` 反向代理也是可以。
>   

>   

> 主要要注意学习的地方就是：`kube-apiserver` 的 `server` 证书地址需要把 `nginx` 的服务器 `ip` 写进 `hosts` 参数。
>   

> 否则会报无权限错误。

# 部署 `HA` 高可用

## 1.`k8s` 的 `HA`

> [!NOTE]
> `k8s` 中 `master` 的三大组件，其中 `kube-controller-manager` 与 `kube-scheduler` 有提供高可用机制。
>   

> `kube-apiserver` 没有，需要自己实现。

### 1-1.`kube-controlle` 与 `kube-scheduler` 的 `HA`

`master` 中的 `kube-apiserver` , `kube-controller-manager` , `kube-scheduler` 这三大组件中。
  

`kube-controller-manager` 与 `kube-scheduler` 有自带的 `leader` 选举机制。
  

  

通过部署的时候设置参数：`--leader-elect=true` 启动 `leader` 自选举。
  

  

在部署完三台 `master` 服务器，可以在任意一台 `master` 输入以下命令查看 `leader`

```shell
kubectl get leases -n kube-system
```

显示如下
  

*可以看出，`kube-controller-manager` 和 `kube-scheduler` 的 `leader` 是在 `master-01` 服务器上*

```text
NAME                      HOLDER                                              AGE
kube-controller-manager   k8s-master01_e0f4cfd5-1190-4f79-9ee5-a2063eb3ca16   156m
kube-scheduler            k8s-master01_54254610-53a8-4c3a-b3ea-a4fa5f549119   99s
```

在三台 `master` 任意一台停止这个三个组件，然后再次查看。
  

  

在 `master-01` 服务器停止三个组件的服务

```shell
systemctl stop kube-scheduler && \
systemctl stop kube-controller-manager && \
systemctl stop kube-apiserver
```

到 `master-02` 服务器查看 `leader` 信息

```shell
kubectl get leases -n kube-system
```

显示如下
  

*可以看出，`leader` 的 `HOLDER` 服务器已经改变了*

```text
NAME                      HOLDER                                              AGE
kube-controller-manager   k8s-master02_e0f4cfd5-1190-4f79-9ee5-a2063eb3ca16   3h5m
kube-scheduler            k8s-master03_e350060d-68ad-4f59-82a8-456f835b7f3d   30m
```

### 1-2.`kube-apiserver` 的 `HA`

`k8s` 没有提供 `kube-apiserver` 的 `HA`，需要手动实现。
  

因为 `kube-apiserver` 是无状态的应用，并且对外提供 `http/https` 的接口调用方式提供服务，所以可以用 `nginx` 来做负载均衡达到高可用。

## 2.初始化系统环境

> [!NOTE]
> 不需要像 `master` 节点和 `node` 节点那样初始化。

### 2-1.安装 `epel` 以及依赖软件

```shell
dnf install epel-release vim iptables jq ipvsadm ipset curl net-tools rsyslog -y
```

### 2-2.关闭防火墙

```shell
systemctl stop firewalld && systemctl disable firewalld
```

### 2-3.清空 `iptables` 规则链

```shell
iptables -F && iptables -X && iptables -F -t nat && iptables -X -t nat
```

### 2-4.关闭 `swap` 分区

```shell
swapoff -a && \
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

### 2-5.关闭 `selinux`

```shell
setenforce 0 && \
sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
```

### 2-6.设置时间同步

设置时区

```shell
timedatectl set-timezone Asia/Shanghai
```

同步时区

```shell
systemctl enable chronyd && \
systemctl start chronyd
```

查看

```shell
timedatectl status
```

*显示*

```text
               Local time: Sun 2022-10-02 13:30:23 CST
           Universal time: Sun 2022-10-02 05:30:23 UTC
                 RTC time: Sun 2022-10-02 05:30:23
                Time zone: Asia/Shanghai (CST, +0800)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no
```

将当前的 `UTC` 时间写入硬件时钟

```shell
timedatectl set-local-rtc 0
```

重启依赖于系统时间的服务

```shell
systemctl restart rsyslog && \
systemctl restart crond
```

### 2-7.设置 `systemd journald`

创建持久化保存日志的目录以及添加配置并生效

```shell
mkdir /var/log/journal && \
mkdir /etc/systemd/journald.conf.d && \
cat > /etc/systemd/journald.conf.d/99-prophet.conf <<EOF
[Journal]
# 持久化保存到磁盘
Storage=persistent
# 压缩历史日志
Compress=yes
SyncIntervalSec=5m
RateLimitInterval=30s
RateLimitBurst=1000
# 最大占用空间 10G
SystemMaxUse=10G
# 单日志文件最大 200M
SystemMaxFileSize=200M
# 日志保存时间 2 周
MaxRetentionSec=2week
# 不将日志转发到 syslog
ForwardToSyslog=no
EOF
systemctl restart systemd-journald
```

## 3.查看网卡是否开启多播

查看

```shell
ip a
```

有显示 `MULTICAST` 即表示有打开网卡多播

```text
2: ens160: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:0c:29:e6:09:4c brd ff:ff:ff:ff:ff:ff
    inet 172.16.222.201/24 brd 172.16.222.255 scope global noprefixroute ens160
       valid_lft forever preferred_lft forever
    inet 172.16.222.110/24 scope global secondary ens160
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fee6:94c/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
```

如果没有开启，执行以下命令打开
  

*`ens160` 为网卡名称*

```shell
ip link set multicast on dev ens160
```

## 4.重启

```shell
shutdown -r now
```

## 5.安装

两台 `HA` 服务器都安装

```shell
dnf install keepalived nginx -y
```

## 6.创建配置文件

### 6-1.创建 `nginx.conf` 配置文件

> [!NOTE]
> 在两台 `HA` 服务器都创建。
>   

> 这里使用的是 `8443` 端口代理 `kube-apiserver` 的 `6443` 端口。

```shell
#备份nginx.conf
mv /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak && \

#新建nginx.conf
cat > /etc/nginx/nginx.conf << 'EOF'
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
    use epoll;
}

# 四层负载均衡，为三台Master apiserver组件提供负载均衡
stream {

    log_format  main  '$remote_addr $upstream_addr - [$time_local] $status $upstream_bytes_sent';

    access_log  /var/log/nginx/k8s-access.log  main;

    upstream k8s-apiserver {
       server 172.16.222.121:6443;   # Master1 APISERVER IP:PORT
       server 172.16.222.122:6443;   # Master2 APISERVER IP:PORT
       server 172.16.222.123:6443;   # Master3 APISERVER IP:PORT
    }

    server {
       listen 8443;
       proxy_pass k8s-apiserver;
    }
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    server {
        listen       80 default_server;
        server_name  _;

        location / {
        }
    }
}
EOF
```

### 6-2.创建 `keepalived` 配置文件

> [!NOTE]
> 其中：
>   

> `k8s-ha-master` 作为 `nginx master` 主负载均衡服务器，
>   

> `k8s-ha-backup` 作为 `nginx slave` 备用负载均衡服务器。

#### 6-2-1.在 `k8s-ha-master` 服务器创建

创建 `keepalived.conf`

```shell
#备份
mv /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.bak && \
cat > /etc/keepalived/keepalived.conf << EOF
global_defs {

   notification_email {

     acassen@firewall.loc
     failover@firewall.loc
     sysadmin@firewall.loc

   }

   script_user root

   enable_script_security

   notification_email_from Alexandre.Cassen@firewall.loc

   smtp_server 127.0.0.1

   smtp_connect_timeout 30

   router_id K8S_HA_NGINX_MASTER
}

vrrp_script nginx_heartbeat {
    script "/etc/keepalived/nginx_heartbeat.sh"
}

vrrp_instance VI_1 {

    # 定义初始状态，这里是 MASTER 备用的 nginx 设置为 BACKUP
    state MASTER

    # 工作接口，通告选举使用哪个接口进行
    interface ens160

    # VRRP 路由 ID实例，每个实例是唯一的
    # 虚拟路由ID，如果是一组虚拟路由就定义一个ID，如果是多组就要定义多个，而且这个虚拟
    # ID还是虚拟MAC最后一段地址的信息，取值范围0-255
    virtual_router_id 51

    # MASTER 的优先级最高，BACKUP 的优先级低一点，下面设置为 90
    priority 100

    # 通告频率，单位为秒
    advert_int 1

    # 通信认证机制，这里是明文认证还有一种是加密认证
    authentication {
        auth_type PASS
        auth_pass 123456
    }
    # 虚拟 ip 地址。即 vip
    virtual_ipaddress {
        172.16.222.110/24
    }
    track_script {
        nginx_heartbeat
    }
}
EOF
```

创建 `nginx_heartbeat.sh` 脚本

```shell
cat > /etc/keepalived/nginx_heartbeat.sh << 'EOF'
#!/bin/bash
count=$(ps -ef |grep nginx |egrep -cv "grep|$$")
if [ "$count" -eq 0 ];
then
    exit 1
else
    exit 0
fi
EOF
chmod +x /etc/keepalived/nginx_heartbeat.sh
```

### 6-3.在 `k8s-ha-backup` 服务器创建

创建 `keepalived.conf`

```shell
mv /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.bak && \
cat > /etc/keepalived/keepalived.conf << EOF
global_defs {

   notification_email {

     acassen@firewall.loc
     failover@firewall.loc
     sysadmin@firewall.loc

   }

   script_user root

   notification_email_from Alexandre.Cassen@firewall.loc

   smtp_server 127.0.0.1

   smtp_connect_timeout 30

   router_id K8S_HA_NGINX_SLAVE
}

vrrp_script nginx_heartbeat {
    script "/etc/keepalived/nginx_heartbeat.sh"}

vrrp_instance VI_1 {

    # 定义初始状态，可以是MASTER或者BACKUP
    # 这里是 nginx 备用，设置为 BACKUP
    state BACKUP

    # 工作接口，通告选举使用哪个接口进行
    interface ens160

    # VRRP 路由 ID实例，每个实例是唯一的
    # 虚拟路由ID，如果是一组虚拟路由就定义一个ID，如果是多组就要定义多个，而且这个虚拟
    # ID还是虚拟MAC最后一段地址的信息，取值范围0-255
    virtual_router_id 51

    # BACKUP 的优先级低一点
    priority 90

    # 通告频率，单位为秒
    advert_int 1

    # 通信认证机制，这里是明文认证还有一种是加密认证
    authentication {
        auth_type PASS
        auth_pass 123456
    }

    # 虚拟 ip。即 vip
    virtual_ipaddress {
        172.16.222.110/24
    }
    track_script {
        nginx_heartbeat
    }
}
EOF
```

创建 `nginx_heartbeat.sh` 脚本

```shell
cat > /etc/keepalived/nginx_heartbeat.sh  << 'EOF'
#!/bin/bash
count=$(ps -ef |grep nginx |egrep -cv "grep|$$")
if [ "$count" -eq 0 ];
then
    exit 1
else
    exit 0
fi
EOF
chmod +x /etc/keepalived/nginx_heartbeat.sh
```

## 7.启动

启动

```shell
systemctl daemon-reload && \
systemctl start nginx && \
systemctl start keepalived
```

查看服务状态
  

*查看状态是否都为 `Active`*

```shell
systemctl status nginx && \
systemctl status keepalived
```

设置开机启动

```shell
systemctl enable nginx && \
systemctl enable keepalived
```

## 8.验证

### 8-1.验证是否可以访问到 `kube-apiserver`

在任意一台 `k8s` 服务器访问

```shell
curl -k https://172.16.222.110:8443/version
```

显示如下表示可以访问到 `kube-apiserver`
  

*没有详细信息和 `401` 是因为没有 `ssl` 验证。此时已经可以访问 `kube-apiserver`*

```text
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "Unauthorized",
  "reason": "Unauthorized",
  "code": 401
}
```

### 8-2.验证 `vip` 是否有漂移

在以上两台服务器都启动服务后，测试一下高可用是否正常。

#### 8-2-1.查看 `k8s-ha-master` 的信息

查看 `vip` 地址

```shell
ip a
```

显示如下
  

*可以看到有 `vip ip` 地址：`172.16.222.110`*

```text
2: ens160: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:0c:29:e6:09:4c brd ff:ff:ff:ff:ff:ff
    inet 172.16.222.201/24 brd 172.16.222.255 scope global noprefixroute ens160
       valid_lft forever preferred_lft forever
    inet 172.16.222.110/24 scope global secondary ens160
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fee6:94c/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
```

查看`keepalived` 状态

```shell
systemctl status keepalived
```

显示如下
  

*可以看到此时 `vip ip` 地址 `172.16.222.110` 在此服务器上*

```text
● keepalived.service - LVS and VRRP High Availability Monitor
   Loaded: loaded (/usr/lib/systemd/system/keepalived.service; enabled; vendor preset: disabled)
   Active: active (running) since Sun 2022-10-02 14:34:04 CST; 14min ago
  Process: 7339 ExecStart=/usr/sbin/keepalived $KEEPALIVED_OPTIONS (code=exited, status=0/SUCCESS)
 Main PID: 7342 (keepalived)
    Tasks: 2 (limit: 23462)
   Memory: 1.9M
   CGroup: /system.slice/keepalived.service
           ├─7342 /usr/sbin/keepalived -D
           └─7343 /usr/sbin/keepalived -D

Oct 02 14:41:51 k8s-master-ha-01 Keepalived_vrrp[7343]: Sending gratuitous ARP on ens160 for 172.16.222.110
Oct 02 14:41:51 k8s-master-ha-01 Keepalived_vrrp[7343]: Sending gratuitous ARP on ens160 for 172.16.222.110
Oct 02 14:41:51 k8s-master-ha-01 Keepalived_vrrp[7343]: Sending gratuitous ARP on ens160 for 172.16.222.110
Oct 02 14:41:51 k8s-master-ha-01 Keepalived_vrrp[7343]: Sending gratuitous ARP on ens160 for 172.16.222.110
Oct 02 14:41:56 k8s-master-ha-01 Keepalived_vrrp[7343]: (VI_1) Sending/queueing gratuitous ARPs on ens160 for 172.16.222.110
Oct 02 14:41:56 k8s-master-ha-01 Keepalived_vrrp[7343]: Sending gratuitous ARP on ens160 for 172.16.222.110
Oct 02 14:41:56 k8s-master-ha-01 Keepalived_vrrp[7343]: Sending gratuitous ARP on ens160 for 172.16.222.110
Oct 02 14:41:56 k8s-master-ha-01 Keepalived_vrrp[7343]: Sending gratuitous ARP on ens160 for 172.16.222.110
Oct 02 14:41:56 k8s-master-ha-01 Keepalived_vrrp[7343]: Sending gratuitous ARP on ens160 for 172.16.222.110
Oct 02 14:41:56 k8s-master-ha-01 Keepalived_vrrp[7343]: Sending gratuitous ARP on ens160 for 172.16.222.110
```

#### 8-2-2.查看 `k8s-ha-backup` 的状态

```shell
systemctl status keepalived
```

显示如下
  

*从最后三条信息中可以看出，`BACKUP` 的权重比较低*
  

*而且 `MASTER` 有在工作，所以进入 `BACKUP` 状态，并且移除了 `vip` 地址*

```text
● keepalived.service - LVS and VRRP High Availability Monitor
   Loaded: loaded (/usr/lib/systemd/system/keepalived.service; enabled; vendor preset: disabled)
   Active: active (running) since Sun 2022-10-02 14:35:52 CST; 21min ago
 Main PID: 5599 (keepalived)
    Tasks: 2 (limit: 23462)
   Memory: 9.7M
   CGroup: /system.slice/keepalived.service
           ├─5599 /usr/sbin/keepalived -D
           └─5600 /usr/sbin/keepalived -D

Oct 02 14:50:01 k8s-master-ha-02 Keepalived_vrrp[5600]: Sending gratuitous ARP on ens160 for 172.16.222.110
Oct 02 14:50:06 k8s-master-ha-02 Keepalived_vrrp[5600]: (VI_1) Sending/queueing gratuitous ARPs on ens160 for 172.16.222.110
Oct 02 14:50:06 k8s-master-ha-02 Keepalived_vrrp[5600]: Sending gratuitous ARP on ens160 for 172.16.222.110
Oct 02 14:50:06 k8s-master-ha-02 Keepalived_vrrp[5600]: Sending gratuitous ARP on ens160 for 172.16.222.110
Oct 02 14:50:06 k8s-master-ha-02 Keepalived_vrrp[5600]: Sending gratuitous ARP on ens160 for 172.16.222.110
Oct 02 14:50:06 k8s-master-ha-02 Keepalived_vrrp[5600]: Sending gratuitous ARP on ens160 for 172.16.222.110
Oct 02 14:50:06 k8s-master-ha-02 Keepalived_vrrp[5600]: Sending gratuitous ARP on ens160 for 172.16.222.110
Oct 02 14:55:49 k8s-master-ha-02 Keepalived_vrrp[5600]: (VI_1) Master received advert from 172.16.222.201 with higher priority 100, ours 90
Oct 02 14:55:49 k8s-master-ha-02 Keepalived_vrrp[5600]: (VI_1) Entering BACKUP STATE
Oct 02 14:55:49 k8s-master-ha-02 Keepalived_vrrp[5600]: (VI_1) removing VIPs.
```

#### 8-2-3.测试转移 `vip`

停掉 `k8s-ha-master` 的 `nginx`

```shell
systemctl stop nginx
```

再到 `k8s-ha-backup` 查看的 `keepalived` 状态

```shell
systemctl status keepalived
```

显示如下
  

*可以看到通过 `nginx_heartbeat.sh` 脚本检测不到 `nginx` 进程*
  

*从而停止了 `vip` 的创建*

```text
Oct 02 14:41:56 k8s-master-ha-01 Keepalived_vrrp[7343]: Sending gratuitous ARP on ens160 for 172.16.222.110
Oct 02 14:41:56 k8s-master-ha-01 Keepalived_vrrp[7343]: Sending gratuitous ARP on ens160 for 172.16.222.110
Oct 02 14:41:56 k8s-master-ha-01 Keepalived_vrrp[7343]: Sending gratuitous ARP on ens160 for 172.16.222.110
Oct 02 14:41:56 k8s-master-ha-01 Keepalived_vrrp[7343]: Sending gratuitous ARP on ens160 for 172.16.222.110
Oct 02 14:41:56 k8s-master-ha-01 Keepalived_vrrp[7343]: Sending gratuitous ARP on ens160 for 172.16.222.110
Oct 02 14:50:00 k8s-master-ha-01 Keepalived_vrrp[7343]: Script `nginx_heartbeat` now returning 1
Oct 02 14:50:00 k8s-master-ha-01 Keepalived_vrrp[7343]: VRRP_Script(nginx_heartbeat) failed (exited with status 1)
Oct 02 14:50:00 k8s-master-ha-01 Keepalived_vrrp[7343]: (VI_1) Entering FAULT STATE
Oct 02 14:50:00 k8s-master-ha-01 Keepalived_vrrp[7343]: (VI_1) sent 0 priority
Oct 02 14:50:00 k8s-master-ha-01 Keepalived_vrrp[7343]: (VI_1) removing VIPs.
```

再到 `k8s-ha-backup` 服务器查看 `ip` 地址

```shell
ip a
```

显示如下
  

*可以看到在 `k8s-ha-backup` 服务器上已经有创建 `vip` 地址*

```text
2: ens160: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:0c:29:2d:ad:07 brd ff:ff:ff:ff:ff:ff
    inet 172.16.222.202/24 brd 172.16.222.255 scope global noprefixroute ens160
       valid_lft forever preferred_lft forever
    inet 172.16.222.110/24 scope global secondary ens160
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fe2d:ad07/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
```

查看 `k8s-ha-backup` 的 `keepalived` 状态

```shell
systemctl status keepalived
```

显示如下
  

*可以看出此时已经在 `k8s-ha-backup` 服务器创建了 `vip` 地址了*

```text
● keepalived.service - LVS and VRRP High Availability Monitor
   Loaded: loaded (/usr/lib/systemd/system/keepalived.service; enabled; vendor preset: disabled)
   Active: active (running) since Sun 2022-10-02 14:35:52 CST; 35min ago
 Main PID: 5599 (keepalived)
    Tasks: 2 (limit: 23462)
   Memory: 9.7M
   CGroup: /system.slice/keepalived.service
           ├─5599 /usr/sbin/keepalived -D
           └─5600 /usr/sbin/keepalived -D

Oct 02 15:11:02 k8s-master-ha-02 Keepalived_vrrp[5600]: Sending gratuitous ARP on ens160 for 172.16.222.110
Oct 02 15:11:02 k8s-master-ha-02 Keepalived_vrrp[5600]: Sending gratuitous ARP on ens160 for 172.16.222.110
Oct 02 15:11:02 k8s-master-ha-02 Keepalived_vrrp[5600]: Sending gratuitous ARP on ens160 for 172.16.222.110
Oct 02 15:11:02 k8s-master-ha-02 Keepalived_vrrp[5600]: Sending gratuitous ARP on ens160 for 172.16.222.110
Oct 02 15:11:07 k8s-master-ha-02 Keepalived_vrrp[5600]: (VI_1) Sending/queueing gratuitous ARPs on ens160 for 172.16.222.110
Oct 02 15:11:07 k8s-master-ha-02 Keepalived_vrrp[5600]: Sending gratuitous ARP on ens160 for 172.16.222.110
Oct 02 15:11:07 k8s-master-ha-02 Keepalived_vrrp[5600]: Sending gratuitous ARP on ens160 for 172.16.222.110
Oct 02 15:11:07 k8s-master-ha-02 Keepalived_vrrp[5600]: Sending gratuitous ARP on ens160 for 172.16.222.110
Oct 02 15:11:07 k8s-master-ha-02 Keepalived_vrrp[5600]: Sending gratuitous ARP on ens160 for 172.16.222.110
Oct 02 15:11:07 k8s-master-ha-02 Keepalived_vrrp[5600]: Sending gratuitous ARP on ens160 for 172.16.222.110
```

到任意一台 `k8s` 服务器使用 `vip` 再次访问 `kube-apiserver`

```shell
curl -k https://172.16.222.110:8443/version
```

显示如下
  

*可以看到一样能够访问到 `kube-apiserver`*

```text
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "Unauthorized",
  "reason": "Unauthorized",
  "code": 401
}
```

***

## 至此。`kube-apiserver` 高可用已经部署成功。也就是 `k8s` 高可用部署成功。
