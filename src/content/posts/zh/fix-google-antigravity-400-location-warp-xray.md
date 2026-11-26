---
title: "基于 Cloudflare WARP 与 Xray 解决 Google AI 服务（Antigravity / Gemini / AI Studio / NotebookLM）地区限制完整方案"
author: "John Yang"
lang: zh
pubDatetime: 2026-11-17T10:00:00Z
modDatetime: 2026-11-27T02:05:00Z
featured: true
tags:
  - DevOps
  - Networking
  - Security
  - Architecture
description: "一套完整的 Google AI 服务地区限制解决方案：通过 Cloudflare WARP 本地代理配合 Xray SNI 嗅探与分流，统一解决 Google Antigravity (HTTP 400)、Gemini、AI Studio 与 NotebookLM 的机房/ISP IP 拦截问题，并附账号锁区与支付资料排查指南。"
---

在使用 Google Antigravity（调用 Gemini 模型）、Gemini 网页版、Google AI Studio 或 NotebookLM 等服务时，很多开发者即使已经将 Google 账号地区设为美国（US），且 VPS 节点的公网 IP 也显示为美国，依然会频繁遭遇地区拦截：

- **Google Antigravity**：任务立即终止并报错：
  ```text
  HTTP 400 Bad Request
  User location is not supported for the API use.
  status: FAILED_PRECONDITION
  ```
- **Gemini / AI Studio / NotebookLM**：打开页面或登录后直接提示“当前所在国家/地区不支持使用该服务”，或者出现 `gemini.google.com` 能打开但 `aistudio.google.com`、`notebooklm.google.com` 依然被拒的情况。

本文将从网络出口、分流规则到 Google 账号与支付资料，给出一套**生产级、零全局污染的完整解决方案**：**利用 Cloudflare WARP 本地代理模式，配合 Xray 入站 SNI 嗅探与出站分流，统一解锁 Google AI 全家桶，并提供分流生效后的账号级排查路径。**

---

## 1. 根因剖析：为什么美国 IP 依然被拦截？

Google 对旗下 AI 与云端开发接口实施了比普通网页更严格的风控策略，拦截通常来自以下四个层面：

1. **数据中心 / 特定 ISP ASN 过滤**：多数 VPS 机房或特定 ISP 广播的 IP 段在风控库中被标记为 Hosting/Datacenter 或高风险段。即便物理位置在美国，请求 `daily-cloudcode-pa.googleapis.com`、`aistudio.google.com`、`notebooklm.google.com` 等服务时仍会被拒绝。
2. **前端域名与后端 API 分离**：仅放行主站域名远远不够。例如 AI Studio 会请求 `*.clients6.google.com`（如 `alkalimakersuite-pa.clients6.google.com`），NotebookLM 会请求 `*.googleapis.com`（如 `tailwind-pa.googleapis.com`、`notebooklm-pa.googleapis.com`）以及 `labs.google`、`notebooklm.google`。若前端走代理出口而后端 API 仍走 VPS 原生直连，依然会触发地区报错。
3. **登录态与出口 IP 一致性校验**：NotebookLM 等服务对会话一致性极其敏感。如果业务域名走 WARP 出口，而 `accounts.google.com`、`gstatic.com` 等鉴权/资源域名走原生直连，出口 IP 不一致也会导致拦截。
4. **账号归属地与隐藏支付资料**：Google 会同时校验网络出口和账号自身的关联国家、隐藏的 Google Payments 付款资料、年龄及 Workspace 策略。

**为什么不建议整机全局开 WARP？**
让 VPS 全局流量走 WARP 容易引发 SSH 路由冲突，还会导致所有出站流量共用 Cloudflare 出口 IP，增加日常服务风控与运维复杂度。**最优架构**是保持 VPS 默认网络不变，仅在 Xray 层将需要解锁的 Google 流量定向转发至本机 `127.0.0.1:40000` 的 WARP SOCKS5 代理。

---

## 2. 整体网络拓扑

完整的数据流向如下：

