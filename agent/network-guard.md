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

- **Ağ kesildi + FS fırtınası birlikte → yüksek güven → canary'yi beklemeden agresif containment.**
- Whitelist: imzalı yedekleme/AV süreçleri + `_PROTECTED_IMAGES` skorlamadan muaf tutulur (FP azaltma).
- Eşikler `client_config.json` / dashboard `protection.network_guard{}` ile ayarlanabilir.

---

## C) Agresif containment (suspend-first)

Fire-and-forget'e karşı: ebeveyn çıkmış olabilir → anormal yazma fan-out'u olan **tüm**
şüpheli süreçler hedeflenir. **Varsayılan davranış = suspend** (kill değil):

1. **Suspend** — şüpheli süreçlerin thread'leri askıya alınır (`NtSuspendProcess`/thread suspend).
   Adli kayıt korunur, geri alınabilir, şifreleme durur.
2. **Acil VSS snapshot** — şifreleme yayılmadan mevcut/etkilenen sürücüde bir shadow copy oluşturulur (best-effort, time-boxed).
3. **Persistence blok** — encryptor'ın Run/RunOnce/Startup/Task persistence'ı temizlenir; imaj IFEO ile bloklanır.
4. **Operatör onayı** — suspend edilenler `pending_confirmation` listesine girer; dashboard onayıyla `kill` veya `release`.
5. **Otomatik yükseltme (opsiyonel)** — `protection.network_guard.auto_kill=true` ise yüksek skorda suspend yerine doğrudan kill+IFEO (varsayılan **kapalı**).

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

Otomatik geri yükleme davranışı `protection.network_guard.auto_restore` (varsayılan
**true**). Geri yükleme sonrası bağlantı doğrulanır (`connectivity` re-probe) ve **E**
alarmı gönderilir. `mapped_drives` yeniden kurulurken **kimlik bilgisi güvensiz saklanmaz**;
yalnız persistent/OS-kayıtlı bağlantılar restore edilir.

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
          "sha256": "…", "state": "suspended", "origin": "network_share" }
      ],
      "vss_emergency_snapshot": true,
      "ts": "2026-07-21T08:00:00Z"
    }
  },
  "raw_events": [
    { "kind": "network_guard", "trigger": "network_cut+fs_storm", "suspect_pid": 6644, "image": "invoice.exe", "state": "suspended" }
  ]
}
```

**Cloud beklenen davranış:**
- Kritik popup (kanary/tamper ile aynı detay motoru; `system_context.network_guard` + `raw_events` okunur).
- Host'u "under_attack" işaretle + operatöre **isolate + suspend edilenleri incele** öner.
- Dedupe: aynı `trigger` için kısa pencere (öneri 60 sn).
- `restored:true` ise "bağlantı otomatik kurtarıldı" rozeti göster.

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
  "auto_restore": true,
  "auto_kill": false
}
```

---

## Acceptance

- [ ] Boot + periyodik ağ baseline yazılır, imzalı, N sürüm saklanır
- [ ] Ağ kesme (adapter/DNS/firewall/route delta) tespit edilir
- [ ] FS fırtınası (write/rename/uzantı/entropi) eşik aşımı tespit edilir
- [ ] Ağ kesme + FS fırtınası → canary beklemeden containment tetiklenir
- [ ] Şüpheli süreçler varsayılan **suspend** edilir (kill değil); operatör onayıyla kill/release
- [ ] Acil VSS snapshot best-effort alınır
- [ ] `auto_restore` ile adapter/DNS/firewall/route/mapped-drive/shares baseline'dan geri yüklenir
- [ ] Geri yükleme sonrası bağlantı doğrulanır ve `ransomware_offline_bomb` urgent alarmı gider (dashboard popup detaylı, `restored` işaretli)
- [ ] Yedekleme/AV whitelist + `_PROTECTED_IMAGES` skorlamadan muaf (FP koruması)
- [ ] `network_snapshot` / `network_restore` / `list_network_baseline` komutları çalışır (`network_restore` = confirm)
- [ ] STATUS/health `network_guard` bloğu dolu
