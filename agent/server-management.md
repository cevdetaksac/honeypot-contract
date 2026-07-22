# Server Management — Dashboard ↔ Client

> Contract **1.4.8** · Status: **normative (additive)**  
> Production floor unchanged: **client ≥ 4.9.0**  
> Target ship: client **≥ 4.9.4** (or next patch that lands this package)  
> Related: [`api/03-control-websocket.md`](../api/03-control-websocket.md),
> [`api/07-lifecycle-sessions.md`](../api/07-lifecycle-sessions.md),
> [`api/05-remote-desktop.md`](../api/05-remote-desktop.md) (`list_local_users`),
> [`disaster-recovery.md`](./disaster-recovery.md) (`create_user`)

Cloud dashboard **Sunucu Yönetimi** surfaces three pages. The agent must
**return full current inventory** on refresh and **execute** every mutate
command already in the catalog. Missing data = empty UI; missing handler =
failed command result.

| Dashboard route | Purpose |
|-----------------|--------|
| `/dashboard/server/users` | Local accounts + active sessions |
| `/dashboard/server/processes` | Running processes |
| `/dashboard/server/services` | Windows services |

All commands travel via control WS / `GET /api/commands/pending` +
`POST /api/commands/result` (same envelope, HMAC, scrub rules as api/03).

---

## Non-goals

- Covert remote shell / arbitrary PowerShell execution
- Domain controller / AD DS management (local SAM + WTS only for v1)
- Silent destructive actions without cloud `confirm:true` where gated

---

## 1) Continuous inventory (health)

Keep these fresh on the normal health cadence ([`polling.md`](./polling.md)):

| Field | Destination |
|-------|-------------|
| `active_sessions[]` | Users page + overview |
| `top_processes[]` | Processes page + overview |

Shapes: [`api/07-lifecycle-sessions.md`](../api/07-lifecycle-sessions.md).

Dashboard “Yenile” also queues explicit list commands (below); do **not** rely
only on the next health tick — honor list commands immediately.

---

## 2) Inventory commands (read)

### `list_local_users`

**Params:**

```json
{ "include_disabled": true }
```

| Param | Default | Notes |
|-------|---------|--------|
| `include_disabled` | `false` (legacy remote) | Server Management always sends **`true`** |

**Result `data`:**

```json
{
  "users": [
    {
      "username": "Administrator",
      "sid": "S-1-5-21-…",
      "enabled": true,
      "is_admin": true,
      "groups": ["Administrators"],
      "full_name": "",
      "last_logon": "2026-07-22T06:00:00Z",
      "has_session": true,
      "session_id": 2,
      "session_status": "Active"
    }
  ]
}
```

- **Never** include passwords / hashes / DPAPI blobs.
- Cloud caches `data.users` under client settings `remote_local_users` (≤200).
- Also used by Remote Desktop user dropdown.

### `list_sessions`

**Params:** `{}`  
**Behavior:** Immediately push updated `active_sessions` via health report
**and** return the same array in `data.sessions` (or `data.active_sessions`).

Each row should include at least: `username`, `session_id`, `status`,
`protocol` / `session_name`, `client_ip`, `login_time`, `can_capture`
(see remote-desktop + lifecycle docs).

### `list_processes`

**Params:** `{}`  
**Behavior:** Immediately refresh `top_processes` (health) **and** return
`data.processes` / `data.top_processes` with the rich row shape from
[`api/07-lifecycle-sessions.md`](../api/07-lifecycle-sessions.md)
(`pid`, `name`, `cpu_percent`, `memory_mb`, `path`, `username`, …).

Cap ≤ 150 unique PIDs (top CPU ∪ top memory ∪ suspicious).

### `list_services` **(additive — required for Services page table)**

Cloud whitelist live. Older clients that ignore unknown types must still
ACK `failed`/`unsupported` honestly; do not hang pending forever.

**Params:**

```json
{
  "include_drivers": false,
  "include_stopped": true
}
```

**Result `data`:**

```json
{
  "services": [
    {
      "name": "TermService",
      "display_name": "Remote Desktop Services",
      "status": "Running",
      "start_type": "Manual",
      "pid": 1234
    }
  ]
}
```

| Field | Required | Notes |
|-------|----------|--------|
| `name` | yes | SCM service name (not display) |
| `display_name` | yes | Localized display |
| `status` | yes | Prefer `Running` / `Stopped` / `StartPending` / … |
| `start_type` | yes | `Automatic` / `Manual` / `Disabled` / … |
| `pid` | no | When running |

