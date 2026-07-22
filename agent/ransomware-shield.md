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

## Cloud tarafı — canary alert işleme (implemented)

Alert SoT = `POST /api/alerts/urgent` (aşağıdaki **Wire** bölümü). Cloud ek olarak:

- Popup süreç detayı: `system_context.ransomware.suspects[0]` (`image`/`pid`/`cmdline`) > `raw_events` process elemanı > description parse sırasıyla çözülür  
- Health-report `canary_files_intact:false` yalnızca **fallback**: son 30 dk içinde `ransomware_canary_triggered` alert'i varsa sentetik alert **üretilmez** (dupe yok)  
- Fallback alert alanları: `threat_score=100`, `target_service=SYSTEM`, süreç bazlı `recommended_action`, `raw_events.kind=ransomware_canary`; snapshot'ta `ransomware` bloğu varsa (file/change_type/suspects) detaya işlenir

---

## Unlock

| Yol | Komut |
|-----|--------|
| GUI | Ransomware detay → koruma kilidini aç |
| IPC | `RS_UNLOCK` / `RS_STATUS` |
| Dashboard | `unlock_ransomware_quarantine` |

Persist: `%ProgramData%\…\ransomware_quarantine.json`

---

## Wire — canary tetiğinde cloud’a ne gider

**Alert SoT = `POST /api/alerts/urgent`** (anında). Health report **detay taşımaz**;
cloud health-report’tan sentetik canary alert **üretmemeli** (dupe + boş popup) —
en fazla fallback olarak, urgent gelmemişse.

### 1) `POST /api/alerts/urgent` (≤ 4.5.66 — mevcut)

```json
{
  "severity": "critical",
  "threat_type": "ransomware_canary_triggered",
  "title": "🚨 Ransomware Tespiti — Canary dosya MODIFIED!",
  "description": "Tuzak dosya '!000_….xlsx' değiştirildi/silindi. …\n\nDosya: C:\\Users\\…\\.cloud-honeypot-canary\\!000_….xlsx\nDeğişiklik: MODIFIED\n\nAcil müdahale önerilir!",
  "threat_score": 100,
  "auto_response_taken": ["emergency_alert"],
  "source_ip": "",
  "target_service": "",
  "username": "",
  "raw_events": [],
  "system_context": {}
}
```

- `change_type` ∈ `MODIFIED | DELETED | SIZE_CHANGED`
- Dosya yolu + change type **description satırlarında** (`Dosya:` / `Değişiklik:`) — cloud parse edebilir
- Suspect süreç/PID ≤4.5.66’da **yok**: `_contain_after_hit` (≤4s scan) urgent’tan **sonra** koşar
- `source_ip` boş = süreç/dosya olayı (network kaynaklı değil)

### 2) `POST /api/health/report` snapshot (periyodik — mevcut)

| Anahtar | Tip | Anlam |
|---------|-----|-------|
| `canary_files_intact` | bool | `canary_alerts == 0` |
| `ransomware_shield_status` | `active` / `disabled` / `error` | Shield çalışıyor mu |
| `vss_shadow_count` | int | VSS shadow sayısı |
| `canary_coverage` | object (observe) | Path-free coverage counts (1.4.5) |

`canary_coverage` shape (additive; missing = legacy):

```json
{
  "mode": "observe",
  "configured": true,
  "files_total": 40,
  "files_intact": 40,
  "files_missing": 0,
  "roots_covered": 3,
  "coverage_ok": true
}
```

No user paths. Desktop canaries remain **forbidden** (config entries under
`\Desktop\` are skipped/removed at deploy). Snapshot’ta dosya yolu / suspect
süreç **yok** (≤4.5.66). ETW shadow aggregates live under snapshot
`etw_shadow` ([`../api/08-architecture.md`](../api/08-architecture.md)) —
alert-only / health-only; never auto-contain.
### 3) Zengin şema (≥ **4.5.67**; tek-yol garantisi ≥ **4.5.68** — implemented)

> **4.5.68 hotfix:** 4.5.67'de canary hem ince `on_alert` (threat-engine) hem
> zengin `send_urgent` çağırıyordu; canlıda ince payload popup'ı kazanabiliyordu.
> ≥4.5.68: **tek** zengin urgent gönderilir (containment ≤4s sonrası).
> `suppress_local_notify` korunur.

Urgent alert’e ek alanlar:

```json
{
  "target_service": "SYSTEM",
  "recommended_action": "isolate_host",
  "system_context": {
    "ransomware": {
      "trigger": "canary MODIFIED: C:\\…\\!000_….xlsx",
      "file": "C:\\Users\\…\\.cloud-honeypot-canary\\!000_….xlsx",
      "change_type": "MODIFIED",
      "suspects": [
        {
          "image": "evil.exe",
          "path": "C:\\Users\\x\\AppData\\Local\\Temp\\evil.exe",
          "pid": 4712,
          "cmdline": "evil.exe --encrypt …",
          "sha256": "…"
        }
      ],
      "quarantine": { "active": true, "entries": 1, "kills": 1 }
    }
  }
}
```

- Client: quarantine arm + suspect scan (≤4s) **önce**, urgent **sonra** (≤ ~5s gecikme kabul)
- Suspect bulunamazsa `suspects: []` — alanlar yine de yapılandırılmış gönderilir
- `raw_events[0]`: canary file/change; sonraki elemanlar suspect process (`process_name`, `pid`, `cmdline`, `path`, `sha256`)
- Health snapshot’a ek: `ransomware_quarantine: { active, trigger, entries: [{image,path,pid,cmdline,sha256,ifeo,at}] }`
- Cloud popup: `system_context.ransomware` varsa onu kullan; yoksa `description` `Dosya:` / `Değişiklik:` parse (≤4.5.66 uyumluluk)

---

## Acceptance

- [ ] Explorer varsayılanında canary klasörü görünmez
- [ ] OneDrive Documents’a canary yazılmaz
- [ ] Canary hit → dashboard alert; yerel balloon yok
- [ ] `RS_UNLOCK` quarantine temizler
- [ ] Intel-only kural emergency_lockdown tetiklemez
- [x] Cloud, canary popup detayını `alerts/urgent`ten alır (health-report fallback 30 dk dedupe)
- [x] ≥4.5.67: urgent `system_context.ransomware` dolu (file, change_type, suspects[])
- [x] ≥4.5.68: canary urgent tek zengin yol (canlı smoke: urgent 200 + `target_service=SYSTEM` + `isolate_host`)
