# Architecture — SYSTEM daemon + GUI frontend

> API: `https://honeypot.yesnext.com.tr`  
> Client: **≥ 4.5.66** (protection.block_rules + WS threat_intel_updated; canary UX ≥4.5.65)  
> Ransomware: [`../agent/ransomware-shield.md`](../agent/ransomware-shield.md)

---

## Model

| Katman | Process | Rol |
|--------|---------|-----|
| **Motor** | `CloudHoneypot-Background` (`--mode=daemon`, SYSTEM Session 0) | Threat, firewall, honeypot, RemoteCommands, control WS, threat-intel, ransomware shield, `:58632`, self_update, RD — **güvenlik kritik** |
| **Guardian** (≥4.6.0) | **Windows Servisi** `CloudHoneypotGuardian` (LocalSystem, Automatic, SCM restart-on-failure) | **Yalnız watchdog** — motoru diriltir, kendisi motoru çalıştırmaz. Motor ↔ Guardian çapraz korur ([`../agent/persistence-and-tamper.md`](../agent/persistence-and-tamper.md)) |
| **Frontend** | Tray / GUI (interactive session) | Sadece arayüz — ProgramData + IPC; motor sağlıklıysa `:58632` **tutmaz** |
| **Watchdog** | Scheduled task (her 2 dk) | Üçüncü emniyet: `motor_ok` / Session 0 yoksa Background’u yeniden başlatır (GUI var diye atlamaz) |

Kural: makinede **tek** SYSTEM motor. Guardian servisi ayrı bir *watchdog* bacağıdır, motoru çoğaltmaz. Her Windows oturumunda **en fazla bir** tray/GUI (`Local\CloudHoneypotClient_GUI`).

**Durdurma politikası (≥4.6.0):** Motor+Guardian yalnız (1) güncelleme (`update_in_progress.lock`) veya (2) PIN ile imzalı `operator_stop.json` ile durur. Başka her sonlanma → tamper → diriliş + `agent_tamper` alarmı. Detay: [`../agent/persistence-and-tamper.md`](../agent/persistence-and-tamper.md).

```
┌─────────────┐   IPC 127.0.0.1:58632   ┌──────────────────────────┐
│ GUI / Tray  │ ◄──────────────────────► │ SYSTEM daemon             │
│ frontend    │   PING / STATUS          │ commands/pending          │
│             │   CLEAR_FIREWALL         │ control WS                │
│             │   BLOCK_IP / UNBLOCK_IP  │ threat-intel + ransomware │
│             │   RS_STATUS / RS_UNLOCK  │ firewall / honeypots      │
│             │   NG_MAINT_* / NG_SNAPSHOT│ network guard             │
│             │   NG_ACCEPT_SURFACE      │                           │
└─────────────┘                          └──────────────────────────┘
        │                                          │
        ▼                                          ▼
 ProgramData (token, blocks, intel, quarantine)   Cloud API
```

---

## IPC (local, newline UTF-8)

```
PING
STATUS
CLEAR_FIREWALL
BLOCK_IP <ip> [reason]
UNBLOCK_IP <ip>
RS_STATUS
RS_UNLOCK
THREAT_TOP
NG_MAINT_START
NG_MAINT_END
NG_MAINT_END_SNAPSHOT
NG_SNAPSHOT
NG_ACCEPT_SURFACE
HONEYPOT START <SVC> <PORT>
HONEYPOT STOP <SVC>
HONEYPOT LIST
SHOW / QUIT   (daemon: SHOW→NOGUI; QUIT for installer)
```

JSON replies start with `{`. Helpers: `client_daemon_ipc.py`.

**Network Guard IPC (≥4.9.12 / accept ≥4.9.15):** GUI bakım ve “Bu bendim”
akışı yerel IPC kullanır (`NG_MAINT_*`, `NG_SNAPSHOT`, `NG_ACCEPT_SURFACE`).
Dashboard aynı işi Control WS komutlarıyla yapar
(`network_maintenance_*`, `network_snapshot`, `network_accept_surface`) —
[`../agent/network-guard.md`](../agent/network-guard.md). Soft inform yolunda
PIN yok; `network_disable_adapter` yalnız WS + confirm.

**`THREAT_TOP` (≥4.7.6):** frontend GUI kendi threat engine'ini tutmaz; "Toplam
Saldırı" detay popup'ı boş kalmasın diye motor top saldırgan context'lerini
döner: `{ok, engine, total, attackers:[{ip, services[], failed_attempts,
threat_score, last_seen, is_blocked}]}`. Kart sayacı bulut `attack-count`
(kümülatif) kalır; motorda anlık IP context yoksa GUI bulut toplamı + açıklama
gösterir, boş ekran değil.

