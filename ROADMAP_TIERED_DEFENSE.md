# Tiered Defense & Resilience Roadmap

> **Status:** Planning (living document)  
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

**Özet tez:**  
LockBit insan onayını beklemez — **doğru**.  
Ama “ağı öldür” default brick + müşteri paniğidir.  
**Süreci dondurmak ≠ sunucuyu kör etmek.** OOB/cloud erişimi process karantinasını güvenli kılar; ağ izolasyonu yalnız bilinçli politika veya armed panic ile gelir.

---

## 1. Kaynak öneriler (özet)

### 1.1 Gemini — RDP / kimlik / izolasyon / forensics

| Öneri | Değerlendirme | Yol haritası |
|-------|---------------|--------------|
| Impossible travel (alışılmadık geo/ASN + başarılı RDP) | Soft alarm uyumlu; bare `successful_logon` zaten soft | **P1** — `suspicious_session` |
| Clipboard / süper hızlı yazım → auto-suspend | Yüksek FP (admin PS yapıştırır); brick | **P3 observe-only** — otomatik yok |
| Canary / VSS → oto ağ izolasyonu (yalnız cloud IP) | Brick riski; internet-koruma hedefiyle çelişir | **P2** — yalnız Paranoid / armed panic |
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
   surface_inform          ask operator            (+ isolate only if armed)