Cloud caches under `windows_services.services` (≤500). Dashboard filters
client-side.

---

## 3) Mutate commands (write) — must execute

Destructive ones already require dashboard confirm + cloud `confirm:true`
where listed in api/03.

### Users / sessions

| `command_type` | Params | Client action |
|----------------|--------|----------------|
| `create_user` | `username`, `password`, `groups[]`, `if_exists` (`fail`\|`reset_enable`), `enable?` | Local SAM create / reset+enable — [`disaster-recovery.md`](./disaster-recovery.md) |
| `enable_account` | `username` | `NetUserSetInfo` / equivalent active |
| `disable_account` | `username` | Active:no — respect protected accounts |
| `reset_password` | `username`, `new_password` (≥8) | Set password; **never log** plaintext |
| `logoff_user` | `username` and/or `session_id` | WTSLogoffSession |
| `disable_all_users` | `logoff?`, `exclude[]` | Panic lock — Administrator included unless excluded |

**Account delete:** not in v1 wire. UI uses **disable**. Optional future
`delete_user` needs a separate additive promotion (cloud whitelist + confirm).

### Processes

| `command_type` | Params | Client action |
|----------------|--------|----------------|
| `kill_process` | `pid`, `process_name` | Terminate; never self / Guardian critical |
| `block_process` | `process_name` / `path` / `name_pattern` | Persistent block rule |
| `suspend_process` / `resume_process` | exact identity fields | ≥4.7.3 network-guard rules |

### Services

| `command_type` | Params | Client action |
|----------------|--------|----------------|
| `start_service` | `name` **or** `service_name` | `StartService` |
| `stop_service` | `name` **or** `service_name` | `ControlService STOP` |
| `restart_service` | `name` **or** `service_name` | stop → start |

Accept **both** param keys (dashboard sends both). Prefer SCM name over
display name. Return clear `error` if access denied / dependent services.

---

## 4) Result envelope

Always report via `POST /api/commands/result`:

```json
{
  "token": "<agent-token>",
  "command_id": "<id>",
  "status": "completed",
  "success": true,
  "message": "ok",
  "data": { }
}
```

On failure: `status: "failed"`, `success: false`, `error` machine code
(`ACCESS_DENIED`, `NOT_FOUND`, `BUSY`, `UNSUPPORTED`, …) + human `message`.
Do not leave commands `running` indefinitely.

After successful mutate, best-effort refresh related inventory (users /
processes / services) so the next dashboard poll is current.

---

## 5) Security

- Password / `new_password` / `pin`: RAM only; never disk, never threat logs.
- Protected local accounts (SYSTEM, etc.): refuse `disable_account` as today.
- Self-process / Guardian / update-lock: refuse `kill_process`.
- Service stop that would kill the agent motor: refuse or defer with
  `PROTECTED_SERVICE` (document in result).

---

## 6) Client TODO / acceptance

### Inventory

- [ ] `list_local_users` with `include_disabled:true` returns disabled accounts
- [ ] `list_sessions` + health `active_sessions` stay populated on live RDP/console
- [ ] `list_processes` returns rich rows (path/username/cpu/memory) ≤150
- [ ] **`list_services` implemented** — table fills on Services → Yenile
- [ ] Cloud cache visible after completed result (`remote_local_users` /
      `windows_services`)

### Mutate

- [ ] Users: create / enable / disable / reset_password / logoff / disable_all
- [ ] Processes: kill / block (suspend/resume already ≥4.7.3)
- [ ] Services: start / stop / restart accepting `name` **and** `service_name`
- [ ] Every command returns terminal `commands/result` within TTL

### UX smoke (one lab host)

1. Sunucu Yönetimi → Kullanıcılar → Yenile → local users + sessions rows  
2. Create test user → appears after refresh  
3. Reset password + disable/enable round-trip  
4. Süreçler → Yenile → kill a benign test process  
5. Servisler → Yenile → `list_services` rows → stop/start a harmless service
   (e.g. Spooler on lab only)

---

## 7) Cloud already live (no client wait)

- Routes: `/dashboard/server/{users,processes,services}`
- Command whitelist includes `list_services` + existing mutates
- Caches `list_local_users` / `list_services` results for UI
- Confirm modal + signing degrade gate for destructive IR
