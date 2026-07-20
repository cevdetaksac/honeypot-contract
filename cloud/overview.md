# Cloud overview (agent-visible)

> Agent bu dosyayı **ops için değil**, mimariyi anlamak için okur.  
> PM2/nginx → cloud `docs/PM2_SERVICE.md` (bu repoda yok).

```
Windows Agent (SYSTEM daemon + GUI)
    │  HTTPS Bearer + WSS
    ▼
honeypot.yesnext.com.tr  (FastAPI :9000 behind nginx/CF)
    ├── register / heartbeat / attack / ports / tunnels
    ├── pending-blocks + commands (+ control WS push)
    ├── threats/config + alerts/* + health (v4 engine)
    ├── agent/threat-intel (curated bundle)
    └── remote desktop (HTTP frame + /ws/remote/*)
```

| Katman | SoT |
|--------|-----|
| Fail→block eşikleri | `protection.block_rules` (register + threats/config) + cloud `notification_rules` |
| Firewall envanter | Agent HP-BLOCK + sync-rules |
| IoC / KEV / ransomware lists | Cloud intel bundle (`api/09`) |
| IR komutları | Control WS / pending → `VALID_COMMAND_TYPES` |

Ingest detayı: [`threat-intel-ingest.md`](./threat-intel-ingest.md)
