# Contract INDEX

> Oku: [`VERSION`](VERSION) → bu dosya → satırdaki MD.  
> **VERSION 1.4.21** · Repo: https://github.com/cevdetaksac/honeypot-contract
> Fleet matrix: [`FLEET.md`](FLEET.md) · Production floor: **client ≥ 4.9.0** (… + Winlogon pre-logon **1.4.21** / client **≥ 4.9.21**; RD WebRTC smoothness **1.4.20** / **≥ 4.9.20**; Defense Policy **1.4.18–1.4.19** → **≥ 4.9.16/4.9.17**; soft inform **≥ 4.9.15**; ZT Design-only — [`cloud/ZERO_TRUST_STATUS.md`](cloud/ZERO_TRUST_STATUS.md); branding — [`cloud/PRODUCT_BRANDING.md`](cloud/PRODUCT_BRANDING.md))


## Shared delivery plans

| Dosya | Konu | Statü |
|-------|------|-------|
| [ROADMAP_TIERED_DEFENSE.md](ROADMAP_TIERED_DEFENSE.md) | Kademeli savunma, observe→balanced onboarding, P0–P3, anti-bait | Planning (living) |
| [cloud/REMOTE_DESKTOP_WINLOGON.md](cloud/REMOTE_DESKTOP_WINLOGON.md) | Console Winlogon / pre-logon RD — cloud/viewer must-do (C-WL-*) | Normative (1.4.21) |
| [cloud/REMOTE_DESKTOP_SMOOTHNESS.md](cloud/REMOTE_DESKTOP_SMOOTHNESS.md) | WebRTC raw/HW encode + input fluidity — cloud/viewer must-do (C-RD-*) | Normative (1.4.20) |
| [SECURITY_RESILIENCE_VNEXT.md](SECURITY_RESILIENCE_VNEXT.md) | Client + Cloud + Dashboard güvenlik/resilience ortak iş paketleri; §7A schemas now normative in api/agent (observe-only) | Plan + promoted observe schemas |
| [cloud/offline-urgent-queue-design.md](cloud/offline-urgent-queue-design.md) | OOB-501 history — **promoted** → [`api/10-offline-urgent-queue.md`](api/10-offline-urgent-queue.md) | Design archive |
| [cloud/ZERO_TRUST_STATUS.md](cloud/ZERO_TRUST_STATUS.md) | ZT asymmetric envelope / keys: Design-only; cloud do/don’t | Decision (1.4.16) |
| [cloud/DEFENSE_POLICY.md](cloud/DEFENSE_POLICY.md) | Tiered policy + observe default + auto-promote — cloud must-do (C-P0…C-P2) | Normative (1.4.19) |
| [cloud/PRODUCT_BRANDING.md](cloud/PRODUCT_BRANDING.md) | HP-BLOCK / CloudHoneypot wire identity stays; no big-bang rename | Decision (1.4.16) |
| [cloud/operator-keyset-design.md](cloud/operator-keyset-design.md) | ZT-602/603 operator public key distribution | Design-only |
| [cloud/command-envelope-v2-design.md](cloud/command-envelope-v2-design.md) | ZT-601 asymmetric envelope + TPM PoP gate | Design-only |

## agent/

