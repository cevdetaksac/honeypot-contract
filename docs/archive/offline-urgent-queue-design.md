# Offline urgent queue — design archive (OOB-501)

> Status: **promoted** → [`../api/10-offline-urgent-queue.md`](../../api/10-offline-urgent-queue.md)  
> Contract: **1.4.7**  
> This file is retained as design history only. Normative wire lives under `api/`.

## Goal

Survive brief cloud/network outages without dropping high-severity local
signals (canary, Network Guard offline bomb, password-burst). Re-POST on
reconnect with ACK/idempotency so dashboards do not duplicate incidents.

## Promotion checklist (done)

1. [x] Promote into `api/` with VERSION bump (**1.4.7**)
2. [x] Cloud implements batch+ACK + dashboard dedupe by `event_id`
3. [ ] Client flag `security.offline_urgent_queue` default off; drain after
      successful heartbeat/WS (client pilot)

## Cloud live endpoints

- `POST /api/alerts/urgent` — soft idempotency (`event_id` / `idempotency_key`)
- `POST /api/alerts/urgent/batch` — ACK `{acked, duplicate, rejected}`

See the normative doc for caps, reject reasons and acceptance.
