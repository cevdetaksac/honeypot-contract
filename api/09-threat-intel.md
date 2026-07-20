# Threat Intel Feed — API contract

> **Contract:** honeypot-contract · see root `VERSION`  
> **Client:** ≥ **4.5.61** · Daemon only (frontend does not poll)  
> **API base:** `https://honeypot.yesnext.com.tr`  
> **Auth:** `Authorization: Bearer <token>` (agent API’de `?token=` yok)  
> **Cloud ingest:** [`../cloud/threat-intel-ingest.md`](../cloud/threat-intel-ingest.md)  
> **≠ local threat engine:** EventLog/skor/auto-block → [`../agent/threat-engine.md`](../agent/threat-engine.md) (`GET /api/threats/config`)  
> **Kural:** Client harici feed (Abuse.ch / CISA) **çekmez** — sadece bu bundle.

---

## GET /api/agent/threat-intel

```http
GET /api/agent/threat-intel?os=windows&client_version=4.5.65&since_version=2026.07.20.011
Authorization: Bearer <token>
If-None-Match: "W/\"sha256:…\""   # or opaque etag from prior 200
```

| Query | Zorunlu | Açıklama |
|-------|---------|----------|
| `os` | hayır | `windows` (default) |
| `client_version` | hayır | SemVer — cloud min-version filtreleyebilir |
| `since_version` | hayır | Önceki `bundle_version` |

| Code | Client |
|------|--------|
| **200** | Bundle kaydet (ProgramData), layer’ları uygula, ACK gönder |
| **304** | Cache tut; `last_check_at` yenile |
| **404/501/503** | Soft-fail; cache tut; sonraki poll |
| Diğer 4xx/5xx | Soft-fail; retry |

**ETag:** Response `ETag` header ve/veya body `etag`. Sonraki istekte `If-None-Match`.

---

## POST /api/agent/threat-intel/ack

```http
POST /api/agent/threat-intel/ack
Authorization: Bearer <token>
Content-Type: application/json

{
  "bundle_version": "2026.07.20.011",
  "applied_at": "2026-07-21T00:00:00Z",
  "stats": {
    "firewall_added": 5,
    "ransomware_rules": 28,
    "process_watch": 3
  }
}
```

Soft-fail if endpoint missing. Cloud metrics / fleet health için.

---

## WebSocket: `threat_intel_updated`

Control WS (`api/03-control-websocket.md`) üzerinde cloud push:

```json
{ "v": 1, "t": "threat_intel_updated", "bundle_version": "2026.07.21.001", "etag": "…" }
```

**Client:** Mesaj gelince poll beklemeden `GET threat-intel` (ETag ile). Yoksa HTTP poll (~20 dk) yeter.

---

## Apply map (client)

| Layer | Davranış |
|-------|----------|
| `firewall_blocks` | `HP-INTEL-*` block (policy: severity ≥ high) |
| `ransomware` | Merge extensions / process_names / cmdline — **lockdown yok** (canary/VSS yerel otorite) |
| `process_watch` | Soft match → alert |
| `kev_cves` / `hardening` / `ui_banners` | Log + dashboard/GUI |

Cache: `%ProgramData%\YesNext\CloudHoneypotClient\threat_intel_bundle.json`

---

## Acceptance

- [ ] Bearer-only GET 200 + `bundle_version` + `layers`
- [ ] Aynı ETag ile GET → **304**
- [ ] ACK 2xx (veya soft-miss)
- [ ] (Opsiyonel) Control WS `threat_intel_updated` → client anında sync
- [ ] Client Abuse.ch/CISA’ya doğrudan istek **yok**