```text
客户端 (v2rayN / Clash TUN 模式)
        ↓ VLESS Reality (自定义端口)
VPS:PORT
        ↓
系统 xray.service (开启入站 SNI 域名嗅探, routeOnly: true)
        │
        ├── 命中 Google AI 服务域名及关联生态 (含 geosite:google 兜底)
        │      ↓
        │   转发至本地 SOCKS5 端口 (127.0.0.1:40000)
        │      ↓
        │   Cloudflare WARP (Local Proxy 模式)
        │      ↓
        │   Cloudflare 出口 (loc=US, warp=on)
        │      ↓
        │   Antigravity / Gemini / AI Studio / NotebookLM (正常响应)
        │
        └── 其他所有网站流量
               ↓
            原有 VPS direct 原生出口直连
```

---

## 3. 部署前检查

在修改配置前，先确认实际监听代理端口的进程归属（区分系统原生 `xray.service` 与 3x-ui / x-ui 等面板托管进程）：

```bash
ss -lntp | grep -E ':(<你的代理端口>) '
```

确认实际接管流量的进程与配置文件路径（本文以系统原生 `/usr/local/etc/xray/config.json` 为例），避免改错配置导致规则不生效。

---

## 4. 第一步：安装并配置 Cloudflare WARP（Local Proxy 模式）

在 VPS（以 Ubuntu 24.04+ 为例）上安装官方 `cloudflare-warp` 客户端，并将其配置为仅监听回环地址的本地 SOCKS5 代理。

### 4.1 安装官方客户端

```bash
. /etc/os-release
install -d -m 0755 /usr/share/keyrings

# 导入 Cloudflare 官方 GPG Key
curl -4 -fsSL https://pkg.cloudflareclient.com/pubkey.gpg \
  | gpg --batch --yes --dearmor \
  -o /usr/share/keyrings/cloudflare-warp-archive-keyring.gpg

# 添加软件源
printf \
  'deb [signed-by=/usr/share/keyrings/cloudflare-warp-archive-keyring.gpg] https://pkg.cloudflareclient.com/ %s main\n' \
  "$VERSION_CODENAME" \
  > /etc/apt/sources.list.d/cloudflare-client.list

apt-get update -qq && apt-get install -y cloudflare-warp
```

### 4.2 注册并启用本地代理模式

> **注意**：新版 `warp-cli` 需显式附带 `--accept-tos` 参数。

```bash
# 注册客户端
warp-cli --accept-tos registration new

# 设置为本地代理模式（绝不接管整机默认网关）
warp-cli --accept-tos mode proxy

# 设置本地 SOCKS5 监听端口为 40000
warp-cli --accept-tos proxy port 40000

# 建立连接
warp-cli --accept-tos connect
```

### 4.3 状态与出口校验

```bash
# 1. 检查连接状态（应显示 Connected / Network: healthy）
warp-cli --accept-tos --no-ansi status

# 2. 确认仅监听 127.0.0.1，严禁暴露至 0.0.0.0
ss -lntp | grep ':40000'
# 预期输出: LISTEN 0 1024 127.0.0.1:40000 0.0.0.0:*

# 3. 验证 WARP 出口地区与状态
curl --socks5-hostname 127.0.0.1:40000 https://www.cloudflare.com/cdn-cgi/trace
# 关键字段: loc=US, warp=on

# 4. 验证目标 API TLS 握手连通性
curl --max-time 15 --socks5-hostname 127.0.0.1:40000 -o /dev/null -sS -w '%{http_code}\n' https://daily-cloudcode-pa.googleapis.com/
# 返回 404 即为正常（因未携带鉴权 Header），说明网络链路已通
```

---

## 5. 第二步：配置 Xray 入站嗅探与出站分流

修改 `/usr/local/etc/xray/config.json` 前，**务必先做带时间戳的备份**：

```bash
cp -a /usr/local/etc/xray/config.json "/usr/local/etc/xray/config.json.bak.$(date -u +%Y%m%dT%H%M%SZ)"
```

### 5.1 启用入站域名嗅探 (`inbounds[].sniffing`)

本地客户端（如 v2rayN / Clash）在本地完成 DNS 解析后，发往 VPS 的流量目的地址往往已经是 IP 而非域名。必须在入站节点开启 `sniffing`，让 Xray 从 TLS Client Hello 中还原 SNI 域名：

```json
"sniffing": {
  "enabled": true,
  "destOverride": [
    "http",
    "tls",
    "quic"
  ],
  "routeOnly": true
}
```

- **`routeOnly: true`**：嗅探出的域名仅用于 `routing.rules` 匹配，不改写原始目的 IP，兼顾分流精准度与连接稳定性。

