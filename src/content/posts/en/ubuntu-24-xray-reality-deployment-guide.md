---
title: "Deploying Xray-Reality on Ubuntu 24.04 from Scratch: Hardening, BBR Tuning, and Anti-Probing"
author: "John Yang"
lang: en
pubDatetime: 2026-11-20T12:00:00Z
featured: true
tags:
  - DevOps
  - Networking
  - Security
  - Linux
description: "Step-by-step guide to deploying Xray VLESS-Reality on Ubuntu 24.04 with Google BBR congestion control, SSH key-only hardening, Fail2ban/UFW protection, and domain-less TLS camouflage."
---

> **Important Disclaimer & Compliance Notice**: This tutorial is provided strictly for **network protocol research, Linux administration learning, and technical exchange**. Please comply with all applicable local laws and regulations (including the Cybersecurity Law of the People's Republic of China where applicable) as well as the Terms of Service of your cloud and network providers. **Commercial use, unauthorized network access, proxy redistribution, or any illegal activity is strictly prohibited.**

When learning Linux server administration and network architecture, balancing **host security (anti-brute-force)**, **throughput speed (low latency)**, and **transport security (resisting active probing)** is a classic engineering challenge.

This step-by-step, sanitized guide walks through deploying **VLESS + XTLS-Vision + Reality** on a fresh **Ubuntu 24.04** server.

---

## 1. Architecture Overview: Security, Speed, and Stealth

1. **Host Security (Anti-Brute-Force)**:
   - Move SSH away from default port `22` to eliminate automated bot noise.
   - Disable password authentication completely and enforce `ed25519` public-key login.
   - Configure `Fail2ban` and `UFW` to deny all inbound traffic by default, exposing only required ports.
2. **Transport Speed (BBR + XTLS-Vision)**:
   - Enable Linux kernel **Google BBR + FQ** congestion control and enlarge TCP send/receive buffers to maximize throughput over high-latency links.
   - Use `xtls-rprx-vision` flow control to eliminate redundant TLS-in-TLS encryption overhead.
3. **Anti-Probing Stealth (Reality)**:
   - Unlike traditional TLS setups that require buying a domain and managing certificates, **Reality** borrows the genuine TLS handshake and certificate of a reputable, high-availability external site (e.g., `www.microsoft.com`), seamlessly forwarding unauthorized probes to the real target.

---

## 2. Step 1: System Update & Google BBR Tuning

Log into your Ubuntu 24.04 server via SSH, update packages, and install baseline tools:

```bash
apt update && apt upgrade -y
apt install -y curl wget git ufw fail2ban jq
```

Enable **Google BBR + FQ** and tune TCP buffers:

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

Verify BBR is active:

```bash
sysctl net.ipv4.tcp_congestion_control
# Expected: net.ipv4.tcp_congestion_control = bbr
```

---

## 3. Step 2: Server Security Hardening

### 1. Custom SSH Port & Key-Only Authentication

> **Important**: Before disabling password login, ensure you have generated an `ed25519` keypair locally (`ssh-keygen -t ed25519`), added your public key to `~/.ssh/authorized_keys`, and verified key-based login works.

Replace `<SSH_PORT>` with a custom high port (e.g., between `10000` and `65535`):

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

### 2. Configure Fail2ban for Custom SSH Port

Ban IPs for 1 day after 3 failed login attempts within 10 minutes:

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

### 3. Enable UFW Firewall

Deny all incoming connections by default and allow only your SSH port and Xray port (`<XRAY_PORT>`):

```bash
ufw default deny incoming
ufw default allow outgoing
ufw allow <SSH_PORT>/tcp comment 'SSH'
ufw allow <XRAY_PORT>/tcp comment 'Xray Reality'
ufw --force enable
```

### 4. Cap Systemd Journal Logs

Prevent `systemd-journald` from filling up disk space over time:

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

## 4. Step 3: Install Xray-core & Generate Reality Keys

### 1. Official Installation Script

```bash
bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install
```

### 2. Generate Credentials

Run these three commands and save the outputs:

```bash
# 1. Generate client UUID
xray uuid

# 2. Generate X25519 keypair (PrivateKey + Password/PublicKey)
xray x25519

# 3. Generate an 8-char hex ShortId
openssl rand -hex 4
```

Keep track of each value:
- **`UUID`**: Used in both server `clients.id` and client config.
- **`PrivateKey`**: **Server-only** (`privateKey`). Never share this.
- **`PublicKey`**: **Client-only** (`pbk` parameter).
- **`ShortId`**: Must match between server and client.

---

## 5. Step 4: Configure & Start Xray Server

Write `/usr/local/etc/xray/config.json` (replace `<XRAY_PORT>`, `<YOUR_UUID>`, `<YOUR_PRIVATE_KEY>`, and `<YOUR_SHORT_ID>`):

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
            "id": "<YOUR_UUID>",
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
          "privateKey": "<YOUR_PRIVATE_KEY>",
          "shortIds": [
            "<YOUR_SHORT_ID>",
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

### Validate & Enable Service

```bash
/usr/local/bin/xray run -test -config /usr/local/etc/xray/config.json
# Expected: Configuration OK.

systemctl daemon-reload
systemctl enable xray --now
systemctl status xray
```

---

## 6. Step 5: Client Connection & Routing

In a compatible local VLESS-Reality client, import the following VLESS-Reality URI after substituting your own values:

```text
vless://<YOUR_UUID>@<SERVER_IP>:<XRAY_PORT>?security=reality&encryption=none&pbk=<YOUR_PUBLIC_KEY>&headerType=none&type=tcp&flow=xtls-rprx-vision&sni=www.microsoft.com&sid=<YOUR_SHORT_ID>&fp=chrome#My-Reality-Node
```

### Recommended Client Settings
1. **Routing Mode**: Use **`Rule Mode`** so local/direct traffic connects directly without unnecessary hops, and only designated development or research traffic traverses the encrypted tunnel.
2. **TUN Mode**: Enable **TUN mode** if you want CLI tools (`git`, `curl`, `docker`) to route according to your configured rules.

---

## 7. Quick Troubleshooting Commands

```bash
# Verify Xray is listening on your configured port
ss -lntp | grep <XRAY_PORT>

# Tail live Xray logs
journalctl -u xray -f --no-pager

# Check Fail2ban SSH jail status
fail2ban-client status sshd
```
