# Threat engine (v4 HTTP) — local detection

> **Contract VERSION:** root `VERSION`  
> **≠** cloud intel bundle → [`../api/09-threat-intel.md`](../api/09-threat-intel.md)  
> **Auth:** Bearer  
> **Kaynak:** `routes_v4.py`, `schemas.py`, archive v4 (normalize)

Agent Windows Event Log + canary/VSS + skor → cloud’a urgent/batch/health/auto-block.  
Cloud curated IoC → ayrı poll (`09`).

---

## Cadence

| Endpoint | Ne zaman |
|----------|----------|
| `GET /api/threats/config` | Startup + ~**5 dk** (+ register `protection`) |
| `POST /api/health/report` | ~1–5 dk |
| `POST /api/events/batch` | Buffer dolunca / ~30–60s |
| `POST /api/alerts/urgent` | Anında (critical/high) |
| `POST /api/alerts/auto-block` | Local firewall block sonrası |
| `POST /api/alerts/lifecycle` | Startup / gui_quit / watchdog / self_update |

---

## GET /api/threats/config

Önemli alanlar (canlı):

- `auto_block_enabled`, `auto_block_threshold` (örn. 80), `auto_block_duration_hours`
- `max_auto_blocks_per_hour` / `_per_day`
- `whitelist_ips` / `whitelist_subnets`
- `ransomware_protection_enabled`, `canary_files_enabled`
- `silent_hours` / `working_hours`
- `monitored_event_channels` (security/system/application/rdp)
- `emergency_lockdown_*`, `logon_challenge`
- **E-posta bildirim tercihleri (GUI Ayarlar sekmesi ≥4.8.0 yazar):**
  `alert_email_enabled`, `instant_email_for_critical`,
  `min_severity_for_email` (`low|medium|high|critical`), `daily_digest_enabled`.
  Bunları **cloud tüketir** (e-postayı cloud gönderir); client apply etmez.
- **Webhook (SIEM forward) — `webhook_enabled`, `webhook_url` (≥4.8.2):** GUI
  Ayarlar sekmesi bu alanları threats/config'e yazar; client `_sync_threat_config`
  bunları yerel `notifications.webhook_*`'e köprüler ve alert gönderiminde local
  forward eder (aşağıya bkz).
- **`protection.block_rules`** — [`register-protection.md`](./register-protection.md) ile aynı şema (SoT poll)
- **`protection.network_guard{}`** — [`network-guard.md`](./network-guard.md): `enabled`,
  `auto_contain`/`auto_kill`/`auto_restore` (client hard-false), `require_strong_signal`,
  `score_threshold`, `fs_write_*`

## POST /api/threats/config (GUI / dashboard write)

Partial deep-merge. Gönderilmeyen sibling alanlar silinmez. Cloud:

1. Effective config'i döndürür.
2. Aynı client'ın control WS kanalına `{ "v":1, "t":"threat_config_updated" }` push eder.
3. Daemon hemen GET + runtime apply yapar (poll fallback).

GUI Security Layers, Ayarlar sekmesi (≥4.8.0) ve whitelist mutasyonları bu
endpoint'i kullanır ([`gui-control-center.md`](./gui-control-center.md)).

**Cloud (honeypot.yesnext.com.tr):** Settings alanları (`alert_email_*`,
`instant_email_for_critical`, `min_severity_for_email`, `daily_digest_enabled`,
`auto_block_*`, `silent_hours`, `webhook_*`) deep-merge ile kabul edilir;
e-posta tercihlerini cloud tüketir; webhook yalnız saklanır/döner (göndermez).
Server-side validation: geçersiz değer **HTTP 422**
`{"detail":{"error":"validation_failed","fields":{...}}}` döner ve hiçbir alan
yazılmaz — `min_severity_for_email` (low|medium|high|critical),
`silent_hours.mode` (night_only|outside_working|always|custom), `HH:MM` saat
alanları, integer sınırları (`auto_block_threshold` 1–100, süreler 0–8760,
limitler ≥1) ve `webhook_url` http(s) şeması.

---

## POST /api/alerts/urgent

```json
{
  "severity": "critical",
  "threat_type": "brute_force",
  "title": "…",
  "description": "…",
  "source_ip": "203.0.113.10",
  "target_service": "RDP",
  "target_port": 3389,
  "username": "administrator",
  "threat_score": 85,
  "windows_event_ids": [4625],
  "recommended_action": "block_ip",
  "auto_response_taken": ["firewall_block"],
  "raw_events": [],
  "system_context": {},
  "timestamp": "2026-07-21T00:00:00Z"
}
```

Nested `alert: { … }` de kabul (cloud flatten eder).

Client SIEM webhook: her urgent alert, `notifications.webhook_enabled=true` ise
`notifications.webhook_url`'e local olarak da forward edilir (Slack/Teams/custom).
≤4.8.1'de bu alanlar **yalnız** local `client_config.json`'dan okunurdu. ≥4.8.2'de
GUI Ayarlar sekmesi `webhook_enabled`/`webhook_url`'ü **cloud** threats/config'e
yazar; daemon `_sync_threat_config` bunları yerel `notifications.*`'e köprüler,
böylece cloud tek kaynak olur ama forward yine client tarafında yapılır. Cloud
kendisi webhook göndermez.

### Offline queue / idempotency (contract ≥1.4.7)

Optional `event_id` / `idempotency_key` on urgent → soft ACK on replay.
Batch drain: [`../api/10-offline-urgent-queue.md`](../api/10-offline-urgent-queue.md)
(`POST /api/alerts/urgent/batch`). Flag `security.offline_urgent_queue`
default **off**.

---

## POST /api/events/batch

