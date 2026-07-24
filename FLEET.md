# Fleet & version matrix

> Hangi client sürümü hangi contract özelliğini karşılar.  
> Contract VERSION: root `VERSION`.

| Özellik | Contract dosyası | Min client |
|---------|------------------|------------|
| Bearer-only agent API | `api/01-auth.md` | 4.4.33+ / fleet default |
| Control WS commands | `api/03-control-websocket.md` | 4.5.x |
| Self-update GUI banner | `api/04-self-update.md` | 4.5.39 |
| Remote `inputs[]` piggyback | `agent/remote-input.md` | **4.5.55** |
| Remote Desktop v2: WS/JPEG healthy path + adaptive telemetry + latest-frame coalescing | `api/05-remote-desktop.md` | **4.9.0** |
| Remote input protocol 2 + mobile gestures + ACK + persistent session helper | `agent/remote-input.md` | **4.9.0** |
| Remote Desktop optional WebRTC signaling (protocol=1, non-trickle, H264, data-channel input, cloud-supplied `ice_servers` in offer) | `api/05-remote-desktop.md` | **4.9.0** |
| Remote Desktop WebRTC smoothness (raw RGB, HW H.264, input fluidity) | `api/05-remote-desktop.md`, `cloud/REMOTE_DESKTOP_SMOOTHNESS.md` | client **≥ 4.9.20** / contract **1.4.20** |
| Remote Desktop console Winlogon / pre-logon (empty session → logon UI) | `api/05-remote-desktop.md`, `cloud/REMOTE_DESKTOP_WINLOGON.md` | sibling pre_logon + prefer/desktop ≥ **4.9.26** / contract **1.4.23** (cloud C-WL shipped; wire from **4.9.21**) |
| WebRTC strict JPEG suppression, 30–60 fps media pacing, optional DXGI capture and effective media telemetry | `api/05-remote-desktop.md` | **4.9.1** |
| P1 observe schemas: heartbeat_proof, access_integrity, etw_shadow correlation, password_burst, device_identity, canary_coverage, deception_health, network_restore dry_run/rollback | `api/08-architecture.md`, `api/01-auth.md`, `api/03-control-websocket.md` | **4.9.1** (flags default off) |
| OOB-501 offline urgent queue: `POST /api/alerts/urgent/batch` + soft idempotency on urgent | `api/10-offline-urgent-queue.md` | cloud live ≥ **1.4.7**; client flag `security.offline_urgent_queue` default **off** |
| Dashboard Server Management (users/processes/services + `list_services`) | `agent/server-management.md` | target ≥ **4.9.4** (cloud UI live ≥ **1.4.8**) |
| Dashboard Users — disabled inventory + enable/disable (`include_disabled:true`, C-USER) | `cloud/SERVER_USER_MANAGEMENT.md`, `agent/server-management.md` | cloud ≥ **1.4.24**; client list/mutate already ≥ **4.9.4** |
| Account unlink from client Settings (`POST /api/agent/unlink-account`) | `api/02-account.md` | cloud live ≥ **1.4.27**; client **≥ 4.9.26** |
| Hardware-bound `machine_id` (MAC+MachineGuid fingerprint; clone split) | `api/01-auth.md` | contract **1.4.26**; client **≥ 4.9.28** |
| Uninstall PIN lifecycle (`uninstall_*` + `windows_user`) | `api/07-lifecycle-sessions.md` | cloud ≥ **1.4.9**; client with uninstall PIN gate |
| Bare `successful_logon` score ≤70 + no auto-block (cloud reject shapes) | `agent/threat-engine.md` | cloud ≥ **1.4.11**; target client **≥4.9.7** |
| Whitelist never stays blocked (reject + `unblock_ip` + pending-unblocks) | `api/06-firewall-blocks.md`, `agent/threat-engine.md` | cloud ≥ **1.4.11**; client must ACK `block-removed` |
| Realtime presence (`presence`/`goodbye` + `POST /api/presence`) | `api/11-presence-realtime.md` | cloud live ≥ **1.4.27**; target client **≥ 4.9.8** |
| System Recovery (policy/service/firewall allowlist snapshot + drift + confirm restore) | `agent/system-recovery.md` | cloud ≥ **1.4.13**; target client **≥ 4.9.12** |
| Network Guard dashboard panel (live IP/adapters + golden baseline + diff + restore) + `auto_restore_network` | `agent/network-guard.md` | cloud ≥ **1.4.14**; target client **≥ 4.9.12** |
| Network Guard maintenance mode (pause → change VPN/IP → snapshot → resume) | `agent/network-guard.md` | cloud ≥ **1.4.15**; target client **≥ 4.9.12** |
| VSS delete intent (kill + quarantine without shadow-count wait) | `agent/ransomware-shield.md` | cloud ≥ **1.4.16**; target client **≥ 4.9.14** |
| Network Guard soft surface inform (additive adapter/DHCP notify; Accept/Disable; no panic while online) | `agent/network-guard.md` | cloud ≥ **1.4.17**; target client **≥ 4.9.15** |
| Defense Policy (tiered + observe default + auto-promote) | `cloud/DEFENSE_POLICY.md`, `agent/defense-policy-client.md`, `ROADMAP_TIERED_DEFENSE.md` | cloud ≥ **1.4.19**; client **≥ 4.9.17** (matrix ≥ **4.9.16**) |
| One GUI/session + motor watchdog | `api/08-architecture.md` | 4.5.58–59 |
| Threat-intel GET/304/ACK | `api/09-threat-intel.md` | **4.5.61** |
| Threat-intel `HP-INTEL-*` apply + orphan/policy reconcile + ACK skipped/removed | `api/09-threat-intel.md`, `api/06-firewall-blocks.md` | **4.9.7** |
| Bare successful_logon no auto HP-BLOCK; silent-hours alert-only FW; whitelist enforce unblock | `agent/threat-engine.md`, `agent/gui-control-center.md` | **4.9.7** |
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
| Whitelist cloud SoT (merge-only persist + tablo cloud okur) | `agent/gui-control-center.md` | **4.8.4** |
| block-removed ACK ips(+ids); dashboard "Kaldırılıyor" takılma fix | `api/06-firewall-blocks.md` | **4.8.5** |

**Önerilen production floor:** **4.9.0** (RD v2 + recovery + NG containment + GUI control center + whitelist SoT + unblock ACK).

Avoid in production: **4.7.0–4.7.2** (NG FP); prefer **≥4.7.3** containment, **≥4.8.1** popup SoT, **≥4.8.5** unblock ACK, **≥4.9.0** RD v2.

Cloud publish: contract clone → `git pull` → `scripts/publish_contract.sh` (HTTPS mirror).
