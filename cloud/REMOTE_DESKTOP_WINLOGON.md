# Console Winlogon / pre-logon Remote Desktop

> **Contract VERSION:** root `VERSION` (**1.4.22**)  
> **Client target:** ≥ **4.9.21** (wire); **non-black Winlogon capture → target ≥ 4.9.26** (see § Client P0)  
> **Canonical RD API:** [`../api/05-remote-desktop.md`](../api/05-remote-desktop.md)  
> **Cloud site:** `https://honeypot.yesnext.com.tr` (dashboard remote + `/remote/app`)

Kimse logon değilken (veya kilit/Winlogon yüzeyinde) agent **console WinSta0** üzerinden
`Winlogon` (veya input) desktop’unu mirror eder; operatör stream üzerinde k.adı/şifre yazar.

---

## Cloud / dashboard (C-WL-*) — **shipped on cloud ≥ 1.4.22**

| ID | İş | Cloud durum |
|----|-----|-------------|
| C-WL-1 | `list_sessions`: `pre_logon:true` / boş `username` + `protocol:Console` + `can_capture:true` → UI **“Logon ekranı”** | ✅ `normalize_sessions` + tek hedef select |
| C-WL-2 | 0 kullanıcı oturumu → Start **bloklanmaz**; `prefer=winlogon` + console `session_id` (yoksa **1**) | ✅ prepare + stream start |
| C-WL-3 | Prepare/start `method`/`desktop=Winlogon` / `winlogon_mode` → viewer ipucu | ✅ banner + toast |
| C-WL-4 | CAD → yalnız `remote_send_sas` (**sentetik** `ctrl+alt+del` key **yok**) | ✅ |
| C-WL-5 | Logon sonrası session list refresh | ✅ winlogon modunda periyodik refresh |

### Cloud viewer davranışı (client’ın bilmesi gereken)

1. **Tek hedef select** — kullanıcı + oturum ayrı dropdown yok. Değerler: `wl` | `s:<id>` | `u:<name>`.
2. **Logon ekranı / 0 oturum → `startWinlogonFlow`:**
   - `remote_stream_stop` (eski `gdi+black` stream’i kes)
   - `remote_session_prepare` `{ prefer: "winlogon", timeout_sec: 30 }` (password opsiyonel)
   - `remote_stream_start` `{ prefer: "winlogon", session_id: <console|1>, force: true, … }`
3. **FPS / quality / WebRTC rozet yığını** header’da gizli; canlı rozet + alt **stats bar** (görüntü dışında).
4. Agent `desktop=Winlogon` + `capture_method` içinde `black` → UI **“Winlogon bağlı — görüntü siyah”** (Session 0 mesajı değil).

### Cloud API notları

| Endpoint / komut | Not |
|------------------|-----|
| `POST /api/remote/prepare` | `prefer=winlogon` iken `username`/`password` **zorunlu değil** |
| `POST /api/remote/session` start | `prefer=winlogon` → `params.prefer`; `session_id` yoksa **1** |
| `POST /api/remote/cad` | yalnız `remote_send_sas` enqueue |
| `normalize_sessions` | `pre_logon`, `can_capture` korunur; boş Console+can_capture → `pre_logon` |

---

## Lab kanıtı (2026-07-24 — host Gözmer-Web, client **4.9.25**)

| Adım | Sonuç |
|------|--------|
| `remote_session_prepare` `{prefer:winlogon}` | ✅ `ready_for_stream (winlogon)`, `desktop=Winlogon`, `session_id=1` |
| `remote_stream_start` `{prefer:winlogon, session_id:1}` | ⚠️ `winlogon_mode:true`, `desktop=Winlogon`, **`capture_method: gdi+black`**, `black_frames≥1`, JPEG fiilen siyah |
| Dashboard | Rozet **Canlı · Winlogon**; logon UI **görünmüyor** (siyah) |

**Sonuç:** Cloud wire + attach path çalışıyor. **Bloker agent capture** — Winlogon desktop açık ama GDI siyah bitmap.

---

## Client özeti (≥4.9.21 wire)

- Session 0 → `SetProcessWindowStation(WinSta0)` + `OpenInputDesktop` / `OpenDesktop(Winlogon\|Default)`
- `query user` boşken console session synthesize (`pre_logon`) — health/`list_sessions` ile cloud’a yaz
- `remote_session_prepare` default: oturum yoksa Winlogon probe (eski `UNSUPPORTED` yerine)
- `prefer=existing` hâlâ oturum zorunlu; `prefer=winlogon` zorla logon UI
- Input: Winlogon attach + local SendInput (user-helper yok)

### Client P0 — non-black Winlogon mirror (target **≥ 4.9.26**)

Cloud’un beklediği acceptance; **4.9.25 lab’de FAIL**:

1. Prepare/start sonrası JPEG/WebRTC’de **gerçek logon UI** (siyah stub değil).
2. `capture_method` **`gdi+black` olmamalı** — örn. `gdi` / `bitblt` / uygun path; `stats.black_frames` sürekli artmamalı.
3. Capture thread: `OpenDesktop(Winlogon)` / input desktop sonrası **`SetThreadDesktop`** + BitBlt **o** desktop’tan. Session 0’dan “boş” BitBlt yasak (contract: sahte siyah JPEG yok → fail et veya düzelt).
4. DXGI secure desktop’ta yetmeyebilir; Winlogon için GDI/BitBlt (doğru winsta/desktop) beklenir.
5. Result/meta dürüst kalsın: `desktop`, `winlogon_mode`, `session_id`, `capture_method`, `stats.black_frames` / `frames_sent`.
6. `list_sessions` / health `active_sessions`: oturum yokken `pre_logon` console satırı (C-WL-1).

### Agent result alanları (cloud okur)

```json
{
  "success": true,
  "data": {
    "desktop": "Winlogon",
    "winlogon_mode": true,
    "session_id": 1,
    "capture_method": "gdi",
    "stats": { "frames_sent": 12, "black_frames": 0, "bytes_sent": 180000 }
  },
  "message": "ready_for_stream (winlogon)"
}
```

`capture_method: "gdi+black"` → cloud UI uyarı; **ürün acceptance fail**.

---

## Acceptance

### Cloud (1.4.22)

- [x] 0 oturumda Start açık; `prefer=winlogon` + `session_id`
- [x] Prepare password’suz winlogon probe
- [x] CAD = `remote_send_sas` only
- [x] Viewer Winlogon ipucu + siyah-kare ayrımı (`gdi+black`)
- [x] Stats bar görüntü dışında; tek hedef select

### Client

- [ ] Logged-out host: prepare/start → **logon UI** JPEG/WebRTC görünür (**not** black)
- [ ] Type username/password + Enter → user desktop (`desktop` → `Default`)
- [ ] CAD via SendSAS
- [ ] Logged-on host path unchanged
- [ ] `pre_logon` session row when `query user` empty
- [ ] Never report success with persistent `gdi+black` as healthy logon mirror
