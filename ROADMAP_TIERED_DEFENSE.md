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

### 1.4 Gemini — Policy delivery (JSON matris · sync · imza · tamper)

| Öneri | Değerlendirme | Yol haritası |
|-------|---------------|--------------|
| Enum yerine **rules matrix** payload | Enterprise / custom policy için doğru | **P0-1** · §3.1 |
| WS push + boot/heartbeat **pull** `policy_version` | Mevcut `threat_config_updated` ile uyumlu | **P0-1b** |
| Ajan “aptal itaatkâr” rule engine | Kabul; hard safety matris üstünde | **P0-1** |
| İmzalı yerel policy cache | Kabul (HMAC) | **P0-1c** |
| Tamper → en agresif + **ağı kes** | **Red** — brick vektörü; LKG/observe + alert | §6 · §3.1 Adım D |
| Balanced örnekte canary/VSS → `auto_isolate_network` | **Red default** — process `kill_quarantine` | §3.1 tablo |

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
`auto_isolate_network` (yalnız paranoid/armed — §0) · `ask_operator`

| Gemini örnek kural | Bizim karar |
|--------------------|-------------|
| `honeytoken_access: auto_isolate_network` | **Red (balanced default)** → `kill_quarantine` / suspend; isolate ≠ default |
| `vss_deletion: auto_isolate_network` | **Red (balanced)** → mevcut `kill_quarantine` (4.9.14) |
| `suspicious_rdp: suspend_process` | **P1** — balanced’ta önce `alert_only` veya `ask_operator`; paranoid’te suspend |
| `high_io_rate: suspend_process` | **Red default** → `alert_only` (FP); skor+origin ile sarıya yükselir |
| Custom “balanced ama X’te isolate” | **P2+** — override matrisi; UI risk onayı |

### Adım B — Senkronizasyon push + pull (**Kabul**)

| Mekanizma | Ne |
|-----------|-----|
| **Push** | Dashboard Kaydet → mevcut `threat_config_updated` (veya `defense_policy_updated`) → ajan `GET /api/threats/config` |
| **Pull / fail-safe** | Boot + periyodik: ajan `policy_version` bildirir; cloud stale ise güncel matrisi döner (heartbeat/health veya config GET) |
| **Offline** | Son **imzalı** matris ProgramData’da; internet yokken o uygulanır |

Yeni WS tipi şart değil; mevcut config push yeterli. İsim netliği için
`defense_policy` bloğu threats/config içinde taşınabilir.

### Adım C — Karar ağacı / “aptal ama itaatkâr” ajan (**Kabul**)

```
Event → RuleEngine.lookup(event_type) → action from signed matrix
       → alert_only | suspend | kill_quarantine | inform_only | …
```

- Zeka matriste; client kodu aksiyon primitiflerini uygular.  
- Hard safety **matrisin üstünde** kalır: örn. `auto_contain` bomb-path,
  IFEO-on-vssadmin yasakları, additive surface → asla isolate — cloud
  `auto_isolate_network` gönderse bile balanced/observe **reddeder** (client invariant).

### Adım D — Offline + imzalı config + tamper (**Kabul, fail-mode düzeltilerek**)

| Gemini önerisi | Değerlendirme |
|----------------|---------------|
| Policy JSON yerel cache | **Kabul** — ProgramData (Registry şart değil) |
| HMAC / imza ile bütünlük | **Kabul** — command/baseline ile aynı anahtar ailesi |
| İmza bozuk → **en agresif + ağı kes** | **Red** — saldırgan imzayı bozarak brick tetikleyebilir; FP = felç |

**Bizim tamper fail-mode (zorunlu):**

1. İmza geçersiz / dosya yok → **last-known-good** imzalı kopya (varsa)  
2. LKG yok → güvenli default **`observe`** (veya built-in balanced-without-isolate)  
3. Urgent `agent_tamper` / `defense_policy_tamper` — `under_attack` yalnız politika ile  
4. **Ağı kesme yok** (isolate yalnız §2 paranoid/armed)  
5. DACL / self-protection mevcut persistence modeline yaslanır  

> Gemini’nin “bankaya satılır olgunluk” hedefi imza + sync ile gelir;  
> “tamper = ağ öldür” ile değil.

### Bugünkü tamper durumu (kısa cevap)

- Komutlar: HMAC (`security.command_signing`)  
- Network/System Recovery baseline: HMAC  
- Guardian / motor stand-down: persistence-and-tamper  
- Threats/config apply: var; **defense rule matrix + signed policy cache** henüz yol haritasında (P0/P1)  
- Yerel config’i full encrypt şart değil; **imza + DACL + tamper alert** yeterli başlangıç

---

## 4. Fazlar (uygulama sırası)

### P0 — Güçlendirir, brick riski düşük *(sonraki sprint adayı)*

| ID | İş | Client | Cloud | Kontrat |
|----|-----|--------|-------|---------|
| P0-1 | `defense_policy` preset + **rules matrix** (balanced defaults §3.1) | apply + hard safety | threats/config + UI | `agent/` + threats/config |
| P0-1b | Push: `threat_config_updated` sonrası matrix apply; pull: boot/`policy_version` | CONFIG-SYNC | version compare | 03-control-websocket |
| P0-1c | İmzalı yerel policy cache + LKG; tamper → observe/LKG (**not** isolate) | ProgramData + verify | sign on emit | persistence-and-tamper |
| P0-2 | Kırmızı process netleştirme (canary/VSS + critical suspend bayrağı) | RS / NG ayrımı | alert routing | ransomware-shield + policy |
| P0-3 | Operatör çıkış: resume / allow (whitelist path+hash) | komutlar | confirm UI | 03-control-websocket |
| P0-4 | Session snapshot (1 JPEG, dedupe ≥5 dk, şüpheli tetik) | RD/capture | urgent attachment / store | api/05 + agent |
| P0-5 | Soft vs urgent hijyen (kırmızı dışı `under_attack` yok) | alert pipeline | dashboard | hygiene doc |

