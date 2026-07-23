# Defense Policy — Cloud implementation (tiered response)

> **Contract VERSION:** root `VERSION` (**1.4.18**)  
> **Status:** Normative for **cloud + dashboard** (client apply target ≥ **4.9.16** when shipped)  
> **Planning SoT:** [`../ROADMAP_TIERED_DEFENSE.md`](../ROADMAP_TIERED_DEFENSE.md)  
> **Related:** [`../agent/threat-engine.md`](../agent/threat-engine.md) · [`../api/03-control-websocket.md`](../api/03-control-websocket.md) · [`../agent/ransomware-shield.md`](../agent/ransomware-shield.md) · [`../agent/network-guard.md`](../agent/network-guard.md)

Bu belge **cloud/dashboard’un yapması gerekenleri** listeler.  
Client motor aksiyon primitiflerini uygular; **politika zekâsı ve operatör UI** cloud’dadır.

---

## 0. Cloud invariants (asla ihlal etme)

1. **Observe / Balanced default’ta ağ izolasyonu yok.**  
   `auto_isolate_network` yalnız `paranoid` + müşteri arm / confirm.  
2. **Tek canary / tek VSS intent → isolate komutu kuyruğa yazılmaz.**  
3. **`under_attack` yalnız kırmızı doğrulanmış olaylarda** (canary write, VSS wipe intent, operatör containment). Soft `network_surface_changed` / `suspicious_session` (info) açmaz.  
4. **N alert ≠ otomatik escalate / isolate** (anti-bait / fatigue).  
5. **Policy imza bozukluğu için cloud “agresif moda geç” demez**; client LKG/observe — cloud tamper alert’i soft-urgent gösterir, isolate push etmez.  
6. Stolen token ile “daha agresif matrix” → yine client hard-reject (balanced isolate); mutating komutlar HMAC + destructive confirm.  
7. Soft surface inform (1.4.17): mavi banner; Accept / optional Disable — panik yok.

---

## 1. Veri modeli

### 1.1 Persist (host veya fleet)

| Alan | Tip | Not |
|------|-----|-----|
| `defense_policy` | `observe` \| `balanced` \| `paranoid` | Preset adı |
| `defense_policy_version` | string | Örn. `1.4.18-def-3` — ajan pull karşılaştırması |
| `defense_rules` | object | Event → action matrisi (aşağıda) |
| `defense_rules_sig` | string | Cloud’un HMAC’i (emit anında) |
| `isolate_armed` | bool | Paranoid ağ izolasyonu müşteri onayı |
| `policy_updated_at` | ISO-8601 | Audit |

Yeni host önerilen UI: **observe** (1–2 hafta) → CTA ile **balanced**.

### 1.2 `GET/POST /api/threats/config` genişlemesi

Mevcut deep-merge’e ekle:

