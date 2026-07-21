# Fleet & version matrix

> Hangi client sürümü hangi contract özelliğini karşılar.  
> Contract VERSION: root `VERSION`.

| Özellik | Contract dosyası | Min client |
|---------|------------------|------------|
| Bearer-only agent API | `api/01-auth.md` | 4.4.33+ / fleet default |
| Control WS commands | `api/03-control-websocket.md` | 4.5.x |
| Self-update GUI banner | `api/04-self-update.md` | 4.5.39 |
| Remote `inputs[]` piggyback | `agent/remote-input.md` | **4.5.55** |
| One GUI/session + motor watchdog | `api/08-architecture.md` | 4.5.58–59 |
| Threat-intel GET/304/ACK | `api/09-threat-intel.md` | **4.5.61** |
| Canary sort-bait + quarantine IPC | `agent/ransomware-shield.md` | 4.5.62–64 |
| Canary UX (no local scare, OneDrive skip) | `agent/ransomware-shield.md` | **4.5.65** |
| `protection.block_rules` apply + WS intel push | `agent/register-protection.md`, `api/09` | **4.5.66** |
| Enriched canary alert + quarantine snapshot | `agent/ransomware-shield.md` | 4.5.67 |
| Single enriched canary urgent (thin-race fix) | `agent/ransomware-shield.md` | **4.5.68** |
| Guardian service + cross-watchdog + tamper wire | `agent/persistence-and-tamper.md` | **4.6.0** |
| `create_user` + `remote_logon` (autologon break-glass) | `agent/disaster-recovery.md` | **4.6.0** |
| Network Guard baseline + offline şüpheli tespiti | `agent/network-guard.md` | **4.7.0** |
| Network Guard locale decode hotfix | `agent/network-guard.md` | **4.7.1** |
| Network Guard false-positive fix + alert-only default | `agent/network-guard.md` | **4.7.2** |
| Hard alert-only + onaylı exact-process suspend/resume | `agent/network-guard.md` | **4.7.3** |
| GUI Security Layers + cloud config push + sayaç invariant | `agent/gui-control-center.md` | **4.7.3** |
| Daemon STATUS recursion / GUI-Guardian IPC health hotfix | `api/08-architecture.md` | **4.7.4** |
| Update lock → replacement motor-ready tamper handoff | `api/08-architecture.md` | **4.7.5** |
| Daily client/threat/lifecycle logs + 7-day retention | `agent/log-retention.md` | **4.7.6** |
| Protection strip + Settings tab + STATUS network_guard/rs alanları | `agent/gui-control-center.md`, `api/08` | **4.8.0** |
| Detay popup veri-kaynağı invariantı (chip=popup, daemon STATUS) | `agent/gui-control-center.md` | **4.8.1** |
| Settings webhook cloud→local köprüsü + config surface dokümante | `agent/threat-engine.md` | **4.8.2** |
| Dashboard PIN set/reset (`set_gui_pin`/`clear_gui_pin`) + IP hızlı aksiyon butonları | `api/03-control-websocket.md`, `agent/gui-control-center.md` | **4.8.3** |

**Önerilen production floor:** **4.8.3** (survival + disaster recovery +
operator-approved Network Guard containment + GUI control center v2 + tutarlı
detay popup'ları + cloud-yönetimli webhook + dashboard PIN yönetimi).
4.7.0/4.7.1 production'da kullanılmamalı; 4.7.3'te STATUS timeout, 4.7.4'te
update false-tamper vardır; 4.7.x GUI'de katman toggle'ları yanlış (hep KAPALI)
render edilir. 4.8.0'da koruma detay popup'ı chip AKTİF iken OFF gösterir (4.8.1
düzeltir).

Cloud publish: contract clone → `git pull` → `scripts/publish_contract.sh` (HTTPS mirror).
