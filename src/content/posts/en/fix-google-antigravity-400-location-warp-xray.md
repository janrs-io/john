---
title: "Unlocking Google AI Services (Antigravity, Gemini, AI Studio, NotebookLM) via Cloudflare WARP and Xray Routing"
author: "John Yang"
lang: en
pubDatetime: 2026-11-17T10:00:00Z
modDatetime: 2026-11-27T02:05:00Z
featured: true
tags:
  - DevOps
  - Networking
  - Security
  - Architecture
description: "A complete solution for Google AI region restrictions: use Cloudflare WARP local proxy with Xray SNI sniffing and targeted routing to unlock Google Antigravity (HTTP 400), Gemini, AI Studio, and NotebookLM, plus account country and Payments profile troubleshooting."
---

When using Google Antigravity (with Gemini models), Gemini Web, Google AI Studio, or NotebookLM, developers often hit region blocks even when their Google account is set to the United States (`loc=US`) and their VPS public IP geolocates to the US:

- **Google Antigravity**: Requests fail immediately with:
  ```text
  HTTP 400 Bad Request
  User location is not supported for the API use.
  status: FAILED_PRECONDITION
  ```
- **Gemini / AI Studio / NotebookLM**: Pages or signed-in sessions display "region not supported," or `gemini.google.com` works while `aistudio.google.com` and `notebooklm.google.com` remain blocked.

This guide presents a **production-grade, zero-global-pollution solution** covering network egress, Xray domain routing, and Google account/payment diagnostics: **pair Cloudflare WARP in local proxy mode with Xray inbound SNI sniffing and outbound routing to unlock the Google AI suite cleanly.**

---

## 1. Root Cause: Why Does a US IP Still Get Blocked?

Google enforces multi-layered risk controls on its AI and developer endpoints. Blocks typically stem from four factors:

1. **Datacenter / ISP ASN Filtering**: IP ranges announced by most VPS providers and certain ISPs are tagged as Hosting/Datacenter or higher-risk ASNs. Even when physically located in the US, requests to `daily-cloudcode-pa.googleapis.com`, `aistudio.google.com`, or `notebooklm.google.com` can be rejected.
2. **Split Frontend and Backend API Domains**: Routing only the main web hostname is not enough. AI Studio queries `*.clients6.google.com` (e.g., `alkalimakersuite-pa.clients6.google.com`), while NotebookLM queries `*.googleapis.com` (e.g., `tailwind-pa.googleapis.com`, `notebooklm-pa.googleapis.com`), `labs.google`, and `notebooklm.google`. If the frontend exits via WARP while backend APIs still exit via the VPS native `direct` IP, requests fail.
3. **Auth and Egress IP Consistency**: Services like NotebookLM strictly verify session IP consistency. If the app domain exits via WARP while `accounts.google.com` or `gstatic.com` exits via the VPS native IP, access can still be denied.
4. **Account Country & Hidden Payments Profiles**: Google checks account-level country association, legacy Google Payments profiles, age eligibility, and Workspace policies alongside your network egress IP.

**Why avoid system-wide global WARP?**
Routing the entire VPS through WARP can disrupt SSH management routes and forces all outbound server traffic through shared Cloudflare IPs. **The optimal architecture** keeps the server's default routing untouched and steers only the target Google traffic through a local WARP SOCKS5 listener on `127.0.0.1:40000`.

---

## 2. End-to-End Network Topology

```text
Local Client (v2rayN / Clash TUN mode)
        ↓ VLESS Reality (Custom Port)
VPS Server (Dedicated Inbound Port)
        ↓
System xray.service (Inbound SNI Sniffing, routeOnly: true)
        │
        ├── Matches Google AI domains & ecosystem (with geosite:google fallback)
        │      ↓
        │   Forwarded to Local SOCKS5 (127.0.0.1:40000)
        │      ↓
        │   Cloudflare WARP (Local Proxy Mode)
        │      ↓
        │   Cloudflare Egress (loc=US, warp=on)
        │      ↓
        │   Antigravity / Gemini / AI Studio / NotebookLM (200 OK)
        │
        └── All Other Internet Traffic
               ↓
            Direct default VPS native egress
```

---

## 3. Pre-Flight Architecture Verification

Before editing any configuration, verify which process owns the client-facing proxy port (e.g., native `xray.service` vs. a 3x-ui / x-ui panel instance):

```bash
ss -lntp | grep -E ':(<YOUR_PORT>) '
```

Confirm the active process and configuration path (this guide uses `/usr/local/etc/xray/config.json`) so changes take effect on the right daemon.

---

## 4. Step 1: Install and Configure Cloudflare WARP (Local Proxy Mode)

