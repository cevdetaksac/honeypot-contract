# Firewall blocks & sync

> **Contract VERSION:** root `VERSION`  
> **Auth:** Bearer  
> **İlgili:** [`../agent/register-protection.md`](../agent/register-protection.md) · threat-intel `HP-INTEL-*` → [`09-threat-intel.md`](./09-threat-intel.md)

Üç kaynak (karıştırma):

| Kaynak | Kural adı | Nasıl gelir |
|--------|-----------|-------------|
| Dashboard / cloud eval (`notification_rules`) | `HP-BLOCK-*` | `pending-blocks` / `block_ip` komutu |
| Local threat engine auto-block | `HP-BLOCK-*` | Agent netsh + `POST /api/alerts/auto-block` |
| Threat intel bundle | `HP-INTEL-*` | `GET /api/agent/threat-intel` apply |

`ip_or_cidr`: tek IP, CIDR, veya `country:XX` (cloud destekliyorsa).

---

## Agent HTTP

| Method | Path | Ne |
|--------|------|-----|
| GET | `/api/agent/pending-blocks` | `status=pending` → uygula |
| POST | `/api/agent/block-applied` | `{ block_ids[] }` veya `ip` + `rule_name` |
| GET | `/api/agent/pending-unblocks` | Kaldırılacaklar |
| POST | `/api/agent/block-removed` | ACK |
| POST | `/api/agent/sync-rules` | Canlı FW envanteri → cloud |
| POST | `/api/premium/clear-all-blocks` | Dashboard; komut `clear_firewall` push |

### block-removed ACK (client ≥4.8.5)

Body (agent → cloud):

```json
{ "token": "…", "block_ids": [2582, 2649], "ips": ["50.16.16.211", "178.62.3.223"], "ip": "…" }
```

- `block_ids`: pending-unblocks öğelerindeki `id` (tercihen int).
- `ips` / `ip`: aynı satırların IP'leri — **zorunlu pratik alan**. Canlı 4.8.4'te cloud
  yalnız `block_ids` ile çoğu zaman `{"updated":0,"status":"ok"}` döndü; dashboard
  "Kaldırılıyor…" durumunda kaldı. `ip` ile ACK `updated>0` üretiyor.
- Client ≥4.8.5 her iki alanı birden gönderir; `updated=0` ise IP başına retry yapar.
- **Cloud implemented (honeypot.yesnext.com.tr, 2026-07-21):** `block_ids` **ve**
  `ips`/`ip` birlikte değerlendirilir (either/or DEĞİL) — 4.8.4'teki
  `if not block_ids` erken-çıkış hatası giderildi. Eşleşen satırlar `id` ile
  dedupe edilir; `updated` = kapatılan benzersiz satır sayısı. `pending` durumu
  da kapatılır (agent firewall'dan kaldırdığını bildiriyorsa henüz uygulanmamış
  kural da düşer). Yanıt ayrıca `removed_ips[]` döndürür. `pending-unblocks`
  GET salt-okunur — öğeler yalnız `block-removed` ACK'iyle `removed` olur, ACK
  gelene kadar kuyrukta kalır (retry mümkün). Cleanup yalnız `remove_pending`'e
  taşır, satırı silmez.

### sync-rules (özet)

Agent `netsh` / HP-BLOCK taraması → JSON blocks listesi. Cloud SoT envanter için.

### clear_firewall (komut)

Params: `wipe_all_honeypot_rules` (default true), `ips[]`, `reason`.  
Sonra sync-rules + isteğe clear-data scopes.

---

## Premium notification rules

Dashboard CRUD: `GET/POST/PUT/DELETE /api/premium/rules`  
Seed: `POST /api/premium/rules/seed-defaults` (dashboard login).  
Yeni client: register otomatik seed ([`register-protection.md`](../agent/register-protection.md)).

Agent eşikleri **local** `protection.block_rules` + cloud worker pending-blocks ile çift katman olabilir; idempotent block.

---

## Acceptance

- [ ] pending-block → HP-BLOCK → block-applied 200  
- [ ] clear_firewall wipe → sync boş / cloud clear  
- [ ] INTEL kuralları BLOCK ile isim çakışması yok
