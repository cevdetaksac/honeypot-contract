# Changelog — honeypot-contract

## 1.3.12 — 2026-07-21 (client **4.8.4** — whitelist cloud SoT invariantı)

- `agent/gui-control-center.md`: yeni **whitelist SoT invariantı** — whitelist
  tek kaynağı cloud `threats/config.whitelist_ips`. Persist merge-only (cloud
  seti + yerel engine setleri + açık add/remove deltası; kör overwrite yasak);
  tablo render'ı engine setleri ∪ cloud seti okur.
- Canlı 4.8.3 bulgusu: frontend-only GUI'de engine nesneleri `None` olduğundan
  hızlı whitelist ekleme buluta boş liste yolluyordu ("eklendi" toast'ı ama
  Whitelist (0) ve cloud wipe riski). Production floor 4.8.4.

## 1.3.11 — 2026-07-21 (client **4.8.3** — dashboard PIN yönetimi + IP hızlı aksiyonları)

- `api/03-control-websocket.md`: yeni komutlar **`set_gui_pin`** (`pin` 4-12
  hane; confirm gate; result/audit'te PIN maskeli — cloud `scrub_command_params`
  listesine `pin` eklenmeli) ve **`clear_gui_pin`** (confirm gate). Cloud TODO:
  `VALID_COMMAND_TYPES` + `DESTRUCTIVE_COMMAND_TYPES` + dashboard "GUI PIN
  tanımla/sıfırla" aksiyonu.
- GUI PIN store dış-değişiklik algılama: daemon `gui_lock.json`'ı yazınca GUI
  süreci mtime'dan yeniden yükler ve oturum kilidini düşürür (restart yok).
- `agent/gui-control-center.md`: hesap bağlıysa PIN diyaloglarında
  "dashboard'dan tanımla/sıfırla" ipucu (`pin_dashboard_hint`).
- IP Listeleri başlığına **＋ IP Engelle** / **＋ Whitelist'e Ekle** hızlı
  aksiyon butonları (modal IP girişi + `ipaddress` doğrulaması + PIN gate;
  satır aksiyonlarıyla aynı IPC/cloud yolu). Production floor 4.8.3.
- **Cloud implemented (honeypot.yesnext.com.tr, 2026-07-21):** `set_gui_pin` /
  `clear_gui_pin` whitelist (42 tip) + destructive confirm gate; `pin` alanı
  `scrub_command_params` ile audit/GET/result'ta `***` (komut kapanınca DB'de de
  maskeli); server-side `pin` format doğrulaması (4-12 hane, yalnız rakam → 400);
  dashboard Tehdit Merkezi → Remote Commands → **GUI PIN** modalı (tanımla /
  sıfırla, confirm dialoglu, TR/EN i18n).
- **Cloud dashboard IP hızlı aksiyonları (2026-07-21):** Bloklama sayfası
  başlığına **IP Engelle** (kırmızı, mevcut `#modalIPBlock`) ve **Whitelist'e
  Ekle** (yeşil, yeni `#modalWhitelistAdd`) butonları eklendi. Whitelist modalı
  IP veya CIDR subnet kabul eder; `POST /api/dashboard/whitelist-ip` artık
  subnet'i de `ipaddress.ip_network(strict=False)` ile doğrulayıp
  `whitelist_subnets` alanına yazar (IP → `whitelist_ips`), geçersiz girişte
  400 `invalid_ip`. Modal "Mevcut IP'mi kullan" hızlı doldurmasını içerir.

## 1.3.10 — 2026-07-21 (client **4.8.2** — Settings config surface documented)

- `agent/threat-engine.md`: `GET /api/threats/config` field list now documents
  the fields the GUI Settings tab writes — email prefs (`alert_email_enabled`,
  `instant_email_for_critical`, `min_severity_for_email`, `daily_digest_enabled`,
  **cloud-consumed**) and the webhook fields.
