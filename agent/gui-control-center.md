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
- **Tehdit Merkezi:** olaylar, şüpheli süreçler, hesaplar, paylaşımlar, servisler,
  komut geçmişi; satır aksiyonları.
- **Honeypot Servisleri:** her servis için durum + start/stop/detail.
- **Güvenlik Katmanları:** açık/kapalı katmanlar, kısa insan-dili açıklaması,
  cloud sync durumu.

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
