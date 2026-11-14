---
title: "【实战加详解】二进制部署k8s高可用集群系列教程五 -初始化服务器"
author: "John Yang"
lang: zh
pubDatetime: 2022-10-17T06:14:00Z
tags:
  - Kubernetes
  - Linux
description: "安装 k8s 所需软件以及依赖包和常用软件 加载 ipvs 内核模块 将当前的 UTC 时间写入硬件时钟 重启依赖于系统时间的服务 创建持久化保存日志的目录以及添加配置并生效 初始化后创建 k8sinit.lock 文件。 下次初始化查看是否有该文件，以免重复初始化造成意外问题。"
---

> [!TIP]
> 二进制部署 `k8s` - 初始化系统环境

***

# 初始化系统环境

> [!NOTE]
> 需要在每台服务器都执行。  

> 只需要初始化部署 `k8s` 集群的服务器。  

> 部署 `etcd` 集群的服务器不需要初始化。

## 设置主机名

```shell
hostnamectl set-hostname k8s-master01
hostnamectl set-hostname k8s-master02
hostnamectl set-hostname k8s-master03
```

## 设置 `/etc/hosts` 解析

> [!NOTE]
> 只需要在 `master` 服务器设置。

```shell
cat >> /etc/hosts <<EOF
172.16.222.121  k8s-master01
172.16.222.122  k8s-master02
172.16.222.123  k8s-master03
EOF
cat /etc/hosts
```

## 安装软件

安装 `epel`

```shell
dnf install epel-release -y
```

安装 `k8s` 所需软件以及依赖包和常用软件  

```shell
dnf install conntrack ipvsadm ipset jq curl sysstat libseccomp wget vim  net-tools git unzip tar curl iptables rsyslog -y
```

## 关闭防火墙

```shell
systemctl stop firewalld && systemctl disable firewalld
```

## 启动并清空 `iptables` 规则链

> [!NOTE]
> 参数 `-F` 清空规则链  

> 参数 `-X` 清空用户自定义的空链  

> 参数 `-F -t nat`清空 `nat` 表的所有链  

> 参数 `-X -t nat` 清空用户自定义的 `nat` 表的所有空链  

```shell
iptables -F && iptables -X && iptables -F -t nat && iptables -X -t nat
```

## 关闭 `swap` 分区

> [!NOTE]
> 关闭 `swap` 分区，否则 `kubelet` 会启动失败  

> *可以设置 `kubelet` 启动参数 `–fail-swap-on` 为 `false` 关闭 `swap` 检查*

