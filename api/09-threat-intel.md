# Threat Intel Feed — API contract

> **Contract:** honeypot-contract · see root `VERSION`  
> **Client:** ≥ **4.5.61** poll/ACK · **≥ 4.9.7** normative `HP-INTEL-*` apply + reconcile  
> **API base:** `https://honeypot.yesnext.com.tr`  
> **Auth:** `Authorization: Bearer <token>` (agent API’de `?token=` yok)  
> **Cloud ingest:** [`../cloud/threat-intel-ingest.md`](../cloud/threat-intel-ingest.md)  
> **≠ local threat engine:** EventLog/skor/auto-block → [`../agent/threat-engine.md`](../agent/threat-engine.md) (`GET /api/threats/config`)  
> **Kural:** Client harici feed (Abuse.ch / CISA) **çekmez** — sadece bu bundle.

---

## GET /api/agent/threat-intel

```http
GET /api/agent/threat-intel?os=windows&client_version=4.9.7&since_version=2026.07.20.011
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
| **200** | Bundle kaydet (ProgramData), layer’ları uygula, **ACK gönder** |
| **304** | Cache tut; `last_check_at` yenile; expire/orphan reconcile (ACK yalnız FW değiştiyse) |
| **404/501/503** | Soft-fail; cache tut; sonraki poll |
| Diğer 4xx/5xx | Soft-fail; retry |

**ETag:** Response `ETag` header ve/veya body `etag`. Client meta +
`threat_intel_bundle.json` içinde saklar; sonraki istekte `If-None-Match`.

---

## POST /api/agent/threat-intel/ack

Apply **bittikten sonra** (yalnız “indirdim” yeterli değil):

```http
POST /api/agent/threat-intel/ack
Authorization: Bearer <token>
Content-Type: application/json

{
  "bundle_version": "2026.07.20.011",
  "applied_at": "2026-07-21T00:00:00Z",
  "stats": {
    "firewall_added": 5,
    "firewall_skipped": 2,
    "firewall_removed": 1,
    "ransomware_rules": 28,
    "process_watch": 3,
    "banners": 1,
    "errors": []
  }
}
```

| `stats` alanı | Anlam |
|---------------|--------|
| `firewall_added` | Yeni `HP-INTEL-*` kuralları |
| `firewall_skipped` | severity / allowlist / expire / cap / fail |
| `firewall_removed` | orphan veya policy-off purge |
| `errors[]` | Soft hatalar — dolu olsa bile ACK gönderilmeli |

Soft-fail if endpoint missing. Dashboard “Agent ACK” / fleet sync bundan beslenir.

---

## WebSocket: `threat_intel_updated`

Control WS (`api/03-control-websocket.md`) üzerinde cloud push:

```json
{ "v": 1, "t": "threat_intel_updated", "bundle_version": "2026.07.21.001", "etag": "…" }
```

**Client:** Mesaj gelince poll beklemeden `GET threat-intel` (ETag ile). Yoksa HTTP poll (~20 dk) yeter.

---

## Apply map (client ≥ 4.9.7)

| Layer | Davranış |
|-------|----------|
| `firewall_blocks` | **`HP-INTEL-<id>`** inbound+outbound (asla `HP-BLOCK-*` / AutoResponse 24h yolu değil) |
| `ransomware` | Merge extensions / process_names / cmdline — **lockdown yok** |
| `process_watch` | Soft match → alert |
| `kev_cves` / `hardening` / `ui_banners` | Log + dashboard/GUI |

### Firewall reconcile kuralları

- Yalnız `policy.auto_block_firewall == true` iken ekle; `false` → tüm `HP-INTEL-*` kaldır
- Severity ≥ `policy.intel_block_requires_severity_at_least` (genelde `high`)
- Bundle/policy allowlist + local `threats/config` whitelist → **asla** intel-block
- `max_firewall_rules_from_intel` aşımında en yüksek severity öncelikli; fazlası `firewall_skipped`
- `expires_at` dolmuş → ekleme / mevcut kuralı kaldır
- Bundle’dan düşen id → orphan `HP-INTEL-*` kaldır
- `clear_firewall` wipe → `HP-INTEL-*` de silinir ([`06-firewall-blocks.md`](./06-firewall-blocks.md))

Cache: `%ProgramData%\YesNext\CloudHoneypotClient\threat_intel_bundle.json`  
Meta: `threat_intel_meta.json` (`etag`, `last_check_at`, `applied`)

---

## Acceptance

- [ ] Bearer-only GET 200 + `bundle_version` + `layers`
- [ ] Aynı ETag ile GET → **304**
- [ ] `auto_block_firewall=true` → netsh’te `HP-INTEL-*` (in+out)
- [ ] `auto_block_firewall=false` → yeni HP-INTEL yok; mevcutlar temizlenir
- [ ] ACK sonrası dashboard sync satırında aynı `bundle_version` + `applied_at`
- [ ] `stats.firewall_added` / skipped / removed tutarlı
- [ ] WS `threat_intel_updated` → 20 dk poll beklemeden sync
- [ ] Client Abuse.ch/CISA’ya doğrudan istek **yok**
- [ ] Whitelist IP’ye HP-INTEL eklenmez