- Webhook semantics clarified: `webhook_enabled`/`webhook_url` are now written to
  **cloud** threats/config (was local-only ≤4.8.1); the daemon bridges them to
  local `notifications.*` and still forwards alerts client-side. Cloud does not
  send webhooks itself. Production floor raised to 4.8.2.
- **Cloud implemented (honeypot.yesnext.com.tr, 2026-07-21):**
  `POST /api/threats/config` deep-merge accepts GUI Settings fields
  (`alert_email_*`, `instant_email_for_critical`, `min_severity_for_email`,
  `daily_digest_enabled`, `auto_block_*`, `silent_hours{…}`, `webhook_*`) and
  returns effective config + `{ "v":1, "t":"threat_config_updated" }` WS push.
  Cloud consumes email prefs (severity gate + instant critical + daily digest
  tick); webhook is store/return only (no cloud forward). Health/STATUS
  `ransomware_running` + `network_guard{}` mirrored to client_status /
  dashboard-live. Dashboard production floor warn → **4.8.2**. Mirror published.

## 1.3.9 — 2026-07-21 (client **4.8.1** — protection popup data-source fix)

- `agent/gui-control-center.md`: new **detay popup veri-kaynağı invariantı** —
  a card/chip and its detail popup must show the same truth. Frontend-only GUI
  holds no local engine objects (`process_protection`, `ransomware_shield`,
  `network_guard`, `memory_guard` are always `None`), so protection popups must
  read daemon `STATUS` (IPC), never `if local_obj:`. Fixes 4.8.0 "Koruma Motoru
  chip AKTİF but Koruma popup OFF".
- Protection/engine popup now derives state from `STATUS.motor_ok` +
  `STATUS.persistence.self_protection`; local object is a fallback, not the sole
  source. Production floor raised to 4.8.1.

## 1.3.8 — 2026-07-21 (client **4.8.0** — GUI control center v2)

- `agent/gui-control-center.md`: new **Protection Status strip** on the landing
  page (motor / ransomware / network guard / guardian / honeypots / quarantine
  chips fed from daemon `STATUS`, each a shortcut to detail or tab) and new
  **Settings tab** managing email alerts, auto-block limits, silent hours and
  webhook via partial `POST /api/threats/config` deep-merge.
- `api/08-architecture.md`: daemon `STATUS` payload extended with
  `ransomware_running` and `network_guard{present,enabled,running,
  suspended_processes,baseline_age_sec,internet_ok}` so the frontend GUI never
  assumes local engines for layer state.
- Toggle render invariant documented: CTk switches must be enabled **before**
  select/deselect — fixes 4.7.x "shield ACTIVE but layer toggles look OFF".
- Layers/Settings tabs re-sync from cloud on every visit (no stale UI).

## 1.3.7 — 2026-07-21 (client **4.7.6** — daily log retention)

- New `agent/log-retention.md`: date-named client/threat/lifecycle logs,
  7-calendar-day retention and daemon/Guardian rename-race avoidance.
- `update-install.log` keeps its fixed active name for liveness compatibility;
  old dated lines are partitioned into 7-day archives.
- Threat logs now use the canonical APPDATA/ProgramData root instead of cwd.
- Production floor raised to 4.7.6.

## 1.3.6 — 2026-07-21 (client **4.7.5** — clean update/tamper handoff)

- Update authorization lock must remain active until the replacement daemon
  completes its boot-time previous-session check and reports motor ready.
- Fixes planned installer restarts being falsely emitted as
  `unexpected_exit` / `agent_tamper`.
- Production floor raised to 4.7.5.

## 1.3.5 — 2026-07-21 (client **4.7.4** — daemon STATUS IPC health)

- Live 4.7.3 smoke test found a recursive health dependency:
  daemon `STATUS` → persistence summary → `is_motor_healthy()` → daemon `STATUS`.
  The single-thread control server accumulated recursive requests/CLOSE_WAIT and
  GUI/Guardian probes timed out although protection and cloud health continued.
