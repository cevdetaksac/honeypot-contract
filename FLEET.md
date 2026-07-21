# Fleet & version matrix

> Hangi client sürümü hangi contract özelliğini karşılar.  
> Contract VERSION: root `VERSION`.

| Özellik | Contract dosyası | Min client |
|---------|------------------|------------|
| Bearer-only agent API | `api/01-auth.md` | 4.4.33+ / fleet default |
| Control WS commands | `api/03-control-websocket.md` | 4.5.x |
| Self-update GUI banner | `api/04-self-update.md` | 4.5.39 |
| Remote `inputs[]` piggyback | `agent/remote-input.md` | **4.5.55** |
| One GUI/session + motor watchdog | `api/08-architecture.md` | 4.5.58–59 |
| Threat-intel GET/304/ACK | `api/09-threat-intel.md` | **4.5.61** |
| Canary sort-bait + quarantine IPC | `agent/ransomware-shield.md` | 4.5.62–64 |
| Canary UX (no local scare, OneDrive skip) | `agent/ransomware-shield.md` | **4.5.65** |
| `protection.block_rules` apply + WS intel push | `agent/register-protection.md`, `api/09` | **4.5.66** |
| Enriched canary alert + quarantine snapshot | `agent/ransomware-shield.md` | **4.5.67** |

**Önerilen production floor:** **4.5.67** (cloud canary popup detail); 4.5.66 core protection floor.

Cloud publish: contract clone → `git pull` → `scripts/publish_contract.sh` (HTTPS mirror).
