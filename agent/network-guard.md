# Network Guard — offline fidye bombası + ağ sürücüsü yedek/kurtarma

> **Contract VERSION:** root `VERSION`
> **API base:** `https://honeypot.yesnext.com.tr`
> **Min client:** **≥ 4.7.0** (Network Guard)
> İlgili: [`ransomware-shield.md`](ransomware-shield.md) · [`persistence-and-tamper.md`](persistence-and-tamper.md) · Mimari: [`../api/08-architecture.md`](../api/08-architecture.md)

Amaç: **fire-and-forget + offline** fidye yazılımına karşı savunma. Saldırgan (veya
paylaşılan bir klasör) sunucuya bir exe/bat bırakır, başlatır ve çıkar; süreç kendi
kendine (1) **ağı/interneti keser** (uzaktan müdahaleyi + bulut alarmını engellemek
için), (2) C2'ye ihtiyaç duymadan **dosyaları şifreler**. Ebeveyn süreç çıktığından
"dropper'ı öldür" yetmez — asıl şifreleyen işçi süreç ayrı ve internetsiz çalışır.

Network Guard beş parçadan oluşur: **A** baseline yedek · **B** offline davranışsal
tespit · **C** agresif containment (suspend-first) · **D** ağ/bağlantı kurtarma · **E**
alarm & istihbarat.

> **Dürüst sınır:** Bu bir tam EDR/AV değildir. Davranışsal kütle-şifreleme tespiti
> false-positive üretebilir (yedekleme yazılımı, toplu dosya işlemleri). Zero-day
> encryptor'da **bazı dosyalar şifrelenmeden** garanti yakalama yoktur. Sözleşmenin
> garantisi: **erken containment + kurtarılabilirlik** (baseline + VSS), kusursuz önleme
> değil. Eşikler ayarlanabilir, varsayılanlar güvenli (suspend-first) olmalıdır.

---

## A) Ağ baseline yedeği (golden)

Motor boot'ta **ilk** golden baseline'ı yazar. Son **N=10** sürüm saklanır.
Dosya: `ProgramData\...\network_baseline.json`,
`sig` = HMAC (aynı `security.command_signing` anahtarı).

### Golden vs live

| Kavram | Kim yazar | Amaç |
|--------|-----------|------|
| **Golden baseline** | Boot (yoksa) + operatör `network_snapshot` | Geri yükleme kaynağı — “bilinen iyi” ağ |
| **Live** | Her STATUS / `list` / `diff` anında toplanır | Dashboard’da görünen anlık IP/adaptör/sürücü |
| **Periyodik loop** | Yalnız `connectivity` probe yeniler | **Saldırganın bozduğu IP/DNS’i golden’a yazmaz** |

> **Zehirlenme yasağı:** Periyodik baseline loop, adapter/IPv4/DNS/mapped-drive
> değişikliğini otomatik yeni golden sürüm yapamaz. Aksi halde saldırgan IP’yi
> bozar → 30 dk sonra “bozuk” hali “iyi” sayılır. Bilinçli IP/DNS değişiminde
> operatör **önce** `network_snapshot` alır (aşağıdaki iş akışı).

Kapsanan durum:

| Alan | Kaynak | İçerik |
|------|--------|--------|
| `mapped_drives[]` | `net use` (+ mümkünse `HKCU\Network\*`) | harf, UNC, persistent |
| `shares[]` | `net share` | ad, path (admin$ hariç) |
| `adapters[]` | `Get-NetIPConfiguration` | ad, state, ipv4, gateway, dns[], dhcp, prefix_length |
| `firewall` | `netsh advfirewall` | domain/private/public state |
| `connectivity` | TCP/DNS probe | internet_ok, dns_ok, gateway_ok |

```json
{
  "version": 12,
  "captured_at": "2026-07-23T12:30:00Z",
  "mapped_drives": [{ "letter": "Z:", "unc": "\\\\srv\\share", "persistent": true }],
  "shares": [{ "name": "Data", "path": "D:\\Data" }],
  "adapters": [{
    "name": "Wi-Fi", "state": "up", "ipv4": "192.168.1.30",
    "gateway": "192.168.1.1", "dns": ["1.1.1.1", "8.8.8.8"],
    "dhcp": true, "prefix_length": 24
  }],
  "firewall": { "domain": "on", "private": "on", "public": "on" },
  "connectivity": { "internet_ok": true, "dns_ok": true, "gateway_ok": true },
  "sig": "<hmac>"
}
```

