---
title: "Ubuntu 24.04 从零部署 Xray-Reality 完整指南：安全加固、BBR 加速与 TLS 传输安全"
author: "John Yang"
lang: zh
pubDatetime: 2026-11-20T12:00:00Z
featured: true
tags:
  - DevOps
  - Networking
  - Security
  - Linux
description: "手把手教你从零在 Ubuntu 24.04 上部署 Xray VLESS-Reality 节点：涵盖 Google BBR 拥塞控制加速、SSH 公钥与 Fail2ban/UFW 安全加固、Reality TLS 传输安全配置以及客户端分流教程。"
---

> **重要免责声明与合规提示**：本教程仅用于**网络协议研究、Linux 运维技术学习与学术交流**。请严格遵守所在国家/地区的法律法规（包括《中华人民共和国网络安全法》及相关规定）以及相关云服务商、第三方平台的服务条款（Terms of Service）。**严禁将本教程内容用于任何商业用途、未授权网络访问、代理服务分发或任何违法违规行为**。

在日常的 Linux 服务器运维与网络架构学习中，如何同时兼顾**服务器自身安全（防爆破）**、**传输速度（低延迟高吞吐）**以及**协议传输安全性（抗主动探测与特征识别）**，是网络工程领域非常经典的话题。

本文将以一台全新的 **Ubuntu 24.04** 服务器为例，采用目前主流的 **VLESS + XTLS-Vision + Reality** 架构，带你从零完成一套“傻瓜式”、全脱敏的生产级部署。

---

## 一、核心原理：如何做到既安全、又快、又可靠？

在动手敲命令之前，先花 1 分钟理解这套架构为什么有效：

1. **既安全（防黑防爆破）**：
   - 修改默认 SSH 端口（告别 22 端口每天上万次的暴力扫描）；
   - 彻底禁用密码登录，强制使用 `ed25519` 非对称密钥免密登录；
   - 部署 `Fail2ban` 自动封禁异常 IP，配合 `UFW` 防火墙默认拒绝所有入站，仅放行必要端口。
2. **又保证速度（BBR + XTLS-Vision）**：
   - 开启 Linux 内核原生的 **Google BBR + FQ** 拥塞控制算法并调大 TCP 缓冲区，显著降低高丢包网络下的延迟并跑满带宽；
   - 使用 `xtls-rprx-vision` 流控，减少二次加解密开销，提升吞吐效率。
3. **抗主动探测（Reality TLS 握手回落）**：
   - 传统方案需要自己买域名、申请 SSL 证书，配置与维护成本较高；
   - **Reality** 无需单独绑定域名，它使用标准大型站点（如云服务或科技公司官网）的真实 TLS 1.3 握手特征，对未经授权的探测流量直接回落到目标真实站点，提升服务端端口的安全性。

---

## 二、第一步：基础环境更新与 BBR 网络加速

使用 SSH 登录到你的 Ubuntu 24.04 服务器后，首先更新系统并安装基础运维工具：

```bash
apt update && apt upgrade -y
apt install -y curl wget git ufw fail2ban jq
```

接着配置内核参数，开启 **Google BBR + FQ** 并优化 TCP 读写缓冲窗口：

```bash
cat << 'EOF' > /etc/sysctl.d/99-network-tuning.conf
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
net.ipv4.tcp_fastopen = 3
net.ipv4.tcp_rmem = 4096 87380 67108864
net.ipv4.tcp_wmem = 4096 65536 67108864
EOF

sysctl --system
```

验证 BBR 是否开启成功：

```bash
sysctl net.ipv4.tcp_congestion_control
# 输出 net.ipv4.tcp_congestion_control = bbr 即为成功
```

---

## 三、第二步：服务器安全加固（防爆破、防爆盘）

### 1. 修改 SSH 端口并强制公钥登录

> **重要提示**：执行本步骤前，请务必先在本地电脑生成好 SSH 密钥（`ssh-keygen -t ed25519`），并把公钥写入服务器的 `~/.ssh/authorized_keys`，确认能免密登录后再关闭密码登录！

假设我们将 SSH 端口修改为一个自定义高位端口（例如 `<SSH_PORT>`，请替换为 10000~65535 之间的任意数字）：

```bash
chmod 700 /root
chmod 700 /root/.ssh
chmod 600 /root/.ssh/authorized_keys

cat << 'EOF' > /etc/ssh/sshd_config.d/security.conf
Port <SSH_PORT>
PermitRootLogin prohibit-password
PasswordAuthentication no
PubkeyAuthentication yes
EOF

systemctl restart ssh
```

### 2. 配置 Fail2ban 自动封禁恶意 IP

针对自定义的 `<SSH_PORT>` 开启防暴力破解（10 分钟内失败 3 次即封禁 1 天）：

