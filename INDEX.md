# Contract INDEX

> Oku: [`VERSION`](VERSION) → bu dosya → satırdaki MD.  
> **VERSION 1.1.0** · Repo: https://github.com/cevdetaksac/honeypot-contract

## agent/

| Dosya | Konu | Min client |
|-------|------|------------|
| [agent/CLIENT.md](agent/CLIENT.md) | İndeks / okuma sırası | — |
| [agent/polling.md](agent/polling.md) | Cadence tablosu | — |
| [agent/register-protection.md](agent/register-protection.md) | `protection.block_rules` | sprint |
| [agent/attacks-and-services.md](agent/attacks-and-services.md) | `/api/attack`, bait tunnels, open-ports | — |
| [agent/threat-engine.md](agent/threat-engine.md) | v4 urgent/batch/health/config/auto-block | — |
| [agent/remote-input.md](agent/remote-input.md) | Frame `inputs[]` | ≥ 4.5.55 |

## api/

| Dosya | Konu | Min client |
|-------|------|------------|
| [api/01-auth.md](api/01-auth.md) | Bearer, register, heartbeat, status | 4.4.33+ |
| [api/02-account.md](api/02-account.md) | Account link / multi-server | — |
| [api/03-control-websocket.md](api/03-control-websocket.md) | Control WS + **VALID_COMMAND_TYPES** | 4.5.x |
| [api/04-self-update.md](api/04-self-update.md) | Self-update ACK | 4.5.39+ |
| [api/05-remote-desktop.md](api/05-remote-desktop.md) | Remote stream / prepare | — |
| [api/06-firewall-blocks.md](api/06-firewall-blocks.md) | HP-BLOCK / sync / clear | 4.5.40+ |
| [api/07-lifecycle-sessions.md](api/07-lifecycle-sessions.md) | Lifecycle / sessions / processes | — |
| [api/08-architecture.md](api/08-architecture.md) | Daemon vs GUI | 4.5.58+ |
| [api/09-threat-intel.md](api/09-threat-intel.md) | Intel bundle ETag/ACK/WS | ≥ **4.5.61** |

## cloud/

| Dosya | Konu |
|-------|------|
| [cloud/overview.md](cloud/overview.md) | Mimari özet (agent-visible) |
| [cloud/threat-intel-ingest.md](cloud/threat-intel-ingest.md) | Harici feed → bundle (agent çekmez) |

## Sprint checklist

- [ ] Register body → `update_block_rules(protection.block_rules)`
- [ ] WS `threat_intel_updated` handler
- [ ] Cadence `polling.md` ile uyum
- [ ] Komut kataloğu `03` ↔ client executor
