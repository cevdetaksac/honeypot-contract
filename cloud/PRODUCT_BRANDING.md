# Product branding & protocol identity

> **Contract VERSION:** root `VERSION` (**1.4.16** — branding karar; soft surface inform için bkz. **1.4.17** / `agent/network-guard.md`)  
> Statü: **normative decision** (not a rename migration plan)

## Karar (Gemini “Honeypot → Core/Agent/Shield” tavsiyesi)

**Canlı SaaS filoda big-bang isim refactoru yapılmaz.**

| Katman | Karar |
|--------|--------|
| Wire / OS firewall | `HP-BLOCK-*`, `HP-INTEL-*` = **kalıcı protokol kimliği** |
| Legacy wipe | `HONEYPOT_*` hâlâ list/clear ile temizlenir |
| Persistence | `%ProgramData%\YesNext\CloudHoneypotClient`, `CloudHoneypot-*` tasks, `honeypot-client.exe` — migration olmadan değişmez |
| Repo / docs | `honeypot-contract` adı tarihsel SoT; ürün dili dashboard’da “Agent / Shield / Guard” olabilir |
| UI copy | Kullanıcıya `HP-BLOCK` göstermek zorunlu değil; “Engellendi / Tehdit istihbaratı” yeterli |

## Neden

1. Filoda binlerce `HP-BLOCK` / `HP-INTEL` kuralı var — prefix değişince orphan + sync kırılır.  
2. Update/watchdog ProgramData + schtasks kimliğine bağlı.  
3. Gemini’nin “okunabilirlik için her şeyi rename” önerisi **pazarlama** için doğru, **wire** için yıkıcı.

## İzin verilen (güvenli)

- Dashboard / installer **görünen ad** (YesNext Cloud Agent vb.)
- Dokümantasyonda “agent / shield / network guard” dili
- Yeni özellik adları (`system_recovery`, `network_guard`) — zaten kullanılıyor

## Yasak (1.4.16)

- Tek release’te `HP-BLOCK` → `CORE-BLOCK` vb. wire rename  
- API path (`/api/...`) veya token dizini big-bang rename  
- Kontrat repo adını zorunlu yeniden adlandırma (opsiyonel mirror sonra)

Cloud implementasyon: yeni UI metinleri serbest; firewall rule prefix’leri **olduğu gibi** kalsın.
