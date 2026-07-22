# Realtime agent presence (sleep / shutdown / offline)

> **Contract VERSION:** root `VERSION` (≥ **1.4.12**)  
> **Min client (target):** **≥ 4.9.8** (additive; older clients keep age-based presence)  
> **Cloud:** live — Control WS + `POST /api/presence` + dashboard `GET /api/client_status`  
> **Why:** Security dashboards must not show green “online” while the host is asleep or the agent is gone. Time-to-truth matters.

---

## Problem (current fleet without this)

| Signal today | Delay | Failure mode |
|--------------|-------|----------------|
| HTTP heartbeat ~60s | up to ~90–150s offline | PC sleep → zombie TCP → false online |
| Control WS ping ~25–30s | idle close ~75s; unexpected drop debounce ~12s | Still not instant on sleep |
| GUI quit | **must not** mark offline | SYSTEM daemon can stay up |

**Requirement:** intentional down events flip dashboard **within ~2s** of the OS/agent knowing.

---

## Presence states (dashboard / `client_status`)

| `presence` | UI | Meaning |
|------------|----|---------|
| `online` | green | Live pulse (WS ping and/or fresh heartbeat) |
| `degraded` | yellow | Pulse delayed (age window) |
| `suspend` | yellow “Uyku (suspend)” | Host sleep / hibernate — agent explicitly signaled |
| `offline` | gray | Shutdown, uninstall, crash/ws_lost, or stale pulse |

`GET /api/client_status` also returns:

```json
{
  "presence": "suspend",
  "presence_reason": "sleep",
  "presence_source": "presence",
  "presence_at": "2026-07-22T21:05:01Z",
  "alive": false,
  "control_ws": false,
  "last_seen_age_sec": 12,
  "online_max_age_sec": 90,
  "degraded_max_age_sec": 150
}
```

---

## Primary channel: Control WS

Endpoint: `wss://…/ws/agent/control` (same session as commands).

### Agent → `presence`

Send **before** the OS finishes suspending (best-effort; ≤500ms work).

```json
{
  "v": 1,
  "t": "presence",
  "state": "suspend",
  "reason": "sleep",
  "ts": "2026-07-22T21:05:01.123Z"
}
```

| `state` | `reason` (suggested) | Cloud action |
|---------|----------------------|--------------|
| `suspend` | `sleep` \| `hibernate` | Instant `presence=suspend` |
| `online` | `resume` \| `wake` | Clear down; online |
| `offline` | `shutdown` \| `operator_stop` \| `update` | Instant offline |
| `degraded` | rare | Delayed (optional) |

Aliases accepted: `sleep`/`hibernate` → suspend; `awake`/`resume`/`wake` → online.

Cloud replies:

```json
{ "t": "presence_ack", "accepted": true, "presence": "suspend", "server_time": "…" }
```

### Agent → `goodbye` (planned stop)

Send **immediately before** closing the control socket / stopping the daemon.

```json
{
  "v": 1,
  "t": "goodbye",
  "reason": "shutdown",
  "ts": "2026-07-22T21:06:00Z"
}
```

| `reason` | When |
|----------|------|
| `shutdown` | Service stop / OS shutdown / logoff that stops daemon |
| `uninstall` | Uninstall authorized path |
| `update` | Self-update stop (expect reconnect with new version) |
| `operator_stop` | User/admin stop |

Cloud: **immediate** `presence=offline` (no 12s flap debounce). Then close WS.

---

## HTTP fallback: `POST /api/presence`

Use when WS may already be dead, or as parallel fire-and-forget on sleep (timeout **≤ 2s**).

Auth: Bearer (and/or body `token`).

```json
{
  "token": "CLIENT_TOKEN",
  "state": "suspend",
  "reason": "sleep",
  "ts": "2026-07-22T21:05:01Z"
}
```

Response `200`:

```json
{ "status": "ok", "ok": true, "presence": "suspend", "reason": "sleep", "server_time": "…" }
```

`422` if `state` unknown.

**Client rule:** on `PBT_APMSUSPEND` / equivalent: try WS `presence` first; if send fails, `POST /api/presence` with short timeout; do not block suspend longer than ~2s.

---

## Lifecycle (optional audit trail)

Also (or instead of only presence) you may emit:

| `event_type` | Maps to presence |
|--------------|------------------|
| `host_sleep` | suspend / sleep |
| `host_hibernate` | suspend / hibernate |
| `host_resume` | online |
| `host_shutdown` | offline |
| `daemon_stopping` | offline |

via existing `POST /api/alerts/lifecycle`. Prefer **WS/HTTP presence first** (faster); lifecycle is for timeline history.

**Do not** treat `gui_quit` as host offline.

---

## Windows implementation notes (normative intent)

1. **SYSTEM daemon** owns presence (not tray GUI alone).
2. Register for power broadcast (`WM_POWERBROADCAST` / `SERVICE_CONTROL_POWEREVENT`):
   - `PBT_APMSUSPEND` / hibernate → `presence` suspend
   - `PBT_APMRESUMEAUTOMATIC` / `PBT_APMRESUMESUSPEND` → reconnect WS + `presence` online (or hello)
3. **Service stop / shutdown:** `goodbye` then close socket.
4. **Crash / kill -9:** no signal possible → cloud falls back to ping idle (~75s) / age (~90–150s). Acceptable for hard kill; not for sleep.
5. Keep WS ping **25–30s**. Reconnect backoff unchanged.

---

## Cloud behavior (already implemented)

| Event | Debounce | Result |
|-------|----------|--------|
| `goodbye` / explicit offline | **0s** | offline immediately |
| `presence` suspend | **0s** | suspend immediately |
| Unexpected WS drop | **~12s** | offline (`reason=ws_lost`) if no reconnect |
| No ping within idle | **~75s** | session dropped → gone path |
| No signal + stale `last_seen` | **90 / 150s** | online → degraded → offline |

Dashboard polls `client_status` ~**2s**.

---

## Acceptance tests (client + cloud)

1. **Sleep:** put host to sleep → dashboard shows **Uyku (suspend)** within **2s** (phone refresh / open dashboard).
2. **Wake:** resume → **Çevrimiçi** within a few seconds of WS hello/ping.
3. **Stop service:** stop YesNext daemon → **Çevrimdışı** within **2s** (goodbye).
4. **GUI quit only:** tray exit → remains **online** if daemon alive.
5. **Pull cable (no goodbye):** offline within ~12–75s (not instant — expected).

---

## Compatibility

- Clients **without** this contract: age + WS heuristics only (no false-green from zombie WS alone; still delayed on sleep).
- Shipping this is a **security UX** priority for personal PCs and laptops that sleep often.
