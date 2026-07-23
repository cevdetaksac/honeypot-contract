# Console Winlogon / pre-logon Remote Desktop

> **Contract VERSION:** root `VERSION` (**1.4.21**)  
> **Client target:** ≥ **4.9.21**  
> **Canonical RD API:** [`../api/05-remote-desktop.md`](../api/05-remote-desktop.md)

Kimse logon değilken (veya kilit/Winlogon yüzeyinde) agent **console WinSta0** üzerinden
`Winlogon` (veya input) desktop’unu mirror eder; operatör stream üzerinde k.adı/şifre yazar.

---

## Cloud / dashboard (C-WL-*)

| ID | İş |
|----|-----|
| C-WL-1 | `list_sessions`: `pre_logon:true` / boş `username` + `protocol:Console` + `can_capture:true` satırını göster (“Logon ekranı”) |
| C-WL-2 | 0 kullanıcı oturumu → Start’ı **bloklama**; `prefer=winlogon` veya console `session_id` ile `remote_session_prepare` / `remote_stream_start` |
| C-WL-3 | Prepare sonucu `method=winlogon` → viewer’da “Logon ekranı — kimlik bilgilerini yazın” ipucu |
| C-WL-4 | CAD: `remote_send_sas` / input `ctrl+alt+del` → agent `SendSAS` (sentetik key yok) |
| C-WL-5 | Logon sonrası (desktop→Default) stream sürer; session list refresh |

---

## Client özeti (≥4.9.21)

- Session 0 → `SetProcessWindowStation(WinSta0)` + `OpenInputDesktop` / `OpenDesktop(Winlogon\|Default)`
- `query user` boşken console session synthesize (`pre_logon`)
- `remote_session_prepare` default: oturum yoksa Winlogon probe (eski `UNSUPPORTED` yerine)
- `prefer=existing` hâlâ oturum zorunlu; `prefer=winlogon` zorla logon UI
- Input: Winlogon attach + local SendInput (user-helper yok)

---

## Acceptance

- [ ] Logged-out host: prepare/start → logon UI JPEG/WebRTC görünür
- [ ] Type username/password + Enter → user desktop
- [ ] CAD works via SendSAS
- [ ] Logged-on host path unchanged
