# Defense Policy — Cloud implementation (tiered response)

> **Contract VERSION:** root `VERSION` (**1.4.19**)  
> **Status:** Normative for **cloud + dashboard** (client apply target ≥ **4.9.17**)  
> **Planning SoT:** [`../ROADMAP_TIERED_DEFENSE.md`](../ROADMAP_TIERED_DEFENSE.md)  
> **Related:** [`../agent/threat-engine.md`](../agent/threat-engine.md) · [`../api/03-control-websocket.md`](../api/03-control-websocket.md) · [`../agent/ransomware-shield.md`](../agent/ransomware-shield.md) · [`../agent/network-guard.md`](../agent/network-guard.md) · [`../agent/defense-policy-client.md`](../agent/defense-policy-client.md)

Bu belge **cloud/dashboard’un yapması gerekenleri** listeler.  
Client motor aksiyon primitiflerini uygular; **politika zekâsı, onboarding ve operatör UI** cloud’dadır.

---

## 0. Ürün modeli (özet)

| İlke | Anlam |
|------|--------|
| **Algı her zaman açık** | Observe / Balanced / Paranoid — soft + kırmızı **uyarılar** üretilir |
| **Agresif tepki moda bağlı** | Kill/Q, suspend, isolate yalnız matrix + seçilen preset izin verirse |
| **Kurulum default = observe** | Panik yok; 2–3 gün (yapılandırılabilir) sonra otomatik **balanced** |
| **Dashboard yönetir** | Preset, auto-promote günü, promote kilidi, armed isolate |
| **Anti-bait ayrı** | Observe→balanced onboarding ≠ tamper/canary→paranoid/isolate |

---

## 0.1 Cloud invariants (asla ihlal etme)

1. **Observe / Balanced default’ta ağ izolasyonu yok.**  
   `auto_isolate_network` yalnız `paranoid` + müşteri arm / confirm.  
2. **Tek canary / tek VSS intent → isolate komutu kuyruğa yazılmaz.**  
3. **`under_attack` yalnız kırmızı doğrulanmış olaylarda** (canary write, VSS wipe intent, operatör containment). Soft `network_surface_changed` / `suspicious_session` (info) açmaz.  
4. **N alert ≠ otomatik escalate / isolate** (anti-bait / fatigue).  
5. **Policy imza bozukluğu için cloud “agresif moda geç” demez**; client LKG/observe — cloud tamper alert’i soft-urgent gösterir, isolate push etmez.  
6. Stolen token ile “daha agresif matrix” → yine client hard-reject (balanced isolate); mutating komutlar HMAC + destructive confirm.  
7. Soft surface inform (1.4.17): mavi banner; Accept / optional Disable — panik yok.  
8. **Auto-promote yalnız observe → balanced** (asla paranoid / asla isolate_armed açmaz).

---

## 1. Veri modeli

### 1.1 Persist (host veya fleet)

| Alan | Tip | Not |
|------|-----|-----|
| `defense_policy` | `observe` \| `balanced` \| `paranoid` | Preset adı |
| `defense_policy_version` | string | Örn. `1.4.19-def-1` — ajan pull karşılaştırması |
| `defense_rules` | object | Event → action matrisi |
| `defense_rules_sig` | string | Cloud HMAC (emit anında) |
| `isolate_armed` | bool | Paranoid ağ izolasyonu müşteri onayı (default **false**) |
| `policy_updated_at` | ISO-8601 | Audit |
| `observe_started_at` | ISO-8601 | İlk observe’a giriş / kurulum |
| `observe_auto_promote_days` | int | Default **3** (dashboard 1–14; `0` = auto-promote kapalı) |
| `observe_auto_promote_enabled` | bool | Default **true** (yeni host) |
| `defense_policy_locked` | bool | Operatör “bu host’ta kal” — auto-promote yok |
| `policy_promote_at` | ISO-8601 \| null | Planlanan balanced yükseltme (audit) |
| `policy_user_set` | bool | Kullanıcı/dashboard elle seçtiyse true — soft CTA azalt |

**Yeni host default:**  
`defense_policy=observe`, `observe_auto_promote_enabled=true`,  
`observe_auto_promote_days=3`, `isolate_armed=false`, `defense_policy_locked=false`.

### 1.2 `GET/POST /api/threats/config` genişlemesi

```json
{
  "protection": {
    "defense_policy": "observe",
    "defense_policy_version": "1.4.19-def-1",
    "isolate_armed": false,
    "observe_started_at": "2026-07-23T12:00:00Z",
    "observe_auto_promote_days": 3,
    "observe_auto_promote_enabled": true,
    "defense_policy_locked": false,
    "defense_rules": {
      "canary_write": "alert_only",
      "vss_deletion": "alert_only",
      "ransomware_critical_process": "alert_only",
      "suspicious_rdp": "alert_only",
      "high_io_rate": "alert_only",
      "mass_password_change": "alert_only",
      "network_surface_additive": "inform_only"
    }
  }
}
```

