# Auth, register, heartbeat, status

> **Contract VERSION:** root `VERSION`  
> **API:** `https://honeypot.yesnext.com.tr`  
> **Client:** Bearer only (`legacy_token_query=false`). Cloud agent API’de `?token=` → deprecated/reject.

---

## Token çözüm sırası (cloud)

1. `Authorization: Bearer <token>`  
2. JSON body `token` (POST uyumluluk)  
3. Query `?token=` — **kapatıldı / reddedilir** (dashboard deep-link hariç)

WS: Bearer header veya `Sec-WebSocket-Protocol: bearer, <token>`.

---

## POST /api/register (public)

```json
{
  "server_name": "HOST (1.2.3.4)",
  "ip": "1.2.3.4",
  "machine_id": "{sha256-hardware-fingerprint}",
  "hwid": "{sha256-hardware-fingerprint}",
  "machine_guid": "{Windows-MachineGuid}",
  "existing_token": null
}
```

**`machine_id` / `hwid` (client ≥ 4.9.28):** SHA-256 over
`v2|{MachineGuid}|{sorted-MACs}|{SMBIOS-UUID}|{C-volume-serial}`.
Not raw MachineGuid alone — unsysprep’d VM clones often share MachineGuid and
would otherwise upsert to the **same** agent token.

**`machine_guid`:** additive telemetry (optional). Cloud may store it; upsert key
remains `machine_id`.

**Upsert:** aynı `machine_id` → **aynı token** (`reused: true`). First-run → yeni token.

**Clone / shared token:** Client binds `token.dat` (CHP2) + `device_binding.json`
to the fingerprint. Mismatch → quarantine local identity + fresh `/register`.
One-time schema v2 upgrade also re-enrolls under the fingerprint (re-link Account).

**200:**

```json
{
  "status": "ok",
  "token": "<uuid>",
  "client_id": 123,
  "machine_id": "…",
  "reused": false,
  "dashboard": "https://honeypot.yesnext.com.tr/dashboard?token=…",
  "protection": { "block_rules": [ "…" ] }
}
```

`protection` → [`../agent/register-protection.md`](../agent/register-protection.md).  
Token: `%ProgramData%\YesNext\CloudHoneypotClient\token.dat` — immutable; decrypt fail → otomatik re-register **yok**.  
Hardware bind: `device_binding.json` + CHP2 fingerprint in `token.dat` (client ≥ 4.9.28).

---

## POST /api/heartbeat

```json
{
  "status": "online",
  "ip": "1.2.3.4",
  "hostname": "HOST",
  "running": true,
  "system_context": {
    "agent_version": "4.9.1",
    "os": "Windows …",
    "heartbeat_proof": {
      "version": 1,
      "issued_at": "2026-07-22T01:00:00.000000Z",
      "algorithm": "hmac-sha256",
      "signature": "<64-hex>",
      "observe": true,
      "enforce": false
    }
  }
}
```

~60s. Offline eşiği cloud tarafında ~2 dk (presence stale).

### Optional signed heartbeat observe (contract 1.4.5 — RES-103)

`system_context.heartbeat_proof` is additive. Missing = legacy. Cloud **soft-
accepts** (store/metrics optional); it must **not** reject presence for
missing/invalid/stale proofs. Client flag `security.signed_heartbeat_observe`
(default **off**). Local verify helpers exist; Guardian reject-stale is **not**
wired.

Algorithm (must match client `client_resilience_p1`):

```text
key = SHA256("{token}|{hostname.lower()}|yesnext-heartbeat-v1")   // raw 32 bytes
msg = "v1|{hostname.lower()}|{status}|{1|0}|{issued_at}"
signature = HMAC-SHA256(key, msg).hexdigest()
```

`issued_at` is used verbatim (UTC ISO-8601, may include microseconds + `Z`).
Suggested soft verify windows (observe metrics only): max age 300s; clock skew
tolerance −30s. Distinct from command HMAC (`yesnext-chp-v1`).

---

## POST /api/update-ip

```json
{ "ip": "1.2.3.4" }
```

Aynı IP peer hard-delete **yok** (soft-archive).

---

## GET /api/client_status

Presence, `control_ws`, `agent_version`, sync ages, **`threat_intel`: { current, last_ack, in_sync }**.

---

## Agent path envanteri (Bearer)

| Grup | Path’ler |
|------|----------|
| Pulse | heartbeat, update-ip, client_status |
| Attack / ports | `/api/attack`, open-ports, tunnel-* |
| Commands | pending, result, get |
| Firewall | pending-blocks, pending-unblocks, block-applied/removed, sync-rules |
| Threat engine | threats/config, alerts/*, health/report, events/batch, auto-block |
| Threat intel | `/api/agent/threat-intel`, `/ack` |
| Account | account-status, link |
| Remote HTTP | `/api/remote/*` |
| Lifecycle | `/api/alerts/lifecycle` |

Public: `/api/public/latest-release`, `/api/public/contract` — auth yok.

---

## Acceptance

- [ ] Aynı machine_id iki process → tek token  
- [ ] Bearer-only heartbeat/commands 200; query-only 401  
- [ ] Register 200 + `protection.block_rules` (RDP threshold 3)
