---
title: "【实战加详解】二进制部署k8s高可用集群教程系列十七 - 超实用k8s操作命令别名"
author: "John Yang"
lang: zh
pubDatetime: 2022-10-17T06:19:00Z
tags:
  - Kubernetes
  - DevOps
description: "部署期间不停的重启服务，查看日志，查看 k8s 部署情况。打命令打到手指断掉。 一怒之下整理了常用命令别名。"
---

> [!TIP]
> 二进制部署 `k8s` - 实用命令别名

***

# `k8s` 命令别名

部署期间不停的重启服务，查看日志，查看 `k8s` 部署情况。打命令打到手指断掉。
  

  

一怒之下整理了常用命令别名。

## `kubectl` 常用命令别名

```shell
cat > /etc/profile.d/kubectl_cmd_alias.sh <<EOF

# 设置kubectl常用别名
alias k='kubectl'
alias kg='k get'
alias kdl='k delete -f'
alias kap='k apply -f'
alias kd='k describe'

# pod command
alias kgpo='kg pods'
alias kgpow='kg pods -o wide'
alias kgpoa='kg pods --all-namespaces'
alias kgpoaw='kg pods --all-namespaces -o wide'
alias kdpo='kd pod'

# namespace command
alias kgns='kg ns'

# service command
alias kgsvca='kg svc --all-namespaces'
alias kgsvc='kg svc'
alias kdsvc='kd svc'

# ingress command
alias kgia='kg ingress --all-namespaces'
alias kgi='kg ingress'

# endpoint command
alias kgepa='kg ep --all-namespaces'
alias kgep='kg ep'

#kubectl describe ep
alias kdep='kd ep'

# node command
alias kgn='kg nodes'
alias kgnl='kg nodes --show-labels'
alias kdn='kd nodes'

# deploy command
alias kgdpa='kg deploy --all-namespaces'
alias kgdp='kg deploy'
alias kddp='kd deploy'

EOF
source /etc/profile
```

## `master` 组件常用命令别名

```shell
cat > /etc/profile.d/master_cmd_alias.sh <<EOF

alias restartapi='systemctl daemon-reload && systemctl restart kube-apiserver'
alias restartcontroller='systemctl daemon-reload && systemctl restart kube-controller-manager'
alias restartscheduler='systemctl daemon-reload && systemctl restart kube-scheduler'

alias startapi='systemctl daemon-reload && systemctl start kube-apiserver'
alias startcontroller='systemctl daemon-reload && systemctl start kube-controller-manager'
alias startscheduler='systemctl daemon-reload && systemctl start kube-scheduler'

alias stopapi='systemctl daemon-reload && systemctl stop kube-apiserver'
alias stopcontroller='systemctl daemon-reload && systemctl stop kube-controller-manager'
alias stopscheduler='systemctl daemon-reload && systemctl stop kube-scheduler'

# 获取组件运行状态以及日志
alias kgapis='echo -e "'"\n\n\n"'" && systemctl status kube-apiserver -n 10 --no-pager -l && echo -e "'"\n"'"'
alias kgapisl='echo -e "'"\n\n\n"'" && systemctl status kube-apiserver --no-pager -l && echo -e "'"\n"'"'

alias kgcontrollers='echo -e "'"\n\n\n"'" && systemctl status kube-controller-manager -n 10 --no-pager -l && echo -e "'"\n"'"'
alias kgcontrollersl='echo -e "'"\n\n\n"'" && systemctl status kube-controller-manager --no-pager -l && echo -e "'"\n"'"'

alias kgschedulers='echo -e "'"\n\n\n"'" && systemctl status kube-scheduler -n 10 --no-pager -l && echo -e "'"\n"'"'
alias kgschedulersl='echo -e "'"\n\n\n"'" && systemctl status kube-scheduler --no-pager -l && echo -e "'"\n"'"'

alias kgapilog='echo -e "'"\n\n\n"'" && journalctl -n 100 -l --no-pager  -u kube-apiserver && echo -e "'"\n"'"'
alias kgcontrollerlog='echo -e "'"\n\n\n"'" && journalctl -n 100 -l --no-pager  -u kube-controller-manager && echo -e "'"\n"'"'
alias kgschedulerlog='echo -e "'"\n\n\n"'" && journalctl -n 100 -l --no-pager  -u kube-scheduler && echo -e "'"\n"'"'

alias kgapierr='echo -e "'"\n\n\n"'" && tail -n 30 /var/log/kubernetes/apiserver/kube-apiserver.ERROR && echo -e "'"\n"'"'
alias kgapiinfo='echo -e "'"\n\n\n"'" && tail -n 30 /var/log/kubernetes/apiserver/kube-apiserver.INFO && echo -e "'"\n"'"'
alias kgapiwar='echo -e "'"\n\n\n"'" && tail -n 30 /var/log/kubernetes/apiserver/kube-apiserver.WARNING && echo -e "'"\n"'"'
alias kgapifatal='echo -e "'"\n\n\n"'" && tail -n 30 /var/log/kubernetes/apiserver/kube-apiserver.FATAL && echo -e "'"\n"'"'

alias kgcontrollererr='echo -e "'"\n\n\n"'" && tail -n 30 /var/log/kubernetes/controller/kube-controller-manager.ERROR && echo -e "'"\n"'"'
alias kgcontrollerinfo='echo -e "'"\n\n\n"'" && tail -n 30 /var/log/kubernetes/controller/kube-controller-manager.INFO && echo -e "'"\n"'"'
alias kgcontrollerwar='echo -e "'"\n\n\n"'" && tail -n 30 /var/log/kubernetes/controller/kube-controller-manager.WARNING && echo -e "'"\n"'"'
alias kgcontrollerfatal='echo -e "'"\n\n\n"'" && tail -n 30 /var/log/kubernetes/controller/kube-controller-manager.FATAL && echo -e "'"\n"'"'

alias kgschedulererr='echo -e "'"\n\n\n"'" && tail -n 30 /var/log/kubernetes/scheduler/kube-scheduler.ERROR && echo -e "'"\n"'"'
alias kgschedulerinfo='echo -e "'"\n\n\n"'" && tail -n 30 /var/log/kubernetes/scheduler/kube-scheduler.INFO && echo -e "'"\n"'"'
alias kgschedulerwar='echo -e "'"\n\n\n"'" && tail -n 30 /var/log/kubernetes/scheduler/kube-scheduler.WARNING && echo -e "'"\n"'"'
alias kgschedulerfatal='echo -e "'"\n\n\n"'" && tail -n 30 /var/log/kubernetes/scheduler/kube-scheduler.FATAL && echo -e "'"\n"'"'

EOF
source /etc/profile
```

