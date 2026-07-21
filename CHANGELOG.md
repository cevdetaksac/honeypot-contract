# Changelog — honeypot-contract

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
