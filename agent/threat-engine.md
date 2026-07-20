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
- **`protection.block_rules`** — [`register-protection.md`](./register-protection.md) ile aynı şema (SoT poll)

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

```json
{
  "snapshot": {
    "cpu_percent": 12,
    "memory_percent": 40,
    "disk_free_gb": 50,
    "canary_ok": true,
    "vss_ok": true,
    "anomalies": [],
    "agent_version": "4.5.65",
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

## Acceptance

- [ ] Config poll → `protection.block_rules` + auto_block_* dolu
- [ ] 4625 flood → urgent + (eş) auto-block → HP-BLOCK
- [ ] health/report 200
- [ ] ThreatFox/CISA’ya client isteği **yok** (sadece `09` bundle)