- Client ≥4.7.4 passes local daemon health into persistence summary while serving
  STATUS; it never probes its own STATUS socket. External health callers still
  probe normally. Regression test added.
- Production floor raised to 4.7.4. Old empty quarantine state left by the
  4.7.0/4.7.1 false-positive was cleared on the validated host.

## 1.3.4 — 2026-07-21 (client **4.7.3** — operator-approved containment + GUI)

- Network Guard safety invariant tightened: detection is **always alert-only**;
  cloud config cannot enable automatic suspend/kill/network restore.
- New confirm-gated `suspend_process` and non-destructive `resume_process`.
  Commands require exact identity (`pid`, image/path, `process_start_time`) to
  prevent delayed approval from acting on a reused PID.
- New `agent/gui-control-center.md`: Security Layers tab, immediate
  `POST /api/threats/config` + rollback, `threat_config_updated` WS push,
  canonical card/popup count rules and row-action requirements.
- Client tracked-IP card/detail use the same blocked∪watching snapshot.
- Cloud follow-up: add `_suspect` popup Suspend action, command whitelist/gate,
  threat-config deep-merge/effective response/WS broadcast; `_suspect` must not
  set `under_attack`.
- **Cloud implemented (honeypot.yesnext.com.tr, 2026-07-21):**
  `suspend_process`/`resume_process` whitelist + suspend confirm-gate +
  exact-identity validation (`pid`+`expected_image`+`process_start_time`);
  `POST /api/threats/config` deep-merge → effective config response +
  `{ "v":1, "t":"threat_config_updated" }` control-WS push;
  `ransomware_offline_suspect` warning→high normalize + trigger/pid dedupe +
  dashboard popup **Suspend/Resume** buttons; `under_attack` only on `_bomb`
  (`_suspect`/warning never sets it). Network Guard safe-defaults mirrored
  (`auto_contain/auto_kill/auto_restore=false`, `require_strong_signal=true`).
  v1.3.5/1.3.6 are client-internal (daemon STATUS IPC, update/tamper handoff) —
  no cloud API change required.

## 1.3.3 — 2026-07-21 (client **4.7.2** — KRİTİK güvenlik düzeltmesi)

- **Network Guard güvenli-varsayılan (client ≥4.7.2):** 4.7.0/4.7.1 canlı bir
  geliştirme makinesinde normal ağır-I/O uygulamalarını (Chrome/Firefox/Cursor/
  GameLoop/EdgeWebView) "offline fidye bombası" sanıp **otomatik suspend edip PC'yi
  kilitledi.** Sözleşme buna göre sıkılaştırıldı:
  - **`net_cut` yalnız gerçek internet erişim kaybında** (`internet_lost`) True olur;
    adapter down / VPN-Wi-Fi churn tek başına net_cut sayılmaz. (Kök hata: güncel
    adapter listesi boşken tüm baseline adapterları "down" sanılıyordu.)
  - **Otomatik containment varsayılan KAPALI:** `auto_contain=false`, `auto_kill=false`,
    `auto_restore=false`, `require_strong_signal=true`. Network Guard varsayılanda
    yalnız **alarm** üretir (`ransomware_offline_suspect`, severity **warning**);
    süreç dondurma/ağ değiştirme yapmaz.
  - Otomatik containment yalnız operatör `auto_contain=true` yaptıysa **ve** yüksek
    güvenli imza (aktif canary/VSS quarantine) varsa çalışır; o zaman
    `ransomware_offline_bomb` (critical) üretir. Ham yazma hızı tek başına asla süreç dondurmaz.
  - `threats/config` `protection.network_guard{}` alanları + güvenli defaultlar,
    STATUS/health `network_guard` bloğuna `auto_contain`, 60 sn trigger debounce.
