# Tiered Defense & Resilience Roadmap

> **Status:** Planning (living document) — **final review 2026-07-23**  
> **Audience:** Client · Cloud/API · Dashboard · QA  
> **Related:** [`agent/network-guard.md`](agent/network-guard.md) · [`agent/ransomware-shield.md`](agent/ransomware-shield.md) · [`SECURITY_RESILIENCE_VNEXT.md`](SECURITY_RESILIENCE_VNEXT.md) · shipped soft surface inform **1.4.17 / client ≥4.9.15**  
> **Repo:** https://github.com/cevdetaksac/honeypot-contract

Bu dosya Gemini + ürün ekibi değerlendirmelerinin **birleşik yol haritasıdır**.  
Yeni öneriler §8’e eklenir; uygulama **faz faz** (P0→P3) ilerler.  
Kod/kontrat yazımı bu belgeden ayrı sprint’lerde yapılır.

---

## 0. Ürün DNA’sı (değişmez filtre)

Her özellik şu süzgeçten geçer:

| Tut | Bırak / ertele |
|-----|----------------|
| Soft alarm, dashboard karar | Varsayılan oto ağ kesme (Ethernet öldürme) |
| Dar, kanıtlı tetik (canary / VSS wipe / intel) | “Disk I/O yüksek” / clipboard / typing-speed → auto-suspend |
| Process suspend/kill+quarantine (kırmızı) | Temp’ten her EXE’yi dondurarak başlat |
| Yanlış alarm çıkışı: resume / whitelist / unlock | `under_attack` panik her soft olayda |
| Observe → Balanced → Paranoid (müşteri seçer) | Tek “herkese agresif” default |
| **Saldırganın bilerek tetiklemesiyle self-DoS yok** | Tek sinyal → felç / ağ kes / “en agresif moda düş” |

**Özet tez:**  
LockBit insan onayını beklemez — **doğru**.  
Ama “ağı öldür” default brick + müşteri paniğidir.  
**Süreci dondurmak ≠ sunucuyu kör etmek.** OOB/cloud erişimi process karantinasını güvenli kılar; ağ izolasyonu yalnız bilinçli politika veya armed panic ile gelir.

**Anti-bait tez:**  
Saldırgan ürünü tanıyıp canary’ye dokunarak, imzayı bozarak veya alarm yorgunluğu yaratarak **bizi kendi silahımızla vurmamalı**. Otomasyon **saldırgan sürecini** durdurur; **sunucuyu ve yönetici erişimini** teslim etmez.

---

## 1. Kaynak öneriler (özet)

### 1.1 Gemini — RDP / kimlik / izolasyon / forensics

| Öneri | Değerlendirme | Yol haritası |
|-------|---------------|--------------|
| Impossible travel (alışılmadık geo/ASN + başarılı RDP) | Soft alarm uyumlu; bare `successful_logon` zaten soft | **P1** — `suspicious_session` |
| Clipboard / süper hızlı yazım → auto-suspend | Yüksek FP (admin PS yapıştırır); brick | **P3 observe-only** — otomatik yok |
| Canary / VSS → oto ağ izolasyonu (yalnız cloud IP) | Brick + **self-DoS bait** | **P2** — yalnız Paranoid / armed; asla tek sinyal |
| Temp/Public’ten bilinmeyen EXE → Suspended start | EDR FP; `auto_contain` hard-off dersi | **P3** — observe; default off |
| Session JPEG snapshot (şüpheli anda) | Brick yok; RD stack hazır; rate-limit şart | **P0** |

### 1.2 Gemini — Kademeli tepki + Defense Policy

| Öneri | Değerlendirme | Yol haritası |
|-------|---------------|--------------|
| Kırmızı = auto isolate (ağ + process) | Process kısmı evet; **ağ default hayır** | Kırmızı process **P0**; ağ **P2** |
| Sarı = dondur + onay bekle | Doğru denge | **P1** |
| Observe / Balanced / Paranoid politikalar | SaaS için zorunlu | **P0** (kontrat + cloud toggle) |

### 1.3 Ürün ekibi — process-first otomasyon

