# Agent Control WebSocket + command catalog

> **Contract VERSION:** root `VERSION`  
> **Endpoint:** `wss://honeypot.yesnext.com.tr/ws/agent/control`  
> **Kim:** yalnızca SYSTEM daemon (`mode=daemon`)  
> **RD video:** `/ws/remote/agent` — ayrı kanal  

Kod SoT (cloud whitelist): `helpers.VALID_COMMAND_TYPES` (29 tip).  
`contain_user` **whitelist’te yok** — client/dashboard `logoff_user` + `reset_password` (+ opsiyonel `disable_account`) birleşimi kullanır.

---

## Bağlantı

| | |
|--|--|
| Auth | Bearer / `Sec-WebSocket-Protocol: bearer,<token>` |
| Ping | ~25–30s → `pong` |
| Reconnect | backoff 1→30s + jitter |
| Fallback | `GET /api/commands/pending` |
| Drain | WS open → pending komutları push |

### Agent → hello

```json
{ "v": 1, "t": "hello", "role": "agent", "version": "4.5.65", "hostname": "…", "pid": 1234, "mode": "daemon" }
```

### Cloud → agent

| `t` | Anlam |
|-----|--------|
| `hello_ack` / hello | bağlandı |
| `ping` / `pong` | keepalive |
| `command` | push IR / update / remote / … |
| `threat_intel_updated` | hemen `GET /api/agent/threat-intel` |
| `error` | |

### Command zarfı

```json
{
  "v": 1,
  "t": "command",
  "command": {
    "command_id": "uuid",
    "command_type": "self_update",
    "priority": "high",
    "expires_at": "…",
    "params": {}
  }
}
```

Result: **`POST /api/commands/result`** asıl SoT; WS `command_result` opsiyonel (aynı payload). Idempotent `command_id`.

---

## Command integrity (HMAC)

Client config: `security.command_signing` (default **true**).

| Yön | Ne zaman | Davranış |
|-----|----------|----------|
| Cloud → agent | `signature` alanı **varsa** | Agent `HMAC` doğrular; fail → reject (`Invalid command signature`) |
| Cloud → agent | `signature` **yok** | Geçiş: kabul (unsigned soft-allow); fleet hedefi signed |
| Agent → cloud | `POST /api/commands/result` | Result payload’a opsiyonel `signature` eklenir |

**İmza mesajı** (UTF-8):

```text
{command_id}|{command_type}|{issued_at}
```

- Algoritma: `HMAC-SHA256` → hex digest  
- Anahtar türetimi (client): `SHA256("{token}|{COMPUTERNAME}|yesnext-chp-v1")`  
- Cloud imza üretirken aynı token + kayıtlı hostname (`COMPUTERNAME`) kullanır  
- Self-process proof (`api/07-lifecycle-sessions.md`) **farklı** HMAC’tir; karıştırma

---

## Destructive commands — server confirmation

Aşağıdakiler **dashboard’da açık onay** olmadan cloud kuyruğa yazılmaz / push edilmez:

- `reset_password`
- `disable_account` / `disable_all_users`
- `enable_lockdown` (emergency lockdown)
- `clear_firewall` (`wipe_all_honeypot_rules=true` dahil)

Client ayrıca whitelist + protected targets uygular; onay **sunucu tarafı** zorunluluğudur.

---

## Canonical command catalog

| Type | Params (özet) | Amaç |
|------|----------------|------|
| `block_ip` | `ip` / `ip_or_cidr`, reason | HP-BLOCK ekle |
| `unblock_ip` | `ip` | Kaldır |
| `clear_firewall` | `wipe_all_honeypot_rules`, `ips[]` | Tüm honeypot FW wipe |
| `sync_firewall_rules` | — | Envanter yenile / hizala |
| `logoff_user` | `username` / `session_id` | Oturum kapat |
| `disable_account` / `enable_account` | `username` | Hesap |
| `reset_password` | `username`, `new_password` (≥8) | Agent uydurmasın |
| `disable_all_users` | `logoff`, `exclude[]` | Administrator **dahil** |
| `kill_process` | `pid` / name | Self PID korumalı |
| `block_process` | name/path | |
| `list_sessions` / `list_processes` / `list_local_users` | — | Remote/IR UI |
| `stop_service` / `start_service` / `restart_service` | `name` | |
| `enable_lockdown` / `disable_lockdown` | management_ip? | Acil |
| `collect_diagnostics` | — | |
| `remote_session_prepare` | `username`, `password`, `session_id?` | Active desktop |
| `remote_stream_start` / `remote_stream_stop` | `session_id` | |
| `remote_input` | event coords (tercihen HTTP queue) | |
| `remote_send_sas` | — | Ctrl+Alt+Del |
| `tunnel_start` / `tunnel_stop` | `service`, port? | Bait listen |
| `self_update` | `force`, `tag`, `download_url` | Installer |
| `check_update` | — | Sürüm kontrol |
| `unlock_ransomware_quarantine` | — | IFEO / quarantine temizle ([`../agent/ransomware-shield.md`](../agent/ransomware-shield.md)) |

Detay: self-update → [`04-self-update.md`](./04-self-update.md); remote → [`05-remote-desktop.md`](./05-remote-desktop.md) + [`../agent/remote-input.md`](../agent/remote-input.md); firewall → [`06-firewall-blocks.md`](./06-firewall-blocks.md).

---

## HTTP komut

- `GET /api/commands/pending` → liste  
- `POST /api/commands/result` → `{ command_id, status, result }` (`running` / `completed` / `failed`)  
- `GET /api/commands/{id}` — şifre alanları redacted  

---

## Acceptance

- [ ] Daemon WS bağlanır; send → &lt;500ms execute  
- [ ] WS kopunca poll kaçırmaz; reconnect drain  
- [ ] Bilinmeyen `command_type` reddedilir (cloud 400)  
- [x] `threat_intel_updated` → ETag GET (client ≥4.5.66)  
- [x] HMAC: signed command fail → reject; unsigned transition soft-allow  
- [x] Destructive IR: dashboard confirmation (cloud gate)
