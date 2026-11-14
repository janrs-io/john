---
title: "Kubernetes Binary HA Cluster Deployment Series (10/18) — High Availability with Nginx & Keepalived"
author: "John Yang"
lang: en
pubDatetime: 2022-10-17T06:16:00Z
tags:
  - Kubernetes
  - HA
description: "Verifying kube-controller-manager and kube-scheduler leader election, and building an L4 HA load balancer for kube-apiserver with Nginx + Keepalived VIP failover."
---

> [!TIP]
> Kubernetes Binary Deployment — High Availability (HA)

***

> [!WARNING]
> Cloud providers such as Alibaba Cloud ECS do not support Keepalived multicast/VRRP VIPs. In cloud environments, test Keepalived locally or use Nginx TCP reverse proxying (or a cloud LB) directly. Ensure every Nginx/LB IP and VIP is included in `kube-apiserver`'s `hosts` SAN list!

# Control-Plane High Availability

## 1. Built-In vs. External HA

- `kube-controller-manager` and `kube-scheduler` provide built-in active-passive HA via `--leader-elect=true`.
- `kube-apiserver` is stateless and serves HTTPS; we front all three instances with **Nginx** (L4 TCP stream proxy) and **Keepalived** (VRRP VIP `172.16.222.110`).

### 1.1 Verify `kube-controller-manager` and `kube-scheduler` Leader Election

```shell
kubectl get leases -n kube-system
```

```text
NAME                      HOLDER                                              AGE
kube-controller-manager   k8s-master01_e0f4cfd5-1190-4f79-9ee5-a2063eb3ca16   156m
kube-scheduler            k8s-master01_54254610-53a8-4c3a-b3ea-a4fa5f549119   99s
```

Stop control-plane services on `master-01`:

```shell
systemctl stop kube-scheduler && \
systemctl stop kube-controller-manager && \
systemctl stop kube-apiserver
```

Check leases from `master-02`—ownership automatically fails over:

```shell
kubectl get leases -n kube-system
```

```text
NAME                      HOLDER                                              AGE
kube-controller-manager   k8s-master02_e0f4cfd5-1190-4f79-9ee5-a2063eb3ca16   3h5m
kube-scheduler            k8s-master03_e350060d-68ad-4f59-82a8-456f835b7f3d   30m
```

## 2. Initialize the Two HA Hosts

Run on `k8s-ha-master` (`172.16.222.201`) and `k8s-ha-backup` (`172.16.222.202`):

```shell
dnf install epel-release vim iptables jq ipvsadm ipset curl net-tools rsyslog -y
systemctl stop firewalld && systemctl disable firewalld
iptables -F && iptables -X && iptables -F -t nat && iptables -X -t nat
swapoff -a && sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
setenforce 0 && sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
timedatectl set-timezone Asia/Shanghai
systemctl enable chronyd && systemctl start chronyd
timedatectl set-local-rtc 0
systemctl restart rsyslog && systemctl restart crond
```

Configure persistent `systemd-journald` and ensure multicast is enabled on `ens160` (`ip link set multicast on dev ens160`), then reboot (`shutdown -r now`).

## 3. Install `keepalived` and `nginx`

On both HA nodes:

```shell
dnf install keepalived nginx -y
```

## 4. Configure Nginx (`8443` -> `6443`)

On both HA nodes:

```shell
mv /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak && \
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

# L4 TCP load balancing across all three kube-apiserver nodes
stream {
    log_format  main  '$remote_addr $upstream_addr - [$time_local] $status $upstream_bytes_sent';
    access_log  /var/log/nginx/k8s-access.log  main;

    upstream k8s-apiserver {
       server 172.16.222.121:6443;   # Master 1
       server 172.16.222.122:6443;   # Master 2
       server 172.16.222.123:6443;   # Master 3
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

## 5. Configure Keepalived

### 5.1 On `k8s-ha-master` (`state MASTER`, `priority 100`)

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
    state MASTER
    interface ens160
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 123456
    }
    virtual_ipaddress {
        172.16.222.110/24
    }
    track_script {
        nginx_heartbeat
    }
}
EOF

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

### 5.2 On `k8s-ha-backup` (`state BACKUP`, `priority 90`)

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
    script "/etc/keepalived/nginx_heartbeat.sh"
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens160
    virtual_router_id 51
    priority 90
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 123456
    }
    virtual_ipaddress {
        172.16.222.110/24
    }
    track_script {
        nginx_heartbeat
    }
}
EOF

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

## 6. Start Services and Test VIP Failover

```shell
systemctl daemon-reload && \
systemctl start nginx && \
systemctl start keepalived && \
systemctl enable nginx && \
systemctl enable keepalived
```

Test reaching `kube-apiserver` via the VIP (`172.16.222.110:8443`):

```shell
curl -k https://172.16.222.110:8443/version
```

Receiving a JSON `401 Unauthorized` response confirms TLS connectivity through Nginx + VIP to `kube-apiserver`:

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

Simulate a failure by running `systemctl stop nginx` on `k8s-ha-master`: `nginx_heartbeat.sh` exits `1`, `keepalived` enters `FAULT STATE` and releases `172.16.222.110`, and `k8s-ha-backup` automatically claims the VIP so `curl -k https://172.16.222.110:8443/version` keeps working seamlessly.

***

Control-plane high availability is now live.