**Kural:** Unelevated GUI **asla** doğrudan `netsh delete/add` ile envanteri silmez/mutasyon yapmaz — motor IPC kullanır. Okuma: ProgramData + throttled live scan.

---

## Ownership rules (freeze / perf)

| İş | Kim | Not |
|----|-----|-----|
| `commands/pending`, control WS | Daemon | GUI emergency bridge sadece motor yoksa |
| Firewall mutate (block/clear) | Daemon (elevated) | GUI → IPC |
| Threat-intel poll + WS push apply | Daemon | frontend poll yok |
| Ransomware canary / quarantine | Daemon | GUI unlock → `RS_UNLOCK` |
| Engellenen liste | ProgramData SoT; GUI poll store; force live scan only on 🔄 / mutate | Periyodik `force=True name=all` **yasak** (4.5.44) |
| Self-update | Daemon → helper | GUI banner via `update_ui_status.json` |
| Threat / RD capture | Daemon | frontend_only skips local motors |

**Tk thread:** netsh / PowerShell / blocking IPC / honeypot start-stop **yok**.  
Motor health cache (background poll). IP mutate off-thread.  
Threat intel: tek worker + coalesce (paralel PS fırtınası yok). Servis toggle off-thread.

---

## Shared helpers

| Modül | Görev |
|-------|--------|
| `client_winproc` | Canonical hidden subprocess: `run_hidden` / `run_ps` / `run_ps_script` / `popen_detached` |
| `client_update_ui` | Cross-process update banner status |
| `client_block_store` | ProgramData blocked inventory |
| `client_daemon_ipc` | Frontend ↔ motor |

---

## ProgramData (`YesNext\CloudHoneypotClient`)

| Dosya | Amaç |
|-------|------|
| `token.dat` | Agent identity (immutable) |
| `blocked_ips.json` | Firewall inventory SoT |
| `threat_intel_bundle.json` / `_meta.json` | Cloud intel cache |
| `protection_block_rules.json` | register / threats/config `protection` |
| `ransomware_quarantine.json` | Canary/VSS IFEO quarantine |
| `update_ui_status.json` | GUI update banner |
| `CANARY_README.txt` | Admin-only notice (bait tree dışı) |

---

## STATUS örneği (≥4.9.15 / contract 1.4.17)

Normative `network_guard` alanları: [`../agent/network-guard.md`](../agent/network-guard.md).
Aşağıdaki örnek cloud health persist + GUI koruma şeridi için yeterlidir.

```json
{
  "ok": true,
  "daemon": true,
  "role": "daemon",
  "motor_ok": true,
  "remote_commands_running": true,
  "version": "4.9.15",
  "running_services": ["SSH"],
  "protection_mode": "monitoring",
  "token_present": true,
  "rs_quarantine": {
    "active": false,
    "entries": 0,
    "canary_files": 137
  },
  "ransomware_running": true,
  "network_guard": {
    "present": true,
    "enabled": true,
    "running": true,
    "maintenance": false,
    "maintenance_started_at": null,
    "suspended_processes": 0,
    "baseline_version": 14,
    "baseline_age_sec": 120,
    "baseline_captured_at": "2026-07-23T14:16:15Z",
    "verified": true,
    "internet_ok": true,
    "drift": false,
    "drift_count": 0,
    "surface_inform": true,
    "surface_inform_count": 1,
    "surface_inform_changes": [
      {
        "id": "adapter_up.Ethernet",
        "kind": "adapter_enabled",
        "interface": "Ethernet",
        "from": "disconnected",
        "to": "up",
        "ipv4": "192.168.1.21"
      }
    ],
    "auto_contain": false,
    "auto_kill": false,
    "auto_restore": false,
    "auto_restore_network": true,
    "live": {
      "adapters": [
        {
          "name": "Ethernet", "state": "up", "ipv4": "192.168.1.21",
          "gateway": "192.168.1.1", "dns": ["1.1.1.1", "8.8.8.8"], "dhcp": true
        },
        {
          "name": "Wi-Fi", "state": "up", "ipv4": "192.168.1.30",
          "gateway": "192.168.1.1", "dns": ["1.1.1.1", "8.8.8.8"], "dhcp": true
        }
      ],
      "mapped_drives": [],
      "firewall": { "domain": "on", "private": "on", "public": "on" },
      "connectivity": { "internet_ok": true, "dns_ok": true, "gateway_ok": true }
    },
    "baseline": {
      "adapters": [
        {
          "name": "Ethernet", "state": "disconnected", "ipv4": "169.254.216.150",
          "gateway": "", "dns": ["1.1.1.1", "8.8.8.8"], "dhcp": true
        },
        {
          "name": "Wi-Fi", "state": "up", "ipv4": "192.168.1.30",
          "gateway": "192.168.1.1", "dns": ["1.1.1.1", "8.8.8.8"], "dhcp": true
        }
      ],
      "mapped_drives": [],
      "firewall": { "domain": "on", "private": "on", "public": "on" }
    }
  },
  "system_recovery": {
    "present": true,
    "enabled": true,
    "baseline_version": 5,
    "verified": true,
    "drift": null
  },
  "persistence": {
    "service_ok": true,
    "daemon_ok": true,
    "tasks_armed": true,
    "tamper_count_24h": 0
  }
}
```