### 5.2 添加 WARP SOCKS 出站 (`outbounds`)

在 `outbounds` 中保留原有的 `direct` 与 `blocked`，新增 `warp-out`：

```json
{
  "outbounds": [
    {
      "protocol": "freedom",
      "tag": "direct"
    },
    {
      "protocol": "blackhole",
      "tag": "blocked"
    },
    {
      "tag": "warp-out",
      "protocol": "socks",
      "settings": {
        "servers": [
          {
            "address": "127.0.0.1",
            "port": 40000
          }
        ]
      }
    }
  ]
}
```

### 5.3 配置分流规则 (`routing.rules`)

将 Google AI 服务及关联域名规则置于 `routing.rules` 最顶部：

```json
{
  "routing": {
    "domainStrategy": "AsIs",
    "rules": [
      {
        "type": "field",
        "domain": [
          "full:daily-cloudcode-pa.googleapis.com",
          "domain:gemini.google.com",
          "domain:aistudio.google.com",
          "domain:notebooklm.google.com",
          "domain:ai.google.dev",
          "domain:deepmind.google",
          "domain:generativelanguage.googleapis.com",
          "domain:clients6.google.com",
          "domain:makersuite.google.com",
          "domain:googleapis.com",
          "domain:labs.google",
          "domain:notebooklm.google",
          "geosite:google"
        ],
        "outboundTag": "warp-out"
      },
      {
        "type": "field",
        "protocol": [
          "bittorrent"
        ],
        "outboundTag": "blocked"
      }
    ]
  }
}
```

#### 规则拆解与选型权衡

各条规则分别解决对应服务的拦截点：
- **Antigravity**：`full:daily-cloudcode-pa.googleapis.com`（若同时遇到 `cloudcode-pa.googleapis.com` 也可一并保留或由下方的 `domain:googleapis.com` 覆盖）。
- **Gemini 与开发者接口**：`domain:gemini.google.com`、`domain:ai.google.dev`、`domain:deepmind.google`、`domain:generativelanguage.googleapis.com`。
- **AI Studio 及其后端**：`domain:aistudio.google.com`、`domain:clients6.google.com`、`domain:makersuite.google.com`。
- **NotebookLM 及其后端**：`domain:notebooklm.google.com`、`domain:notebooklm.google`、`domain:labs.google`、`domain:googleapis.com`。
- **全量一致性兜底（`geosite:google`）**：确保 `accounts.google.com`、`gstatic.com`、`googleusercontent.com` 等所有 Google 关联域名与上述 AI 服务保持**完全一致的 WARP 出口 IP**，彻底消除因鉴权域名与业务域名 IP 不一致导致的拦截。

> **关于 `geosite:google` 的取舍**：
> - **保留 `geosite:google`（全量兜底，最彻底）**：适合同时重度使用 NotebookLM、AI Studio、Gemini 等全套服务的场景。需注意两个小影响：① 访问 `www.google.com` 搜索时因 WARP 共享 IP 可能比原生 IP 更容易弹出一次性的图形验证码（reCAPTCHA）；② YouTube、Gmail、Google Drive 等流量也会经过本地 `127.0.0.1:40000`（延迟略微增加，通常感知不明显），但**完全不影响非 Google 网站与服务器上的其他业务**。
> - **精简模式（按需移除 `geosite:google`）**：如果你只用 Antigravity 或希望 Google 搜索走原生出口，可移除 `geosite:google`，仅保留上面明确列出的业务域名（若需兼容 NotebookLM 登录态，可单独补充 `"domain:accounts.google.com"`）。

---

## 6. 第三步：校验语法、重启服务与端到端验收

修改完成后，先用 Xray 自带校验命令检查 JSON 语法与规则合法性：

```bash
/usr/local/bin/xray run -test -config /usr/local/etc/xray/config.json
# 必须确认输出: Configuration OK.
```

确认无误后重启并检查服务状态：

```bash
systemctl restart xray
systemctl is-active xray warp-svc
# 均应返回: active
```

### 链路验收

1. **命令行与日志验收**：在本地客户端连上该节点后，测试目标接口并观察 VPS 端日志是否命中 `[warp-out]`：
   ```bash
   # 本地开发机测试（端口替换为本地客户端实际 SOCKS 端口）
   curl --max-time 15 --socks5-hostname 127.0.0.1:10808 -o /dev/null -sS -w '%{http_code}\n' https://daily-cloudcode-pa.googleapis.com/

   # VPS 端查看分流日志
   journalctl -u xray --since '1 minute ago' --no-pager | grep warp-out
   ```
