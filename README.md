# YesNext Honeypot — Shared Contract

**Single source of truth** for Windows client ↔ Cloud API behavior.

| | |
|--|--|
| **VERSION** | See [`VERSION`](VERSION) |
| **Changelog** | [`CHANGELOG.md`](CHANGELOG.md) |
| **API base** | `https://honeypot.yesnext.com.tr` |
| **Auth** | `Authorization: Bearer <token>` — agent API’de `?token=` **yok** (geçiş dual-read cloud’da olabilir; client göndermez) |

## Bu repo kimler için?

| Taraf | Ne yapar |
|-------|----------|
| **Windows client** | Sözleşmeye uyum kodu; contract’a aykırı endpoint uydurmaz |
| **Cloud / API** | Aynı MD’leri implement eder; PM2/nginx/dashboard HTML bu repoda **yok** |
| **Cursor / agent** | Her görevde `VERSION` + `INDEX.md` → ilgili MD |

## Kurallar (zorunlu)

1. API / davranış değişikliği → **önce** bu repoda MD + CHANGELOG + VERSION bump → sonra kod.
2. Belirsizlik → contract’a `## Open questions` notu; tahmin etme.
3. Client harici threat feed çekmez; sadece cloud `threat-intel` bundle.
4. Min client sürümü ilgili MD’de yazar (ör. threat-intel ≥ 4.5.61).
5. Cloud-only (PM2, nginx, dashboard HTML) varsayma / buraya koyma.

## Clone

```text
# Önerilen: client workspace yanında veya içinde
git clone <REMOTE_URL> honeypot-contract

# Client repo içinde pointer:
#   cloud-client/contract/README.md  → CONTRACT_ROOT
```

`CONTRACT_ROOT` = bu dizinin yolu.
