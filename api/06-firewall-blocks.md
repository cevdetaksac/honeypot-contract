# Firewall blocks & sync

> **Contract VERSION:** root `VERSION`  
> **Auth:** Bearer  
> **İlgili:** [`../agent/register-protection.md`](../agent/register-protection.md) · threat-intel `HP-INTEL-*` → [`09-threat-intel.md`](./09-threat-intel.md)

Üç kaynak (karıştırma):

| Kaynak | Kural adı | Nasıl gelir |
|--------|-----------|-------------|
| Dashboard / cloud eval (`notification_rules`) | `HP-BLOCK-*` | `pending-blocks` / `block_ip` komutu |
| Local threat engine auto-block | `HP-BLOCK-*` | Agent netsh + `POST /api/alerts/auto-block` |
| Threat intel bundle | `HP-INTEL-*` | `GET /api/agent/threat-intel` apply (client ≥ **4.9.7**) |

`ip_or_cidr`: tek IP, CIDR, veya `country:XX` (cloud destekliyorsa).

### Whitelist — asla engelleme (client ≥ 4.9.7)

`threats/config.whitelist_ips` / `whitelist_subnets`:

- `block_ip` (auto / komut / intel) whitelist IP’ye **uygulanmaz**.
- Whitelist güncellenince veya block denemesi whitelist’e çarparsa client
  mevcut `HP-BLOCK-*` **ve** eşleşen `HP-INTEL-*` kurallarını **derhal siler**
  (`enforce_whitelist_unblocks`).
- Bare `successful_logon` zaten HP-BLOCK üretmez → [`../agent/threat-engine.md`](../agent/threat-engine.md).

---

## Whitelist invariant (contract ≥1.4.11)

`threats/config.whitelist_ips` / `whitelist_subnets` (ve silent-hours WL) içindeki
IP **asla** engelli kalmamalı.

| Kim | Ne yapar |
|-----|----------|
| Client | Auto-block / manual block öncesi WL kontrolü; WL ise HP-BLOCK yok |
| Cloud | `POST /api/alerts/auto-block` → `rejected`/`whitelisted`; manuel block-rule create → reject |
| Cloud lift | BlockRule `remove_pending` + AutoBlock pasif + `unblock_ip` komutu + `GET pending-unblocks` |
| Client | `unblock_ip` **ve/veya** pending-unblocks → netsh sil → `POST block-removed` (`block_ids` **+** `ips`) |

Reconcile tetikleri (cloud): whitelist-add, `GET/POST /api/threats/config`,
`sync-rules`, cleanup sweep.

Control WS (opsiyonel push):

```json
{ "v": 1, "t": "pending_unblocks_updated", "ips": ["84.44.42.18"], "reason": "whitelist_guard" }
```

Agent: mesaj gelince hemen `GET /api/agent/pending-unblocks` (poll bekleme).

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

Agent `netsh` / `HP-BLOCK-*` (+ envanterde `HP-INTEL-*` görünürlüğü) taraması →
JSON blocks listesi. Cloud SoT envanter için. İsim önekleri karıştırılmaz.

### clear_firewall (komut)

Params: `wipe_all_honeypot_rules` (default true), `ips[]`, `reason`.  
Wipe **hem** `HP-BLOCK-*` **hem** `HP-INTEL-*` (ve legacy `HONEYPOT_*`) siler.
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
- [ ] clear_firewall wipe → sync boş / cloud clear (HP-INTEL dahil)  
- [ ] INTEL kuralları BLOCK ile isim çakışması yok (`HP-INTEL-*` ≠ `HP-BLOCK-*`)
- [ ] Whitelist IP → block yok; engelli ise anında kaldırılır (client ≥4.9.7)
- [ ] Whitelist IP auto-block denemesi → cloud `rejected`/`whitelisted` + client FW’de kural yok
- [ ] Whitelist’e eklenen önceden bloklu IP → `remove_pending` + `unblock_ip` / pending-unblocks → `block-removed` ACK `updated>0`
- [ ] Çıplak `successful_logon` auto-block → cloud `successful_logon_no_autoblock` reject
