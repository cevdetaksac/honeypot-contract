# Architecture — SYSTEM daemon + GUI frontend

> API: `https://honeypot.yesnext.com.tr`  
> Client: **≥ 4.5.66** (protection.block_rules + WS threat_intel_updated; canary UX ≥4.5.65)  
> Ransomware: [`../agent/ransomware-shield.md`](../agent/ransomware-shield.md)

---

## Model

| Katman | Process | Rol |
|--------|---------|-----|
| **Motor** | `CloudHoneypot-Background` (`--mode=daemon`, SYSTEM Session 0) | Threat, firewall, honeypot, RemoteCommands, control WS, threat-intel, ransomware shield, `:58632`, self_update, RD — **güvenlik kritik** |
| **Frontend** | Tray / GUI (interactive session) | Sadece arayüz — ProgramData + IPC; motor sağlıklıysa `:58632` **tutmaz** |
| **Watchdog** | Scheduled task (her 2 dk) | `motor_ok` / Session 0 yoksa Background’u yeniden başlatır (GUI var diye atlamaz) |

Kural: makinede **tek** SYSTEM motor. Her Windows oturumunda **en fazla bir** tray/GUI (`Local\CloudHoneypotClient_GUI`).

```
┌─────────────┐   IPC 127.0.0.1:58632   ┌──────────────────────────┐
│ GUI / Tray  │ ◄──────────────────────► │ SYSTEM daemon             │
│ frontend    │   PING / STATUS          │ commands/pending          │
│             │   CLEAR_FIREWALL         │ control WS                │
│             │   BLOCK_IP / UNBLOCK_IP  │ threat-intel + ransomware │
│             │   RS_STATUS / RS_UNLOCK  │ firewall / honeypots      │
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
HONEYPOT START <SVC> <PORT>
HONEYPOT STOP <SVC>
HONEYPOT LIST
SHOW / QUIT   (daemon: SHOW→NOGUI; QUIT for installer)
```

JSON replies start with `{`. Helpers: `client_daemon_ipc.py`.

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

## STATUS örneği (≥4.5.66)

```json
{
  "ok": true,
  "daemon": true,
  "role": "daemon",
  "motor_ok": true,
  "remote_commands_running": true,
  "version": "4.5.66",
  "running_services": ["SSH"],
  "protection_mode": "monitoring",
  "token_present": true,
  "rs_quarantine": {
    "active": false,
    "entries": 0,
    "canary_files": 138
  }
}
```

---

## Lifecycle

- `client_startup` — daemon veya frontend
- `gui_quit` — frontend kapandı; **motor ayakta kalmalı**
- `self_update_*` — ACK + helper; bitiş = yeni process version

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
