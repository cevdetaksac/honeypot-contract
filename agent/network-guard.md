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

## A) Ağ baseline yedeği

Motor boot'ta ve periyodik (öneri **≤ 30 dk**) ağ durumunu snapshot'lar. Anlamlı değişiklikte
yeni sürüm yazılır (son **N=10** sürüm saklanır). Dosya: `ProgramData\...\network_baseline.json`,
`sig` = HMAC (aynı `security.command_signing` anahtarı) ile imzalı (kötücül üzerine yazma tespiti).

Kapsanan durum:

| Alan | Kaynak | İçerik |
|------|--------|--------|
| `mapped_drives[]` | `HKCU\Network\*` (aktif kullanıcılar) + `net use` | harf, UNC, kullanıcı, persistent bayrağı |
| `shares[]` | `net share` / `HKLM\...\LanmanServer\Shares` | ad, path, izinler |
| `adapters[]` | `Get-NetIPConfiguration` / WMI | ad, durum (up/down), IPv4, gateway, DNS, metric |
| `routes[]` | route table | hedef, mask, gateway, metric |
| `firewall` | `netsh advfirewall` profilleri | domain/private/public state + default in/out policy |
| `connectivity` | ping/DNS probe | internet_ok, dns_ok, gateway_ok |

```json
{
  "version": 7,
  "captured_at": "2026-07-21T08:00:00Z",
  "mapped_drives": [{ "letter": "Z:", "unc": "\\\\srv\\share", "user": "…", "persistent": true }],
  "shares": [{ "name": "Data", "path": "D:\\Data" }],
  "adapters": [{ "name": "Ethernet", "state": "up", "ipv4": "10.0.0.5", "gateway": "10.0.0.1", "dns": ["10.0.0.1"] }],
  "firewall": { "domain": "on", "private": "on", "public": "on" },
  "connectivity": { "internet_ok": true, "dns_ok": true, "gateway_ok": true },
  "sig": "<hmac>"
}
```

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

Malware `interneti kesti` durumunu tersine çevir — **daemon buluta yeniden bağlanıp
alarm atabilsin** diye:

| Adım | Aksiyon | Kaynak |
|------|---------|--------|
| Adapter | down/disable edilmiş adapter'ı enable et | baseline `adapters[].state` |
| DNS | değiştirilmiş DNS'i baseline'a geri al | baseline `adapters[].dns` |
| Firewall | flush/kapatılmış profili baseline'a geri al | baseline `firewall` |
| Route | silinen default route'u geri ekle | baseline `routes[]` |
| Mapped drive | kopan `net use` bağlantılarını baseline'dan yeniden kur | baseline `mapped_drives[]` |
| Shares | silinen paylaşımları geri kur | baseline `shares[]` |

Ağ geri yükleme de otomatik değildir. Operatör `network_restore` için açık onay
verir; geri yükleme sonrası bağlantı doğrulanır (`connectivity` re-probe).
`mapped_drives` yeniden kurulurken **kimlik bilgisi güvensiz saklanmaz**; yalnız
persistent/OS-kayıtlı bağlantılar restore edilir.

Dashboard komutları (control WS, [`../api/03-control-websocket.md`](../api/03-control-websocket.md)):
`network_snapshot` (anlık baseline al), `network_restore` (baseline'dan geri yükle — `REQUIRES_CONFIRMATION`),
`list_network_baseline`.

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
- `network_restored` + `network_restore_actions` dashboard-live alanları → modalda “bağlantı otomatik kurtarıldı” rozeti.
- Dedupe: aynı `trigger` için **60 sn** pencere (`routes_v4._find_recent_duplicate_urgent`).
- Komutlar: `network_snapshot` / `list_network_baseline` (whitelist) + `network_restore` (**destructive confirm**).
- Host'u `under_attack` yalnız doğrulanmış/operatör-contain edilmiş `_bomb` olayında
  işaretle; `_suspect` warning tek başına bu bayrağı açmaz.
- `protection.network_guard{}` threats/config poll + update. **Alanlar (client ≥4.7.3):**
  `enabled`, `auto_contain`, `auto_kill`, `auto_restore`, `require_strong_signal`,
  `score_threshold`, `fs_write_bytes_per_sec`, `fs_write_count_per_sec`.
  `auto_contain`, `auto_kill`, `auto_restore` alanları wire compatibility için
  kalır ama client bunları hard-false yapar; cloud ile açılamaz.
- Health snapshot `network_guard` + `persistence` → settings persist (client_status rozeti).

---

## STATUS / health genişletmesi

Motor STATUS (`:58632`) ve `POST /api/health/report` snapshot'ına eklenir:

```json
"network_guard": {
  "enabled": true,
  "baseline_version": 7,
  "baseline_age_sec": 540,
  "internet_ok": true,
  "mapped_drives": 2,
  "suspended_processes": 0,
  "last_trigger_ts": null,
  "auto_contain": false,
  "auto_restore": false,
  "auto_kill": false
}
```

---

## Acceptance

- [x] Boot + periyodik ağ baseline yazılır, imzalı, N sürüm saklanır
- [x] `net_cut` yalnız gerçek internet erişim kaybında True (adapter/VPN churn tek başına tetiklemez) — **FP koruması (client ≥4.7.2)**
- [x] FS fırtınası (yazma hız) eşik aşımı tespit edilir (yüksek eşik; salt sinyal olarak)
- [x] Ağ kesme + FS fırtınası → canary beklemeden **ALARM** (`ransomware_offline_suspect`, warning) — containment DEĞİL
- [x] **Hiçbir süreç otomatik dondurulmaz**; config ile açılamayan hard safety invariant (client ≥4.7.3)
- [x] Client `suspend_process` / `resume_process`: PID+image+path+start-time doğrulamalı; suspend confirm-gated
- [x] Cloud popup/satır: açık onaylı **Suspend/Resume** butonu + exact-identity (pid+image+start-time) — cloud implemented; canlı client ile uçtan uca doğrulama bekliyor
- [ ] Acil VSS snapshot yalnız ayrı/onaylı aksiyon sırasında best-effort alınır
- [x] Ağ restore yalnız `network_restore` açık onayıyla çalışır
- [x] Geri yükleme sonrası bağlantı doğrulanır ve `ransomware_offline_bomb` urgent alarmı gider (dashboard popup detaylı, `restored` işaretli)
- [ ] Yedekleme/AV whitelist + `_PROTECTED_IMAGES` skorlamadan muaf (FP koruması)
- [x] `network_snapshot` / `network_restore` / `list_network_baseline` komutları çalışır (`network_restore` = confirm)
- [x] STATUS/health `network_guard` bloğu dolu
