# Contract INDEX

> Oku: [`VERSION`](VERSION) → bu dosya → satırdaki MD.  
> **VERSION 1.4.28** · Repo: https://github.com/cevdetaksac/honeypot-contract  
> Fleet: [`FLEET.md`](FLEET.md) · Production floor: **client ≥ 4.9.0**

### Recent contract highlights

| Contract | Konu | Client |
|----------|------|--------|
| **1.4.27** | Unlink API live + realtime presence wire | unlink ≥**4.9.26**; presence ≥**4.9.8** |
| **1.4.26** | Hardware-bound `machine_id` (clone split) | ≥**4.9.28** |
| **1.4.25** | Account unlink Spec (Settings) | ≥**4.9.26** |
| **1.4.24** | Server Users C-USER | cloud; list ≥**4.9.4** |
| **1.4.23** | Winlogon sibling pre_logon + C-WL | ≥**4.9.26** |
| **1.4.20** | WebRTC smoothness | ≥**4.9.20** |
| **1.4.19** | Defense Policy observe→balanced | ≥**4.9.17** |
| Design-only | ZT envelope/keys — [`cloud/ZERO_TRUST_STATUS.md`](cloud/ZERO_TRUST_STATUS.md) | — |
| Decision | Branding — [`cloud/PRODUCT_BRANDING.md`](cloud/PRODUCT_BRANDING.md) | — |

---

## Shared delivery plans

| Dosya | Konu | Statü |
|-------|------|-------|
| [ROADMAP_TIERED_DEFENSE.md](ROADMAP_TIERED_DEFENSE.md) | Kademeli savunma, onboarding, anti-bait | Planning (living) |
| [SECURITY_RESILIENCE_VNEXT.md](SECURITY_RESILIENCE_VNEXT.md) | Ortak güvenlik paketleri; §7A observe şemaları api/agent’ta | Plan + promoted observe |
| [docs/archive/](docs/archive/) | Promoted design dumps / legacy prompts | Archive |

---

## agent/

| Dosya | Konu | Min client |
|-------|------|------------|
| [agent/CLIENT.md](agent/CLIENT.md) | İndeks / okuma sırası | ≥ **4.5.68** |
| [agent/polling.md](agent/polling.md) | Cadence tablosu | — |
| [agent/register-protection.md](agent/register-protection.md) | `protection.block_rules` | ≥ **4.5.66** |
| [agent/ransomware-shield.md](agent/ransomware-shield.md) | Canary UX, quarantine, unlock | ≥ **4.5.65** |
| [agent/persistence-and-tamper.md](agent/persistence-and-tamper.md) | Guardian, watchdog, tamper | ≥ **4.6.0** |
| [agent/disaster-recovery.md](agent/disaster-recovery.md) | `create_user`, `remote_logon` | ≥ **4.6.0** |
| [agent/network-guard.md](agent/network-guard.md) | Offline alarm, golden network, soft inform, contain | ≥ **4.7.3** (panel ≥**4.9.12**; soft ≥**4.9.15**) |
| [agent/system-recovery.md](agent/system-recovery.md) | Surface snapshot / drift / restore | target ≥ **4.9.12** |
| [agent/gui-control-center.md](agent/gui-control-center.md) | GUI katmanlar, şerit, Ayarlar, popup SoT | ≥ **4.7.3** (şerit ≥**4.8.0**) |
| [agent/log-retention.md](agent/log-retention.md) | Yerel log 7 gün | ≥ **4.7.6** |
| [agent/attacks-and-services.md](agent/attacks-and-services.md) | Attack, bait tunnels, ports | — |
| [agent/threat-engine.md](agent/threat-engine.md) | Urgent/batch/health/config; whitelist enforce | ≥ **4.9.7** |
| [agent/defense-policy-client.md](agent/defense-policy-client.md) | Matrix apply, observe default, CTA | ≥ **4.9.17** |
| [agent/remote-input.md](agent/remote-input.md) | Input protocol 2 + session helper | ≥ **4.9.0** |
| [agent/server-management.md](agent/server-management.md) | Users / processes / services | target ≥ **4.9.4** |

