# Changelog — honeypot-contract

## 1.2.0 — 2026-07-21

Client **4.6.0** survival katmanı için cloud desteği (Guardian service, tamper wire, break-glass recovery).

- **Cloud implemented:** Yeni komut tipleri whitelist’te — `create_user`, `remote_logon`, `set_autologon`, `clear_autologon`, `reboot` (`VALID_COMMAND_TYPES` → 35 tip).
- **Cloud implemented:** Destructive confirm gate genişletildi — `create_user`, `remote_logon`, `set_autologon`, `reboot` `confirm:true` olmadan 400 (`clear_autologon` non-destructive). `create_user`/`remote_logon` → username+password, `set_autologon` → username alan doğrulaması.
- **Cloud implemented:** Break-glass şifre maskeleme — `password`/`new_password` audit log + `GET /api/commands/{id}` görünümünde `***`; agent’a giden `pending`/WS payload gerçek şifreyi taşır; komut terminal duruma geçince DB’de şifre `***` ile ezilir (`helpers.scrub_command_params`). `remote_logon` TTL 15 dk, `reboot` TTL 15 dk.
- **Cloud implemented:** `agent_tamper` popup builder — `system_context.tamper.offender` (+ `raw_events[].offender_pid`/`image`) → popup süreç/PID; `agent_tamper` popup açar.
- **Cloud implemented:** `pending_tunnel_commands` legacy kuyruğu TTL (30 dk) + servis-başına dedupe (`routes_agent._prune_tunnel_commands`) — hem tunnel-set (yazma) hem tunnel-status (okuma).
- **Cloud implemented:** Opsiyonel dead-man — health report `snapshot.persistence` `daemon_ok=false`/`service_ok=false` (operator_stop yok) → sentetik `agent_persistence_degraded` alert (high/70, 30 dk dedupe). Tamper urgent’e ikincil güvenlik ağı.
- Doküman: `api/03-control-websocket.md` (komut katalog + destructive + maskeleme), `agent/attacks-and-services.md` (tunnel TTL/dedupe), `agent/threat-engine.md` (tamper + dead-man).

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
