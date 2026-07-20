# Register → protection.block_rules

> **Contract VERSION:** see root `VERSION`  
> **API:** `https://honeypot.yesnext.com.tr`  
> **Auth (register):** public — no Bearer (token mint)  
> **Status:** Contract defined — client apply path may still use `GET` block-rules; align to this payload.

---

## Amaç

Agent ilk kayıt / upsert (`POST /api/register`) yanıtında **koruma eşik kurallarını** alsın.
Örnek: honeypot bait kapalıyken gerçek RDP’de **30 dk içinde 3 fail → firewall block + alert**.

Client harici listeleri hardcode etmek yerine (veya default’un üstüne) cloud’dan gelen
`protection.block_rules` uygular.

---

## `POST /api/register` response (ek alan)

Mevcut alanlara ek:

```json
{
  "status": "ok",
  "token": "<uuid>",
  "machine_id": "<guid>",
  "protection": {
    "block_rules": [
      {
        "id": "rdp-fail-3",
        "name": "RDP brute force",
        "enabled": true,
        "service": "RDP",
        "event": "failed_auth",
        "threshold": 3,
        "window_seconds": 1800,
        "action": "block_ip",
        "alert": true,
        "severity": "high"
      },
      {
        "id": "ssh-fail-3",
        "name": "SSH brute force",
        "enabled": true,
        "service": "SSH",
        "event": "failed_auth",
        "threshold": 3,
        "window_seconds": 1800,
        "action": "block_ip",
        "alert": true,
        "severity": "high"
      }
    ]
  }
}
```

| Alan | Zorunlu | Açıklama |
|------|---------|----------|
| `protection` | hayır* | Yoksa client `DEFAULT_BLOCK_RULES` kullanır |
| `protection.block_rules` | * | Dizi; boş dizi = defaults’a düş |
| `id` | evet | Stabil kural id |
| `threshold` | evet | Örn. `3` |
| `window_seconds` | evet | Örn. `1800` (30 dk) |
| `action` | evet | `block_ip` (MVP) |
| `service` | önerilir | `RDP` / `SSH` / `FTP` / `MSSQL` / `MYSQL` / `Network` / `*` |

\* Cloud production’da `protection` göndermesi beklenir (fleet tutarlılığı).

---

## Client davranışı

1. Register (veya token refresh upsert) yanıtında `protection.block_rules` varsa → `ThreatEngine.update_block_rules(rules)`.
2. Sonraki sync: `GET /api/…/block-rules` (veya threats/config) **üzerine yazar** (SoT cloud).
3. Kural: aynı `service` + `failed_auth` sayacı `threshold`’a ulaşınca `action` + urgent alert.
4. Honeypot bait credential capture ayrı kanaldır; bu kurallar **gerçek** auth fail event’leri içindir (`api/06-firewall-blocks.md`).

---

## Open questions

- [ ] Register her çağrıda mı yoksa sadece mint’te mi `protection` döner? (öneri: her 200 register/upsert)
- [ ] `protection` ayrıca `GET /api/agent/threats/config` altında da mı gelir? (öneri: evet, tek SoT)

---

## Acceptance

- [ ] Cloud register 200 body’sinde `protection.block_rules` en az bir RDP kuralı (`threshold: 3`)
- [ ] Yeni client register sonrası ProgramData / runtime’da kural listesi defaults’tan farklıysa cloud id’leri görünür
- [ ] 3 fail (window içinde) → `HP-BLOCK-*` + alert (test host)
- [ ] `protection` yok → client regressiyon yok (defaults)
