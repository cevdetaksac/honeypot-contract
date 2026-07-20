# Contract INDEX

> Oku sırası: [`VERSION`](VERSION) → bu dosya → ilgili satırdaki MD.  
> Contract VERSION: **1.0.0** (dosya `VERSION` ile aynı tut)

## agent/ — Windows agent davranışı

| Dosya | Konu | Min client |
|-------|------|------------|
| [agent/CLIENT.md](agent/CLIENT.md) | Auth, Control WS, Self-update, IR, Remote özet | 4.5.44+ |
| [agent/register-protection.md](agent/register-protection.md) | `POST /register` yanıtı `protection.block_rules` (örn. 3 fail→block) | TBD |
| [agent/remote-input.md](agent/remote-input.md) | Frame / frame-json ACK `inputs[]` anında uygula | ≥ 4.5.55 |

## api/ — HTTP/WS endpoint sözleşmeleri

| Dosya | Konu | Min client |
|-------|------|------------|
| [api/01-auth.md](api/01-auth.md) | Bearer auth, register upsert, token storage | 4.4.33+ |
| [api/02-account.md](api/02-account.md) | Account link / membership | — |
| [api/03-control-websocket.md](api/03-control-websocket.md) | `wss://…/ws/agent/control` | 4.5.x |
| [api/04-self-update.md](api/04-self-update.md) | Self-update ACK / lifecycle | 4.5.39+ |
| [api/05-remote-desktop.md](api/05-remote-desktop.md) | Remote desktop stream | — |
| [api/06-firewall-blocks.md](api/06-firewall-blocks.md) | HP-BLOCK, sync-rules, clear | 4.5.40+ |
| [api/07-lifecycle-sessions.md](api/07-lifecycle-sessions.md) | Lifecycle / sessions | — |
| [api/08-architecture.md](api/08-architecture.md) | SYSTEM daemon vs GUI IPC | 4.5.58+ |
| [api/09-threat-intel.md](api/09-threat-intel.md) | GET ETag/304, ACK, WS `threat_intel_updated` | ≥ **4.5.61** |

## cloud/ — Cloud-only ingest (agent uygulamaz)

| Dosya | Konu |
|-------|------|
| [cloud/threat-intel-ingest.md](cloud/threat-intel-ingest.md) | Cloud’un harici feed’lerden bundle üretmesi (agent bu feed’leri **çekmez**) |

## Sprint checklist (özellikle)

- [ ] `agent/CLIENT.md` — Auth / Control WS / Self-update / IR / Remote
- [ ] `agent/register-protection.md` — register → `protection.block_rules`
- [ ] `agent/remote-input.md` — `inputs[]` piggyback apply
- [ ] `api/09-threat-intel.md` — ETag/304 + ACK + WS notify