**STATUS zenginleştirme (≥4.8.0 / rich ≥4.9.12 / soft inform ≥4.9.15):** frontend
GUI "Koruma Durumu" şeridi katman durumlarını STATUS’tan okur.

- **1.4.14:** `network_guard.live.adapters` (IPv4/DNS) + golden özeti → dashboard
  **Ağ kurtarma** paneli; tam history `list_network_baseline`.
- **1.4.15:** `maintenance` — VPN/IP işi için koruma pause.
- **1.4.17:** `surface_inform` + `surface_inform_count` + `surface_inform_changes`
  (mavi / additive); kırmızı `drift` yalnız subtractive. Internet açıkken panik yok.
  `network_accept_surface` / GUI `NG_ACCEPT_SURFACE` yeni golden yapar.

Frontend yerel engine varsaymaz; motor yoksa katman KAPALI.

### Additive resilience health — observe mode (contract 1.4.2 + 1.4.5)

Cloud `POST /api/health/report` içinde aşağıdaki opsiyonel blokları kabul edip
son health snapshot + client effective settings içinde saklar:

```json
{
  "resilience": {
    "guardian_installed": true,
    "guardian_running": true,
    "guardian_exit_code": 0,
    "daemon_restarts_24h": 0,
    "guardian_restarts_24h": 0,
    "last_recovery_ms": 1840,
    "restart_backoff_sec": 0,
    "restart_storm": false,
    "binary_integrity": "valid|invalid|unknown",
    "stand_down_reason": "update|operator_pin|uninstall|null"
  },
  "event_log_health": {
    "available": true,
    "running": true,
    "channels_active": 1,
    "channels_total": 1,
    "events_processed": 0,
    "events_filtered": 0,
    "errors": 0,
    "watched_ids": [4625, 4723, 4724],
    "password_burst": {
      "mode": "observe",
      "auto_lockout": false,
      "window_sec": 300,
      "threshold": 5,
      "events": 0,
      "unique_targets": 0,
      "unique_actors": 0,
      "failed_or_unknown": 0,
      "peak_actor_events": 0,
      "burst_detected": false
    }
  },
  "etw_shadow": {
    "available": false,
    "provider_attached": false,
    "mode": "shadow",
    "source": "stub",
    "fallback": "none",
    "auto_containment": false,
    "window_sec": 60.0,
    "events_in_window": 0,
    "ops": {},
    "top_pids": [],
    "dropped_events": 0,
    "provider_restarts": 0,
    "buffer_pressure": false,
    "correlation": {
      "mode": "shadow",
      "auto_containment": false,
      "candidates": [],
      "candidate_count": 0,
      "thresholds": {
        "file_fanout": 25,
        "rename_burst": 20,
        "write_burst": 30
      }
    },
    "error": "etw consumer not attached (shadow stub)"
  },
  "command_signing": {
    "observe": true,
    "enforce": false,
    "ok": 0,
    "missing": 0,
    "invalid": 0,
    "no_token": 0,
    "disabled": 0,
    "last_error": ""
  },
  "access_integrity": {
    "observe": true,
    "enforce": false,
    "baseline_valid": true,
    "entries_checked": 2,
    "changed": 0,
    "missing": 0,
    "status": "healthy"
  },
  "device_identity": {
    "mode": "observe",
    "enrolled": false,
    "tpm_present": true,
    "tpm_ready": true,
    "manufacturer_id": null,
    "key_non_exportable": null,
    "attestation": "not_implemented",
    "reenrollment_required": false,
    "error": ""
  },
  "canary_coverage": {
    "mode": "observe",
    "configured": true,
    "files_total": 0,
    "files_intact": 0,
    "files_missing": 0,
    "roots_covered": 0,
    "coverage_ok": false
  },
  "deception_health": [
    {
      "service": "SSH",
      "port": 22,
      "status": "started",
      "handler_limit": 48,
      "handlers_rejected": 0,
      "backlog": 0,
      "rate_limit_per_ip_min": 10,
      "resource_budgeted": true,
      "protocol_aware": true,
      "fingerprint_profile": "static_legacy",
      "bypass_coverage_required": true
    }
  ]
}
```