> Ağı öldürmeden, LockBit ihtimali aşırı yüksek süreçlerde anında **suspend**;  
> meşruysa yönetici **resume / whitelist** ile devam ettirsin.

**Karar:** Kabul. Ayrı bayrak `auto_suspend_critical` (balanced+).  
Eski Network Guard bomb-path `auto_contain` ile karıştırılmaz.  
Mevcut canary / VSS delete intent (kill + quarantine) kırmızı çekirdek olarak kalır.  
Suspend **yalnız şüpheli PID** — korumalı sistem süreçleri asla.

### 1.4 Gemini — Policy delivery (JSON matris · sync · imza · tamper)

| Öneri | Değerlendirme | Yol haritası |
|-------|---------------|--------------|
| Enum yerine **rules matrix** payload | Enterprise / custom policy için doğru | **P0-1** · §3.1 |
| WS push + boot/heartbeat **pull** `policy_version` | Mevcut `threat_config_updated` ile uyumlu | **P0-1b** |
| Ajan “aptal itaatkâr” rule engine | Kabul; hard safety matris üstünde | **P0-1** |
| İmzalı yerel policy cache | Kabul (HMAC) | **P0-1c** |
| Tamper → en agresif + **ağı kes** | **Red** — self-DoS / brick vektörü | §3.2 · §6 |
| Balanced örnekte canary/VSS → `auto_isolate_network` | **Red default** — process `kill_quarantine` | §3.1 tablo |

### 1.5 Ürün — adversarial self-DoS / “sinir ucu” bait

> Saldırgan uygulamayı bilerek tetikleyip kendi kendimizi imha ettirmesin.

**Karar:** Birinci sınıf tasarım kısıtı — §3.2.  
Tek canary dokunuşu / imza bozma / alert spam → **asla** ağ kesme veya “paranoid’e düş”.

---

## 2. Hedef mimari — Tiered Response

```
                    ┌─────────────────────────┐
                    │  Defense Policy (host)  │
                    │ observe|balanced|paranoid│
                    └───────────┬─────────────┘
                                │
        ┌───────────────────────┼───────────────────────┐
        ▼                       ▼                       ▼
   Mavi (Inform)           Sarı (Suspect)          Kırmızı (Critical)
   soft notify             suspend PID?            suspend/kill+Q
   no under_attack         network intact          network: default OFF
   surface_inform          ask operator            (+ isolate only if armed
                                                   AND multi-signal / confirm)
```

### 2.1 Kırmızı — yüksek kesinlik

**Tetik (dar allowlist):**

- Canary yazma / silme / bozma  
- VSS wipe intent (`vssadmin delete shadows` / WMI / wbadmin) — mevcut ≥4.9.14  
- (sonra) bilinen ransomware hash / intel imza  

**Aksiyon (Balanced+):**

1. Process suspend **veya** kill (**yalnız tetikleyen / şüpheli PID**)  
2. Quarantine arm (IFEO — admin VSS araçlarına değil)  
3. Urgent alert (`force_urgent` yalnız kırmızı)  
4. Session JPEG snapshot (rate-limit)  
5. **Ağ kesme: default kapalı** — tek kırmızı sinyal isolate etmez (§3.2)  

**Çıkış kapıları:** `resume_process` · `allow_process` / whitelist · `unlock_ransomware_quarantine`

### 2.2 Sarı — anomali

**Tetik (örnek):**

- Impossible travel / şüpheli RDP oturumu  
- Bilinmeyen imaj + yüksek disk fan-out (tek başına yetmez; skor eşiği)  
- Gece toplu password reset vb.  

**Aksiyon (Balanced):**

- Yalnız o PID suspend (veya Observe’da yalnız alarm)  
- Ağı kesme yok  
- Dashboard: “Devam ettir / Öldür / Whitelist”

### 2.3 Mavi — bilgilendirme (shipped)

- Ethernet up / DHCP lease / yeni NIC → `network_surface_changed`  
- `surface_inform` · Accept golden · Disable yalnız confirm  
- Contract **1.4.17** / client **≥4.9.15**

---

## 3. Defense Policy (SaaS — host başına)

