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
  "machine_id": "{Windows-MachineGuid}",
  "hwid": "{Windows-MachineGuid}",
  "existing_token": null
}
```

**Upsert:** aynı `machine_id` → **aynı token** (`reused: true`). First-run → yeni token.

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

---

## POST /api/heartbeat

```json
{
  "status": "online",
  "ip": "1.2.3.4",
  "hostname": "HOST",
  "running": true,
  "system_context": {
    "agent_version": "4.5.65",
    "os": "Windows …"
  }
}
```

~60s. Offline eşiği cloud tarafında ~2 dk (presence stale).

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