**Kabul kriteri:** Balanced lab’de canary/VSS → process durur, RDP/internet ayakta, yönetici resume ile devam. Policy imzası bozulunca ağ kesilmez; tamper alert + observe/LKG.

### P1 — Sarı bölge + kimlik + custom rules UI

| ID | İş |
|----|-----|
| P1-1 | Impossible travel / suspicious RDP (`suspicious_session`, soft) |
| P1-2 | Sarı: tek-PID suspend + dashboard İzin ver / Öldür |
| P1-3 | GUI chip: “Süreç donduruldu — devam ettir” (PIN yok; kill’de confirm) |
| P1-4 | Geo/ASN baseline (host veya hesap) — privacy-min |
| P1-5 | Dashboard rule matrix editor (preset + güvenli override; isolate risk banner) |

**Kabul kriteri:** Meşru admin PS yapıştırması ağ/RDP’yi düşürmez; sarı yanlış alarmda 1 tık resume.

### P2 — Ağ izolasyonu (bilinçli)

| ID | İş |
|----|-----|
| P2-1 | `isolate_host` / `lift_isolation` (confirm) — cloud IP allowlist |
| P2-2 | Yalnız `paranoid` **veya** armed panic; matrix’te `auto_isolate_network` yalnız bu modlarda apply |
| P2-3 | Isolate sonrası OOB mesaj + NG/System Recovery geri dönüş |
| P2-4 | Dual-NIC / VPN lab matrisi (Radmin vb.) |
| P2-5 | Custom policy: “şu tetikte isolate” — zorunlu risk onayı + audit log |

**Kabul kriteri:** Observe/Balanced’ta asla sürpriz ağ kesilmez; Paranoid’te tek tık lift.

### P3 — Ertele / observe-only

| ID | İş | Not |
|----|-----|-----|
| P3-1 | Clipboard / typing-speed | Otomatik aksiyon yok |
| P3-2 | Temp/Public Suspended-start | Default off; shadow metrics |
| P3-3 | Bilinen ransomware hash intel genişletme | P0 kırmızıya besler |
| P3-4 | Marka/rename | Ayrı final iş (PRODUCT_BRANDING) |
| P3-5 | Policy dosyası full encrypt (imza ötesi) | İsteğe bağlı; DACL+HMAC önce |

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
6. **Policy imza bozuk → en agresif + ağı kes** (Gemini fail-mode) — brick vektörü  
7. Balanced preset’te `*_access` / `vss_deletion` → `auto_isolate_network`  

---

## 7. Başarı metrikleri (faz kapıları)

| Metrik | Hedef |
|--------|--------|
| Kırmızı FP (meşru admin VSS list / yedek) | Quarantine yanlış arm ≈ 0 (IFEO admin tool yasağı korunur) |
| Balanced brick olayı (RDP/DNS ölümü) | 0 (ağ izole default off) |
| Sarı resume süresi (yönetici) | Dashboard’dan &lt; 30 sn |
| Observe→Balanced geçiş | Kurulum +14 gün CTA |
| Session snapshot boyutu / sıklık | ≤1 frame / 5 dk / tetik ailesi |
| Policy tamper (imza bozuk) | Ağ kesilmez; tamper alert + LKG/observe ≤ 1 sn apply |
| Stale `policy_version` (push kaçmış) | Boot/pull ile ≤ 1 config cycle’da düzelir |

---

## 8. Gelen öneriler kuyruğu *(sonra ekle)*

> Yeni Gemini / ekip / müşteri önerileri buraya tarih + özet + (Tut/Ertele/Red) ile yazılır.  
> Onaylananlar ilgili P0–P3 tablosuna ID ile taşınır.

| Tarih | Kaynak | Özet | Karar | Faz |
|-------|--------|------|-------|-----|
| 2026-07-23 | Gemini | Impossible travel, clipboard, oto izolasyon, Temp suspend, session snapshot, tiered policy | Snapshot+policy+process-first kabul; ağ/clipboard/Temp ertele | P0–P3 |
| 2026-07-23 | Ürün | Kırmızı process auto-suspend + resume/whitelist | Kabul | P0 |
| 2026-07-23 | Gemini | Policy JSON matrisi, push/pull sync, rule engine, imzalı cache; tamper→agresif+ağ kes | Matris+sync+imza+LKG **kabul**; balanced isolate default **red**; tamper→isolate **red** | §3.1 · P0-1* · P1-5 · P2-5 |
| | | *(sonraki öneriler…)* | | |

---

## 9. Sonraki somut adım

1. Bu belgeyi gözden geçir · §8’e yeni önerileri ekle  
2. P0 kontrat taslağı: `defense_policy` + **rules matrix** + signed cache + snapshot + allow/resume  
3. Client lab: Balanced + canary/VSS + resume; policy tamper smoke (ağ ayakta kalmalı)  
4. Cloud UI: preset seçici → sonra P1 matrix editor  

**Uygulama başlamadan:** §0 filtresi + §6 red listesi ihlal edilmez.