**Aksiyon sözlüğü (wire):**

| Değer | Anlam | Cloud notu |
|-------|--------|------------|
| `alert_only` | Alarm; süreç/ağ yok | Soft/urgent hijyen — **observe default** |
| `inform_only` | Soft inform (mavi) | `network_surface_changed` |
| `ask_operator` | Dashboard onay kartı | Komut bekler |
| `suspend_process` | Client PID suspend | Exact identity |
| `kill_quarantine` | Kill + RS quarantine | Balanced+ kırmızı |
| `auto_isolate_network` | Ağ izolasyonu | **Yalnız paranoid + armed** |

**Preset derleme (cloud):**

| Preset | `canary_write` / `vss_deletion` | Uyarılar | `*_isolate*` |
|--------|----------------------------------|----------|--------------|
| observe | `alert_only` | **Açık** (hepsi) | yok |
| balanced | `kill_quarantine` | Açık | yok |
| paranoid | `kill_quarantine` | Açık | yalnız `isolate_armed=true` |

POST validation: observe/balanced body’de `auto_isolate_network` varsa **422** veya strip + audit.  
`observe_auto_promote_days` ∈ 0..14; paranoid seçiminde auto-promote **devre dışı** (zaten üst seviye).

Kaydet sonrası: effective config + WS `{ "v":1, "t":"threat_config_updated" }`.

### 1.3 Ajan pull (fail-safe)

- Health/heartbeat veya config GET: `defense_policy_version` + onboarding alanları.  
- Stale version → güncel imzalı matris.  
- Emit: `sig` = HMAC (command signing ailesi).

### 1.4 Auto-promote (observe → balanced)

Cloud job / request-path (tercihen cron + config GET öncesi):

```
IF defense_policy == observe
AND observe_auto_promote_enabled
AND NOT defense_policy_locked
AND observe_auto_promote_days > 0
AND now >= observe_started_at + days
THEN
  set defense_policy = balanced
  recompile defense_rules (balanced preset)
  bump defense_policy_version
  sign + persist
  audit: reason=auto_promote_observe_elapsed
  WS threat_config_updated
  optional notify: "Savunma dengesi otomatik açıldı"
```

- Kullanıcı observe’da **kalmak** isterse: dashboard “İzlemede kilitle” → `defense_policy_locked=true` **veya** `observe_auto_promote_enabled=false`.  
- Elle balanced/paranoid seçimi → `policy_user_set=true`; auto-promote iptal.  
- **Asla** auto → paranoid veya `isolate_armed=true`.

Client yedek (cloud job gecikirse): aynı koşullarda yerel CTA + opsiyonel `POST threats/config` balanced önerisi; cloud SoT kalır ([`../agent/defense-policy-client.md`](../agent/defense-policy-client.md)).

---

## 2. Dashboard UI (zorunlu)

### Eğitim (tüm preset seçicilerde)

Her mod kartında kısa metin (TR/EN):

| Mod | Başlık | Tek cümle |
|-----|--------|-----------|
| İzleme | Observe | Tüm uyarıları görürsünüz; süreç otomatik öldürülmez / ağ kesilmez. |
| Denge | Balanced | Kırmızı tehditte şüpheli süreç durdurulur; RDP/internet ayakta kalır. |
| Tetikte | Paranoid | Daha agresif süreç tepkisi; ağ izolasyonu ayrı onay ister (brick riski). |

Paranoid seçilince: brick/RDP risk + `isolate_armed` checkbox (default kapalı).

### P0 — minimum

1. Savunma politikası seçici + eğitim metinleri  
2. Yeni host: **observe** seçili; “2–3 gün sonra dengede önerilir / otomatik geçer” notu  
3. Auto-promote kontrolü: gün sayısı + enable + “İzlemede kilitle”  
4. Soft vs urgent alert ayrımı  
5. Kırmızı kart: Resume / Allow / Unlock  
6. `network_surface_changed`: Accept / Ignore / Disable (confirm)  
7. Auto-promote audit log satırı  

### P1

8. Rule matrix editor (preset + güvenli override)  
9. Isolate override = kırmızı risk banner + audit  
10. Sarı: “Süreç donduruldu — Devam / Öldür / Whitelist”  
11. Alert fatigue grouping  

### P2

12. Isolate host / Lift isolation (destructive confirm + PIN)  
13. OOB isolate banner  
14. Custom isolate kuralı — risk onayı; tek-canary auto-queue **yok**

---

## 3. Alert routing (cloud)

| `threat_type` | Severity | `under_attack` | UI |
|---------------|----------|----------------|-----|
| `ransomware_canary_triggered` / VSS wipe intent | critical/high (observe’da da alert) | balanced+ ve doğrulanmışta evet; **observe’da process action yok** ama kart göster | Soft-urgent hijyen |
| `network_surface_changed` | info | **hayır** | Mavi soft |
| `defense_policy_promoted` | info | hayır | “Dengeye geçildi” |
| `defense_policy_tamper` | high | hayır | Soft-urgent; isolate yok |
| Process suspended | warning/high | kırmızıda opsiyonel | Resume/Allow |