- **Not (cloud'a):** popup builder artık iki tip görmeli — `ransomware_offline_suspect`
  (warning, `recommended_action=review_suspects`, otomatik aksiyon YOK) ve
  `ransomware_offline_bomb` (critical, containment yapıldı). `under_attack` bayrağı
  yalnız `_bomb` (critical) için tetiklenmeli; `_suspect` host'u under_attack yapmamalı.

## 1.3.2 — 2026-07-21

- **Cloud gap-fill (network-guard + survival):**
  - `under_attack` bayrağı: `agent_tamper` / `ransomware_offline_bomb` / canary urgent → `settings`; `client_status` + `dashboard-live` expose; `network_restore`/`unlock_ransomware_quarantine` completed → clear; 24 saat TTL.
  - Health snapshot `persistence` + `network_guard` → settings persist → `client_status`/`dashboard-live`.
  - `GET/POST threats/config`: `protection.network_guard{enabled,auto_restore,auto_kill}` (defaults + update).
  - Dashboard confirm: `create_user` / `remote_logon` / `set_autologon` / `reboot` (+ mevcut `network_restore`).
  - Tunnel cloud acceptance maddeleri kapatıldı (TTL/dedupe/desired).

## 1.3.1 — 2026-07-21

- **Cloud implemented (network-guard / client ≥4.7.0):**
  - `VALID_COMMAND_TYPES` → **38 tip**: `network_snapshot`, `network_restore`, `list_network_baseline`.
  - `network_restore` destructive confirm gate (`confirm:true` yoksa 400).
  - `ransomware_offline_bomb` popup builder: `system_context.network_guard.suspects[]` + `raw_events[].suspect_pid`/`image` → süreç/PID; popup listesinde.
  - 60 sn trigger dedupe (`routes_v4._find_recent_duplicate_urgent`).
  - Dashboard: `network_restored` rozeti + `network_restore` onay dialogu + kritik modal butonu.
  - Canlı doğrulama (client 41): offline-bomb urgent → popup `cursor.exe`/PID + `network_restored=true` + restore_actions dolu.

## 1.3.0 — 2026-07-21 (spec; client **4.7.0** planlı)

- **Yeni:** [`agent/network-guard.md`](agent/network-guard.md) — **offline fidye bombası**
  savunması (fire-and-forget + internetsiz kütle şifreleme). Beş parça:
  **A** imzalı ağ baseline yedeği (mapped drive / shares / adapter / DNS / route / firewall),
  **B** internetsiz davranışsal tespit (ağ-kesme delta + FS yazma/rename fırtınası + fidye notu deseni skorlama; ağ-kesme+FS-fırtınası → canary beklemeden tetik),
  **C** agresif containment **suspend-first** (kill değil; acil VSS snapshot; operatör onayıyla kill/release; opsiyonel auto-kill),
  **D** ağ/bağlantı kurtarma (adapter/DNS/firewall/route/mapped-drive/shares baseline'dan restore → daemon yeniden bağlanır),
  **E** `ransomware_offline_bomb` urgent alarmı (`system_context.network_guard`, `restored` işaretli).
- Yeni komutlar: `network_snapshot`, `network_restore` (confirm), `list_network_baseline`.
- STATUS/health `network_guard{}` bloğu.
- Dürüst sınır: tam EDR/AV değil; davranışsal tespit ayarlanabilir eşik + güvenli
  (suspend-first) varsayılan; garanti = erken containment + kurtarılabilirlik.

## 1.2.0 — 2026-07-21 (client **4.6.0** implemented)

- **Yeni:** [`agent/persistence-and-tamper.md`](agent/persistence-and-tamper.md) — survival modeli:
  Windows Servisi `CloudHoneypotGuardian` (SCM restart-on-failure) + Session 0 motor
  **çapraz watchdog**; durdurma yalnız (1) update-lock (2) imzalı PIN `operator_stop.json`;
  başka her sonlanma → **tamper** → ≤5 sn diriliş + `agent_tamper` urgent (SACL offender PID).
- **Yeni:** [`agent/disaster-recovery.md`](agent/disaster-recovery.md) — felaket kurtarma:
  `create_user` (yeni/yeniden Administrator), `remote_logon` (var olan oturum reconnect →
  yoksa **autologon `AutoLogonCount=1` + reboot** break-glass, boot sonrası LSA secret temizlenir),
  `set_autologon`/`clear_autologon`/`reboot` primitifleri, uçtan uca kurtarma runbook.
- `api/03-control-websocket.md`: komut kataloğuna `create_user`, `remote_logon`,
  `set_autologon`/`clear_autologon`, `reboot`; hepsi HMAC + **destructive confirm** listesinde.
- `api/08-architecture.md`: Guardian servis bacağı + durdurma politikası.
- `agent/attacks-and-services.md`: cloud `pending_tunnel_commands` **TTL/expiry + dedupe**
  sözleşmesi (10 aylık bayat komut bulgusu); client zaten yok sayıyor, `desired` otorite.
- `FLEET.md` / `INDEX.md`: 4.6.0 hedef satırları; production floor **4.5.68** (değişmedi).
- **Cloud implemented (aynı gün):**
  - `VALID_COMMAND_TYPES` → **35 tip**: `create_user`, `remote_logon`, `set_autologon`, `clear_autologon`, `reboot` eklendi.
  - Destructive confirm gate genişletildi: `create_user`/`remote_logon`/`set_autologon`/`reboot` `confirm:true` olmadan 400 (`clear_autologon` non-destructive). Alan doğrulaması: create_user/remote_logon → username+password, set_autologon → username. `remote_logon`/`reboot` TTL 15 dk.
  - Break-glass şifre maskeleme (`helpers.scrub_command_params`): audit log + `GET /api/commands/{id}` → `***`; agent’a giden gerçek şifre `pending`/WS payload’ta; terminal durumda DB’de şifre ezilir.
  - `agent_tamper` popup builder: `system_context.tamper.offender` (+ `raw_events[].offender_pid`/`image`) → popup süreç/PID; `agent_tamper` popup listesinde.
  - `pending_tunnel_commands` TTL (**24 saat**, kontrat yaşam döngüsü bölümü) + servis-başına dedupe (`routes_agent._prune_tunnel_commands`) — tunnel-set (yazma) + tunnel-status (okuma).
  - Opsiyonel dead-man: health `snapshot.persistence` `daemon_ok=false`/`service_ok=false` (operator_stop yok) → sentetik `agent_persistence_degraded` (high/70, 30 dk dedupe). Tamper urgent’e ikincil ağ.
  - Doküman: `agent/threat-engine.md` (tamper + dead-man wire), `agent/attacks-and-services.md` (tunnel TTL/dedupe detay).

## 1.1.7 — 2026-07-21

- **Fix (cloud):** Komut zarfına `type` alias'ı eklendi (= `command_type`) — client `verify_command_signature` tipi `type`/`command` anahtarından okuduğu için alias'sız imza doğrulaması reject ediyordu ("Invalid command signature"). Canlı 4.5.68 ile signed `list_sessions` → completed doğrulandı.
- Threat-intel cloud checklist güncellendi: MVP + ack + WS push maddeleri implemented olarak işaretlendi (ingest bg_worker'da ~30 dk, 3 kaynak senkron, 30 bundle).
- api/03 acceptance: WS push + unknown-command-400 kapatıldı.

## 1.1.6 — 2026-07-21

- Client **4.5.68** hotfix: canary tetiğinde **tek** zengin urgent yolu —
  4.5.67'deki ince `handle_alert` + zengin `send_urgent` yarışı kaldırıldı
  (canlı smoke boş-alanlı payload'ı yakaladı).
- Kural: canary urgent **her zaman** `system_context.ransomware` + `raw_events` +
  `target_service=SYSTEM` + `recommended_action=isolate_host` taşır.
- Fleet production floor → **4.5.68**.
- Canlı doğrulama (DESKTOP-F5SCL3G): canary MODIFIED → quarantine arm →
  urgent 200 `received` (dashboard+email), `RS_UNLOCK` OK.

## 1.1.5 — 2026-07-21

- **Cloud implemented:** HMAC command signing canlı — WS `command` push + `GET /api/commands/pending` artık `issued_at` + `signature` içeriyor (key = raw SHA256 digest; hostname kaynağı: heartbeat `hostname`)
- **Cloud implemented:** Destructive IR server gate canlı — `POST /api/commands/send` `confirm: true` olmadan 400 (`reset_password`, `disable_account`, `disable_all_users`, `enable_lockdown`, `clear_firewall` wipe)
- **Cloud implemented:** `unlock_ransomware_quarantine` whitelist'te (30 tip); dashboard kritik ransomware popup'ına unlock butonu
- **Cloud implemented:** Canary popup detayı `system_context.ransomware` (≥4.5.67 wire) + `raw_events`'ten okunuyor; health-report canary fallback'i 30 dk dedupe ile sentetik alert üretmiyor (Wire kuralına uyum)

## 1.1.4 — 2026-07-21

- Client **4.5.67** implements enriched canary urgent payload:
  `system_context.ransomware`, process-compatible `raw_events`, SYSTEM target/action.
- Health snapshot implements `ransomware_quarantine` with persisted suspect entries.
- Fleet production floor moved to 4.5.67 for detailed cloud canary popup.

## 1.1.3 — 2026-07-21

- `agent/ransomware-shield.md`: **Wire** bölümü — canary alert SoT = `alerts/urgent`
  (score 100 + `Dosya:`/`Değişiklik:` description’da); health snapshot sadece
  `canary_files_intact` / `ransomware_shield_status` / `vss_shadow_count`
- Cloud kuralı: health-report’tan sentetik canary alert üretme (boş popup / dupe)
- ≥4.5.67 hedef şema: urgent `system_context.ransomware` (file, change_type, suspects[pid/cmdline/sha256], quarantine)
- `agent/threat-engine.md`: health/report ransomware alanları not
- Cloud tarafı (aynı gün): HMAC signing + destructive gate + unlock whitelist (detay 1.1.5)

## 1.1.2 — 2026-07-21

- `api/03-control-websocket.md`: HMAC command signing (`security.command_signing`) + result signature
- Destructive IR: dashboard confirmation gate (cloud) documented
- `agent/threat-engine.md`: note optional local `notifications.webhook_url` SIEM forward

## 1.1.1 — 2026-07-21

- `FLEET.md`: feature → min-client matrix; production floor **4.5.66**
- `agent/ransomware-shield.md`: canary UX, quarantine, unlock paths
- `api/08-architecture.md`: `RS_STATUS` / `RS_UNLOCK`, ProgramData, STATUS `rs_quarantine`
- `api/03-control-websocket.md`: `unlock_ransomware_quarantine`; WS intel acceptance closed
- INDEX / CLIENT: ransomware + fleet; sprint WS handler closed
- register-protection: unit acceptance for 3-fail threshold

## 1.1.0

- Mimari / core loop contract’a taşındı (client referans paketi).
- **Yeni:** `agent/attacks-and-services.md`, `agent/threat-engine.md`, `agent/polling.md`, `cloud/overview.md`
- **Yeniden yazıldı (canlı şema):** `api/01-auth.md`, `api/03-control-websocket.md` (29’lu komut kataloğu), `api/06-firewall-blocks.md`
- `agent/CLIENT.md` → indeks; `register-protection` merged with client ≥4.5.66 apply-closed notes
- `INDEX.md` güncellendi

## 1.0.1 — 2026-07-21

- `agent/register-protection.md`: cloud+client apply path closed (client ≥4.5.66); open questions resolved
- INDEX: note client 4.5.66 for register-protection + WS threat_intel_updated

## 1.0.0 — 2026-07-21

- İlk yayın: api/01–09, agent CLIENT / register-protection / remote-input, cloud threat-intel-ingest
