---
title: "Kubernetes Binary HA Cluster Deployment Series (5/18) — Server Initialization"
author: "John Yang"
lang: en
pubDatetime: 2022-10-17T06:14:00Z
tags:
  - Kubernetes
  - Linux
description: "Preparing Kubernetes master and worker nodes on AlmaLinux 8.6: packages, swap/SELinux/firewalld, br_netfilter, sysctl tuning, IPVS, chrony, and journald."
---

> [!TIP]
> Kubernetes Binary Deployment — Initializing the Server Environment

***

# System Initialization

> [!NOTE]
> Run these steps on every Kubernetes node (`master` and `node` hosts). Dedicated `etcd` hosts do not need this setup.

## Set Hostnames

```shell
hostnamectl set-hostname k8s-master01
hostnamectl set-hostname k8s-master02
hostnamectl set-hostname k8s-master03
```

## Configure `/etc/hosts`

> [!NOTE]
> Configure on the `master` servers.

```shell
cat >> /etc/hosts <<EOF
172.16.222.121  k8s-master01
172.16.222.122  k8s-master02
172.16.222.123  k8s-master03
EOF
cat /etc/hosts
```

## Install Required Packages

Install EPEL and Kubernetes/system utilities:

```shell
dnf install epel-release -y
dnf install conntrack ipvsadm ipset jq curl sysstat libseccomp wget vim net-tools git unzip tar curl iptables rsyslog -y
```

## Disable Firewall & Flush `iptables`

```shell
systemctl stop firewalld && systemctl disable firewalld
iptables -F && iptables -X && iptables -F -t nat && iptables -X -t nat
```

## Disable Swap

> [!NOTE]
> `kubelet` fails to start when swap is enabled unless `--fail-swap-on=false` is explicitly passed.

```shell
swapoff -a && \
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

## Disable SELinux

> [!NOTE]
> Prevents `Permission denied` errors when `kubelet` mounts container volumes.

```shell
setenforce 0 && \
sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
```

## Load `br_netfilter` and `ip_conntrack`

```shell
modprobe br_netfilter && modprobe ip_conntrack
```

Persist across reboots:

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

## Tune Kernel Parameters (`sysctl`)

> [!WARNING]
> On kernels older than `4.1`, set `fs.may_detach_mounts=1` and `net.ipv4.tcp_tw_recycle=0`.
> - Enabling `net.ipv4.tcp_tw_recycle=1` breaks internal NAT/load-balanced TCP connections in Kubernetes.
> - Leaving `fs.may_detach_mounts=0` can cause Pods to get stuck in `Terminating` with `UnmountVolume.TearDown failed for volume` (see [kubernetes/kubernetes#51835](https://github.com/kubernetes/kubernetes/issues/51835) and [Red Hat Bugzilla #1441737](https://bugzilla.redhat.com/show_bug.cgi?id=1441737)).

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

## Increase File Descriptor Limits

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

## Load `ipvs` Kernel Modules

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

## Configure Timezone & NTP Sync

```shell
timedatectl set-timezone Asia/Shanghai
systemctl enable chronyd && systemctl start chronyd
timedatectl status
```

Write UTC time to the hardware clock and restart time-dependent daemons:

```shell
timedatectl set-local-rtc 0
systemctl restart rsyslog && systemctl restart crond
```

## Configure Persistent `systemd-journald`

```shell
mkdir /var/log/journal && \
mkdir /etc/systemd/journald.conf.d && \
cat > /etc/systemd/journald.conf.d/99-prophet.conf <<EOF
[Journal]
# Persist logs to disk
Storage=persistent
# Compress archived logs
Compress=yes
SyncIntervalSec=5m
RateLimitInterval=30s
RateLimitBurst=1000
# Max disk usage: 10G
SystemMaxUse=10G
# Max size per log file: 200M
SystemMaxFileSize=200M
# Retain logs for 2 weeks
MaxRetentionSec=2week
# Do not forward logs to syslog
ForwardToSyslog=no
EOF
systemctl restart systemd-journald
```

## Create Lock File and Reboot

```shell
touch /tmp/k8s_init.lock
shutdown -r now
```

***

Server initialization is now complete.
