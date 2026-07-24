# Windows Client — sözleşme indeksi

> **Repo:** [cevdetaksac/honeypot-contract](https://github.com/cevdetaksac/honeypot-contract)  
> **VERSION:** root [`VERSION`](../VERSION) · giriş: [`INDEX.md`](../INDEX.md) · fleet: [`FLEET.md`](../FLEET.md)  
> **API:** `https://honeypot.yesnext.com.tr`  
> **Production floor:** client ≥ **4.9.0**

Bu dosya **özet + link**. Şema/detay için ilgili MD’ye git; buraya kopyalama.

---

## Okuma sırası (yeni agent / sprint)

1. [`../FLEET.md`](../FLEET.md) — min sürüm matrisi  
2. [`polling.md`](./polling.md) — cadence  
3. [`../api/01-auth.md`](../api/01-auth.md) — register / heartbeat / Bearer  
4. [`register-protection.md`](./register-protection.md) — `protection.block_rules`  
5. [`attacks-and-services.md`](./attacks-and-services.md) — bait + attack POST  
6. [`../api/03-control-websocket.md`](../api/03-control-websocket.md) — WS + **komut kataloğu**  
7. [`threat-engine.md`](./threat-engine.md) — v4 alerts/health/config  
8. [`../api/09-threat-intel.md`](../api/09-threat-intel.md) — cloud IoC bundle  
9. [`ransomware-shield.md`](./ransomware-shield.md) — canary / quarantine / unlock  
10. [`remote-input.md`](./remote-input.md) + [`../api/05-remote-desktop.md`](../api/05-remote-desktop.md)  
11. [`persistence-and-tamper.md`](./persistence-and-tamper.md) — survival + tamper (≥4.6.0)  
12. [`disaster-recovery.md`](./disaster-recovery.md) — `create_user` / `remote_logon` (≥4.6.0)  
13. [`network-guard.md`](./network-guard.md) — golden ağ baseline + dashboard panel + `auto_restore_network`; soft surface inform (`network_surface_changed` / accept / disable); süreç contain onaylı (≥4.7.3 / panel ≥4.9.12 / soft inform ≥4.9.15 / contract 1.4.17)  
13b. [`../cloud/DEFENSE_POLICY.md`](../cloud/DEFENSE_POLICY.md) + [`defense-policy-client.md`](./defense-policy-client.md) — tiered policy; observe default + auto-promote (contract **1.4.19**; client ≥4.9.17)  
14. [`system-recovery.md`](./system-recovery.md) — saldırı yüzeyi policy/service/firewall snapshot + drift + onaylı restore (target ≥4.9.12)  
15. [`gui-control-center.md`](./gui-control-center.md) — dinamik GUI / güvenlik katmanları (≥4.7.3)  
16. [`log-retention.md`](./log-retention.md) — günlük loglar / 7 gün retention (≥4.7.6)
17. [`server-management.md`](./server-management.md) — Dashboard Sunucu Yönetimi inventory + mutates (target ≥4.9.4)
18. [`../api/11-presence-realtime.md`](../api/11-presence-realtime.md) — sleep/shutdown presence (target ≥4.9.8; cloud live ≥1.4.27)
19. [`../api/06-firewall-blocks.md`](../api/06-firewall-blocks.md) · [`04-self-update.md`](../api/04-self-update.md) · [`07-lifecycle-sessions.md`](../api/07-lifecycle-sessions.md) · [`08-architecture.md`](../api/08-architecture.md) · [`02-account.md`](../api/02-account.md) (unlink Settings ≥4.9.26)

---

## Auth (tek satır)

Bearer only. Token ProgramData. Query `?token=` gönderme.

- **İlk kurulum:** `POST /api/register`  
- **Token yeniden üretim (rekey / identity v2):** `POST /api/agent/rotate-token`
  (`old_token`→`new_token`, aynı `client_id`) — **≥ 4.9.31** / contract **1.4.29**.
  Bare register ile yeni satır açma.

---

## Control WS (tek satır)

`wss://…/ws/agent/control` → `command` push; result HTTP; `threat_intel_updated` → anında intel GET.

---

## IR notu

`contain_user` cloud whitelist’te **yok** → `logoff_user` + `reset_password` (+ `disable_account`).  
Tam liste: [`../api/03-control-websocket.md`](../api/03-control-websocket.md).

---

## Ransomware (tek satır)

Canary Hidden+System; yerel scare yok; unlock = GUI / `RS_UNLOCK` / `unlock_ransomware_quarantine`. Detay: [`ransomware-shield.md`](./ransomware-shield.md).

---

## Mimari (tek satır)

SYSTEM daemon = motor (WS, FW, update, intel, ransomware). GUI = tray/UI + IPC (`RS_*`). Detay: `api/08-architecture.md`.

---

## Survival + kurtarma (tek satır, ≥4.6.0)

Motor+Guardian servisi çapraz watchdog; durdurma yalnız update-lock / imzalı PIN token; başka çıkış → tamper alarmı + diriliş. Felaket: `create_user` + `remote_logon` (autologon+reboot). Detay: [`persistence-and-tamper.md`](./persistence-and-tamper.md), [`disaster-recovery.md`](./disaster-recovery.md).

---

## Network Guard + GUI (tek satır, ≥4.7.3)

Şüpheli yoğun aktivite **yalnız alarm** (`ransomware_offline_suspect`); süreç dondurma açık kullanıcı onayıyla `suspend_process`. Güvenlik katmanı toggle'ları GUI → `POST /api/threats/config` → `threat_config_updated` push. Detay: [`network-guard.md`](./network-guard.md), [`gui-control-center.md`](./gui-control-center.md).

---

## Remote Desktop v2 (tek satır, ≥4.9.0)

Sağlıklı yol **WS + JPEG** (her WS karesinde ayrıca HTTP post YOK); `hello`/`meta` protocol 2 (native vs encoded boyut, negatif origin, requested/effective telemetri); input protocol 2 + mobil jestler + `input_ack` + gizlilik (tuş/metin loglanmaz); farklı WTS oturumu için kalıcı session helper; opsiyonel WebRTC signaling agent/view WS relay üzerinden (protocol=1, non-trickle ICE, H264, data-channel input, offer içinde cloud-verilen sınırlı `ice_servers` — credential loglanmaz) → başarısızlıkta JPEG-WS fallback. Detay: [`../api/05-remote-desktop.md`](../api/05-remote-desktop.md), [`remote-input.md`](./remote-input.md).
