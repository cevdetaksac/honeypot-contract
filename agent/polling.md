# Agent polling & presence cadence

> **Contract VERSION:** root `VERSION`  
> Değerler önerilen; WS sağlıklıyken HTTP seyreltilir.

| Kanal | Aralık | Not |
|-------|--------|-----|
| Control WS | kalıcı | Primary komut; ping ~25–30s |
| `GET /api/commands/pending` | WS up: 15–30s · WS down: ~1s | Safety net |
| `POST /api/heartbeat` | ~60s | Presence; `system_context` / version |
| `GET /api/client_status` | GUI / isteğe | presence, control_ws, threat_intel ack |
| `GET /api/agent/pending-blocks` | ~30s | + pending-unblocks |
| `POST /api/agent/sync-rules` | mutate sonrası / periyodik | HP-BLOCK envanter |
| `GET /api/threats/config` | startup + ~5 dk | + `protection` |
| `GET /api/agent/threat-intel` | startup + 15–30 dk | ETag/304; WS `threat_intel_updated` anında |
| `POST /api/health/report` | ~1–5 dk | |
| `POST /api/agent/open-ports` | ~5 dk | |
| `GET /api/premium/tunnel-status` | ~30–60s | Desired bait services |
| `POST /api/agent/tunnel-status` | state değişince | |
| Attack / urgent | event-driven | |
| RD frames | stream aktifken | `inputs[]` piggyback |

**Offline:** heartbeat ~2 dk yok → cloud presence stale (dashboard).