```json
{
  "protection": {
    "defense_policy": "balanced",
    "defense_policy_version": "1.4.18-def-3",
    "isolate_armed": false,
    "defense_rules": {
      "canary_write": "kill_quarantine",
      "vss_deletion": "kill_quarantine",
      "ransomware_critical_process": "suspend_process",
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
| `alert_only` | Alarm; süreç/ağ yok | Soft/urgent hijyen |
| `inform_only` | Soft inform (mavi) | `network_surface_changed` |
| `ask_operator` | Dashboard onay kartı | Komut bekler |
| `suspend_process` | Client PID suspend | Exact identity |
| `kill_quarantine` | Kill + RS quarantine | Mevcut canary/VSS |
| `auto_isolate_network` | Ağ izolasyonu | **Yalnız paranoid + armed**; observe/balanced emit etme |

**Preset derleme (cloud):**

| Preset | `canary_write` / `vss_deletion` | `suspicious_rdp` | `*_isolate*` |
|--------|----------------------------------|------------------|--------------|
| observe | `alert_only` (veya mevcut RS observe) | `alert_only` | yok |
| balanced | `kill_quarantine` | `alert_only` veya `ask_operator` | yok |
| paranoid | `kill_quarantine` | `suspend_process` / `ask_operator` | yalnız `isolate_armed=true` iken matrix’e girebilir |

POST validation: observe/balanced body’de `auto_isolate_network` varsa **422** veya sunucu sessizce strip + audit warning.

Kaydet sonrası: effective config dön + WS `{ "v":1, "t":"threat_config_updated" }`  
(opsiyonel alias `defense_policy_updated` — şart değil).

### 1.3 Ajan pull (fail-safe)

- Health/heartbeat veya config GET cevabında `defense_policy_version` döner.  
- Ajan stale `policy_version` bildirirse cloud **güncel imzalı matrisi** döner (push kaçmış olsa bile).  
- Emit edilen her matrix: `sig` = HMAC (command signing anahtar ailesi).

---

## 2. Dashboard UI (zorunlu)

### P0 — minimum

1. **Savunma politikası** seçici: Observe / Balanced / Paranoid  
2. Paranoid seçilince: brick/RDP risk metni + `isolate_armed` checkbox (default kapalı)  
3. Soft vs urgent alert ayrımı (mavi surface inform ≠ kırmızı ransomware)  
4. Kırmızı olay kartı: Resume / Allow (whitelist) / Unlock quarantine  
5. `network_surface_changed`: Accept surface / Ignore / Disable adapter (confirm)

### P1

6. Rule matrix editor (preset + güvenli override)  
7. Isolate override = kırmızı risk banner + audit log  
8. Sarı: “Süreç donduruldu — Devam ettir / Öldür / Whitelist”  
9. Alert fatigue: aynı bait ailesi tek kartta birleşir  

### P2

10. **Isolate host / Lift isolation** butonları (destructive confirm + PIN)  
11. Isolate armed host’ta OOB banner: “Ağ izole — panele gel”  
12. Custom “şu tetikte isolate” — zorunlu risk onayı; tek-canary auto-queue **yok**

---

## 3. Alert routing (cloud)

| `threat_type` | Severity | `under_attack` | UI |
|---------------|----------|----------------|-----|
| `ransomware_canary_triggered` / VSS wipe intent | critical/high | evet (politika) | Kırmızı + aksiyonlar |
| `network_surface_changed` | info | **hayır** | Mavi soft |
| `network_surface_restored` | warning | hayır | Rozet |
| `suspicious_session` (P1) | info/warning | hayır | Sarı |
| `defense_policy_tamper` / agent tamper | high | hayır (default) | Soft-urgent; isolate push **yok** |
| Process suspended (sarı/kırmızı) | warning/high | kırmızıda opsiyonel | Resume/Allow |

**Dedupe:** Aynı host + aynı `threat_type` + aynı bait fingerprint → mevcut urgent dedupe pencereleri; **auto-escalate to isolate yok**.

Session snapshot (P0 client): urgent/attachment store — rate-limit client’ta; cloud flood’u drop/merge.

---

## 4. Control WS komutları (cloud whitelist + UI)

### P0 — ekle / doğrula

| `command_type` | Confirm | Cloud UI |
|----------------|---------|----------|
| `network_accept_surface` | hayır | Soft inform “Yeni golden” |
| `network_disable_adapter` | mutate evet / dry_run hayır | Soft inform opsiyonel |
| `unlock_ransomware_quarantine` | (mevcut) | Kırmızı kart |
| `resume_process` | hayır (veya soft) | Dondurulmuş süreç |
| `allow_process` / whitelist | tercihen confirm | “Bu bendim — güven” |
| `suspend_process` / `kill_process` | mevcut destructive | Sarı/kırmızı |

`VALID_COMMAND_TYPES` + `DESTRUCTIVE_COMMAND_TYPES` güncelle
([`../api/03-control-websocket.md`](../api/03-control-websocket.md)).

Exact-identity: `pid` + `image` + `path` + `process_start_time` (mevcut suspend kuralları).

### P2 — izolasyon

| `command_type` | Confirm | Not |
|----------------|---------|-----|
| `isolate_host` | **zorunlu** destructive | cloud IP allowlist param; observe/balanced UI’da gizle veya disabled |
| `lift_isolation` | confirm önerilir | Tek tık kurtarma |

Cloud, isolate’i **asla** salt canary alert handler’ından auto-queue etmez.

---

## 5. Anti-bait — cloud checklist

- [ ] Canary/VSS alert pipeline → **process komutları** (client zaten local kill edebilir); **isolate_host yok**  
- [ ] Alert count eşiği → isolate yok  
- [ ] `defense_policy_tamper` → bildirim; policy push “paranoid+isolate” yok  
- [ ] Config POST strip/reject `auto_isolate_network` on observe/balanced  
- [ ] Stolen-token agresif matrix: client reject’e güven; audit log  
- [ ] Fatigue: duplicate urgent merge  

---

## 6. Fazlara göre cloud iş paketleri

### Cloud P0 (client 4.9.16 hedefiyle birlikte)

| ID | İş |
|----|-----|
| C-P0-1 | DB: `defense_policy`, version, rules, sig, `isolate_armed` |
| C-P0-2 | threats/config GET/POST + preset derleyici + validation |
| C-P0-3 | WS `threat_config_updated` on policy save; heartbeat/version pull |
| C-P0-4 | HMAC sign matrix on emit |
| C-P0-5 | Dashboard preset UI + soft/urgent + Resume/Allow/Unlock |
| C-P0-6 | Soft inform panel (1.4.17 Accept/Disable) — henüz yoksa tamamla |
| C-P0-7 | Snapshot attachment ingest (opsiyonel storage) |
| C-P0-8 | Anti-bait: no auto-isolate from alerts; no tamper→escalate |

### Cloud P1

| ID | İş |
|----|-----|
| C-P1-1 | `suspicious_session` kartı + geo/ASN (privacy-min) |
| C-P1-2 | Sarı suspend onay kartı |
| C-P1-3 | Rule matrix editor + audit |
| C-P1-4 | Alert fatigue grouping |

### Cloud P2

| ID | İş |
|----|-----|
| C-P2-1 | `isolate_host` / `lift_isolation` whitelist + destructive UI |
| C-P2-2 | `isolate_armed` gate; multi-signal veya insan confirm |
| C-P2-3 | OOB isolate banner |
| C-P2-4 | Bait simülasyon QA (tek canary ≠ isolate queue) |

---

## 7. Acceptance (cloud)

- [ ] Preset observe/balanced/paranoid persist + config poll  
- [ ] Balanced effective rules asla `auto_isolate_network` içermez  
- [ ] Policy save → WS push ≤ 2s; stale version pull düzelir  
- [ ] Canary urgent → isolate komutu oluşmaz  
- [ ] Soft `network_surface_changed` → `under_attack=false`  
- [ ] Resume/Allow/Unlock dashboard’dan çalışır  
- [ ] Paranoid + armed olmadan isolate butonu disabled  
- [ ] Tamper/malformed policy alert isolate push etmez  

---

## 8. Client hedef (referans — bu dosya cloud SoT)

| Client | Ne |
|--------|-----|
| ≥4.9.15 | Soft surface inform (shipped) |
| ≥4.9.16 (plan) | defense_rules apply, signed cache, LKG, resume/allow, snapshot, anti-bait invariants |

Detay fazlar: [`../ROADMAP_TIERED_DEFENSE.md`](../ROADMAP_TIERED_DEFENSE.md).