Install the official `cloudflare-warp` client on the VPS (e.g., Ubuntu 24.04+) and configure it strictly as a loopback-only SOCKS5 proxy.

### 4.1 Install the Official Package

```bash
. /etc/os-release
install -d -m 0755 /usr/share/keyrings

# Import Cloudflare GPG Key
curl -4 -fsSL https://pkg.cloudflareclient.com/pubkey.gpg \
  | gpg --batch --yes --dearmor \
  -o /usr/share/keyrings/cloudflare-warp-archive-keyring.gpg

# Add repository
printf \
  'deb [signed-by=/usr/share/keyrings/cloudflare-warp-archive-keyring.gpg] https://pkg.cloudflareclient.com/ %s main\n' \
  "$VERSION_CODENAME" \
  > /etc/apt/sources.list.d/cloudflare-client.list

apt-get update -qq && apt-get install -y cloudflare-warp
```

### 4.2 Register and Enable Local Proxy Mode

> **Note**: Modern `warp-cli` releases require the explicit `--accept-tos` flag.

```bash
# Register a fresh client
warp-cli --accept-tos registration new

# Switch mode to local proxy (leaves default system routing untouched)
warp-cli --accept-tos mode proxy

# Bind SOCKS5 listener to local port 40000
warp-cli --accept-tos proxy port 40000

# Establish connection
warp-cli --accept-tos connect
```

### 4.3 Health and Egress Verification

```bash
# 1. Verify connection state (Expected: Connected / Network: healthy)
warp-cli --accept-tos --no-ansi status

# 2. Ensure port 40000 binds strictly to 127.0.0.1, never 0.0.0.0
ss -lntp | grep ':40000'
# Expected: LISTEN 0 1024 127.0.0.1:40000 0.0.0.0:*

# 3. Validate WARP egress location and state
curl --socks5-hostname 127.0.0.1:40000 https://www.cloudflare.com/cdn-cgi/trace
# Expected attributes: loc=US, warp=on

# 4. Verify target API TLS handshake
curl --max-time 15 --socks5-hostname 127.0.0.1:40000 -o /dev/null -sS -w '%{http_code}\n' https://daily-cloudcode-pa.googleapis.com/
# HTTP 404 is expected (missing bearer token), confirming network reachability
```

---

## 5. Step 2: Configure Xray SNI Sniffing and Outbound Routing

Create a timestamped backup before editing `/usr/local/etc/xray/config.json`:

```bash
cp -a /usr/local/etc/xray/config.json "/usr/local/etc/xray/config.json.bak.$(date -u +%Y%m%dT%H%M%SZ)"
```

### 5.1 Enable Inbound SNI Sniffing (`inbounds[].sniffing`)

Proxy clients often resolve DNS locally and send raw IP destinations to the VPS. Enable server-side `sniffing` on the inbound so Xray extracts the real hostname from the TLS Client Hello SNI:

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

- **`routeOnly: true`**: Uses the sniffed domain solely for `routing.rules` matching without rewriting the destination IP, preserving connection stability.

### 5.2 Add the WARP SOCKS Outbound (`outbounds`)

Keep your existing `direct` and `blocked` outbounds and add `warp-out`:

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

### 5.3 Configure Routing Rules (`routing.rules`)

Place the Google AI and ecosystem rule at the top of `routing.rules`:

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

#### Rule Breakdown & Trade-offs

Each entry targets a specific part of the Google AI stack:
- **Antigravity**: `full:daily-cloudcode-pa.googleapis.com` (and `cloudcode-pa.googleapis.com`, also covered by `domain:googleapis.com`).
- **Gemini & Developer APIs**: `domain:gemini.google.com`, `domain:ai.google.dev`, `domain:deepmind.google`, `domain:generativelanguage.googleapis.com`.
- **AI Studio & Backend**: `domain:aistudio.google.com`, `domain:clients6.google.com`, `domain:makersuite.google.com`.
- **NotebookLM & Backend**: `domain:notebooklm.google.com`, `domain:notebooklm.google`, `domain:labs.google`, `domain:googleapis.com`.
- **Full Consistency Fallback (`geosite:google`)**: Ensures auxiliary domains such as `accounts.google.com`, `gstatic.com`, and `googleusercontent.com` share the **exact same WARP egress IP** as the AI apps, eliminating session IP-mismatch blocks.