| Mod | Process | Ağ izolasyonu | Tipik kullanım |
|-----|---------|---------------|----------------|
| **observe** | Alarm only | Off | İlk 1–2 hafta / lab |
| **balanced** (önerilen) | Kırmızı: auto suspend/kill+Q; Sarı: suspend+onay | Off | Çoğu sunucu |
| **paranoid** | Kırmızı+sarı agresif | Opsiyonel isolate (müşteri arm + risk onayı + tercihen multi-signal) | Kritik DB |

**Cloud alan (taslak):** `protection.defense_policy` = `observe|balanced|paranoid`  

**Kurallar:**

- Yeni kurulumda UI **observe** önerir (1–2 hafta), sonra balanced’a geçiş CTA  
- Paranoid network isolate kurulurken açık “brick/RDP kesilir” onayı  
- Client: process `auto_contain` bomb-path hard-false kalır; kırmızı yol `auto_suspend_critical` / mevcut RS yolları ile gider  
- **Hiçbir hata/tamper yolu sessizce paranoid+isolate’e yükseltmez**

Preset’ler §3.1 **kural matrisine** derlenir; ileride Custom Policy aynı şemayı satabilir.

---

## 3.1 Policy delivery mimarisi (Gemini 4 adım — değerlendirme)

Gemini’nin “enum değil, ajanın işlediği kural matrisi” önerisi **Enterprise SaaS için doğru**.  
Mevcut `threat_config_updated` + `GET /api/threats/config` omurgası üzerine oturur
([`api/03-control-websocket.md`](api/03-control-websocket.md),
[`agent/threat-engine.md`](agent/threat-engine.md),
[`agent/persistence-and-tamper.md`](agent/persistence-and-tamper.md)).

### Adım A — Veri modeli / payload (**Kabul, varsayılanlar düzeltilerek**)

- Cloud: host/fleet `defense_policy` + isteğe bağlı **rules override** (custom).  
- Ajanına giden şey yalnız `balanced` string’i değil; **işlenebilir matris**:

```json
{
  "policy_version": "1.4.18-def-3",
  "policy_name": "balanced",
  "rules": {
    "canary_write": "kill_quarantine",
    "vss_deletion": "kill_quarantine",
    "ransomware_critical_process": "suspend_process",
    "suspicious_rdp": "alert_only",
    "high_io_rate": "alert_only",
    "mass_password_change": "alert_only",
    "network_surface_additive": "inform_only"
  },
  "sig": "<hmac>"
}
```

**Aksiyon sözlüğü (taslak):**  
`alert_only` · `inform_only` · `suspend_process` · `kill_quarantine` ·
`auto_isolate_network` (yalnız paranoid/armed — §0 / §3.2) · `ask_operator`

| Gemini örnek kural | Bizim karar |
|--------------------|-------------|
| `honeytoken_access: auto_isolate_network` | **Red (balanced default)** → `kill_quarantine`; isolate ≠ default / ≠ tek sinyal |
| `vss_deletion: auto_isolate_network` | **Red (balanced)** → mevcut `kill_quarantine` (4.9.14) |
| `suspicious_rdp: suspend_process` | **P1** — balanced’ta önce `alert_only` veya `ask_operator`; paranoid’te suspend |
| `high_io_rate: suspend_process` | **Red default** → `alert_only` (FP + bait) |
| Custom “balanced ama X’te isolate” | **P2+** — override + risk onayı + audit |

### Adım B — Senkronizasyon push + pull (**Kabul**)

| Mekanizma | Ne |
|-----------|-----|
| **Push** | Dashboard Kaydet → mevcut `threat_config_updated` → ajan `GET /api/threats/config` |
| **Pull / fail-safe** | Boot + periyodik: ajan `policy_version` bildirir; cloud stale ise güncel matrisi döner |
| **Offline** | Son **imzalı** matris ProgramData’da; internet yokken o uygulanır |

### Adım C — Karar ağacı / “aptal ama itaatkâr” ajan (**Kabul**)

```
Event → RuleEngine.lookup(event_type) → action from signed matrix
       → alert_only | suspend | kill_quarantine | inform_only | …
```

Hard safety **matrisin üstünde**: bomb-path contain, IFEO-on-vssadmin yasak,
additive → asla isolate, **§3.2 anti-bait**, korumalı PID listesi.
Cloud `auto_isolate_network` gönderse bile observe/balanced **reddeder**.