Tüm bloklar additive/opsiyoneldir; missing = `legacy`, **failure değildir**.

- `binary_integrity`: `valid` = Authenticode publisher OK; `invalid` =
  present-but-broken signature (tamper); `unknown` = unsigned/dev build or
  check unavailable. Observe-only — never blocks startup.
- `access_integrity`: path-free DACL fingerprint drift (`status` enum:
  `healthy|degraded|baseline_created|baseline_unavailable|disabled`). Never
  uploads paths, principals, or raw ACL text. Flag
  `security.acl_drift_observe` (default off).
- `etw_shadow`: bounded sensor health/aggregates only; raw unbounded file-I/O
  events cloud'a alınmaz. `source` = `stub|psutil_io|…`, `fallback` =
  `none|psutil`. `auto_containment` always false; never auto-dispatch
  suspend/kill. Flag `threat_detection.etw_file_io_shadow` (default off);
  optional `threat_detection.etw_psutil_fallback`.
- `password_burst` (nested under `event_log_health`): wire SoT name — **not**
  `identity_burst`. Counts only; `auto_lockout` always false.
- `device_identity`: read-only TPM capability (`security.tpm_identity_observe`).
  Missing TPM → `error: "tpm_unsupported"` / non-Windows → `"non_windows"`;
  never hard-fails health; enrollment/attestation not implemented.
- `canary_coverage` / `deception_health`: path-free / credential-free observe
  only. Desktop canaries remain forbidden.

Restart storm, installed-but-not-running Guardian veya invalid binary integrity
iki ardışık snapshot'ta görülürse cloud 30 dakika dedupe'lu
`agent_resilience_degraded` observe alarmı üretir. Bu telemetri hiçbir
process/network containment komutunu otomatik tetiklemez.


### STATUS dependency invariant (client ≥4.7.4)

`:58632 STATUS` handler'ı tek-thread control server üzerinde çalışır. Handler
ve çağırdığı summary fonksiyonları **aynı STATUS soketini tekrar probe edemez**.
Özellikle persistence summary, `daemon_ok` değerini mevcut motor state'inden
(`_is_daemon_motor && remote_commands_running`) alır; handler içinden
`is_motor_healthy()` çağırmaz.

Neden: `STATUS → persistence → is_motor_healthy → STATUS` recursive self-call
kuyruğu listener'ı tüketir, `CLOSE_WAIT` biriktirir ve GUI/Guardian timeout olur.
External health caller'lar normal şekilde STATUS probe etmeye devam eder.

---

## Lifecycle

- `client_startup` — daemon veya frontend
- `gui_quit` — frontend kapandı; **motor ayakta kalmalı**
- `self_update_*` — ACK + helper; bitiş = yeni process version
- Update handoff (≥4.7.5): `update_in_progress.lock`, replacement daemon
  `Ensure-DaemonMotor` ile hazır olup previous-session kontrolünü tamamlayana
  kadar silinmez. Planlı restart `unexpected_exit` / `agent_tamper` üretmez.

---

## God-files (bilinçli sınır)

Büyütmeden önce çıkar: `client_gui.py`, `client.py`, `client_utils.py`.  
Yeni GUI özellikleri: `client_gui_*.py` alt modülleri tercih et.

---

## Acceptance

- [ ] Motor sağlıklı → GUI `frontend_only`, `:58632` daemon’da
- [ ] Engelle / Tümünü temizle → IPC; unelevated netsh fail etmez / flaş yok
- [ ] `RS_STATUS` / `RS_UNLOCK` frontend’den çalışır
- [ ] Status açıkken periyodik full `name=all` fırtınası yok
- [ ] GUI kapandı → dashboard poll devam
- [ ] threats/config `protection.block_rules` motor’da uygulanır
- [ ] İki RDP kullanıcısı → RD `session_id` doğru masaüstü
- [x] 5 ardışık STATUS hızlı döner; recursive self-probe / CLOSE_WAIT yok (≥4.7.4)
- [x] Update sonrası `tamper_count_24h=0`, `last_tamper_ts=null` (≥4.7.5)
