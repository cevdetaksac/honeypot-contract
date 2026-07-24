# Server User Management — Cloud/Dashboard (C-USER-*)

> **Contract VERSION:** root `VERSION` (**1.4.24**)  
> **Client wire:** `list_local_users` + `enable_account` / `disable_account` — see
> [`../agent/server-management.md`](../agent/server-management.md)  
> **Cloud site:** `https://honeypot.yesnext.com.tr` · route `/dashboard/server/users`

Bu belge **cloud + dashboard Users sayfası** zorunluluklarını listeler. Client
disabled hesapları ve `PROTECTED_ACCOUNT` sonuçlarını doğru döndürdüğünde sayfa
tam oturur; cloud bu checklist’i önceden karşılamalıdır.

---

## 1. Cloud / dashboard zorunlulukları (C-USER-*)

| ID | Cloud ne yapmalı |
|----|------------------|
| **C-USER-1** | `list_local_users` her zaman **`include_disabled: true`** (Server Management refresh + fallback send). Legacy remote dropdown ayrı kalsa bile Users yolu `true` gönderir. |
| **C-USER-2** | Tabloda **Active + Disabled** satırları birlikte görünür (filtre ile gizlemek serbest; varsayılan: hepsi). |
| **C-USER-3** | Satır aksiyonu: **Enable / Disable toggle** (`enable_account` / `disable_account` + `username`). |
| **C-USER-4** | Mutate sonrası **cache/refresh**: komut tamamlanınca `list_local_users` (veya sayfa reload) ile UI güncellenir. |
| **C-USER-5** | Cache (`remote_local_users`) **disabled satırları da tutar** — `enabled:false` / boş liste `or` ile düşürülmez. |
| **C-USER-6** | Disable/enable öncesi **confirm**; agent `PROTECTED_ACCOUNT` (veya eşdeğer) dönerse toast/modal ile net hata. |
| **C-USER-7** | Üstte **active / disabled sayaç** (ör. `3 aktif · 2 pasif` veya ayrı badge). |

---

## 2. Wire (dashboard → agent)

### Yenile

```json
{
  "command_type": "list_local_users",
  "params": { "include_disabled": true }
}
```

Cloud `POST /api/remote/users/refresh` ve Users sayfası fallback’i aynı param’ı
kullanır.

### Toggle

```json
{ "command_type": "disable_account", "params": { "username": "alice", "confirm": true } }
```

```json
{ "command_type": "enable_account", "params": { "username": "alice", "confirm": true } }
```

Destructive gate: mevcut dashboard confirm + cloud `confirm:true` (api/03).

### Korunan hesap

Agent sonuç örneği:

```json
{
  "status": "failed",
  "success": false,
  "error": "PROTECTED_ACCOUNT",
  "message": "SYSTEM / DefaultAccount cannot be disabled"
}
```

UI `error` / `message` gösterir; satırı “başarılı” sanmaz.

---

## 3. Cache

| Key | Kaynak | Not |
|-----|--------|-----|
| `remote_local_users` | `list_local_users` result `data.users` | ≤200; **disabled dahil** |
| Sayfa seed | `local_users` + `active_sessions` template | SSR → `H.smRenderLocalUsers` |

Boş `users: []` geçerli tamamlanmış sonuçtur; falsy `or` ile önceki cache’i
silmeyin (services’teki boş-liste hatası ile aynı sınıf).

---

## 4. Acceptance (cloud)

- [ ] Users → Yenile → pending `include_disabled: true`
- [ ] Disabled hesap satırı görünür (client ≥ ship)
- [ ] Disable → confirm → başarıda refresh; `PROTECTED_ACCOUNT` → hata toast
- [ ] Enable/disable sonrası sayaçlar güncellenir
- [ ] Cache’te `enabled: false` satırlar kalır

---

## 5. Client notu

Client tarafı inventory + mutate [`agent/server-management.md`](../agent/server-management.md)
içinde. Cloud bu doc’u uyguladıktan sonra client ship ile Users sayfası tam
oturur.
