# Ransomware shield & canary UX

> **Canonical:** `honeypot-contract/agent/ransomware-shield.md`  
> **Min client:** ≥ **4.5.65** (UX) · quarantine unlock IPC ≥4.5.62 · user canary watch ≥4.5.64

---

## Amaç

Erken ransomware tespiti (canary / VSS / şüpheli process) — **günlük kullanıcıyı korkutmadan**.

---

## Canary yerleşimi

| Konum | Not |
|-------|-----|
| Interactive user `Documents\.cloud-honeypot-canary` | OneDrive path **skip** |
| `C:\Users\Public\Documents\…` | Ortak |
| `%ProgramData%\.cloud-honeypot-canary` | Ortak |
| Desktop | **Yasak** (legacy temizlenir) |

- İsimler: `!000_*` sort-bait  
- Attr: **Hidden + System + NotContentIndexed**  
- Admin notice: yalnızca ProgramData `CANARY_README.txt` (bait tree içinde README yok)

---

## Tetik → yanıt

| Sinyal | Client |
|--------|--------|
| Canary MODIFIED / DELETED / SIZE | Score 100; dashboard urgent API |
| | Yerel tray/toast: **suppress** (`suppress_local_notify`) |
| | Quarantine **hemen arm**; suspect `open_files` ≤4s |
| VSS silme | Critical + quarantine arm |
| Cloud intel alone | Lockdown **yok** (canary/VSS yerel otorite) |

IFEO / kill: SearchIndexer, Defender, OneDrive, shell host’lar **korumalı**.

---

## Unlock

| Yol | Komut |
|-----|--------|
| GUI | Ransomware detay → koruma kilidini aç |
| IPC | `RS_UNLOCK` / `RS_STATUS` |
| Dashboard | `unlock_ransomware_quarantine` |

Persist: `%ProgramData%\…\ransomware_quarantine.json`

---

## Acceptance

- [ ] Explorer varsayılanında canary klasörü görünmez
- [ ] OneDrive Documents’a canary yazılmaz
- [ ] Canary hit → dashboard alert; yerel balloon yok
- [ ] `RS_UNLOCK` quarantine temizler
- [ ] Intel-only kural emergency_lockdown tetiklemez