2. **浏览器无痕窗口验证**：测试 Gemini、AI Studio、NotebookLM 时，**务必使用浏览器的无痕/隐私窗口**，避免旧的本地 Cookie 和缓存继续触发地区报错。

---

## 7. 第四步：分流已生效但仍提示“地区不支持”？账号级排查指南

如果在**无痕窗口**下连接该节点依然提示“地区不支持”，通常问题已经不在代理与分流链路，而是 **Google 账号本身的归属地锁区** 或 **隐藏的支付资料冲突**（Google 会优先校验账号关联国家，而非只看当前 IP）。请按以下 4 步逐一排查：

### 7.1 用无痕窗口未登录测试，区分“网络问题”与“账号锁区”

在无痕窗口、**不登录任何 Google 账号**的情况下直接打开 `https://aistudio.google.com/`、`https://gemini.google.com/` 或 `https://notebooklm.google.com/`：
- **未登录时就报错**：说明仍有相关域名未走 WARP 或存在出口 IP 不一致，检查 Xray 配置是否已启用 `geosite:google` 及 `sniffing` 规则，并确认 `xray.service` 与 `warp-svc` 均为 `active`。
- **未登录时能正常打开落地页，一登录就报错**：说明 **100% 是 Google 账号归属地或资格限制**，换任何 VPN/节点都无效，必须处理账号本身设置。

### 7.2 检查并修改 Google 账号关联国家（官方渠道）

1. **查看当前账号关联地区**：登录后访问 Google 服务条款页 [policies.google.com/terms](https://policies.google.com/terms)，拉至页面底部或条款正文查看当前关联的国家/地区。
2. **提交官方改区申请**：
   - 先保持代理连接在目标支持地区（如美国 US）；
   - 打开官方国家/地区关联更改表单：[policies.google.com/country-association-form](https://policies.google.com/country-association-form)；
   - 选择目标国家（如 `United States`），原因建议选择最后一项（`None of the above`），用英文说明因工作或学习需要访问 Gemini / AI Studio 并常驻该地区。提交后通常几小时到几天内会收到审核邮件。

### 7.3 检查隐藏的 Google Payments 支付资料冲突

即使主账号已经改区，如果 Google Payments 中仍残留旧国家/地区（如不支持地区）的付款资料，依然会被系统拦截：
1. 访问 [pay.google.com](https://pay.google.com) → **设置（Settings）**；
2. 检查 **国家/地区（Country/Region）**（若有多个付款资料，需在页面顶部下拉切换逐一核对）；
3. 若发现旧地区的付款资料，可在页面最底部点击**关闭付款资料**（状态同步通常需要 3~7 天生效）。

### 7.4 核对年龄与 Workspace 账号限制

- **年龄要求**：确认账号生日已满 18 岁（可在 [myaccount.google.com/birthday](https://myaccount.google.com/birthday) 核对）。
- **账号类型**：确认当前使用的不是被学校或企业管理员禁用生成式 AI 功能的 Google Workspace 账号。
- **官方社区**：若确认以上均无异常仍遇账号同步 Bug，可前往官方开发者社区 [discuss.ai.google.dev](https://discuss.ai.google.dev/) 反馈排查。

---

## 8. 总结与注意事项

1. **体系化解决思路**：面对 Google AI 服务（Antigravity、Gemini、AI Studio、NotebookLM）的地区限制，需同时覆盖“**WARP 清洁出口 + Xray SNI 嗅探与关联域名/geosite 分流 + 无痕测试与账号/支付资料排查**”三个环节，缺一不可。
2. **安全隔离**：WARP 本地 SOCKS5 代理必须仅绑定 `127.0.0.1:40000`，严禁暴露到公网 `0.0.0.0`。
3. **快速回滚**：任何配置变更前均保留带时间戳的 `.bak` 备份；若需调整分流粒度或回退，覆盖恢复配置文件并执行 `xray run -test` 与 `systemctl restart xray` 即可在数秒内生效。
4. **合规使用**：本方案旨在为具备合法合规开发资质、已在支持地区使用服务但受限于机房或 ISP 网络出口属性的开发者提供技术解决思路。