```shell
swapoff -a && \
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

## 关闭 `selinux`

> [!NOTE]
> 关闭 `SELinux`，否则 `kubelet` 挂载目录时可能报错 `Permission denied`

```shell
setenforce 0 && \
sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
```

## 加载 `br_netfilter` 模块

> [!NOTE]
> 由于开启内核 `ipv4` 转发需要加载 `br_netfilter` 模块，所以要加载该模块

临时加载

```shell
modprobe br_netfilter && modprobe ip_conntrack
```

永久开机加载

```shell
cat >> /etc/rc.sysinit <<EOF
#!/bin/bash
for file in /etc/sysconfig/modules/*.modules ; do
[ -x $file ] && $file
done
EOF
echo "modprobe br_netfilter" >/etc/sysconfig/modules/br_netfilter.modules && \
echo "modprobe ip_conntrack" >/etc/sysconfig/modules/ip_conntrack.modules && \
chmod 755 /etc/sysconfig/modules/br_netfilter.modules && \
chmod 755 /etc/sysconfig/modules/ip_conntrack.modules
```

## 优化内核参数

> [!WARNING]
> **内核低于 `4.1` 版本需要添加 `fs.may_detach_mounts=1` 和 `net.ipv4.tcp_tw_recycle=0`**  

>   

> 在内核低于 `4.1` 中，不要设置 `net.ipv4.tcp_tw_recycle` 这个参数为 `1`
> ，网上有不少教程没提到或者内核版本过低系统默认设置为 `1`   

> 开启此参数，对于外网的 `sockets` 链接会快速回收。但是对于内网会导致大量的 `TCP` 链接建立错误。`k8s`
> 使用的都是在内网，所以要禁用！设置为 `0`  

> 有关 `net.ipv4.tcp_tw_recycle` 参数查看文章(https://cloud.tencent.com/developer/article/1683704)  

>   

> 参数 `fs.may_detach_mounts` 则是跟容器相关的。该参数如果设置为 `0`，会导致服务变更后旧 `pod`
> 在回收时会一直卡在 `Terminating` 的状态，会重复出现 `UnmountVolume.TearDown failed for volume` 错误。  

> 有关 `fs.may_detach_mounts` 参数产生的 `bug` 查看文章(https://github.com/kubernetes/kubernetes/issues/51835)
> 以及 (https://bugzilla.redhat.com/show_bug.cgi?id=1441737) 。

```shell
cat > /etc/sysctl.d/kubernetes.conf <<EOF
net.ipv4.tcp_slow_start_after_idle=0
net.core.rmem_max=16777216
fs.inotify.max_user_watches=1048576
kernel.softlockup_all_cpu_backtrace=1
kernel.softlockup_panic=1
fs.file-max=2097152
fs.nr_open=2097152
fs.inotify.max_user_instances=8192
fs.inotify.max_queued_events=16384
vm.max_map_count=262144
net.core.netdev_max_backlog=16384
net.ipv4.tcp_wmem=4096 12582912 16777216
net.core.wmem_max=16777216
net.core.somaxconn=32768
net.ipv4.ip_forward=1
net.ipv4.tcp_max_syn_backlog=8096
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.bridge.bridge-nf-call-arptables=1
net.ipv4.tcp_rmem=4096 12582912 16777216
vm.swappiness=0
kernel.sysrq=1
net.ipv4.neigh.default.gc_stale_time=120
net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.default.rp_filter=0
net.ipv4.conf.default.arp_announce=2
net.ipv4.conf.lo.arp_announce=2
net.ipv4.conf.all.arp_announce=2
net.ipv4.tcp_max_tw_buckets=5000
net.ipv4.tcp_syncookies=1
net.ipv4.tcp_synack_retries=2
net.ipv6.conf.lo.disable_ipv6=1
net.ipv6.conf.all.disable_ipv6=1
net.ipv6.conf.default.disable_ipv6=1
net.ipv6.conf.all.forwarding=0
net.ipv4.ip_local_port_range=1024 65535
net.ipv4.tcp_keepalive_time=600
net.ipv4.tcp_keepalive_probes=10
net.ipv4.tcp_keepalive_intvl=30
net.nf_conntrack_max=25000000
net.netfilter.nf_conntrack_max=25000000
net.netfilter.nf_conntrack_tcp_timeout_established=180
net.netfilter.nf_conntrack_tcp_timeout_time_wait=120
net.netfilter.nf_conntrack_tcp_timeout_close_wait=60
net.netfilter.nf_conntrack_tcp_timeout_fin_wait=12
net.ipv4.tcp_timestamps=0
net.ipv4.tcp_orphan_retries=3
kernel.pid_max=4194303
net.ipv4.tcp_tw_reuse=1
net.ipv4.tcp_fin_timeout=1
vm.min_free_kbytes=262144
kernel.msgmnb=65535
kernel.msgmax=65535
kernel.shmmax=68719476736
kernel.shmall=4294967296
kernel.core_uses_pid=1
net.netfilter.nf_conntrack_tcp_timeout_close=3
net.ipv4.neigh.default.gc_thresh1=0
net.ipv4.neigh.default.gc_thresh2=4096
net.ipv4.neigh.default.gc_thresh3=8192
net.ipv4.conf.all.route_localnet=1
EOF
sysctl -p /etc/sysctl.d/kubernetes.conf
```

## 设置系统文件打开数

```shell
cat >> /etc/security/limits.conf <<EOF
 *       soft    nofile  655350
 *       hard    nofile  655350
 *       soft    nproc   655350
 *       hard    nproc   655350
 *       soft    core    unlimited
 *       hard    core    unlimited
EOF
```

## 配置 `ipvs`

加载 `ipvs` 内核模块

```shell
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack
EOF
chmod 755 /etc/sysconfig/modules/ipvs.modules && \
bash /etc/sysconfig/modules/ipvs.modules && \
lsmod | grep -e ip_vs -e nf_conntrack
```

## 设置系统时区与同步

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

*显示如下*
> [!NOTE]
> `Time zone: Asia/Shanghai (CST, +0800)` 表示使用东八区时区  

> `System clock synchronized: yes` 表示时区有同步  

> `NTP service: active` 表示开启了时区同步服务

```text
               Local time: 三 2022-09-21 01:12:09 CST
           Universal time: 二 2022-09-20 17:12:09 UTC
                 RTC time: 二 2022-09-20 17:12:09
                Time zone: Asia/Shanghai (CST, +0800)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no
```

将当前的 UTC 时间写入硬件时钟

```shell
timedatectl set-local-rtc 0
```

重启依赖于系统时间的服务

```shell
systemctl restart rsyslog && \
systemctl restart crond
```

## 设置 `systemd journald`

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

## 创建 `k8s_init.lock`

初始化后创建 `k8s_init.lock` 文件。
  

下次初始化查看是否有该文件，以免重复初始化造成意外问题。

```shell
touch /tmp/k8s_init.lock
```

## 重启

```shell
shutdown -r now
```

***

## 至此。初始化系统环境完成。
