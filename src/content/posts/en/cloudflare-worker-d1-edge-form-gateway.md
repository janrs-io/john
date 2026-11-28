---
title: "Building a Resilient Edge Form Gateway with Cloudflare Worker and D1: Anti-Spam, Durable Buffering, and Mutual HMAC Pull"
author: "John Yang"
lang: en
pubDatetime: 2026-11-28T09:30:00+08:00
featured: true
tags:
  - Cloudflare
  - Architecture
  - Security
  - Serverless
  - Go
description: "A framework-agnostic edge architecture for public web forms: use Cloudflare Worker as a zero-trust edge gateway to block bots, buffer submissions durably in Cloudflare D1 with a lease-based queue, and pull records safely from your core backend using mutual HMAC-SHA256 signatures and 300s replay protection."
---

Whether you run a static site (Astro, Next.js, Hugo), a corporate landing page, or WordPress, any public "Contact Us" or "Request a Quote" form eventually runs into the same architectural challenge: **how do you reliably and securely get public form submissions into your core backend without losing leads?**

Pointing browser `POST` requests directly at your core backend API introduces four classic failure modes:

1. **Data Loss During Downtime**: If your core backend is restarting, deploying, or hosted behind an intermittent private network link, visitor submissions fail immediately and leads are lost.
2. **Untrusted Clients & Secret Exposure**: Browsers cannot hold backend API secrets, and you can never trust client-supplied identifiers like `tenant_id` or `site_id`.
3. **Public Endpoint Abuse**: Exposed form endpoints attract botnets, spam injection, rapid-fire duplicate submissions, and oversized payload attacks that hit your primary database directly.
4. **Frontend Stack Coupling**: Different sites (Astro, React, WordPress, plain HTML) need a simple, standardized HTTP JSON contract independent of backend frameworks.

This article presents a **generic, production-tested edge architecture**: **use Cloudflare Worker as a zero-trust edge gateway, Cloudflare D1 as a durable buffer queue, and have your core backend pull and explicitly ACK records over a mutual HMAC-SHA256 channel.**

---

## 1. Architecture Overview: Push to Edge, Pull to Core

Instead of "browser pushes directly to core backend," the gateway decouples ingestion from processing via **Push to Edge, Pull from Edge**:

```text
┌───────────────────────────┐
│  Any Public Site / Form   │  (Astro / Next.js / WordPress / Plain HTML)
│  Holds public site_key    │
└─────────────┬─────────────┘
              │ HTTPS POST /v1/submissions (JSON, <= 32KB)
              ▼
┌───────────────────────────────────────────────────────────┐
│                   Cloudflare Worker                       │
│  1. Origin allowlist + 32KB payload size cap              │
│  2. Honeypot trap + IP/email rate limiting                │
│  3. Enrich with edge metadata (cf_country, cf_asn, IP)    │
│  4. Assign unique edge_id and persist to D1               │
└─────────────┬─────────────────────────────────────────────┘
              │ Durable Edge Storage (SQLite)
              ▼
┌───────────────────────────────────────────────────────────┐
│                    Cloudflare D1                          │
│  State machine: pending -> leased (90s) -> acked / dead   │
└─────────────▲─────────────────────────────────────────────┘
              │ Scheduled / triggered pull (Mutual HMAC + 300s window)
              │ GET /internal/pull  -->  POST /internal/ack
┌─────────────┴─────────────┐
│   Core Backend (Go/Node)  │
│  1. Verify edge signature │
│  2. Idempotent insert     │
│  3. ACK only after commit │
└───────────────────────────┘
```

### Why Pull-Based Edge Buffering Beats Direct Webhooks
- **Zero Lead Loss**: Even if your core backend is offline for hours during maintenance, submissions are durably committed to Cloudflare D1 in milliseconds and drained automatically once the backend recovers.
- **Zero Public Attack Surface on Core**: If your backend runs inside a private VPC or only allows outbound connections, you never need to expose inbound webhook ports to the public internet.
- **Natural Spike Absorption**: Traffic bursts and bot probes are absorbed or rejected at Cloudflare's edge, while your backend consumes buffered records at its own steady pace.

---

## 2. Zero-Trust Edge Defenses (Worker Layer)

Public form endpoints must follow one rule: **never trust any identity claim or anti-spam signal supplied by the browser.**

Execute these five checks in order at the Worker boundary:

1. **Origin & Method Enforcement**: Allow only `POST` and `OPTIONS` preflight requests. Validate the `Origin` header against a strict per-site allowlist.
2. **Hard Payload Cap**: Reject requests exceeding `32KB` with `413 Payload Too Large` before parsing JSON to protect Worker memory.
3. **Server-Side Identity Mapping**: The browser sends only a public `site_key`. The Worker or core backend maps `site_key` to internal site/workspace IDs. Any `tenant_id` or internal IDs in the request body are stripped or rejected.
4. **Honeypot & Behavioral Telemetry**:
   - Include a visually hidden field (e.g., `website_url`) that human users never fill but automated bots do. When triggered, **do not return an explicit error**—silently quarantine it as `spam` or return a neutral `202 Accepted` so attackers cannot probe and tune their scripts.
   - Optionally collect lightweight signals (`fill_duration_ms`, `is_pasted`) alongside Cloudflare's trusted `request.cf` attributes (`country`, `city`, `asn`). Quarantine suspected spam rather than hard-deleting it immediately so false positives remain recoverable.
5. **Rate Limiting**: Enforce per-IP and per-email sliding-window limits to block accidental double-submits and brute-force floods.

---

## 3. D1 Buffer Queue & Lease State Machine

