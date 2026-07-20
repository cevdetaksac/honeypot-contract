# Windows Client — sözleşme indeksi

> **Repo:** [cevdetaksac/honeypot-contract](https://github.com/cevdetaksac/honeypot-contract)  
> **VERSION:** root [`VERSION`](../VERSION) · giriş: [`INDEX.md`](../INDEX.md)  
> **API:** `https://honeypot.yesnext.com.tr`

Bu dosya **özet + link**. Şema/detay için ilgili MD’ye git; buraya kopyalama.

---

## Okuma sırası (yeni agent / sprint)

1. [`polling.md`](./polling.md) — cadence  
2. [`../api/01-auth.md`](../api/01-auth.md) — register / heartbeat / Bearer  
3. [`register-protection.md`](./register-protection.md) — `protection.block_rules`  
4. [`attacks-and-services.md`](./attacks-and-services.md) — bait + attack POST  
5. [`../api/03-control-websocket.md`](../api/03-control-websocket.md) — WS + **komut kataloğu**  
6. [`threat-engine.md`](./threat-engine.md) — v4 alerts/health/config  
7. [`../api/09-threat-intel.md`](../api/09-threat-intel.md) — cloud IoC bundle  
8. [`remote-input.md`](./remote-input.md) + [`../api/05-remote-desktop.md`](../api/05-remote-desktop.md)  
9. [`../api/06-firewall-blocks.md`](../api/06-firewall-blocks.md) · [`04-self-update.md`](../api/04-self-update.md) · [`07-lifecycle-sessions.md`](../api/07-lifecycle-sessions.md) · [`08-architecture.md`](../api/08-architecture.md) · [`02-account.md`](../api/02-account.md)

---

## Auth (tek satır)

Bearer only. Token ProgramData, immutable. Query `?token=` gönderme.

---

## Control WS (tek satır)

`wss://…/ws/agent/control` → `command` push; result HTTP; `threat_intel_updated` → anında intel GET.

---

## IR notu

`contain_user` cloud whitelist’te **yok** → `logoff_user` + `reset_password` (+ `disable_account`).  
Tam liste: [`../api/03-control-websocket.md`](../api/03-control-websocket.md).

---

## Mimari (tek satır)

SYSTEM daemon = motor (WS, FW, update, intel). GUI = tray/UI + IPC. Detay: `api/08-architecture.md`.
