# Register → protection.block_rules

> **Contract VERSION:** root `VERSION`  
> **API:** `POST /api/register` (public) + `GET /api/threats/config` (Bearer)  
> **Min client:** register body apply — sprint (TBD fleet); config poll zaten var

---

## Amaç

Cloud her register/upsert’ta varsayılan fail→block katmanlarını seed eder ve agent’a iletir.  
Örnek: gerçek RDP’de **30 dk / 3 fail → `block_ip` + alert**.

---

## Kararlar (open questions kapatıldı)

| Soru | Karar |
|------|--------|
| Register her çağrıda `protection`? | **Evet** — first-run ve reuse |
| `threats/config` altında da? | **Evet** — `protection.block_rules` (poll SoT) |
| Apply sırası | Register snapshot → sonra config/rules poll üzerine yazar |

---

## `protection.block_rules` öğesi

```json
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
}
```

Cloud seed (idempotent `default_*` notification_rules):

| id | service | threshold | window_seconds |
|----|---------|-----------|----------------|
| `rdp-fail-3` | RDP | 3 | 1800 |
| `ssh-fail-3` | SSH | 3 | 1800 |
| `ftp-fail-3` | FTP | 3 | 1800 |
| `mssql-fail-3` | MSSQL | 3 | 1800 |
| `mysql-fail-3` | MYSQL | 3 | 1800 |
| `network-fail-10` | Network | 10 | 1800 |

Register ekstra alanlar (bilgi): `defaults_applied`, `created`, `skipped`, `threat_config_ready`, `hint`.

---

## Client

1. Register 200 → `ThreatEngine.update_block_rules(protection.block_rules)` (varsa).  
2. Yoksa / boş → local defaults; regressiyon yok.  
3. `GET /api/threats/config` → aynı şema ile üzerine yaz (SoT).  
4. Bait honeypot capture ≠ `failed_auth` sayacı ([`attacks-and-services.md`](./attacks-and-services.md)).

---

## Acceptance

- [ ] Register → en az `rdp-fail-3` threshold 3 / 1800s / `block_ip`  
- [ ] Reuse → kurallar hâlâ dolu (`skipped`≥0)  
- [ ] Client runtime’da cloud `id`’leri görünür  
- [ ] 3 fail window → HP-BLOCK + alert
