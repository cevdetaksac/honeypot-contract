# Agent polling & presence cadence

> **Contract VERSION:** root `VERSION`  
> Değerler önerilen; WS sağlıklıyken HTTP seyreltilir.

| Kanal | Aralık | Not |
|-------|--------|-----|
| Control WS | kalıcı | Primary komut; ping ~25–30s; **`presence` / `goodbye` anlık** ([`api/11-presence-realtime.md`](../api/11-presence-realtime.md)) |
| `GET /api/commands/pending` | WS up: 15–30s · WS down: ~1s | Safety net |
| `POST /api/heartbeat` | ~60s | Presence fallback; `system_context` / version |
| `POST /api/presence` | event-driven | Sleep/shutdown HTTP fallback (timeout ≤2s) |
| `GET /api/client_status` | GUI / dashboard ~2s | presence, suspend, control_ws, threat_intel ack |
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

**Offline / suspend:**  
- Explicit `goodbye` / `presence(suspend)` / `POST /api/presence` → dashboard **anında** (≤2s poll).  
- No signal: heartbeat/WS age → degraded ~90–150s; unexpected WS drop ~12s debounce.