**Dedupe:** bait ailesi birleşir; **auto-escalate to isolate yok**.  
Observe→balanced promote **bilinçli ürün yolu**dır; anti-bait listesindeki “N alert → isolate” ile karıştırma.

---

## 4. Control WS komutları

### P0

| `command_type` | Confirm | Cloud UI |
|----------------|---------|----------|
| `network_accept_surface` | hayır | Soft inform |
| `network_disable_adapter` | mutate evet | Soft inform opsiyonel |
| `unlock_ransomware_quarantine` | (mevcut) | Kırmızı kart |
| `resume_process` | hayır/soft | Dondurulmuş süreç |
| `allow_process` | tercihen confirm | “Bu bendim — güven” |
| `suspend_process` / `kill_process` | destructive | Sarı/kırmızı |

`VALID_COMMAND_TYPES` + `DESTRUCTIVE_COMMAND_TYPES` güncelle
([`../api/03-control-websocket.md`](../api/03-control-websocket.md)).

### P2

| `command_type` | Confirm | Not |
|----------------|---------|-----|
| `isolate_host` | **zorunlu** | observe/balanced UI disabled |
| `lift_isolation` | confirm | Tek tık kurtarma |

---

## 5. Anti-bait — cloud checklist

- [ ] Canary/VSS → process komutları OK; **isolate_host auto-queue yok**  
- [ ] Alert count → isolate yok  
- [ ] Tamper → bildirim; paranoid+isolate push yok  
- [x] observe/balanced `auto_isolate_network` strip/reject  
- [x] Auto-promote **yalnız** observe→balanced; paranoid/armed açmaz  
- [ ] Fatigue: duplicate urgent merge  

---

## 6. Fazlara göre cloud iş paketleri

### Cloud P0 (client ≥4.9.17)

| ID | İş |
|----|-----|
| C-P0-1 | DB: policy + version + rules + sig + armed + **onboarding alanları** |
| C-P0-2 | threats/config GET/POST + preset derleyici + validation |
| C-P0-3 | WS `threat_config_updated`; heartbeat/version pull |
| C-P0-4 | HMAC sign matrix on emit |
| C-P0-5 | Dashboard preset UI + **eğitim metinleri** + soft/urgent + Resume/Allow/Unlock |
| C-P0-6 | Soft inform panel (1.4.17) |
| C-P0-7 | Snapshot attachment (opsiyonel) |
| C-P0-8 | Anti-bait: no auto-isolate; no tamper→escalate |
| C-P0-9 | **Yeni host default observe** + auto-promote job (default 3 gün) |
| C-P0-10 | Dashboard: promote günü / kilitle / audit + `defense_policy_promoted` notify |

### Cloud P1 / P2

Önceki tablolar aynı (sarı UI, matrix editor, isolate).

---

## 7. Acceptance (cloud)

- [x] Yeni host `defense_policy=observe` (unit ensure_default)  
- [x] Balanced effective rules asla `auto_isolate_network` içermez  
- [ ] Policy save → WS ≤ 2s; stale version pull düzelir  
- [ ] Canary urgent → isolate komutu oluşmaz (observe’da kill de yok)  
- [ ] Soft surface → `under_attack=false`  
- [x] `observe_auto_promote_days=3` → süre dolunca balanced + WS + audit (unit)  
- [x] Locked / disabled promote → observe’da kalır (unit)  
- [x] Auto-promote paranoid veya armed açmaz  
- [ ] Resume/Allow/Unlock dashboard’dan çalışır  
- [ ] Eğitim metinleri üç modda görünür  

---

## 8. Client hedef

| Client | Ne |
|--------|-----|
| ≥4.9.15 | Soft surface inform |
| ≥4.9.16 | Matrix apply, signed cache, LKG, allow_process, snapshot, anti-bait |
| ≥4.9.17 | Default observe; onboarding CTA; local promote backup; education UI |

Detay: [`../agent/defense-policy-client.md`](../agent/defense-policy-client.md) · [`../ROADMAP_TIERED_DEFENSE.md`](../ROADMAP_TIERED_DEFENSE.md).

---

## 9. Cloud ekibine kısa handoff (kopyala-yapıştır)

> Contract **1.4.19** · SoT: bu dosya.  
> Client **≥4.9.17** observe-default + matrix hazır.  
> Uygula: **C-P0-1 … C-P0-10**.  
> Default observe; 3 gün sonra auto balanced (kilitleme dashboard’da).  
> Tüm uyarılar her modda; agresif aksiyon moda bağlı.  
> Anti-bait: canary/tamper/N-alert → isolate yok. Auto-promote ≠ paranoid.
