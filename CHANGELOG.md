# Changelog — honeypot-contract

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
