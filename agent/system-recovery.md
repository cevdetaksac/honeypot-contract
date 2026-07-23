# System Recovery — saldırı yüzeyi snapshot / drift / restore

> **Contract VERSION:** root `VERSION` (**1.4.13**)  
> **Min client:** target **≥ 4.9.12** (additive; production floor ≥ 4.9.0)  
> İlgili: [`network-guard.md`](network-guard.md) · [`persistence-and-tamper.md`](persistence-and-tamper.md) ·
> Control WS: [`../api/03-control-websocket.md`](../api/03-control-websocket.md)

## Amaç

Saldırganların **tek exe ile bozduğu kontrol düzlemini** tersine çevirmek:

- Görev Yöneticisi / Regedit / CMD kapatma (policy)
- Firewall profilini kapatma
- VSS / Security Center vb. kritik servisleri durdurma
- (Mevcut Network Guard ile) mapped drive / DNS / adapter bozma

**Kapsam dışı:** tüm registry dump, tüm Windows image, kullanıcı dosya içeriği.
Yalnız **allowlist** yüzeyler. Full `.reg` export **yasak**.

## Model

1. **Snapshot (imzalı)** — boot + periyot (öneri ≤ 30 dk) + operatör `system_recovery_snapshot`
2. **Drift watch** — allowlist’te beklenen “sağlıklı” değere göre delta →
   `system_recovery_drift` alert (severity **warning** / **high**; urgent yalnız high+confirm path değil, default warning→batch veya urgent=high)
3. **Restore** — dashboard one-click; **mutating** = `confirm:true`; `dry_run:true` = plan only

Dosya: `%ProgramData%\YesNext\CloudHoneypotClient\system_recovery.json` (+ history N=10).  
`sig` = HMAC (`security.command_signing` ile aynı anahtar ailesi; NG baseline ile uyumlu).

## Allowlist yüzeyler (v1)

| `id` | Tür | Örnek |
|------|-----|--------|
| `policy.taskmgr` | reg | `HKCU/HKLM\...\Policies\System\DisableTaskMgr` → sağlıklı = yok veya 0 |
| `policy.regedit` | reg | `DisableRegistryTools` |
| `policy.cmd` | reg | `DisableCMD` |
| `policy.norun` | reg | `Policies\Explorer\NoRun` |
| `firewall.profiles` | firewall | domain/private/public state on (NG ile örtüşebilir; SR özeti) |
| `service.vss` | service | `VSS` Start=auto/manual + Running tercih |
| `service.swprv` | service | Shadow Copy Provider |
| `service.wscsvc` | service | Security Center |
| `service.eventlog` | service | EventLog |
| `service.schedule` | service | Schedule |

Allowlist **genişletilebilir** (cloud config ileride); client bilinmeyen `id` restore etmez.

## Komutlar (Control WS)

| `type` | Params | Confirm | Min |
|--------|--------|---------|-----|
| `system_recovery_snapshot` | — | hayır | ≥4.9.12 |
| `list_system_recovery` | — | hayır | ≥4.9.12 |
| `system_recovery_diff` | `version?` | hayır | ≥4.9.12 |
| `system_recovery_restore` | `targets[]?`, `dry_run?`, `rollback_version?` | **mutate evet** / dry_run hayır | ≥4.9.12 |

`targets[]` örnek: `["policy","firewall","service"]` veya id listesi
`["policy.taskmgr","service.vss"]`.

### Dry-run result

```json
{
  "success": true,
  "data": {
    "dry_run": true,
    "baseline_version": 3,
    "plan": [
      {"id": "policy.taskmgr", "action": "reg_set", "hive": "HKCU", "path": "...", "name": "DisableTaskMgr", "to": 0},
      {"id": "service.vss", "action": "service_start", "name": "VSS"}
    ]
  }
}
```

Hatalar: `no_baseline`, `baseline_signature_invalid`,
`rollback_baseline_not_found_or_invalid`, `unknown_target`.

## Alert — `system_recovery_drift`

```json
{
  "severity": "warning",
  "threat_type": "system_recovery_drift",
  "threat_score": 55,
  "title": "Sistem yüzeyi değişti (policy/service/firewall)",
  "description": "DisableTaskMgr=1; VSS stopped",
  "system_context": {
    "system_recovery": {
      "baseline_version": 3,
      "changes": [{"id": "policy.taskmgr", "from": 0, "to": 1}]
    }
  },
  "recommended_action": "system_recovery_restore",
  "force_urgent": false
}
```

Çoklu kritik policy (TaskMgr+Regedit+Firewall off) → severity **high**, score ≤ 80  
(ransomware_canary / VSS delete ile karıştırma — score 100 rezerv).

## STATUS / health (additive)

```json
"system_recovery": {
  "present": true,
  "enabled": true,
  "baseline_version": 3,
  "baseline_age_sec": 400,
  "drift": false,
  "drift_count": 0,
  "last_snapshot_at": "2026-07-23T12:00:00Z"
}
```

## Güvenlik kuralları

- Restore **asla** otomatik değil (Network Guard ile aynı felsefe).
- Defender’ı zorla enable etmek v1’de yok (yanlış pozitif / yönetilen filo).
- Snapshot’a secret/password yazılmaz.
- GUI quit ≠ restore; operator confirm zorunlu.

## Acceptance

1. Lab: `DisableTaskMgr=1` yaz → drift alert ≤ 2 dk  
2. `system_recovery_restore` dry_run → plan’da `policy.taskmgr`  
3. confirm restore → Task Manager tekrar açılır  
4. `VSS` stop → drift; restore → start  
5. Full registry export komutu **yok**
