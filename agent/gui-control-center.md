# GUI Control Center — UX + live config contract

> **Client:** Windows GUI ≥4.7.3  
> **Amaç:** Teknik olmayan kullanıcıya “ne korunuyor, şu an ne oluyor ve hangi
> aksiyonu alabilirim?” sorularını tek bakışta cevaplamak.

## Temel UX ilkeleri

1. **Önce durum, sonra detay, sonra aksiyon:** Her kart anlaşılır bir değer +
   sağlık rengi taşır; tıklanınca aynı veri kümesinin detay görünümü açılır.
2. **Sayaç tek kaynak:** Kart ve popup aynı canonical snapshot'tan hesaplanır.
   Kart `N` gösteriyorsa popup başlığı da `N` göstermeli; filtreler ayrıca
   `gösterilen M / toplam N` yazmalıdır.
3. **Her satır eyleme hazır:** Uygulanabilir her tablo satırında `Detay` ve
   bağlama göre `Block / Whitelist / Unblock / Suspend / Resume / Kill` bulunur.
   Yıkıcı aksiyonlar açık onay ister.
4. **Asenkron UI:** OS taraması/API çağrısı Tk/UI thread'ini bloklamaz. Loading,
   success ve rollback/error durumları görünürdür.
5. **Cloud source of truth:** Ayar toggle'ı önce `POST /api/threats/config` ile
   kaydedilir. Başarıdan sonra dönen effective config gösterilir; hata halinde
   switch eski değerine döner.

## Navigasyon

Minimum ana bölümler:

- **Anlık Durum:** canlı bağlantı/motor, saldırı, servis, kaynak ve koruma kartları.
  ≥4.8.0: **Koruma Durumu şeridi** — motor, ransomware shield, network guard,
  guardian, honeypot ve karantina chip'leri daemon `STATUS`'tan beslenir;
  her chip ilgili detay/sekmeye kısayoldur.
- **Tehdit Merkezi:** olaylar, şüpheli süreçler, hesaplar, paylaşımlar, servisler,
  komut geçmişi; satır aksiyonları.
- **Honeypot Servisleri:** her servis için durum + start/stop/detail.
- **Güvenlik Katmanları:** açık/kapalı katmanlar, kısa insan-dili açıklaması,
  cloud sync durumu.
- **Ayarlar (≥4.8.0):** e-posta bildirimleri, otomatik engelleme limitleri,
  sessiz saatler ve webhook. Kaydet → `POST /api/threats/config` (partial
  deep-merge) → effective config yeniden okunur. GUI hiçbir ayarı yerelde
  saklamaz; bulut tek kaynaktır.

## Ayarlar sekmesi wire (≥4.8.0)

GUI şeması `client_settings_util.SECTIONS`'tan üretilir; kaydetmeden önce
`build_threat_config_patch` doğrular (saat `HH:MM`, int aralıkları, webhook
URL şeması). Yönetilen alanlar:

```json
POST /api/threats/config
{
  "alert_email_enabled": true,
  "instant_email_for_critical": true,
  "min_severity_for_email": "medium",
  "daily_digest_enabled": false,
  "auto_block_enabled": true,
  "auto_block_threshold": 3,
  "auto_block_duration_hours": 0,
  "max_auto_blocks_per_hour": 20,
  "max_auto_blocks_per_day": 100,
  "silent_hours": { "enabled": false, "mode": "night_only",
                     "night_start": "00:00", "night_end": "07:00" },
  "webhook_enabled": false,
  "webhook_url": ""
}
```

Geçersiz alanlar patch'e girmez ve kullanıcıya alan adlarıyla raporlanır.

## Toggle render invariantı (≥4.8.0)

CustomTkinter `CTkSwitch.select()/deselect()` widget `disabled` iken **no-op**.
Cloud'dan okunan durumu çizerken sıra her zaman: `state="normal"` → knob set.
Aksi halde tüm toggle'lar config değerinden bağımsız KAPALI görünür (4.7.x'te
yaşanan "SAFE ama katman kapalı görünüyor" hatası). Katmanlar ve Ayarlar
sekmeleri her ziyarette cloud'dan yeniden eşitlenir; bayat görünüm yasaktır.

## Detay popup veri-kaynağı invariantı (≥4.8.1)

