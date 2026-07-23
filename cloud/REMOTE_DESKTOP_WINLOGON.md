# Console Winlogon / pre-logon Remote Desktop

> **Contract VERSION:** root `VERSION` (**1.4.23**)  
> **Client wire:** ≥ **4.9.21** · **sibling pre_logon + named Winlogon:** ≥ **4.9.26**  
> **Canonical RD API:** [`../api/05-remote-desktop.md`](../api/05-remote-desktop.md)  
> **Cloud site:** `https://honeypot.yesnext.com.tr`

Kimse logon değilken **veya** kullanıcı oturumu varken kilit/Winlogon yüzeyi istenince
agent console WinSta0 üzerinden `Winlogon` desktop’unu mirror eder; operatör stream’de
kimlik yazar.

---

## Client 4.9.26 (lab bulguları → düzeltme)

Önceki lab (caksac Active): health boş username / bare console’u atıyordu → dashboard’da
logon satırı yoktu; kullanıcı oturumu varken **sibling** `pre_logon` yoktu (aynı
`session_id`); `OpenInputDesktop` önce gelince Default’a yapışılıyordu; aynı SID start
user satırını seçiyordu (`prefer=winlogon` yok).

**4.9.26 düzeltmeleri:**
- Health + `list_sessions`: her zaman **“Logon / Lock screen”** satırı (`pre_logon:true`)
- Sibling örnek: `[('caksac', 2), ('', 2, pre_logon, 'Logon / Lock screen')]`
- `remote_stream_start`: `prefer` / `desktop` / `pre_logon` dinle
- Named Winlogon önce; hello’da `winlogon` / `pre_logon` capability

---

## Cloud / dashboard (C-WL-*) — **shipped ≥ 1.4.23**

| ID | İş | Cloud |
|----|-----|-------|
| C-WL-1 | Sibling `pre_logon` satırını göster (aynı `session_id` user ile yan yana) | ✅ hedef select: `s:<id>` + `wl:<id>` |
| C-WL-2 | Start’ta `prefer=winlogon` (+ `pre_logon=true`, `desktop=Winlogon`); username **gönderme** | ✅ |
| C-WL-3 | Winlogon ipucu | ✅ |
| C-WL-4 | CAD → `remote_send_sas` only | ✅ |
| C-WL-5 | Logon sonrası session refresh | ✅ |

### Start wire (dashboard → agent)

Lock / Logon satırı seçiliyken:

```json
{
  "command_type": "remote_stream_start",
  "params": {
    "prefer": "winlogon",
    "pre_logon": true,
    "desktop": "Winlogon",
    "session_id": 2,
    "stream_id": "…",
    "fps": 30, "quality": 40, "max_width": 1280
  }
}
```

- **`username` yok** — agent user Default’a yapışmasın.
- User satırı (`s:2` + username) → normal start; sibling pre_logon yüzünden **otomatik
  winlogon’a çevrilmez**.

Prepare (0 oturum / Logon seçimi):

```json
{ "prefer": "winlogon", "pre_logon": true, "desktop": "Winlogon", "timeout_sec": 30 }
```

---

## Lab notları

| Client | Sonuç |
|--------|--------|
| ≤4.9.25 | Attach `desktop=Winlogon` ama sık `gdi+black` |
| ≥4.9.26 | Sibling pre_logon + prefer/desktop; cloud C-WL ile birlikte doğrula |

---

## Acceptance

### Cloud
- [x] Sibling `pre_logon` + user aynı SID → iki ayrı select option
- [x] Lock satırı Start → `prefer=winlogon` + `pre_logon` + `desktop` + SID, no username
- [x] User satırı Start → username/session, **not** forced winlogon
- [x] CAD = `remote_send_sas`

### Client (≥4.9.26+)
- [x] Always emit Logon/Lock `pre_logon` row (even when user Active)
- [x] Honor `prefer` / `desktop` / `pre_logon` on start
- [ ] Non-black Winlogon pixels (no persistent `gdi+black`)
- [ ] Type creds on stream → Default desktop
- [ ] CAD SendSAS