Cloudflare D1 is distributed SQLite at the edge. To turn it into a reliable buffer queue without duplicate processing or stuck jobs, use **leases + retry counters + explicit ACKs**.

### 1. D1 Schema

```sql
CREATE TABLE IF NOT EXISTS buffered_submissions (
  edge_id        TEXT PRIMARY KEY,          -- Worker-generated UUIDv7/ULID
  site_key       TEXT NOT NULL,
  payload_json   TEXT NOT NULL,             -- Sanitized payload + CF metadata
  status         TEXT NOT NULL DEFAULT 'pending', -- pending | leased | acked | dead
  lease_token    TEXT,                      -- Per-batch lease token
  lease_until    INTEGER,                   -- Unix timestamp (seconds)
  attempt_count  INTEGER NOT NULL DEFAULT 0,
  last_error     TEXT,
  created_at     INTEGER NOT NULL,
  acked_at       INTEGER
);

CREATE INDEX IF NOT EXISTS idx_buffered_status_lease
  ON buffered_submissions (status, lease_until, created_at);
```

### 2. State Transitions
- **`pending -> leased`**: On a pull request, the Worker atomically claims up to N rows (e.g., 50) where `status = 'pending'` or `status = 'leased' AND lease_until < now`, setting a **90-second `lease_until`** and a random `lease_token`.
- **`leased -> acked`**: Only after the core backend commits the record to its primary database (e.g., PostgreSQL) does it call `/internal/ack` with `edge_id` and `lease_token` to mark the row `acked`.
- **Crash Recovery & Idempotency**: If the backend crashes mid-batch, the 90-second lease expires and the next pull re-delivers the item. The backend enforces a unique constraint on `edge_id` (`ON CONFLICT (edge_id) DO NOTHING`), achieving **at-least-once delivery with end-to-end idempotency**.
- **Dead-Letter Quarantine (`dead`)**: After `attempt_count >= 10` failures, move the row to `dead` with `last_error` so a single poison message never blocks the queue.

---

## 4. Mutual HMAC-SHA256 Authentication & Replay Protection

The `/internal/pull` and `/internal/ack` endpoints between your backend and the Worker must never rely on a bare static Bearer token alone.

### Canonical Signing Scheme
The backend attaches three headers to every internal request:
- `X-Timestamp`: Current UTC Unix timestamp in seconds
- `X-Nonce`: Random 16-byte hex string
- `X-Signature`: HMAC-SHA256 over the canonical request string using a shared `RELAY_SECRET`

```text
{HTTP_METHOD}\n{REQUEST_PATH}\n{X-Timestamp}\n{X-Nonce}\n{SHA256_HEX(BODY)}
```

The Worker enforces three checks:
1. **Replay Window**: Reject requests where `|now - X-Timestamp| > 300` seconds with `401 Unauthorized`.
2. **Constant-Time Verification**: Verify the signature via Web Crypto (`crypto.subtle.verify`) to prevent timing side-channels.
3. **Mutual Response Signing (Recommended)**: Have the Worker sign its JSON pull response with the same `RELAY_SECRET` so the backend can verify payload integrity end-to-end.

```ts
// Constant-time HMAC-SHA256 verification in Cloudflare Worker
async function verifyHmacSignature(
  secret: string,
  canonical: string,
  signatureHex: string,
): Promise<boolean> {
  const encoder = new TextEncoder();
  const key = await crypto.subtle.importKey(
    "raw",
    encoder.encode(secret),
    { name: "HMAC", hash: "SHA-256" },
    false,
    ["verify"],
  );

  const sigBytes = new Uint8Array(
    signatureHex.match(/.{1,2}/g)?.map((b) => parseInt(b, 16)) ?? [],
  );

  return crypto.subtle.verify("HMAC", key, sigBytes, encoder.encode(canonical));
}
```

---

## 5. Versioned Form Schemas Without `ALTER TABLE`

Form fields evolve frequently across landing pages and product lines. Adding a new relational database column for every custom field quickly becomes unmaintainable.

A cleaner pattern is **Fixed Core Columns + Versioned Custom Fields**:
1. **Fixed Core Columns**: Store universal fields (`contact_name`, `email`, `message`, `privacy_consent`) in strongly typed SQL columns for fast indexing and querying.
2. **Immutable Versioned Schemas**: Publish each form revision as an immutable `form_version` (e.g., `v1`, `v2`) and store `custom_fields` alongside `form_version`.
   - Updating field labels, types, or dropdown options creates a new version without mutating historical submissions, ensuring old records always render with their original field definitions.
   - Enforce strict limits (e.g., max 30 custom fields per form, plain-text only—no raw HTML/JS) and sanitize values on export to prevent XSS and CSV formula injection (`=`, `+`, `-`, `@` prefixes).

---

## 6. Operational Best Practices

1. **Scheduled D1 Pruning**: Use a Cloudflare Worker `scheduled` Cron Trigger to purge `acked` rows older than 7–30 days, keeping D1 storage compact and queries fast.
2. **End-to-End Traceability**: Generate a `request_id` at the Worker edge and persist both `request_id` and `edge_id` into your core database so any submission can be traced across browser, Worker logs, D1, and backend.
3. **Graceful Frontend UX**: If D1 experiences a transient write error, return `503` so the frontend preserves the visitor's typed message and prompts a retry instead of clearing the form.

---

## Summary

Combining **Cloudflare Worker + D1** turns a fragile public form endpoint into a **zero-trust edge ingestion layer and durable message buffer**. Your public sites stay fast and framework-agnostic, bots and abuse are stopped at the edge, and your core backend pulls verified, idempotent submissions on its own schedule without risking lost leads.
