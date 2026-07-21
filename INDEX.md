# Contract INDEX

> Oku: [`VERSION`](VERSION) → bu dosya → satırdaki MD.  
> **VERSION 1.4.1** · Repo: https://github.com/cevdetaksac/honeypot-contract
> Fleet matrix: [`FLEET.md`](FLEET.md) · Production floor: **client ≥ 4.9.0** (safe containment + healthy IPC + bounded logs + GUI control center v2 + consistent detail popups + cloud-managed webhook + dashboard PIN yönetimi + IP hızlı aksiyonları + whitelist cloud SoT + block-removed ACK ips + **Remote Desktop v2**: WS/JPEG healthy path, input protocol 2, persistent session helper, optional WebRTC signaling)

## Shared delivery plans

| Dosya | Konu | Statü |
|-------|------|-------|
| [SECURITY_RESILIENCE_VNEXT.md](SECURITY_RESILIENCE_VNEXT.md) | Client + Cloud + Dashboard güvenlik/resilience ortak iş paketleri, sahiplik ve rollout kapıları | Planlama; normative wire değişikliği değil |

## agent/

| Dosya | Konu | Min client |
|-------|------|------------|
| [agent/CLIENT.md](agent/CLIENT.md) | İndeks / okuma sırası | ≥ **4.5.68** |
| [agent/polling.md](agent/polling.md) | Cadence tablosu | — |
| [agent/register-protection.md](agent/register-protection.md) | `protection.block_rules` | ≥ **4.5.66** |
| [agent/ransomware-shield.md](agent/ransomware-shield.md) | Canary UX, quarantine, unlock | ≥ **4.5.65** |
| [agent/persistence-and-tamper.md](agent/persistence-and-tamper.md) | Guardian servisi, çapraz watchdog, tamper wire | ≥ **4.6.0** |
| [agent/disaster-recovery.md](agent/disaster-recovery.md) | `create_user`, `remote_logon` (autologon break-glass) | ≥ **4.6.0** |
| [agent/network-guard.md](agent/network-guard.md) | Offline fidye alarmı, ağ baseline; otomatik müdahale yok, onaylı suspend | ≥ **4.7.3** |
| [agent/gui-control-center.md](agent/gui-control-center.md) | Dinamik GUI, güvenlik katmanları, koruma şeridi + Ayarlar sekmesi, sayaç/action + detay popup veri-kaynağı invariant'ları | ≥ **4.7.3** (şerit/Ayarlar ≥ **4.8.0**, popup kaynağı ≥ **4.8.1**) |
| [agent/log-retention.md](agent/log-retention.md) | Günlük yerel loglar, 7 gün retention, updater uyumluluğu | ≥ **4.7.6** |
| [agent/attacks-and-services.md](agent/attacks-and-services.md) | `/api/attack`, bait tunnels, open-ports, tunnel cmd expiry | — |
| [agent/threat-engine.md](agent/threat-engine.md) | v4 urgent/batch/health/config/auto-block | — |
| [agent/remote-input.md](agent/remote-input.md) | Input protocol 2 + persistent session helper (frame `inputs[]` baseline ≥4.5.55) | ≥ **4.9.0** |

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
| [api/09-threat-intel.md](api/09-threat-intel.md) | Intel bundle ETag/ACK/WS | ≥ **4.5.61** / push ≥4.5.66 |

## cloud/

| Dosya | Konu |
|-------|------|
| [cloud/overview.md](cloud/overview.md) | Mimari özet (agent-visible) |
| [cloud/threat-intel-ingest.md](cloud/threat-intel-ingest.md) | Harici feed → bundle (agent çekmez) |

## Sprint checklist

- [x] Register body → `update_block_rules(protection.block_rules)` (client ≥4.5.66)
- [x] WS `threat_intel_updated` handler (client ≥4.5.66)
- [x] `agent/ransomware-shield.md` + architecture `RS_STATUS` / `RS_UNLOCK`
- [x] `FLEET.md` version matrix
- [ ] Cadence `polling.md` ile canlı host doğrulama (opsiyonel)
- [ ] Live host: 3 fail → `HP-BLOCK-*` (unit covered)