```json
{
  "batch_id": "uuid",
  "events": [ { "event_id": 4625, "channel": "Security", "…": "…" } ],
  "summary": { "count": 10 }
}
```

---

## POST /api/health/report

Ransomware alanları (snapshot): `canary_files_intact` (bool), `ransomware_shield_status`
(`active|disabled|error`), `vss_shadow_count` (int), optional path-free
`canary_coverage{}` — **detay yok**; canary alert detayı
`alerts/urgent`tedir → [`ransomware-shield.md`](./ransomware-shield.md) “Wire” bölümü.

Additive observe blocks (contract 1.4.5; missing = legacy): `resilience`,
`event_log_health` (+ nested `password_burst`), `etw_shadow`,
`command_signing`, `access_integrity`, `device_identity`, `deception_health[]`.
Full shapes: [`../api/08-architecture.md`](../api/08-architecture.md).

Password-change visibility (ID-401/402): Security Event Log **4723** /
**4724** may be watched. Burst aggregates are counts only
(`password_burst.auto_lockout` always false). No automatic admin lockout.

```json
{
  "snapshot": {
    "cpu_percent": 12,
    "memory_percent": 40,
    "disk_free_gb": 50,
    "canary_ok": true,
    "vss_ok": true,
    "anomalies": [],
    "agent_version": "4.9.1",
    "…": "…"
  }
}
```

Self-process koruma alanları: [`../api/07-lifecycle-sessions.md`](../api/07-lifecycle-sessions.md).

---

## POST /api/alerts/auto-block

```json
{
  "blocked_ip": "203.0.113.10",
  "reason": "threat_score>=80",
  "threat_score": 90,
  "duration_hours": 24,
  "firewall_rule_name": "HP-BLOCK-203.0.113.10",
  "related_alert_id": "…",
  "events_summary": {}
}
```

Cloud `auto_blocks` + `block_rules` upsert. Response `extend_duration` / `permanent_block` → client kural süresini ayarlar.

---

## Skor / auto-block (client)

| Kural (özet) | Not |
|--------------|-----|
| Threshold | `threats/config.auto_block_threshold` (default 80) |
| Rate limit | max per hour/day |
| Whitelist | IP/subnet → block yok |
| Silent hours | config’e göre agresif logoff/disable |
| Brute fail | ayrıca `protection.block_rules` eşikleri |

---

## Windows Event (özet)

| ID | Anlam (tipik) |
|----|----------------|
| 4624 | Logon success |
| 4625 | Logon fail |
| 4648 | Explicit cred |
| 4720 | User created |
| 1102 | Audit log cleared |
| 1149 | RDP gateway / logon |

Channel’lar `monitored_event_channels` ile filtrelenir.

---

## Canary / VSS

- Local otorite: canary touch / VSS delete → urgent + opsiyonel lockdown komutu
- Cloud threat-intel ransomware layer **lockdown zorlamaz** (`09`)

---

## Agent survival — tamper + dead-man (client ≥4.6.0)

Guardian service + SYSTEM daemon kendini korur; motor durdurulmaya çalışılırsa veya
beklenmedik şekilde ölürse cloud’a bildirir.

### `agent_tamper` alert (SoT: `POST /api/alerts/urgent`)

```json
{
  "severity": "critical",
  "threat_type": "agent_tamper",
  "title": "AGENT TAMPER — motor durdurulmaya calisildi",
  "description": "reason=… leg=… resurrected=…",
  "threat_score": 100,
  "target_service": "SYSTEM",
  "recommended_action": "isolate_host",
  "system_context": {
    "tamper": {
      "reason": "service_stop",
      "leg": "daemon",
      "legitimate": false,
      "resurrected": true,
      "resurrect_ms": 850,
      "ts": "…Z",
      "offender": { "pid": 6120, "image": "taskkill.exe" }
    }
  },
  "raw_events": [
    { "kind": "agent_tamper", "reason": "service_stop", "leg": "daemon",
      "resurrected": true, "offender_pid": 6120, "image": "taskkill.exe" }
  ]
}
```

- Cloud, `system_context.tamper.offender` (veya `raw_events[].offender_pid`/`image`) →
  popup süreç/PID alanları (`helpers.parse_alert_process_info`).
- `agent_tamper` **popup açar** (`helpers._POPUP_THREAT_TYPES`); modalda “Kill Process” ile offender durdurulabilir.
- `legitimate=true` (planlı stop/update/operator) ise client **hiç göndermez**.

### Dead-man (cloud fallback) — `snapshot.persistence`

Health report’ta `snapshot.persistence` bloğu (client ≥4.6.0):

```json
{ "service_ok": true, "service_installed": true, "daemon_ok": true,
  "tasks_armed": true, "self_protection": true, "operator_stop": false,
  "last_tamper_ts": null, "tamper_count_24h": 0 }
```

Cloud kuralı (`routes_v4._maybe_alert_persistence_degraded`): `service_installed=true` iken
`daemon_ok=false` **veya** `service_ok=false` **ve** `operator_stop=false` → sentetik
`agent_persistence_degraded` alert (severity **high**, score 70, 30 dk dedupe).
`operator_stop=true` (bilinçli durdurma) → alarm yok. Bu, tamper urgent gelmeden
motor sessizce ölürse **ikincil güvenlik ağıdır**; SoT tamper urgent’tir.

---

## Acceptance

- [ ] Config poll → `protection.block_rules` + auto_block_* dolu
- [ ] 4625 flood → urgent + (eş) auto-block → HP-BLOCK
- [ ] health/report 200
- [ ] ThreatFox/CISA’ya client isteği **yok** (sadece `09` bundle)