```

### 2.1 Kırmızı — yüksek kesinlik

**Tetik (dar allowlist):**

- Canary yazma / silme / bozma  
- VSS wipe intent (`vssadmin delete shadows` / WMI / wbadmin) — mevcut ≥4.9.14  
- (sonra) bilinen ransomware hash / intel imza  

**Aksiyon (Balanced+):**

1. Process suspend **veya** kill (mevcut VSS/canary yolunu bozma)  
2. Quarantine arm (IFEO — admin VSS araçlarına değil)  
3. Urgent alert (`force_urgent` yalnız kırmızı)  
4. Session JPEG snapshot (rate-limit)  
5. **Ağ kesme: default kapalı**  

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
| **paranoid** | Kırmızı+sarı agresif | Opsiyonel isolate (müşteri arm + risk onayı) | Kritik DB |

**Cloud alan (taslak):** `protection.defense_policy` = `observe|balanced|paranoid`  

**Kurallar:**

- Yeni kurulumda UI **observe** önerir (1–2 hafta), sonra balanced’a geçiş CTA  
- Paranoid network isolate kurulurken açık “brick/RDP kesilir” onayı  
- Client: process `auto_contain` bomb-path hard-false kalır; kırmızı yol `auto_suspend_critical` / mevcut RS yolları ile gider  

---

## 4. Fazlar (uygulama sırası)

### P0 — Güçlendirir, brick riski düşük *(sonraki sprint adayı)*

| ID | İş | Client | Cloud | Kontrat |
|----|-----|--------|-------|---------|
| P0-1 | `defense_policy` + `auto_suspend_critical` alanları | apply config | threats/config + UI seçici | `agent/` + WS |
| P0-2 | Kırmızı process netleştirme (canary/VSS + critical suspend bayrağı) | RS / NG ayrımı dokümante | alert routing | ransomware-shield + policy |
| P0-3 | Operatör çıkış: resume / allow (whitelist path+hash) | komutlar | confirm UI | 03-control-websocket |
| P0-4 | Session snapshot (1 JPEG, dedupe ≥5 dk, şüpheli tetik) | RD/capture | urgent attachment / store | api/05 + agent |
| P0-5 | Soft vs urgent hijyen (kırmızı dışı `under_attack` yok) | alert pipeline | dashboard | hygiene doc |

**Kabul kriteri:** Balanced lab’de canary/VSS → process durur, RDP/internet ayakta, yönetici resume ile devam.

### P1 — Sarı bölge + kimlik

| ID | İş |
|----|-----|
| P1-1 | Impossible travel / suspicious RDP (`suspicious_session`, soft) |
| P1-2 | Sarı: tek-PID suspend + dashboard İzin ver / Öldür |
| P1-3 | GUI chip: “Süreç donduruldu — devam ettir” (PIN yok; kill’de confirm) |
| P1-4 | Geo/ASN baseline (host veya hesap) — privacy-min |

**Kabul kriteri:** Meşru admin PS yapıştırması ağ/RDP’yi düşürmez; sarı yanlış alarmda 1 tık resume.

### P2 — Ağ izolasyonu (bilinçli)

| ID | İş |
|----|-----|
| P2-1 | `isolate_host` / `lift_isolation` (confirm) — cloud IP allowlist |
| P2-2 | Yalnız `paranoid` **veya** armed panic; default off |
| P2-3 | Isolate sonrası OOB mesaj + NG/System Recovery geri dönüş |
| P2-4 | Dual-NIC / VPN lab matrisi (Radmin vb.) |

**Kabul kriteri:** Observe/Balanced’ta asla sürpriz ağ kesilmez; Paranoid’te tek tık lift.

### P3 — Ertele / observe-only

| ID | İş | Not |
|----|-----|-----|
| P3-1 | Clipboard / typing-speed | Otomatik aksiyon yok |
| P3-2 | Temp/Public Suspended-start | Default off; shadow metrics |
| P3-3 | Bilinen ransomware hash intel genişletme | P0 kırmızıya besler |
| P3-4 | Marka/rename | Ayrı final iş (PRODUCT_BRANDING) |

---

## 5. Mevcut temel (zaten elimizi güçlendiren)

Bunlar yol haritasının **önkoşulu**; yeniden icat edilmez:

| Özellik | Sürüm / kontrat |
|---------|-----------------|
| VSS delete intent → kill + quarantine | client ≥4.9.14 / 1.4.16 |
| Soft network surface inform (additive) | client ≥4.9.15 / 1.4.17 |
| NG maintenance + golden baseline + subtractive restore | ≥4.9.12 / 1.4.14–15 |
| System Recovery (policy/service/firewall) | ≥4.9.12 / 1.4.13 |
| Process contain hard-off (bomb-path) | ≥4.7.3 |
| successful_logon soft skor / no bare auto-block | ≥4.9.7 / 1.4.9+ |
| Remote Desktop v2 / JPEG | ≥4.9.0 |
| Alert signal hygiene | ≥4.9.11 |

---

## 6. Bilinçli red listesi (şimdilik yapma)

1. Default **oto Ethernet kes** (canary dokununca)  
2. Clipboard / rubber-ducky **auto-suspend**  
3. Her Temp EXE’yi **Suspended start** (fleet default)  
4. Soft inform / DHCP / kablo takma → urgent / under_attack  
5. Big-bang `HP-BLOCK` / path rename (branding finalde)

---

## 7. Başarı metrikleri (faz kapıları)

| Metrik | Hedef |
|--------|--------|
| Kırmızı FP (meşru admin VSS list / yedek) | Quarantine yanlış arm ≈ 0 (IFEO admin tool yasağı korunur) |
| Balanced brick olayı (RDP/DNS ölümü) | 0 (ağ izole default off) |
| Sarı resume süresi (yönetici) | Dashboard’dan &lt; 30 sn |
| Observe→Balanced geçiş | Kurulum +14 gün CTA |
| Session snapshot boyutu / sıklık | ≤1 frame / 5 dk / tetik ailesi |

---

## 8. Gelen öneriler kuyruğu *(sonra ekle)*

> Yeni Gemini / ekip / müşteri önerileri buraya tarih + özet + (Tut/Ertele/Red) ile yazılır.  
> Onaylananlar ilgili P0–P3 tablosuna ID ile taşınır.

| Tarih | Kaynak | Özet | Karar | Faz |
|-------|--------|------|-------|-----|
| 2026-07-23 | Gemini | Impossible travel, clipboard, oto izolasyon, Temp suspend, session snapshot, tiered policy | Snapshot+policy+process-first kabul; ağ/clipboard/Temp ertele | P0–P3 yukarıda |
| 2026-07-23 | Ürün | Kırmızı process auto-suspend + resume/whitelist | Kabul | P0 |
| | | *(sonraki öneriler…)* | | |

---

## 9. Sonraki somut adım

1. Bu belgeyi gözden geçir · §8’e yeni önerileri ekle  
2. P0 için kontrat taslağı (`defense_policy`, snapshot, allow/resume) — ayrı PR  
3. Client lab: Balanced + canary/VSS + resume smoke  
4. Cloud UI: politika seçici + soft/urgent ayrımı  

**Uygulama başlamadan:** §0 filtresi + §6 red listesi ihlal edilmez.