Bir kart/chip ve onun detay popup'ı **aynı gerçeği** göstermek zorundadır.
Frontend-only GUI (Session ≠ 0) hiçbir koruma motorunun yerel nesnesini tutmaz;
`process_protection`, `ransomware_shield`, `network_guard`, `memory_guard` gibi
alanlar bu süreçte daima `None`'dır. Bu yüzden koruma durumunu gösteren tüm
popup'lar da chip/kart ile aynı kaynağı — daemon `STATUS` (IPC) — kullanmalıdır;
yerel nesne varlığına `if obj:` şeklinde bakmak yasaktır.

- **Koruma / Koruma Motoru** popup'ı `STATUS.motor_ok` + `STATUS.persistence.
  self_protection` okur (yerel `process_protection` fallback'tir, tek kaynak
  değildir). 4.8.0'da chip **AKTİF** derken popup yereldeki `None` nesne yüzünden
  **OFF** gösteriyordu; 4.8.1 bunu giderir.
- Ransomware / karantina popup'ı `RS_STATUS` / `STATUS.rs_quarantine`,
  Guardian popup'ı `STATUS.persistence` okur.

Kural: kart yeşil/aktif ise popup da aynı durumu doğrulamalı; iki farklı okuma
yolu (biri IPC, biri yerel nesne) tutulmaz.

## Güvenlik Katmanları wire

GUI açılışında `GET /api/threats/config`; değişiklikte partial deep-merge:

```json
POST /api/threats/config
{
  "ransomware_protection_enabled": true,
  "canary_files_enabled": true,
  "protection": {
    "network_guard": { "enabled": true }
  }
}
```

- POST partial update'tir; gönderilmeyen sibling alanları silmez.
- Cloud effective config'i döndürür ve aynı client'ın daemon WS kanalına
  `{ "v":1, "t":"threat_config_updated", "revision":"…" }` push eder.
- Daemon hemen GET + runtime apply yapar; periyodik poll fallback'tir.
- GUI switch POST sürerken disabled + “kaydediliyor”; hata halinde rollback.
- Network Guard açıklaması açıkça: **“Yalnız bildir; otomatik suspend yapmaz.”**

Runtime uygulanabilen ilk katmanlar:

| Katman | Config | Davranış |
|---|---|---|
| Ransomware Shield | `ransomware_protection_enabled` | `start()` / `stop()` |
| Canary files | `canary_files_enabled` | monitor flag |
| Network Guard | `protection.network_guard.enabled` | detect/alert `start()` / `stop()` |

## Şüpheli süreç onay akışı

1. Client hızlıca `ransomware_offline_suspect` (warning) yollar; süreç
   `state:"observed"` kalır.
2. UI satırı PID, image, path, cmdline, write rate ve neden/skoru gösterir.
3. Kullanıcı **Suspend** tıklar → impact metni + açık onay.
4. Cloud `suspend_process` komutunu exact identity ile yollar:
   `pid`, `expected_image`, `expected_path`, `process_start_time`.
5. Başarıda satır `suspended`; aksiyonlar `Resume` ve `Kill` olur.
6. Client identity uyuşmazlığında `STALE_PROCESS_TARGET`; UI listeyi yeniler.

Tespit motoru hiçbir koşulda kendi kendine suspend/kill/network restore yapmaz.

## Sayaç kabul kriterleri

- Kart ve popup aynı fonksiyon/snapshot/revision kullanır.
- `tracked_ips = blocked IP union watching IP`; IP iki gruptaysa bir kez sayılır.
- Popup hem blocked hem watching satırlarını birlikte gösterir; bir grubun varlığı
  diğerini saklamaz.
- API sayacı gecikmişse kart “stale / son güncelleme …” gösterir; uydurma `0` yok.
- Her popup başlığında `Toplam N`; filtre varsa `Gösterilen M / Toplam N`.

## Acceptance

- [x] Client GUI'de Güvenlik Katmanları sekmesi + cloud-write rollback durumu
- [x] `threat_config_updated` WS client handler + immediate GET/apply
- [x] Tracked-IP kart/popup aynı union snapshot'ı
- [x] Client exact-identity `suspend_process` / `resume_process`
- [ ] Cloud POST deep-merge + effective response + WS broadcast
- [ ] Cloud popup/table Suspend→Resume/Kill uçtan uca
- [ ] Tüm kart/popup ve tablo/satır aksiyonları için data-source audit + UI testi
