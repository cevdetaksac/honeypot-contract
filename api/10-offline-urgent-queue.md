# Offline urgent queue (OOB-501)

> Contract **1.4.7** · Status: **normative (additive)**  
> Production floor unchanged: **client ≥ 4.9.0**  
> Client flag `security.offline_urgent_queue` remains **default off** until
> fleet pilot. Cloud endpoint is live; drain only after successful
> heartbeat / control WS.

Survives brief cloud/network outages without dropping high-severity local
signals (canary, Network Guard offline bomb, password-burst). Re-POST on
reconnect with ACK/idempotency so dashboards do not duplicate incidents.

Design history: [`../cloud/offline-urgent-queue-design.md`](../cloud/offline-urgent-queue-design.md).

## Non-goals

- Covert DNS/ICMP channels
- Unbounded spool / forensic full dumps
- Silent “fire and forget” without ACK

## Caps

| Cap | Value |
|-----|-------|
| Max events per batch | **500** |
| Max payload JSON size | **200 KB** per event |
| Local disk records (client) | 500 |
| Local TTL (client) | 7 days — drop expired locally |
| Cloud `expired` reject | `queued_at` older than 7 days |

## Health observe (additive)

`POST /api/health/report` may include:

```json
{
  "offline_urgent_queue": {
    "mode": "observe",
    "enabled": false,
    "pending": 0,
    "max_records": 500,
    "max_batch": 500,
    "max_payload_bytes": 204800,
    "ttl_sec": 604800,
    "oldest_dropped": 0,
    "expired_dropped": 0,
    "too_large_rejected": 0
  }
}
```

`oldest_dropped` is durable across client restarts (local stats file). Flag
`enabled` stays false until pilot.

## POST `/api/alerts/urgent` — soft idempotency

Additive fields (optional; missing = legacy single-shot urgent):

| Field | Meaning |
|-------|---------|
| `event_id` | Stable offline-queue id (prefer sha256 hex, ≤ 64 chars) |
| `idempotency_key` | Alias of `event_id` |

When either is present (or `alert_id` equals that stable id):

- First delivery creates the dashboard incident with `alert_id = event_id`.
- Replay returns:

```json
{
  "status": "ok",
  "alert_id": "<event_id>",
  "duplicate": true,
  "acked": true,
  "message": "Idempotent replay — existing alert returned"
}
```

Payload shape otherwise matches existing
[`../agent/threat-engine.md`](../agent/threat-engine.md) urgent wire.
`payload` / body must already be redacted (no passwords, tokens, PIN,
TURN credentials, private keys).

## POST `/api/alerts/urgent/batch`

### Request

```json
{
  "token": "<agent-token>",
  "events": [
    {
      "event_id": "<sha256-hex>",
      "queued_at": "2026-07-22T01:00:00.000000Z",
      "payload": { /* same shape as POST /api/alerts/urgent after redact */ }
    }
  ]
}
```

Bearer token auth also accepted (`Authorization: Bearer …`).

### Response

```json
{
  "status": "ok",
  "mode": "observe",
  "acked": ["<event_id>", "..."],
  "duplicate": ["..."],
  "rejected": [{ "event_id": "…", "reason": "schema|too_large|expired|transient" }]
}
```

| Bucket | Client action |
|--------|----------------|
| `acked` | Delete local queue row |
| `duplicate` | Delete local queue row (already on dashboard) |
| `rejected` `schema` / `too_large` / `expired` | Do **not** retry; drop or quarantine locally |
| `rejected` `transient` | Retry later |

### Reject reasons

| reason | When |
|--------|------|
| `schema` | Missing `event_id` / `payload`, malformed JSON object |
| `too_large` | Serialized payload > 200 KB |
| `expired` | `queued_at` older than 7 days |
| `transient` | Server/storage error — safe to retry |

## Cloud requirements

1. Dedupe by `event_id` ≡ `ThreatAlert.alert_id` (unique).
2. Batch processes ≤ 500 events; HTTP 400 if over cap.
3. Never create a second dashboard incident for the same `event_id`.
4. Redaction middleware must scrub secrets in request logs.
5. Do **not** require the client flag to be on — endpoints stay additive.

## Client wiring

1. Keep `security.offline_urgent_queue` **default off**.
2. Enqueue only high-severity local signals on urgent POST failure / offline.
3. Drain via `/api/alerts/urgent/batch` after successful heartbeat or control WS.
4. Delete only `acked` + `duplicate`.

## Acceptance

- [x] Idempotent: double-delivery → one dashboard incident (cloud E2E)
- [~] Offline 10m canary → appears after reconnect — **harness green**
  (`tests/test_offline_queue_pilot.py`, client **4.9.3**); **one-host live
  pilot** remains the final gate (`security.offline_urgent_queue=true` →
  ~10m offline + canary → reconnect drain → single dashboard incident →
  disable flag on that host; fleet default **off**)
- [x] Full disk / 500 cap → oldest dropped with durable counter
  (`offline_urgent_queue.oldest_dropped` on health/report; **accepted
  client 4.9.3**)
- [x] No DNS/ICMP fallback (out of scope / rejected)

### One-host live pilot (flag stays fleet-off)

1. Pick **one** test host on client **≥ 4.9.3** (not production fleet).
2. Set `security.offline_urgent_queue: true` **only on that host**; leave default **off** elsewhere.
3. Block / drop cloud reachability ~10 minutes; trigger a canary (or other high-severity urgent).
4. Restore network → heartbeat or control WS reconnect → expect `/api/alerts/urgent/batch` drain.
5. Pass criteria: **one** dashboard incident for the stable `event_id`; health shows queue counters; no duplicate alert rows.
6. After pass: turn flag **off** again on the pilot host (or leave on only that lab box).
