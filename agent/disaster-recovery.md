# Disaster recovery — remote user create + logon (break-glass)

> **Contract VERSION:** root `VERSION`
> **API base:** `https://honeypot.yesnext.com.tr`
> **Min client:** **≥ 4.6.0** (`create_user`, `remote_logon`, autologon break-glass)
> Komut zarfı/HMAC/onay: [`../api/03-control-websocket.md`](../api/03-control-websocket.md)
> Remote desktop akışı: [`../api/05-remote-desktop.md`](../api/05-remote-desktop.md)

Senaryo: makine hacklendi; **tüm kullanıcılar disable, parolalar değişti**. Tek dayanak
noktası çalışan SYSTEM motoru. Motor dışa-doğru (outbound) cloud kanalını koruduğu için
(gelen portlar kapalı olsa bile) buradan **başsız kurtarma** mümkündür.

Kurtarma katmanları:
1. **Başsız IR** (masaüstü gerektirmez): `enable_account`, `reset_password`, `create_user`, `clear_firewall`, `start_service` → yeni admin + RDP yolu aç.
2. **Görsel geri dönüş**: var olan oturumu reconnect; hiç oturum yoksa **autologon+reboot** ile taze oturum yarat → RD aynalar.

---

## `create_user` — yeni/yeniden Administrator

**Onay + HMAC zorunlu** (destructive). Parola tek-seferlik, RAM-only, log/disk'e yazılmaz.

```json
{
  "command_type": "create_user",
  "params": {
    "username": "Administrator",
    "password": "<one-shot>",
    "groups": ["Administrators"],
    "enable": true,
    "comment": "break-glass",
    "password_never_expires": true,
    "if_exists": "reset_enable"
  }
}
```

| Alan | Zorunlu | Varsayılan | Not |
|------|---------|------------|-----|
| `username` | Evet | — | |
| `password` | Evet | — | Tek-seferlik; ≥ yerel politika min; maskelenir |
| `groups` | Hayır | `["Users"]` | Kurtarma: `["Administrators"]` |
| `enable` | Hayır | `true` | `/active:yes` |
| `comment` | Hayır | — | |
| `password_never_expires` | Hayır | `false` | |
| `if_exists` | Hayır | `"fail"` | `"reset_enable"` = varsa parola reset + enable |

Agent:
1. Yoksa `net user <u> <pw> /add`; varsa `if_exists` politikasına göre `fail` veya reset+`/active:yes`.
2. Her grup için `net localgroup <g> <u> /add`.
3. Parolayı **asla** loglamaz/diske yazmaz; history redacted (`***`).

Result:
```json
{ "success": true, "data": { "username": "Administrator", "sid": "S-1-5-…", "groups": ["Administrators"], "created": true, "enabled": true } }
```

Guard:
- HMAC + sunucu `confirm: true`.
- Rate-limit (öneri: ≤ 3 / saat).
- Bilgi amaçlı audit: cloud'a `account_created_by_agent` (informational alert) — meşru kurtarma da olsa iz bırakır.

---

## `remote_logon` — kimlikle uzaktan oturum açtırma

**Onay + HMAC zorunlu** (reboot içerebilir → destructive). Parola tek-seferlik, RAM-only.

```json
{
  "command_type": "remote_logon",
  "params": {
    "username": "Administrator",
    "password": "<one-shot>",
    "domain": ".",
    "mode": "auto",
    "reboot": true,
    "timeout_sec": 120
  }
}
```

`mode`:
| Değer | Davranış |
|-------|----------|
| `auto` (varsayılan) | Var olan oturum varsa reconnect (reboot yok); yoksa `autologon_reboot`'a düş |
| `reconnect_only` | Yalnız var olan/disconnected oturumu geri getir; yoksa `UNSUPPORTED` (reboot yok) |
| `autologon_reboot` | Doğrudan autologon+reboot break-glass |

### Akış

1. **Kimlik doğrula** — `LogonUser`. Fail → `AUTH_FAILED` / `ACCOUNT_LOCKED` / `ACCOUNT_DISABLED`.
2. **Var olan oturum** (Active/Disconnected, bu kullanıcı) → `WTSConnectSession` + `tscon` → Active. `ready_for_stream: true`, `method: "reconnect"`. **Reboot yok.**
3. **Hiç oturum yok** ve mode autologon'a izin veriyorsa → **autologon break-glass**:
   - Winlogon: `DefaultUserName` / `DefaultDomainName`, `AutoAdminLogon=1`, **`AutoLogonCount=1`** (tek-seferlik).
   - Parola: tercihen **LSA secret** `DefaultPassword` (registry düz-metin `DefaultPassword` yerine); fallback registry.
   - ProgramData `autologon_pending.json` marker: `{ username, command_id, requested_at, one_shot:true }`.
   - Ara result: `status:"running"`, `phase:"rebooting"`.
   - `shutdown /r /t <grace>` (öneri grace 15–30 sn).
4. **Açılışta** (yeni boot): motor startup rutini marker'ı görür → console oturumu (Session 1) belirince:
   - LSA secret / registry autologon anahtarlarını **temizler** (AutoLogonCount=1'e ek savunma).
   - `command_id` için `status:"completed"`, `data.ready_for_stream:true`, `session_id`.
   - Cloud artık `remote_stream_start` gönderebilir.

### Güvenlik
- Parola log/diske düz-metin yazılmaz; LSA secret ilk boot'ta silinir; `AutoLogonCount=1` zaten tek kullanımda tükenir.
- Yalnız HMAC + `confirm: true`.
- Autologon başarısız/oturum gelmezse marker silinir + `LOGON_TIMEOUT`.

### Yardımcı primitifler (opsiyonel, ayrı komut olarak da açık)
| Komut | Params | İş |
|-------|--------|-----|
| `set_autologon` | `username`, `password`, `count` (default 1) | Autologon arm (reboot'suz) |
| `clear_autologon` | — | Autologon anahtarlarını + LSA secret temizle |
| `reboot` | `grace_sec`, `reason` | Onaylı yeniden başlatma |

---

## Uçtan uca kurtarma runbook (felaket)

```
1. list_local_users            → durum tespiti (hepsi disabled mı?)
2. create_user {Administrator2, Administrators}   (veya enable_account + reset_password)
3. clear_firewall / unblock_ip → RDP/erişim yolunu aç
4. start_service TermService (gerekiyorsa) + RDP portu
5a. Var olan oturum → remote_session_prepare → remote_stream_start (görsel)
5b. Hiç oturum yok  → remote_logon {mode:auto, reboot:true} → boot sonrası remote_stream_start
6. remote_send_sas / remote_input ile tam kontrol
```

Adım 2–4 tamamen **başsız** ve **gelen-port bağımsız** (motor outbound). Görsel (5) yalnız
oturum gerektirir; 5b onu autologon+reboot ile sıfırdan yaratır.

---

## Acceptance

- [ ] `create_user` → yeni Administrator; HMAC+confirm; parola loglanmaz
- [ ] `create_user if_exists=reset_enable` → var olan hesabı reset+enable
- [ ] `remote_logon mode=auto` + var olan oturum → reconnect, reboot yok
- [ ] `remote_logon` + sıfır oturum → autologon(count=1)+reboot → boot sonrası `ready_for_stream`
- [ ] Boot sonrası autologon anahtarları + LSA secret temizlenir
- [ ] `reconnect_only` + sıfır oturum → `UNSUPPORTED` (reboot yok)
- [ ] Tüm komutlar HMAC + server confirm olmadan reddedilir
