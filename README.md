# YesNext Honeypot — Shared Contract

**Single source of truth** for Windows client ↔ Cloud API behavior.

| | |
|--|--|
| **VERSION** | [`VERSION`](VERSION) (**1.4.28**) |
| **Index** | [`INDEX.md`](INDEX.md) |
| **Changelog** | [`CHANGELOG.md`](CHANGELOG.md) |
| **Fleet matrix** | [`FLEET.md`](FLEET.md) — production floor client ≥ **4.9.0** |
| **API base** | `https://honeypot.yesnext.com.tr` |
| **Auth** | `Authorization: Bearer <token>` — agent API must not rely on `?token=` query |

## Who uses this?

| Party | Role |
|-------|------|
| **Windows client** | Implements the contract; does not invent endpoints |
| **Cloud / API** | Implements the same MDs; PM2/nginx/dashboard HTML are **not** in this repo |
| **Cursor / agents** | Read `VERSION` → `INDEX.md` → relevant MD before coding |

## Rules

1. Behavior / API change → **first** MD + CHANGELOG + VERSION bump here → then code.
2. Ambiguity → add `## Open questions` in the MD; do not guess.
3. Client does not pull external threat feeds; only the cloud threat-intel bundle.
4. Minimum client version is stated per MD (see `FLEET.md`).
5. No cloud-only ops (PM2, nginx, dashboard HTML) in this repo.
6. Promoted designs / legacy prompts live under [`docs/archive/`](docs/archive/) — not normative.

## Clone

```bash
git clone https://github.com/cevdetaksac/honeypot-contract.git
cd honeypot-contract
# pin: cat VERSION   # currently 1.4.28
```

Client workspace pointer: `cloud-client/contract/README.md` → this repo.

## Layout

| Path | Content |
|------|---------|
| `api/` | HTTP / WS wire contracts |
| `agent/` | Client-side behavior |
| `cloud/` | Dashboard / cloud must-do (C-* checklists) + design-only ZT |
| `docs/archive/` | Promoted designs / legacy prompt dumps |

## Cloud publish (operators)

```bash
cd /data/honeypot.yesnext.com.tr/contract && git pull && ../scripts/publish_contract.sh
```

HTTPS mirror: `https://honeypot.yesnext.com.tr/static/shared-contract.zip`  
Meta: `GET /api/public/contract`
