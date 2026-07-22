# Offline urgent queue — design (OOB-501)

> Status: **design / non-normative**  
> Contract: 1.4.5  
> Client primitive: `cloud-client/client_offline_queue.py` (DPAPI + HMAC,
> bounded, idempotent) — **not wired** into `alerts/urgent` until this design
> is promoted.

## Goal

Survive brief cloud/network outages without dropping high-severity local
signals (canary, Network Guard offline bomb, password-burst). Re-POST on
reconnect with ACK/idempotency so dashboards do not duplicate incidents.

## Non-goals

- Covert DNS/ICMP channels
- Unbounded spool / forensic full dumps
- Silent “fire and forget” without ACK (duplicate risk)

## Proposed wire (future promote)

### Client → cloud ingest

`POST /api/alerts/urgent/batch` (or extend existing urgent with
`idempotency_key`):

```json
{
  "events": [
    {
      "event_id": "<sha256-hex>",
      "queued_at": "2026-07-22T01:00:00.000000Z",
      "payload": { /* same shape as POST /api/alerts/urgent after redact */ }
    }
  ]
}
```

Caps (client already approximates):

| Cap | Value |
|-----|-------|
| Max records on disk | 500 |
| Max payload size | existing urgent redaction limits |
| TTL | 7 days (drop expired locally) |

### Cloud → ACK

```json
{
  "acked": ["<event_id>", "..."],
  "duplicate": ["..."],
  "rejected": [{ "event_id": "…", "reason": "schema|too_large|expired" }]
}
```

Client deletes only `acked` + `duplicate`. Retries `rejected` only when
`reason` is transient.

## Redaction

`payload` must already pass `redact_sensitive`. No passwords, tokens, typed
text, TURN credentials, or private keys. Canary paths allowed only as in
current urgent wire (description), not in offline metadata beyond
`event_id`.

## Client wiring gate

Do **not** enqueue on every urgent success timeout until cloud ACK exists.
Preferred sequence:

1. Promote this file into `api/` with VERSION bump.
2. Cloud implements batch+ACK + dashboard dedupe by `event_id`.
3. Client flag `security.offline_urgent_queue` default off; drain after
   successful heartbeat/WS.

## Acceptance

- [ ] Idempotent: double-delivery → one dashboard incident
- [ ] Offline 10m canary → appears after reconnect
- [ ] Full disk / 500 cap → oldest dropped with local counter
- [ ] No DNS/ICMP fallback