### Adım D — Offline + imzalı config + tamper (**Kabul, fail-mode düzeltilerek**)

| Gemini önerisi | Değerlendirme |
|----------------|---------------|
| Policy JSON yerel cache | **Kabul** — ProgramData |
| HMAC / imza ile bütünlük | **Kabul** |
| İmza bozuk → **en agresif + ağı kes** | **Red** — saldırgan bilinçli self-imha |

**Tamper fail-mode:** LKG → else **observe** → tamper alert → **ağ kesme yok**.

### Bugünkü tamper durumu

- Komut / baseline HMAC · guardian persistence · config apply var  
- Defense rule matrix + signed policy cache → P0  
- Full encrypt şart değil; imza + DACL + LKG yeterli başlangıç  

---

## 3.2 Anti-bait / adversarial self-DoS (zorunlu tasarım)

Saldırgan ürünü bilerek “sinir uçlarına” basabilir. Amaç çoğu zaman şifrelemek
değil; **yanlış otomatizmle erişimi öldürmek**, alarm yormak veya restore savaşını
başlatmak.

### Tipik bait senaryoları

| Bait | Saldırgan ne yapar | Tehlikeli otomatizm | Bizim yanıt |
|------|--------------------|---------------------|-------------|
| Canary dokunuşu | Bilinen tuzak dosyaya yazar | Ağ izolasyonu / tüm oturumu kes | Yalnız **yazar PID** kill/Q; RDP/DNS ayakta |
| Policy imza boz | `defense_policy.json` truncate/edit | “En agresif + isolate” | LKG/observe + tamper alert |
| Alert flood | Canary’ye tekrar tekrar dokun / sahte skor | Urgent spam → yorgunluk / auto-escalate | Dedupe · rate-limit · escalate yok |
| VSS list spam | `vssadmin list` (silme değil) | Yanlış quarantine | Zaten list ≠ intent (hijyen) |
| Korumalı süreci yemle | İsmi “encryptor” gibi gösteren ama kritik PID | Suspend lsass/services | **Protected images** hard deny |
| Golden zehirleme | IP boz → periyodik baseline’a yazdır | Restore ile kalıcı hasar | Golden poison yasağı (1.4.14) — korunur |
| Subtractive sahte | Adapter kapat → auto_restore döngüsü | Flap / DoS | Restore dedupe (≥5 dk); maintenance yolu |
| Isolate armed host | Tek canary ile paranoid isolate | Yönetici kilit dışı | Isolate: **multi-signal veya insan arm**; tek canary yetmez |
| Cloud’a sahte “daha agresif policy” | Çalıntı token ile matrix | Ani isolate | Command HMAC + destructive confirm; client balanced hard-reject isolate |

### Tasarım kuralları (P0’dan itibaren invariant)

1. **Blast radius = tetikleyen süreç** (mümkün olduğunca); asla “makineyi kör et”.  
2. **Tek zayıf/bilinen bait sinyali → ağ aksiyonu yok** (canary tek başına isolate etmez).  
3. **Fail-safe sakinleşir, kızışmaz:** hata/tamper/stale → observe/LKG, never paranoid.  
4. **Rate-limit / dedupe** tüm auto aksiyonlarda (alert + suspend + snapshot + restore).  
5. **Protected process / path** listesi aşılamaz (matrix bile).  
6. **Auto-escalate yok:** N alert ≠ otomatik isolate.  
7. **Paranoid isolate** için: müşteri arm + (tercihen) ikinci sinyal veya dashboard confirm.  
8. **OOB hayatta:** ne olursa olsun cloud komut kanalı / motor yeniden bağlanma yolu korunur.

> Canary’ye dokunmak saldırgan için **kendi dropper’ını yakmak** olmalı;  
> **sunucuyu admin’den koparmak** değil.

---

## 4. Fazlar (uygulama sırası)

### P0 — Güçlendirir, brick + self-DoS riski düşük