> **Trade-off on `geosite:google`**:
> - **Keep `geosite:google` (Recommended for NotebookLM / full Google AI suite)**: Most reliable for preventing hidden subdomain or auth-IP mismatches. Two minor trade-offs apply: (1) `www.google.com` search may occasionally show a one-time reCAPTCHA due to shared WARP IPs; (2) YouTube, Gmail, and Google Drive also traverse `127.0.0.1:40000` with minimal latency overhead. Non-Google traffic and other VPS services remain completely unaffected.
> - **Minimal Mode (Omit `geosite:google`)**: If you only use Antigravity or prefer direct egress for Google Search, remove `geosite:google` and keep only the explicit service domains above (add `"domain:accounts.google.com"` if needed for NotebookLM login consistency).

---

## 6. Step 3: Validate, Restart, and Verify End-to-End

Validate the JSON syntax before restarting Xray:

```bash
/usr/local/bin/xray run -test -config /usr/local/etc/xray/config.json
# Confirm: Configuration OK.
```

Restart and verify both services are active:

```bash
systemctl restart xray
systemctl is-active xray warp-svc
# Both should return: active
```

### Verification

1. **CLI & Journal Audit**: Connect your local client to the node, run a probe, and verify `[warp-out]` in the VPS logs:
   ```bash
   # Local workstation probe (replace port with your client's local SOCKS port)
   curl --max-time 15 --socks5-hostname 127.0.0.1:10808 -o /dev/null -sS -w '%{http_code}\n' https://daily-cloudcode-pa.googleapis.com/

   # Check Xray routing logs on the VPS
   journalctl -u xray --since '1 minute ago' --no-pager | grep warp-out
   ```
2. **Browser Incognito Test**: Always test Gemini, AI Studio, and NotebookLM in an **incognito/private window** to avoid stale cookies or cached region errors.

---

## 7. Step 4: Still Blocked After Routing Works? Account & Payments Audit

If an **incognito window** still shows "region not supported" after routing is verified, the block is usually caused by **Google account country lock** or a **conflicting Google Payments profile** (Google prioritizes account region metadata over IP alone). Walk through these four checks:

### 7.1 Isolate Network Routing vs. Account Lock (Signed-Out Incognito Test)

Open `https://aistudio.google.com/`, `https://gemini.google.com/`, or `https://notebooklm.google.com/` in an **incognito window without signing in**:
- **Blocked while signed out**: A related domain is still bypassing WARP or hitting an IP-consistency check. Verify `geosite:google` and `sniffing` are active, and confirm both `xray` and `warp-svc` are `active`.
- **Loads fine signed out, blocked immediately after sign-in**: This is **100% an account region or eligibility restriction**. Switching VPNs will not fix it; update the account settings below.

### 7.2 Check and Update Google Account Country Association

1. **Check current region**: Sign in and visit [policies.google.com/terms](https://policies.google.com/terms), then scroll to check the country/region currently associated with your account.
2. **Submit an official change request**:
   - Keep your proxy/VPN connected to the target supported region (e.g., United States).
   - Open the [Google Country Association Form](https://policies.google.com/country-association-form).
   - Select the target country (e.g., `United States`), choose `None of the above`, and explain in English that you need access to Gemini / AI Studio for work or study while residing in that region. Decisions usually arrive by email within a few hours to a few days.

### 7.3 Audit Hidden Google Payments Profiles

Even after updating your primary account country, an old payment profile tied to an unsupported region can still trigger blocks:
1. Open [pay.google.com](https://pay.google.com) → **Settings**.
2. Check **Country/Region** across **all** profiles (use the top dropdown if multiple profiles exist).
3. Close any legacy profile tied to an unsupported region at the bottom of Settings (propagation typically takes 3–7 days).

### 7.4 Verify Age and Workspace Restrictions

- **Age requirement**: Ensure your account birthday is 18+ at [myaccount.google.com/birthday](https://myaccount.google.com/birthday).
- **Account type**: Confirm you are not using a school or enterprise Google Workspace account with generative AI disabled by an administrator.
- **Developer forum**: For persistent account sync issues, check or report on [discuss.ai.google.dev](https://discuss.ai.google.dev/).

---

## 8. Operational Summary and Precautions

1. **Systematic Solution**: Resolving Google AI restrictions across Antigravity, Gemini, AI Studio, and NotebookLM requires three aligned layers: **clean WARP egress + Xray SNI sniffing & domain/`geosite:google` routing + incognito/account/Payments verification**.
2. **Strict Loopback Isolation**: Bind WARP only to `127.0.0.1:40000`. Never expose local proxy ports to `0.0.0.0`.
3. **Fast Rollback**: Keep timestamped `.bak` backups before every edit; restore, run `xray run -test`, and restart `xray.service` to revert in seconds.
4. **Legitimate Use**: This architecture is intended for developers with legitimate access who are constrained by datacenter or ISP network egress characteristics.
