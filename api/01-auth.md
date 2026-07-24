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

## POST /api/agent/rotate-token (contract **1.4.29**)

**Amaç:** Yerel `token.dat` yeniden üretildiğinde **yeni `clients` satırı açma**.  
Aynı `client_id` üzerinde `token` kolonunu güncelle → saldırı geçmişi, alert, blok, AccountClient, alias korunur.

### Ne zaman

| Durum | Doğru yol |
|-------|-----------|
| İlk kurulum / bilinmeyen host | `POST /api/register` |
| Aynı makinede token **yeniden üretildi** (identity v2, rekey, schema upgrade) | **`POST /api/agent/rotate-token`** sonra yeni token’ı diske yaz |
| Sadece `machine_id` aynı, token aynı kalmalı | `POST /api/register` → `reused: true` (token değişmez) |

**Yasak:** Eski token hâlâ biliniyorken bare `/register` ile yeni satır açmak (ghost server + geçmiş kaybı).

### Request

```http
POST /api/agent/rotate-token
Authorization: Bearer <old_token>   # opsiyonel; body.old_token yeterli
Content-Type: application/json
```

```json
{
  "old_token": "<current-uuid>",
  "new_token": "<new-uuid>",
  "machine_id": "{sha256-hardware-fingerprint}",
  "hwid": "{sha256-hardware-fingerprint}",
  "reason": "identity_v2"
}
```

| Alan | Zorunlu | Not |
|------|---------|-----|
| `old_token` | Evet* | *veya Bearer = eski token |
| `new_token` | Evet | Henüz başka `clients.token` olmamalı |
| `machine_id` / `hwid` | Önerilir | Cloud’da kayıtlıysa **eşleşmeli** (403 `machine_id_mismatch`) |
| `reason` | Hayır | `identity_v2` \| `rekey` \| `schema_upgrade` \| … |

### Response 200

```json
{
  "status": "ok",
  "ok": true,
  "rotated": true,
  "idempotent": false,
  "client_id": 57,
  "token": "<new-uuid>",
  "previous_token": "<old-uuid>",
  "machine_id": "…",
  "dashboard": "https://honeypot.yesnext.com.tr/dashboard?token=<new-uuid>",
  "message": "Token updated in place — attacks/alerts/account link preserved."
}
```

- **Idempotent:** Aynı `old→new` tekrar → `rotated: false`, `idempotent: true` (200).  
- Cloud `settings_json`: `previous_token`, `token_rotations[]`, sticky `explicit_presence` temizlenir.

### Errors

| HTTP | detail |
|------|--------|
| 422 | `old_token and new_token required` / `invalid_token_format` |
| 401 | `bearer_old_token_mismatch` |
| 403 | `machine_id_mismatch` |
| 404 | `old_token_not_found` |
| 409 | `new_token_in_use` (başka satırın token’ı) |

### Client algorithm (≥ **4.9.31**)

1. Generate `new_token` (uuid4) **in memory** — disk’e yazma.  
2. `POST /api/agent/rotate-token` with `old_token` + `new_token` + `machine_id` + `reason`.  
3. **Only on 200** → write `token.dat` / CHP2 binding with `new_token`.  
4. On 404: old already gone — if local already has `new_token` and heartbeat works, OK; else fall back to register reclaim.  
5. On 409: pick another `new_token` and retry once.  
6. Never leave local disk on `new_token` if cloud still expects `old_token`.

### Acceptance

- [x] Cloud: in-place `clients.token` update; `client_id` unchanged  
- [x] AccountClient / attacks / alias remain on same id  
- [ ] Client ≥4.9.31: rekey path uses rotate-token (not bare register)

---

## GET /api/client_status

Presence, `control_ws`, `agent_version`, sync ages, **`threat_intel`: { current, last_ack, in_sync }**.

---

## Agent path envanteri (Bearer)

| Grup | Path’ler |
|------|----------|
| Pulse | heartbeat, update-ip, client_status |
| Identity | **`/api/agent/rotate-token`** (in-place rekey, 1.4.29) |
| Attack / ports | `/api/attack`, open-ports, tunnel-* |
| Commands | pending, result, get |
| Firewall | pending-blocks, pending-unblocks, block-applied/removed, sync-rules |
| Threat engine | threats/config, alerts/*, health/report, events/batch, auto-block |
| Threat intel | `/api/agent/threat-intel`, `/ack` |
| Account | account-status, link, unlink-account |
| Remote HTTP | `/api/remote/*` |
| Lifecycle | `/api/alerts/lifecycle` |

Public: `/api/public/latest-release`, `/api/public/contract` — auth yok.

---

## Acceptance

- [ ] Aynı machine_id iki process → tek token  
- [x] Token rotate in-place (`/api/agent/rotate-token`) → aynı `client_id`  
- [ ] Client ≥4.9.28 fingerprint `machine_id`  
- [ ] Client ≥4.9.31 rekey uses rotate-token (not bare register)
- [ ] Bearer-only heartbeat/commands 200; query-only 401  
- [ ] Register 200 + `protection.block_rules` (RDP threshold 3)
