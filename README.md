# YesNext Honeypot — Shared Contract

**Single source of truth** for Windows client ↔ Cloud API behavior.

| | |
|--|--|
| **VERSION** | See [`VERSION`](VERSION) |
| **Changelog** | [`CHANGELOG.md`](CHANGELOG.md) |
| **Fleet** | [`FLEET.md`](FLEET.md) — production floor client ≥ **4.5.66** |
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

```bash
git clone https://github.com/cevdetaksac/honeypot-contract.git
cd honeypot-contract
# pin: git checkout v1.1.3
```

`CONTRACT_ROOT` = bu dizin. Client: `cloud-client/contract/README.md` pointer.  
`docs/api/*` client tarafında stub — SoT yalnızca bu repo.

Cloud sunucu:

```bash
cd /data/honeypot.yesnext.com.tr/contract && git pull && ../scripts/publish_contract.sh
```

HTTPS mirror (cloud): `https://honeypot.yesnext.com.tr/static/shared-contract.zip`  
Meta: `GET /api/public/contract`
