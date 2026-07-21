# Agent Control WebSocket + command catalog

> **Contract VERSION:** root `VERSION`  
> **Endpoint:** `wss://honeypot.yesnext.com.tr/ws/agent/control`  
> **Kim:** yalnızca SYSTEM daemon (`mode=daemon`)  
> **RD video:** `/ws/remote/agent` — ayrı kanal  

Kod SoT (cloud whitelist): `helpers.VALID_COMMAND_TYPES` (42 tip).  
`contain_user` **whitelist’te yok** — client/dashboard `logoff_user` + `reset_password` (+ opsiyonel `disable_account`) birleşimi kullanır.

> **Cloud implemented (≥1.3.11, 2026-07-21):** `set_gui_pin` + `clear_gui_pin` →
> `VALID_COMMAND_TYPES` (42 tip) + `DESTRUCTIVE_COMMAND_TYPES` (confirm gate) +
> `scrub_command_params` (`pin` → `***`, sonuç sonrası DB'den de maskelenir).
> Sunucu tarafı format doğrulaması: `pin` 4-12 hane yalnız rakam → aksi 400.
> Dashboard: Tehdit Merkezi → Remote Commands → **GUI PIN** modalı
> (PIN tanımla / PIN sıfırla, confirm dialoglu). Client ≥4.8.3 uygular.

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
| `threat_config_updated` | security-layer ayarı değişti; hemen `GET /api/threats/config` + runtime apply (client ≥4.7.3) |
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
- Anahtar türetimi (client): `SHA256("{token}|{COMPUTERNAME}|yesnext-chp-v1")` — key = **raw digest** (32 bayt), hex string değil  
- Cloud imza üretirken aynı token + kayıtlı hostname (`COMPUTERNAME`) kullanır (heartbeat `hostname` > `server_name` prefix)  
- **Cloud → agent zarf alanları:** `issued_at` (imzalanan tam string) + `signature` — hem WS `command` push hem `GET /api/commands/pending` liste öğelerinde  
- **Zarf ayrıca `type` alias'ı içerir** (= `command_type`): client `verify_command_signature` tipi `type` anahtarından okur; alias olmadan imza boş tip üzerinden hesaplanıp reject edilir (canlı 4.5.68 ile doğrulandı)  
- Agent doğrularken zarftaki `issued_at` string’ini **verbatim** kullanır (parse/normalize etme)  
- Self-process proof (`api/07-lifecycle-sessions.md`) **farklı** HMAC’tir; karıştırma

---

## Destructive commands — server confirmation

Aşağıdakiler **dashboard’da açık onay** olmadan cloud kuyruğa yazılmaz / push edilmez:

- `reset_password`
- `disable_account` / `disable_all_users`
- `enable_lockdown` (emergency lockdown)
- `clear_firewall` (`wipe_all_honeypot_rules=true` dahil)
- `create_user` (yeni/yeniden hesap — [`../agent/disaster-recovery.md`](../agent/disaster-recovery.md))
- `remote_logon` / `set_autologon` / `reboot` (autologon + yeniden başlatma break-glass)
- `network_restore` (ağ baseline'dan geri yükle — [`../agent/network-guard.md`](../agent/network-guard.md))
- `suspend_process` (şüpheli süreci askıya al — açık kullanıcı onayı zorunlu)
- `set_gui_pin` / `clear_gui_pin` (yerel GUI anti-tamper PIN'ini ez/sıfırla — ≥4.8.3)

Uygulama (cloud): `POST /api/commands/send` gövdesinde **`confirm: true`** yoksa **400** döner
(`helpers.DESTRUCTIVE_COMMAND_TYPES`). Dashboard onay modalı geçildiğinde `confirm: true` gönderir.
Cloud-internal maintenance akışları (blok temizliği → `clear_firewall`) bu endpoint’i kullanmaz; gate dashboard/API çağrıları içindir.

> `clear_autologon` yıkıcı **değildir** (autologon geri alma) — onay gerekmez.

**Cloud alan doğrulaması (implemented):** `create_user`/`remote_logon` → `username` + `password` zorunlu; `set_autologon` → `username` zorunlu (aksi halde 400). `remote_logon` ve `reboot` komut TTL’i **15 dk** (reboot/autologon boot süresi).

Client ayrıca whitelist + protected targets uygular; onay **sunucu tarafı** zorunluluğudur.

### Şifre maskeleme (break-glass, implemented)

`password` / `new_password` taşıyan komutlar (`create_user`, `remote_logon`, `set_autologon`, `remote_session_prepare`, `reset_password`) — ve `pin` taşıyan `set_gui_pin` (≥4.8.3, cloud scrub listesine `pin` alanı eklenmeli):

- **Agent’a** iletilen `params` (WS push + `GET /api/commands/pending`) **gerçek** şifreyi taşır — uygulama için gerekli.
- **Dashboard/audit** görünümleri maskeli: `GET /api/commands/{id}` + `CommandAuditLog` → `***`.
- Komut terminal duruma geçince (`completed`/`failed`/…) cloud, kayıtlı `params` şifresini DB’de `***` ile ezer (`helpers.scrub_command_params`).

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
| `suspend_process` | `pid`, `expected_image`, `expected_path?`, `process_start_time` | Açık onay sonrası exact-process suspend; PID reuse korumalı — ≥4.7.3 |
| `resume_process` | `pid`, `expected_image`, `expected_path?`, `process_start_time` | Exact-process resume; non-destructive — ≥4.7.3 |
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
| `create_user` | `username`, `password`, `groups[]`, `if_exists` | Yeni/yeniden Administrator — ≥4.6.0 ([`../agent/disaster-recovery.md`](../agent/disaster-recovery.md)) |
| `remote_logon` | `username`, `password`, `mode`, `reboot` | Kimlikle uzaktan oturum (reconnect / autologon+reboot) — ≥4.6.0 |
| `set_autologon` / `clear_autologon` | `username`, `password`, `count` | Autologon arm/temizle — ≥4.6.0 |
| `reboot` | `grace_sec`, `reason` | Onaylı yeniden başlatma — ≥4.6.0 |
| `network_snapshot` | — | Anlık ağ baseline al — ≥4.7.0 ([`../agent/network-guard.md`](../agent/network-guard.md)) |
| `network_restore` | `targets[]?` | Baseline'dan ağ/sürücü geri yükle (confirm) — ≥4.7.0 |
| `list_network_baseline` | — | Baseline sürümleri/özeti — ≥4.7.0 |
| `set_gui_pin` | `pin` (4-12 hane, yalnız rakam) | Dashboard'dan yerel GUI PIN tanımla/değiştir (confirm; result PIN içermez) — ≥4.8.3 |
| `clear_gui_pin` | — | Dashboard'dan yerel GUI PIN'i sıfırla/kaldır (confirm) — ≥4.8.3 |

**GUI PIN semantiği (≥4.8.3):** PIN store `ProgramData/.../gui_lock.json` — SYSTEM daemon komutu uygular, GUI süreci dosya mtime'ından değişikliği otomatik algılar (restart gerekmez) ve aktif oturum kilidini düşürür. Hesap bağlıysa GUI PIN diyalogları "dashboard üzerinden tanımlayabilir/sıfırlayabilirsiniz" ipucu gösterir (`is_account_linked()`), böylece PIN unutulduğunda kurtarma yolu görünürdür. Client doğrulaması: `pin` eksik → `missing_pin`, format dışı → `invalid_pin_format`.

Detay: self-update → [`04-self-update.md`](./04-self-update.md); remote → [`05-remote-desktop.md`](./05-remote-desktop.md) + [`../agent/remote-input.md`](../agent/remote-input.md); firewall → [`06-firewall-blocks.md`](./06-firewall-blocks.md); kurtarma → [`../agent/disaster-recovery.md`](../agent/disaster-recovery.md); kalıcılık/tamper → [`../agent/persistence-and-tamper.md`](../agent/persistence-and-tamper.md).

---

## HTTP komut

- `GET /api/commands/pending` → liste  
- `POST /api/commands/result` → `{ command_id, status, result }` (`running` / `completed` / `failed`)  
- `GET /api/commands/{id}` — şifre alanları redacted  

---

## Acceptance

- [x] Daemon WS bağlanır; send → anında push (canlı 4.5.68: signed `list_sessions` → completed)  
- [ ] WS kopunca poll kaçırmaz; reconnect drain  
- [x] Bilinmeyen `command_type` reddedilir (cloud 400 — test edildi)  
- [x] `threat_intel_updated` → ETag GET (client ≥4.5.66)  
- [x] HMAC: signed command fail → reject; unsigned transition soft-allow  
- [x] Destructive IR: dashboard confirmation (cloud gate)