### Operatör iş akışı — bilinçli IP/DNS değişimi

**Tercih edilen: bakım modu (1.4.15)**

1. Dashboard/GUI → **Korumayı durdur (bakım)** (`network_maintenance_start`)
2. Host’ta VPN / IP / DNS / mapped drive değişikliklerini yap
3. **Mevcut ayarları yedekle** (`network_snapshot`) veya doğrudan
   **Yedekle ve korumayı başlat** (`network_maintenance_end` + `snapshot:true`)
4. Client live ≈ yeni golden → drift yok

Bakım açıkken: detect loop ve `auto_restore_network` **kapalı**. Bayrak
`ProgramData\...\network_guard_maintenance.json` — reboot sonrası da kalır;
cloud `protection.network_guard.enabled=true` sync bakımı **silmez**.

**Kısa yol (bakım olmadan):** önce `network_snapshot`, sonra host değişikliği
(hata riski: snapshot unutulursa auto-restore eski golden’a çeker).

Saldırı senaryosu (bakım yok, snapshot yok): live ≠ golden **ve** değişiklik
**subtractive** (adapter down / DNS hijack / firewall off / static gasp) →
`auto_restore_network` (veya dashboard **Geri yükle**) golden’a döner.

### Additive vs subtractive (contract 1.4.17) — kullanıcıyı darlama

Ürün hedefi: **internete bağlı kalındığından emin olmak.**  
`connectivity.internet_ok == true` iken Ethernet kablosu / DHCP lease / yeni
adapter ayağa kalkması **panik değildir** — soft bilgilendirme yeter.

| Sınıf | Örnek | Client davranışı | Cloud / dashboard |
|-------|-------|------------------|-------------------|
| **Additive (inform)** | Adapter `disconnected→up`, yeni NIC, DHCP lease IP değişimi | Soft alert `network_surface_changed` (**info**, `force_urgent=false`); **asla** auto-disable; **asla** auto_restore | Soft banner / diff (mavi); **under_attack yok**; **Yeni golden kabul** / isteğe bağlı **Adaptörü kapat** (confirm) |
| **Subtractive (restore)** | Baseline’da `up` olan düştü, DNS değişti, firewall off, static IP gasp | `auto_restore_network` (default on) | Kırmızı drift; restore sonucu `network_surface_restored` |
| **Panic (net_cut)** | `internet_ok` true→false (≥ persist) | Offline skor / suspect alarm | Gerçek kesinti — panik burada |

Kurallar:

1. Additive değişiklik **otomatik adaptör kapatmaz** ve golden’ı zehirlemez.
2. Operatör / kullanıcı “bu bendim” derse → `network_accept_surface`
   (= `network_snapshot`) → live yeni golden olur.
3. Dashboard “bu adaptör enable kalmasın” derse → yalnız
   `network_disable_adapter` (**confirm**); client asla kendi başına disable etmez.
4. Yerel GUI: soft chip/toast + **Bu bendim → yedeği güncelle**. Inform yolunda
   **PIN / geri sayımlı zorlama yok** (kullanıcıyı darlama). PIN yalnız
   cloud confirm’li yıkıcı komutlarda (disable/restore) kalır.
5. Soft alert dedupe: aynı değişiklik parmak izi ≥ **15 dk** (flap yok).

```json
{
  "severity": "info",
  "threat_type": "network_surface_changed",
  "title": "Ağ yüzeyi genişledi (internet açık)",
  "description": "Ethernet up · 192.168.1.21",
  "threat_score": 15,
  "force_urgent": false,
  "recommended_action": "review_network",
  "system_context": {
    "network_guard": {
      "trigger": "surface_inform",
      "internet_ok": true,
      "inform_changes": [
        {"id": "adapter_up.Ethernet", "kind": "adapter_enabled",
         "interface": "Ethernet", "from": "disconnected", "to": "up",
         "ipv4": "192.168.1.21"}
      ]
    }
  }
}
```

**Cloud talimatı (zorunlu):**

- `network_surface_changed` → soft panel / toast; host’u `under_attack` yapma.
- Butonlar: **Yeni golden kabul et** (`network_accept_surface`) ·
  **Adaptörü kapat** (`network_disable_adapter`, confirm+PIN) · Yoksay.
- `internet_ok=false` / `ransomware_offline_*` / subtractive restore ayrı kanallar —
  soft inform ile karıştırma.
- Health: `network_guard.surface_inform` + `surface_inform_changes` göster;
  kırmızı `drift` yalnız subtractive.