| ID | İş | Client | Cloud | Kontrat |
|----|-----|--------|-------|---------|
| P0-1 | `defense_policy` preset + **rules matrix** (balanced defaults §3.1) | apply + hard safety | threats/config + UI | `agent/` + threats/config |
| P0-1b | Push/pull `policy_version` | CONFIG-SYNC | version compare | 03-control-websocket |
| P0-1c | İmzalı cache + LKG; tamper → observe/LKG (**not** isolate) | verify | sign on emit | persistence-and-tamper |
| P0-2 | Kırmızı process netleştirme (canary/VSS + critical suspend) | RS; yalnız şüpheli PID | alert routing | ransomware-shield |
| P0-3 | resume / allow (whitelist path+hash) | komutlar | confirm UI | 03-control-websocket |
| P0-4 | Session snapshot (1 JPEG, dedupe ≥5 dk) | RD/capture | store | api/05 + agent |
| P0-5 | Soft vs urgent hijyen | alert pipeline | dashboard | hygiene |
| P0-6 | **Anti-bait invariants** dokümante + test: tek canary≠isolate; tamper≠escalate; protected PID; action dedupe | unit/lab | — | bu roadmap §3.2 |

**Kabul:** Balanced lab — canary/VSS → şüpheli process durur, RDP/internet ayakta, resume çalışır. İmza bozulunca ağ kesilmez. Bilerek canary spam → isolate yok, alert dedupe.

### P1 — Sarı + kimlik + custom rules UI

| ID | İş |
|----|-----|
| P1-1 | Impossible travel / `suspicious_session` (soft) |
| P1-2 | Sarı: tek-PID suspend + İzin ver / Öldür |
| P1-3 | GUI chip: donduruldu → devam ettir |
| P1-4 | Geo/ASN baseline — privacy-min |
| P1-5 | Rule matrix editor (isolate override = risk banner + audit) |
| P1-6 | Alert-fatigue UX: aynı bait ailesi tek kartta birleşir |

### P2 — Ağ izolasyonu (bilinçli, bait-resistant)

| ID | İş |
|----|-----|
| P2-1 | `isolate_host` / `lift_isolation` (confirm) — cloud IP allowlist |
| P2-2 | Yalnız paranoid / armed; matrix isolate observe/balanced’ta **client reject** |
| P2-3 | Isolate: **multi-signal veya insan confirm** (tek canary yetmez) |
| P2-4 | OOB mesaj + NG/System Recovery geri dönüş |
| P2-5 | Dual-NIC / VPN lab |
| P2-6 | Custom isolate kuralı — risk onayı + audit; bait simülasyon testi zorunlu |

### P3 — Ertele / observe-only

| ID | İş | Not |
|----|-----|-----|
| P3-1 | Clipboard / typing-speed | Otomatik yok |
| P3-2 | Temp/Public Suspended-start | Default off |
| P3-3 | Ransomware hash intel | P0 kırmızıya besler |
| P3-4 | Marka/rename | PRODUCT_BRANDING final |
| P3-5 | Policy full encrypt | İsteğe bağlı |

---

## 5. Mevcut temel (önkoşul)

| Özellik | Sürüm / kontrat |
|---------|-----------------|
| VSS delete intent → kill + quarantine | ≥4.9.14 / 1.4.16 |
| Soft network surface inform | ≥4.9.15 / 1.4.17 |
| NG maintenance + golden + subtractive restore | ≥4.9.12 / 1.4.14–15 |
| Golden poison yasağı | 1.4.14 |
| System Recovery | ≥4.9.12 / 1.4.13 |
| Process contain hard-off (bomb-path) | ≥4.7.3 |
| successful_logon soft / no bare auto-block | ≥4.9.7 |
| Remote Desktop v2 / JPEG | ≥4.9.0 |
| Alert signal hygiene + dedupe | ≥4.9.11 |
| Command HMAC + destructive confirm | api/03 |

---

## 6. Bilinçli red listesi

1. Default **oto Ethernet kes** (canary dokununca)  
2. Clipboard / rubber-ducky **auto-suspend**  
3. Her Temp EXE’yi **Suspended start** (fleet default)  
4. Soft inform / DHCP / kablo → urgent / under_attack  
5. Big-bang `HP-BLOCK` / path rename  
6. **Policy imza bozuk → agresif + ağı kes**  
7. Balanced’ta canary/VSS → `auto_isolate_network`  
8. **Tek sinyal / bait → isolate veya paranoid’e auto-escalate**  
9. **N alert sonra otomatik izolasyon** (fatigue escalate)  
10. Korumalı sistem PID’lerine matrix ile bile suspend  