```bash
cat << 'EOF' > /etc/fail2ban/jail.d/custom-sshd.conf
[sshd]
enabled = true
port = <SSH_PORT>
filter = sshd
maxretry = 3
findtime = 10m
bantime = 1d
EOF

systemctl enable fail2ban --now
```

### 3. 开启 UFW 防火墙精准放行

默认拒绝所有外部主动连接，只放行你的 SSH 端口和即将部署的 Xray 端口（例如 `<XRAY_PORT>`）：

```bash
ufw default deny incoming
ufw default allow outgoing
ufw allow <SSH_PORT>/tcp comment 'SSH'
ufw allow <XRAY_PORT>/tcp comment 'Xray Reality'
ufw --force enable
```

### 4. 限制系统日志体积（防止长期运行占满硬盘）

```bash
mkdir -p /etc/systemd/journald.conf.d
cat << 'EOF' > /etc/systemd/journald.conf.d/max-size.conf
[Journal]
SystemMaxUse=100M
RuntimeMaxUse=50M
EOF

systemctl restart systemd-journald
```

---

## 四、第三步：安装 Xray-core 并生成 Reality 密钥

### 1. 使用官方脚本安装 Xray

```bash
bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install
```

### 2. 一键生成所需的三组核心凭证

在服务器终端依次执行以下命令，并**把输出的结果复制记录下来**：

```bash
# 1. 生成客户端连接用的 UUID
xray uuid

# 2. 生成 X25519 密钥对（会输出 PrivateKey 和 Password/PublicKey）
xray x25519

# 3. 生成 8 位 16 进制 ShortId
openssl rand -hex 4
```

请记好这 4 个值：
- **`UUID`**：填在服务端 `clients.id` 和客户端节点里；
- **`PrivateKey`**：**仅填在服务端** `privateKey` 中，切勿泄露；
- **`PublicKey`**：**仅填在客户端** `pbk` 参数中；
- **`ShortId`**：服务端和客户端保持一致。

---

## 五、第四步：写入 Xray 服务端配置

将下面的配置写入 `/usr/local/etc/xray/config.json`（请将 `<XRAY_PORT>`、`<你的UUID>`、`<你的PrivateKey>`、`<你的ShortId>` 替换为真实值）：

> **回落目标选择建议**：`dest` 与 `serverNames` 建议选择支持 TLS 1.3 且网络连通性稳定的大型正规站点（例如 `www.microsoft.com` 或大型云服务官网）。

```bash
cat << 'EOF' > /usr/local/etc/xray/config.json
{
  "log": {
    "loglevel": "warning"
  },
  "inbounds": [
    {
      "port": <XRAY_PORT>,
      "protocol": "vless",
      "settings": {
        "clients": [
          {
            "id": "<你的UUID>",
            "flow": "xtls-rprx-vision"
          }
        ],
        "decryption": "none"
      },
      "streamSettings": {
        "network": "tcp",
        "security": "reality",
        "realitySettings": {
          "show": false,
          "dest": "www.microsoft.com:443",
          "xver": 0,
          "serverNames": [
            "www.microsoft.com"
          ],
          "privateKey": "<你的PrivateKey>",
          "shortIds": [
            "<你的ShortId>",
            ""
          ]
        }
      },
      "sniffing": {
        "enabled": true,
        "destOverride": [
          "http",
          "tls",
          "quic"
        ],
        "routeOnly": true
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "freedom"
    }
  ]
}
EOF
```

### 启动并设置开机自启

先做语法校验，确认无误后再启动：

```bash
/usr/local/bin/xray run -test -config /usr/local/etc/xray/config.json
# 确认输出 Configuration OK.

systemctl daemon-reload
systemctl enable xray --now
systemctl status xray
```

---

## 六、第五步：客户端连接与日常分流建议

在本地支持 VLESS-Reality 协议的客户端中，可以按以下标准 VLESS-Reality 链接格式导入（替换尖括号内容后复制导入）：

```text
vless://<你的UUID>@<你的服务器IP>:<XRAY_PORT>?security=reality&encryption=none&pbk=<你的PublicKey>&headerType=none&type=tcp&flow=xtls-rprx-vision&sni=www.microsoft.com&sid=<你的ShortId>&fp=chrome#My-Reality-Node
```

### 日常使用推荐设置
1. **路由模式**：选择 **`按规则分流（Rule Mode）`**，确保本地直连流量不绕路，仅需要加密传输的特定开发与研究流量走加密隧道。
2. **终端与命令行接管**：如果需要让 `git`、`curl`、`docker` 等命令行工具按规则走加密通道，可在合规前提下开启客户端的 **TUN 虚拟网卡模式**。

---

## 七、常见排查命令速查

```bash
# 查看 Xray 是否正在监听端口
ss -lntp | grep <XRAY_PORT>

# 查看 Xray 实时运行日志
journalctl -u xray -f --no-pager

# 查看 Fail2ban 当前封禁状态
fail2ban-client status sshd
```