---

## api/

| Dosya | Konu | Min client |
|-------|------|------------|
| [api/01-auth.md](api/01-auth.md) | Bearer, register, heartbeat, `machine_id` | 4.4.33+ / hw-id ≥**4.9.28** |
| [api/02-account.md](api/02-account.md) | Account link / unlink / multi-server | unlink ≥**4.9.26** |
| [api/03-control-websocket.md](api/03-control-websocket.md) | Control WS + komutlar + HMAC | 4.5.x |
| [api/04-self-update.md](api/04-self-update.md) | Self-update ACK | 4.5.39+ |
| [api/05-remote-desktop.md](api/05-remote-desktop.md) | RD v2 (WS/JPEG + WebRTC) | **≥4.9.0** (smooth ≥**4.9.20**) |
| [api/06-firewall-blocks.md](api/06-firewall-blocks.md) | HP-BLOCK / sync / clear | 4.5.40+ |
| [api/07-lifecycle-sessions.md](api/07-lifecycle-sessions.md) | Lifecycle / sessions / processes | — |
| [api/08-architecture.md](api/08-architecture.md) | Daemon vs GUI IPC | ≥ **4.5.66** |
| [api/09-threat-intel.md](api/09-threat-intel.md) | Intel bundle ETag/ACK/WS | ≥ **4.5.61** / apply ≥**4.9.7** |
| [api/10-offline-urgent-queue.md](api/10-offline-urgent-queue.md) | OOB-501 batch + idempotency | cloud live; client flag **off** |
| [api/11-presence-realtime.md](api/11-presence-realtime.md) | Sleep/suspend/shutdown presence | cloud live ≥**1.4.27**; client ≥**4.9.8** |

---

## cloud/

| Dosya | Konu | Statü |
|-------|------|-------|
| [cloud/overview.md](cloud/overview.md) | Mimari özet | Normative |
| [cloud/DEFENSE_POLICY.md](cloud/DEFENSE_POLICY.md) | Tiered policy + auto-promote (C-P0…) | Normative (1.4.19) |
| [cloud/SERVER_USER_MANAGEMENT.md](cloud/SERVER_USER_MANAGEMENT.md) | Users C-USER-1…7 | Normative (1.4.24) |
| [cloud/REMOTE_DESKTOP_WINLOGON.md](cloud/REMOTE_DESKTOP_WINLOGON.md) | Winlogon / pre_logon C-WL | Normative (1.4.23) |
| [cloud/REMOTE_DESKTOP_SMOOTHNESS.md](cloud/REMOTE_DESKTOP_SMOOTHNESS.md) | WebRTC smoothness C-RD | Normative (1.4.20) |
| [cloud/threat-intel-ingest.md](cloud/threat-intel-ingest.md) | Harici feed → bundle | Normative |
| [cloud/PRODUCT_BRANDING.md](cloud/PRODUCT_BRANDING.md) | Wire identity / no big-bang rename | Decision |
| [cloud/ZERO_TRUST_STATUS.md](cloud/ZERO_TRUST_STATUS.md) | ZT do/don’t | Design-only |
| [cloud/command-envelope-v2-design.md](cloud/command-envelope-v2-design.md) | ZT-601 envelope | Design-only |
| [cloud/operator-keyset-design.md](cloud/operator-keyset-design.md) | ZT-602/603 keys | Design-only |
| [cloud/offline-urgent-queue-design.md](cloud/offline-urgent-queue-design.md) | Pointer → archive + api/10 | Promoted stub |

---

## Opsiyonel lab (zorunlu değil)

- Cadence: `agent/polling.md` canlı host doğrulama  
- Live host: 3 fail → `HP-BLOCK-*` smoke (`agent/register-protection.md`)