---

## 7. Başarı metrikleri (faz kapıları)

| Metrik | Hedef |
|--------|--------|
| Kırmızı FP (meşru VSS list / yedek) | Yanlış IFEO ≈ 0 |
| Balanced brick (RDP/DNS ölümü) | 0 |
| **Bait canary → isolate** | 0 (Balanced/Observe) |
| **Tamper → isolate / paranoid** | 0 |
| Sarı resume | &lt; 30 sn dashboard |
| Observe→Balanced CTA | Kurulum +14 gün |
| Snapshot | ≤1 / 5 dk / tetik ailesi |
| Policy tamper | LKG/observe ≤1 sn; ağ ayakta |
| Stale policy_version | ≤1 config cycle |
| Alert flood (aynı canary) | Tek urgent penceresi / dedupe |

---

## 8. Gelen öneriler kuyruğu

| Tarih | Kaynak | Özet | Karar | Faz |
|-------|--------|------|-------|-----|
| 2026-07-23 | Gemini | Impossible travel, clipboard, oto izolasyon, Temp suspend, session snapshot, tiered policy | Snapshot+policy+process-first; ağ/clipboard/Temp ertele | P0–P3 |
| 2026-07-23 | Ürün | Kırmızı process auto-suspend + resume/whitelist | Kabul | P0 |
| 2026-07-23 | Gemini | Policy matris, push/pull, rule engine, imzalı cache; tamper→agresif+ağ kes | Matris+sync+imza+LKG kabul; tamper/isolate default red | §3.1 · P0-1* |
| 2026-07-23 | Ürün | Saldırgan bilerek tetikleyerek self-imha / sinir ucu bait | Anti-bait birinci sınıf kısıt | §0 · §3.2 · P0-6 · P2-3 |
| | | *(sonraki öneriler…)* | | |

---

## 9. Sonraki somut adım

1. Bu belge **final review** onayından sonra P0 kontrat taslağı  
2. P0: matrix + signed cache + resume/allow + snapshot + **anti-bait testleri**  
3. Lab: Balanced canary/VSS + bilinçli canary spam + policy tamper (ağ ayakta)  
4. Cloud UI: preset → sonra P1 matrix editor  

**Uygulama başlamadan:** §0 + §3.2 + §6 ihlal edilmez.

---

## 10. Final review checklist (2026-07-23)

| # | Soru | Durum |
|---|------|--------|
| 1 | Process-first otomasyon (suspend/kill+Q) balanced’ta var mı? | Evet — P0 |
| 2 | Ağ izolasyonu default off mu? | Evet — P2 / paranoid only |
| 3 | Soft surface inform (mavi) shipped referansı var mı? | Evet — 1.4.17 / 4.9.15 |
| 4 | Policy matris + push/pull + imza planlı mı? | Evet — §3.1 / P0-1* |
| 5 | Tamper fail-mode brick üretmiyor mu? | Evet — LKG/observe |
| 6 | Saldırgan bait / self-DoS ele alındı mı? | Evet — §3.2 / P0-6 |
| 7 | Tek canary → isolate yasak mı? | Evet — red list #8 · P2-3 |
| 8 | Resume/whitelist çıkış kapısı var mı? | Evet — P0-3 |
| 9 | Observe onboarding panik azaltıyor mu? | Evet — §3 |
| 10 | Clipboard/Temp auto-aksiyon ertelendi mi? | Evet — P3 |
| 11 | Mevcut VSS/canary/NG/SR temeli yeniden icat edilmiyor mu? | Evet — §5 |
| 12 | Faz sırası brick-safe mi? (P0 process → P1 sarı → P2 ağ) | Evet |

**Review sonucu:** Yol haritası uygulama için yeterli olgunlukta.  
İlk kod/kontrat sprint’i = **P0** (matrix + anti-bait + resume + snapshot).  
P2 isolate, anti-bait testleri geçmeden production default olamaz.