---

## B) Offline davranışsal tespit (internetsiz)

Motor içi, internet gerektirmez. Bir **skorlama** penceresi (öneri 10 sn kayan pencere)
şu sinyalleri toplar; eşik aşılınca **C** tetiklenir:

| Sinyal | Ağırlık | Nasıl ölçülür |
|--------|---------|----------------|
| **Ağ ani kesildi** | yüksek | adapter down / DNS değişti / firewall flush / route silindi (A baseline'a göre delta) + `connectivity.internet_ok` true→false |
| **FS yazma/rename fırtınası** | yüksek | kısa sürede yüksek sayıda write/rename (öneri > 60 dosya/10 sn), farklı klasörlere yayılan |
| **Kütle uzantı değişimi / entropi** | orta | çok sayıda dosyaya `.encrypted`/rastgele uzantı ekleme veya yeniden adlandırma |
| **Şüpheli köken** | orta | süreç imajı `Temp`/`Downloads`/removable/UNC/network share yolundan çalışıyor |
| **Fidye notu deseni** | orta | kısa sürede çok klasörde `*README*`, `*DECRYPT*`, `*.txt/.hta` fidye notu oluşturma |
| **VSS silme** | yüksek | (mevcut) shadow copy azalması |
| **Canary** | kesin | (mevcut) canary'ye yazma → doğrudan yüksek skor |

- **Ağ kesildi + FS fırtınası birlikte → şüpheli → canary'yi beklemeden ALARM.**
  (client ≥4.7.2: bu kombinasyon *containment* değil, yalnız uyarı üretir.)
- **`net_cut` yalnız gerçek internet erişim kaybında** (`internet_lost`) True olur.
  Adapter down / VPN-Wi-Fi churn tek başına net_cut sayılmaz (false-positive koruması).
- Alarm ayrımı: containment yapılmayan durum `ransomware_offline_suspect`
  (severity **warning**); operatör onaylı containment `ransomware_offline_bomb`
  (severity **critical**).
- Whitelist: imzalı yedekleme/AV süreçleri + `_PROTECTED_IMAGES` skorlamadan muaf tutulur (FP azaltma).
- Eşikler `client_config.json` / dashboard `protection.network_guard{}` ile ayarlanabilir.

---

## C) Agresif containment (suspend-first)

> **GÜVENLİK (client ≥4.7.3):** Otomatik containment **yasak / hard-disabled**
> (`auto_contain=false`, `auto_kill=false`, `auto_restore=false`). Cloud config
> bu alanları `true` gönderse bile client uygulamaz. Sebep: salt
> "ağ-kesme + yazma fırtınası" sinyali meşru ağır-I/O uygulamalarında (tarayıcı,
> editör, oyun, yedekleme) **false-positive** üretip süreçleri dondurabilir
> (4.7.0/4.7.1 canlıda bunu yaşattı). Network Guard **yalnız alarm** gönderir;
> operatör popup/tablo satırından açık onay verince cloud `suspend_process`
> komutunu yollar. Ham yazma hızı veya yüksek güvenli sinyal dahil, tespit
> pipeline'ı kendi başına **asla** süreç dondurmaz.

Fire-and-forget'e karşı: ebeveyn çıkmış olabilir → anormal yazma fan-out'u olan **tüm**
şüpheli süreçler hedeflenir. **Onaylı davranış = suspend** (kill değil):

1. **Operatör onayı** — dashboard şüpheli süreç satırında “Suspend” seçer;
   cloud `confirm:true` gate'inden sonra `suspend_process` yollar.
2. **Suspend** — şüpheli süreçlerin thread'leri askıya alınır (`psutil.Process.suspend`).
   Adli kayıt korunur, geri alınabilir, şifreleme durur.
3. **Kill / release ayrı onay** — suspend sonrası operatör `kill_process` veya
   `resume_process` seçer. Otomatik kill yoktur.

PID reuse koruması zorunlu: alarmdaki her suspect
`pid + image + path + process_start_time` (Unix timestamp) taşır. Dashboard bu
alanları komuta aynen koyar; client işlem anında ad/path/start-time eşleşmezse
`STALE_PROCESS_TARGET` ile reddeder. Yalnız PID ile suspend/resume reddedilir.

Ransomware quarantine ile aynı `unlock`/`list` IPC + komut kanalı kullanılır
([`ransomware-shield.md`](ransomware-shield.md)). Suspend edilen süreçler quarantine
entry'lerine `state: "suspended"` ile eklenir.

---

## D) Ağ / bağlantı kurtarma

Malware ağ sürücülerini keser / IPv4-DNS bozar / adapter kapatır → golden’a dön:

| Adım | Aksiyon | Kaynak |
|------|---------|--------|
| Adapter | down/disable → enable | baseline `adapters[].state==up` |
| IPv4 | DHCP ise `dhcp`; static ise ipv4+prefix+gateway | baseline `adapters[].dhcp/ipv4/...` |
| DNS | baseline DNS listesine al | baseline `adapters[].dns` |
| Firewall | kapatılmış profili aç | baseline `firewall` |
| Mapped drive | persistent `net use` yeniden | baseline `mapped_drives[]` |
| Shares | (best-effort) silinen paylaşımları kur | baseline `shares[]` |

### `auto_restore_network` (network-surface only)

| Bayrak | Default | Anlam |
|--------|---------|--------|
| `auto_restore_network` | **true** (≥4.9.12) | Live≠golden **subtractive** (adapter down / DNS / IPv4 mode / mapped drive / firewall) → client **anında** restore. **Additive** (adapter up / DHCP lease) restore **etmez** (1.4.17). |
| `auto_contain` / `auto_kill` | hard **false** | Süreç dondurma/öldürme asla otomatik değil |
| Legacy `auto_restore` | hard **false** wire uyumu | Eski bomb-path alanı; cloud `auto_restore_network` kullanır |

- Intentional IP change: önce `network_snapshot` (yukarı). Snapshot yoksa auto-restore
  eski golden’a çeker — bu **istenen** güvenlik davranışıdır.
- Auto-restore sonrası alert: `threat_type` `network_surface_restored` (veya mevcut
  suspect alarmında `network.restored=true` + `restore_actions[]`).
- Dedupe: aynı yüzey için ≥ **5 dk** (flap yok).

### Dashboard / manuel restore

Operatör `network_restore` için açık onay verir (`confirm:true`);
`dry_run:true` plan-only (confirm yok); `rollback_version` history’den seçer.
`mapped_drives` yeniden kurulurken **kimlik bilgisi güvensiz saklanmaz**; yalnız
persistent/OS-kayıtlı bağlantılar restore edilir.

### Dashboard panel (cloud zorunlu UI — contract 1.4.14)

Dashboard host detayında **Ağ kurtarma** paneli:

1. **Canlı** — adaptör tablosu (ad, state, IPv4, gateway, DNS, dhcp)
2. **Son yedek (golden)** — version, `captured_at`, aynı tablo alanları, imza OK
3. **Diff** — `network_diff` sonucu (kırmızı satırlar)
4. **Aksiyonlar** — Yedek al (`network_snapshot`) · Plan (`network_restore` dry_run) ·
   Geri yükle (confirm) · Geçmiş sürüm seç (`rollback_version`)

Veri kaynağı: health/STATUS `network_guard.live` + `network_guard.baseline` **veya**
Control WS `list_network_baseline` / `network_diff` (tercihen komut — tam payload).

### Komutlar

| `type` | Params | Confirm | Min |
|--------|--------|---------|-----|
| `network_snapshot` | — | hayır | ≥4.7.0 |
| `network_accept_surface` | — | hayır | ≥4.9.15 (1.4.17) — additive “bu bendim”; `network_snapshot` ile aynı etki |
| `list_network_baseline` | — | hayır | ≥4.7.0 (rich ≥4.9.12) |
| `network_diff` | `version?` | hayır | ≥4.9.12 — `changes` (subtractive) + `inform_changes` (additive, ≥4.9.15) |
| `network_restore` | `targets[]?`, `dry_run?`, `rollback_version?` | mutate evet / dry_run hayır | ≥4.7.0 |
| `network_disable_adapter` | `name` (zorunlu), `dry_run?` | mutate evet / dry_run hayır | ≥4.9.15 — asla otomatik |

`targets[]`: `adapter` | `ipv4` | `dns` | `firewall` | `mapped_drive`

#### `list_network_baseline` result (rich)

```json
{
  "success": true,
  "data": {
    "version": 12,
    "captured_at": "2026-07-23T12:30:00Z",
    "verified": true,
    "baseline": {
      "adapters": [{ "name": "Wi-Fi", "state": "up", "ipv4": "192.168.1.30",
                     "gateway": "192.168.1.1", "dns": ["1.1.1.1"], "dhcp": true }],
      "mapped_drives": [],
      "shares": [],
      "firewall": { "domain": "on", "private": "on", "public": "on" }
    },
    "live": {
      "adapters": [{ "name": "Wi-Fi", "state": "up", "ipv4": "192.168.1.30",
                     "gateway": "192.168.1.1", "dns": ["1.1.1.1", "8.8.8.8"], "dhcp": true }],
      "mapped_drives": [],
      "firewall": { "domain": "on", "private": "on", "public": "on" },
      "connectivity": { "internet_ok": true, "dns_ok": true, "gateway_ok": true }
    },
    "drift": false,
    "changes": [],
    "history": [
      { "version": 11, "captured_at": "…", "verified": true }
    ]
  }
}
```

Dry-run / mutate result: `plan[]` + `connectivity`; mutate + `restore_actions[]`.
Errors: `no_baseline`, `baseline_signature_invalid`,
`rollback_baseline_not_found_or_invalid`.

---

## E) Alarm & istihbarat — `POST /api/alerts/urgent`

Canary/tamper ile aynı taşıyıcı. Ek `system_context.network_guard` bloğu:

```json
{
  "severity": "critical",
  "threat_type": "ransomware_offline_bomb",
  "title": "🔴 OFFLINE FİDYE BOMBASI — ağ kesildi + kütle şifreleme",
  "description": "…",
  "threat_score": 100,
  "target_service": "SYSTEM",
  "recommended_action": "isolate_host",
  "system_context": {
    "network_guard": {
      "trigger": "network_cut+fs_storm | fs_storm | vss_delete | canary",
      "score": 100,
      "network": {
        "internet_lost": true,
        "adapters_down": ["Ethernet"],
        "dns_changed": true,
        "firewall_flushed": false,
        "restored": true,
        "restore_actions": ["adapter_enable:Ethernet", "dns_restore:Ethernet", "netuse:Z:"]
      },
      "fs": { "writes_10s": 412, "renamed": 380, "ext_added": ".locked", "roots": ["D:\\Data", "C:\\Users\\..."] },
      "suspects": [
        { "pid": 6644, "image": "invoice.exe", "path": "C:\\Users\\Public\\invoice.exe",
          "process_start_time": 1784635200.25, "sha256": "…",
          "state": "observed", "origin": "network_share" }
      ],
      "vss_emergency_snapshot": true,
      "ts": "2026-07-21T08:00:00Z"
    }
  },
  "raw_events": [
    { "kind": "network_guard", "trigger": "network_cut+fs_storm", "suspect_pid": 6644, "image": "invoice.exe", "state": "observed" }
  ]
}
```

**Cloud davranış:**
- `ransomware_offline_suspect` popup/tablo: `system_context.network_guard.suspects[]`
  → süreç/PID/path/start-time + açık onaylı **Suspend** butonu.
- `suspend_process` komutu `confirm:true` olmadan kuyruğa/push'a yazılmaz.
- `ransomware_offline_bomb` yalnız operatör containment sonucunu temsil eder;
  tespit pipeline'ı bu tipi kendi başına üretmez.
- `network_surface_changed` (1.4.17): soft banner only; Accept / optional Disable;
  never `under_attack`; never auto-queue disable.
- `network_restored` + `network_restore_actions` dashboard-live alanları → modalda “bağlantı otomatik kurtarıldı” rozeti.
- Dedupe: aynı `trigger` için **60 sn** pencere (`routes_v4._find_recent_duplicate_urgent`).
- Komutlar: `network_snapshot` / `list_network_baseline` (whitelist) +
  `network_restore` (**mutating = destructive confirm**; `dry_run:true` =
  plan-only, confirm gerekmez — contract 1.4.5).
- Host'u `under_attack` yalnız doğrulanmış/operatör-contain edilmiş `_bomb` olayında
  işaretle; `_suspect` warning tek başına bu bayrağı açmaz.
- `protection.network_guard{}` threats/config poll + update. **Alanlar:**
  `enabled`, `auto_contain`, `auto_kill`, `auto_restore` (legacy hard-false),
  `auto_restore_network` (default **true**, cloud toggle), `require_strong_signal`,
  `score_threshold`, `fs_write_bytes_per_sec`, `fs_write_count_per_sec`.
  `auto_contain` / `auto_kill` client hard-false; `auto_restore_network` cloud ile
  kapatılabilir (bilinçli bakım penceresi).
- Health snapshot `network_guard` + `persistence` → settings persist (client_status rozeti).
- Dashboard panel: §D — live IP/adaptör + golden + diff + restore (1.4.14).

---

## STATUS / health genişletmesi

Motor STATUS (`:58632`) ve `POST /api/health/report` snapshot'ına eklenir
(≥4.9.12 rich; eski client’lar kısa özet gönderebilir):

```json
"network_guard": {
  "present": true,
  "enabled": true,
  "running": true,
  "baseline_version": 12,
  "baseline_age_sec": 540,
  "baseline_captured_at": "2026-07-23T12:30:00Z",
  "verified": true,
  "internet_ok": true,
  "drift": false,
  "drift_count": 0,
  "surface_inform": true,
  "surface_inform_count": 1,
  "surface_inform_changes": [
    {"id": "adapter_up.Ethernet", "kind": "adapter_enabled",
     "interface": "Ethernet", "from": "disconnected", "to": "up",
     "ipv4": "192.168.1.21"}
  ],
  "mapped_drives": 0,
  "suspended_processes": 0,
  "last_trigger_ts": null,
  "auto_contain": false,
  "auto_kill": false,
  "auto_restore": false,
  "auto_restore_network": true,
  "maintenance": false,
  "maintenance_started_at": null,
  "live": {
    "adapters": [
      { "name": "Wi-Fi", "state": "up", "ipv4": "192.168.1.30",
        "gateway": "192.168.1.1", "dns": ["1.1.1.1", "8.8.8.8"], "dhcp": true }
    ],
    "mapped_drives": [],
    "connectivity": { "internet_ok": true, "dns_ok": true, "gateway_ok": true }
  },
  "baseline": {
    "adapters": [
      { "name": "Wi-Fi", "state": "up", "ipv4": "192.168.1.30",
        "gateway": "192.168.1.1", "dns": ["1.1.1.1", "8.8.8.8"], "dhcp": true }
    ],
    "mapped_drives": [],
    "firewall": { "domain": "on", "private": "on", "public": "on" }
  }
}
```

Cloud health persist: `live.adapters` + `baseline` saklanır ki dashboard panel
komut beklemeden son bilinen IP’yi göstersin. Tam history için `list_network_baseline`.

---

## Acceptance

- [x] Boot golden baseline yazılır, imzalı, N sürüm saklanır
- [x] Periyodik loop golden adapter/IP/DNS’i saldırı ile zehirlemez (1.4.14)
- [x] `net_cut` yalnız gerçek internet erişim kaybında True (adapter/VPN churn tek başına tetiklemez) — **FP koruması (client ≥4.7.2)**
- [x] FS fırtınası (yazma hız) eşik aşımı tespit edilir (yüksek eşik; salt sinyal olarak)
- [x] Ağ kesme + FS fırtınası → canary beklemeden **ALARM** (`ransomware_offline_suspect`, warning) — containment DEĞİL
- [x] **Hiçbir süreç otomatik dondurulmaz**; config ile açılamayan hard safety invariant (client ≥4.7.3)
- [x] Client `suspend_process` / `resume_process`: PID+image+path+start-time doğrulamalı; suspend confirm-gated
- [x] Cloud popup/satır: açık onaylı **Suspend/Resume** butonu + exact-identity (pid+image+start-time) — cloud implemented; canlı client ile uçtan uca doğrulama bekliyor
- [ ] Acil VSS snapshot yalnız ayrı/onaylı aksiyon sırasında best-effort alınır
- [x] Ağ restore: manuel `network_restore` confirm; `dry_run:true` plan-only
- [x] `auto_restore_network` (default on): mapped drive / DNS / adapter / IPv4 mode drift → anında golden restore (1.4.14); **additive adapter-up / DHCP lease restore etmez** (1.4.17)
- [x] Soft `network_surface_changed` + `network_accept_surface` / `network_disable_adapter` (confirm) — internet açıkken panik yok (1.4.17)
- [x] Bilinçli IP değişimi: önce `network_snapshot` (dashboard panel)
- [x] Dashboard panel: live IP/adaptör + golden + diff + restore (cloud ≥1.4.14)
- [x] Geri yükleme sonrası bağlantı doğrulanır
- [ ] Yedekleme/AV whitelist + `_PROTECTED_IMAGES` skorlamadan muaf (FP koruması)
- [x] `network_snapshot` / `network_restore` / `list_network_baseline` / `network_diff` komutları
- [x] STATUS/health `network_guard` rich bloğu (adapters ipv4/dns)