| Dosya | Konu | Min client |
|-------|------|------------|
| [agent/CLIENT.md](agent/CLIENT.md) | İndeks / okuma sırası | ≥ **4.5.68** |
| [agent/polling.md](agent/polling.md) | Cadence tablosu | — |
| [agent/register-protection.md](agent/register-protection.md) | `protection.block_rules` | ≥ **4.5.66** |
| [agent/ransomware-shield.md](agent/ransomware-shield.md) | Canary UX, quarantine, unlock | ≥ **4.5.65** |
| [agent/persistence-and-tamper.md](agent/persistence-and-tamper.md) | Guardian servisi, çapraz watchdog, tamper wire | ≥ **4.6.0** |
| [agent/disaster-recovery.md](agent/disaster-recovery.md) | `create_user`, `remote_logon` (autologon break-glass) | ≥ **4.6.0** |
| [agent/network-guard.md](agent/network-guard.md) | Offline fidye alarmı, golden ağ baseline, dashboard panel (IP/adaptör), `auto_restore_network`; soft surface inform; süreç contain onaylı | ≥ **4.7.3** (panel/auto network ≥ **4.9.12**; soft inform ≥ **4.9.15**) |
| [agent/system-recovery.md](agent/system-recovery.md) | Saldırı yüzeyi (policy/service/firewall) snapshot, drift, onaylı restore — full registry yok | target ≥ **4.9.12** |
| [agent/gui-control-center.md](agent/gui-control-center.md) | Dinamik GUI, güvenlik katmanları, koruma şeridi + Ayarlar sekmesi, sayaç/action + detay popup veri-kaynağı invariant'ları | ≥ **4.7.3** (şerit/Ayarlar ≥ **4.8.0**, popup kaynağı ≥ **4.8.1**) |
| [agent/log-retention.md](agent/log-retention.md) | Günlük yerel loglar, 7 gün retention, updater uyumluluğu | ≥ **4.7.6** |
| [agent/attacks-and-services.md](agent/attacks-and-services.md) | `/api/attack`, bait tunnels, open-ports, tunnel cmd expiry | — |
| [agent/threat-engine.md](agent/threat-engine.md) | v4 urgent/batch/health/config/auto-block; bare success no HP-BLOCK + whitelist enforce | ≥ **4.9.7** (policy) |
| [agent/defense-policy-client.md](agent/defense-policy-client.md) | defense_rules apply, observe default, onboarding CTA / promote backup | ≥ **4.9.17** |
| [agent/remote-input.md](agent/remote-input.md) | Input protocol 2 + persistent session helper (frame `inputs[]` baseline ≥4.5.55) | ≥ **4.9.0** |
| [agent/server-management.md](agent/server-management.md) | Dashboard Sunucu Yönetimi: users / processes / services inventory + mutates (`list_services` additive) | target ≥ **4.9.4** |

## api/

| Dosya | Konu | Min client |
|-------|------|------------|
| [api/01-auth.md](api/01-auth.md) | Bearer, register, heartbeat, status | 4.4.33+ |
| [api/02-account.md](api/02-account.md) | Account link / multi-server | — |
| [api/03-control-websocket.md](api/03-control-websocket.md) | Control WS + komut kataloğu + HMAC + `threat_intel_updated` | 4.5.x / intel push ≥4.5.66 |
| [api/04-self-update.md](api/04-self-update.md) | Self-update ACK | 4.5.39+ |
| [api/05-remote-desktop.md](api/05-remote-desktop.md) | Remote Desktop v2 (WS/JPEG + WebRTC signaling, prepare/stream) | prepare ≥4.5.48 / **v2 ≥4.9.0** |
| [api/06-firewall-blocks.md](api/06-firewall-blocks.md) | HP-BLOCK / sync / clear | 4.5.40+ |
| [api/07-lifecycle-sessions.md](api/07-lifecycle-sessions.md) | Lifecycle / sessions / processes | — |
| [api/08-architecture.md](api/08-architecture.md) | Daemon vs GUI IPC (`RS_*`) | ≥ **4.5.66** |
| [api/09-threat-intel.md](api/09-threat-intel.md) | Intel bundle ETag/ACK/WS; normative `HP-INTEL-*` apply | ≥ **4.5.61** / push ≥4.5.66 / apply ≥ **4.9.7** |
| [api/10-offline-urgent-queue.md](api/10-offline-urgent-queue.md) | OOB-501 urgent batch + idempotency ACK | cloud live; client flag default **off** |
| [api/11-presence-realtime.md](api/11-presence-realtime.md) | Anlık presence: sleep/suspend/shutdown goodbye | cloud ≥ **1.4.12**; target client **≥ 4.9.8** |

## cloud/

| Dosya | Konu |
|-------|------|
| [cloud/overview.md](cloud/overview.md) | Mimari özet (agent-visible) |
| [cloud/threat-intel-ingest.md](cloud/threat-intel-ingest.md) | Harici feed → bundle (agent çekmez) |
| [cloud/command-envelope-v2-design.md](cloud/command-envelope-v2-design.md) | ZT-601 v2 envelope + TPM/WebAuthn design gate (wire değil) |

## Sprint checklist

- [x] Register body → `update_block_rules(protection.block_rules)` (client ≥4.5.66)
- [x] WS `threat_intel_updated` handler (client ≥4.5.66)
- [x] `agent/ransomware-shield.md` + architecture `RS_STATUS` / `RS_UNLOCK`
- [x] `FLEET.md` version matrix
- [ ] Cadence `polling.md` ile canlı host doğrulama (opsiyonel)
- [ ] Live host: 3 fail → `HP-BLOCK-*` (unit covered)
