# Defense Policy — Client apply

> **Contract VERSION:** root `VERSION` (**1.4.19**)  
> **Min client:** ≥ **4.9.17** (matrix core ≥ **4.9.16**)  
> **Cloud SoT:** [`../cloud/DEFENSE_POLICY.md`](../cloud/DEFENSE_POLICY.md)  
> **Roadmap:** [`../ROADMAP_TIERED_DEFENSE.md`](../ROADMAP_TIERED_DEFENSE.md)

Client **aptal itaatkâr rule engine** + hard safety + onboarding UX.  
Politika zekâsı / fleet default cloud’da; client imzalı cache uygular.

---

## 1. Davranış özeti

| Mod | Uyarı | Otomatik süreç | Ağ |
|-----|-------|----------------|-----|
| observe | Hepsi | Yok (`alert_only`) | Yok |
| balanced | Hepsi | Kırmızı kill/Q veya suspend | Yok |
| paranoid | Hepsi | Daha agresif; isolate yalnız armed | Opsiyonel |

**Kurulum / ilk hydrate default:** `observe` (builtin balanced kaldırıldı ≥4.9.17).

---

## 2. Config apply

Kaynak: `GET /api/threats/config` → `protection.*`  
Push: WS `threat_config_updated` → aynı sync.

Alanlar: `defense_policy`, `defense_policy_version`, `defense_rules`,  
`defense_rules_sig` / `sig`, `isolate_armed`,  
`observe_started_at`, `observe_auto_promote_days`,  
`observe_auto_promote_enabled`, `defense_policy_locked`, `policy_user_set`.

Hard safety (matrix üstü):

- observe/balanced → `auto_isolate_network` strip → process fallback  
- tamper → LKG / observe (asla paranoid+isolate)  
- protected images asla  
- high_io / surface_additive asla kill/suspend  

ProgramData: `defense_policy.json` (+ `.lkg.json`), `defense_allowlist.json`,  
`session_snapshots/`.

STATUS / health: `defense_policy`, `defense_policy_version`, `isolate_armed`,  
`observe_*` özet, `sig_ok`, `source`.

---

## 3. Onboarding (client)

1. İlk boot: observe + `observe_started_at` (yoksa yaz).  
2. GUI: üç mod eğitim metni (İzleme / Denge / Tetikte).  
3. Observe’dayken banner/CTA:  
   “Sorun görmüyorsanız **Denge** moduna geçin — kırmızı tehditte süreç korunur, ağ kesilmez.”  
4. Auto-promote backup (cloud job gecikirse):  
   koşullar cloud ile aynı → kullanıcıya net diyalog **veya**  
   `policy_user_set=false` ve unlocked ise `POST /api/threats/config` balanced önerisi  
   (tercihen cloud SoT; client yalnız yedek).  
5. “İzlemede kal” → local flag + cloud’a `defense_policy_locked` / disable promote patch.  
6. Auto-promote **asla** paranoid / `isolate_armed` açmaz.

---

## 4. Komutlar

| Komut | Not |
|-------|-----|
| `resume_process` | Exact identity |
| `allow_process` | path/image/hash whitelist |
| `unlock_ransomware_quarantine` | Mevcut |
| `isolate_host` | Reject unless paranoid+armed; P2 full path |
| `lift_isolation` | network_restore yolu |

---

## 5. Acceptance (client)

- [ ] Fresh install STATUS `defense_policy=observe`  
- [ ] Observe canary → alert, kill/Q yok  
- [ ] Balanced canary → kill/Q, ağ ayakta  
- [ ] Tamper → LKG/observe, ağ ayakta  
- [ ] CTA görünür; kilitle → auto-promote yok  
- [ ] Cloud promote sonrası WS ≤ poll cycle apply  