## `node` 组件常用命令别名

```shell
cat > /etc/profile.d/node_cmd_alias.sh <<EOF

alias restartkubelet='systemctl daemon-reload && systemctl restart kubelet'
alias restartproxy='systemctl daemon-reload && systemctl restart kube-proxy'

alias startkubelet='systemctl daemon-reload && systemctl start kubelet'
alias startproxy='systemctl daemon-reload && systemctl start kube-proxy'

alias stopkubelet='systemctl stop kubelet'
alias stopproxy='systemctl stop kube-proxy'

# 获取组件运行状态以及日志
alias kgkubelets='echo -e "'"\n\n\n"'" && systemctl status kubelet -n 10 --no-pager -l && echo -e "'"\n"'"'
alias kgkubeletsl='echo -e "'"\n\n\n"'" && systemctl status kubelet -n 10 --no-pager -l && echo -e "'"\n"'"'

alias kgproxys='echo -e "'"\n\n\n"'" && systemctl status kube-proxy -n 10 --no-pager -l && echo -e "'"\n"'"'
alias kgproxysl='echo -e "'"\n\n\n"'" && systemctl status kube-proxy -n 10 --no-pager -l && echo -e "'"\n"'"'

alias kgkubeletlog='echo -e "'"\n\n\n"'" && journalctl -n 100 -l --no-pager  -u kubelet && echo -e "'"\n"'"'
alias kgproxylog='echo -e "'"\n\n\n"'" && journalctl -n 100 -l --no-pager  -u kube-proxy && echo -e "'"\n"'"'

alias kgkubeleterr='echo -e "'"\n\n\n"'" && tail -n 30 /var/log/kubernetes/kubelet/kubelet.ERROR && echo -e "'"\n"'"'
alias kgkubeletinfo='echo -e "'"\n\n\n"'" && tail -n 30 /var/log/kubernetes/kubelet/kubelet.INFO && echo -e "'"\n"'"'
alias kgkubeletfatal='echo -e "'"\n\n\n"'" && tail -n 30 /var/log/kubernetes/kubelet/kubelet.FATAL && echo -e "'"\n"'"'
alias kgkubeletwar='echo -e "'"\n\n\n"'" && tail -n 30 /var/log/kubernetes/kubelet/kubelet.WARNING && echo -e "'"\n"'"'

alias kgproxyerr='echo -e "'"\n\n\n"'" && tail -n 30 /var/log/kubernetes/kube-proxy/kube-proxy.ERROR && echo -e "'"\n"'"'
alias kgproxyinfo='echo -e "'"\n\n\n"'" && tail -n 30 /var/log/kubernetes/kube-proxy/kube-proxy.INFO && echo -e "'"\n"'"'
alias kgproxyfatal='echo -e "'"\n\n\n"'" && tail -n 30 /var/log/kubernetes/kube-proxy/kube-proxy.FATAL && echo -e "'"\n"'"'
alias kgproxywar='echo -e "'"\n\n\n"'" && tail -n 30 /var/log/kubernetes/kube-proxy/kube-proxy.WARNING && echo -e "'"\n"'"'

EOF
source /etc/profile
```
