# Persistence & tamper protection (survival model)

> **Contract VERSION:** root `VERSION`
> **API base:** `https://honeypot.yesnext.com.tr`
> **Min client:** **≥ 4.6.0** (Guardian service + tamper wire)
> Mimari bağlamı: [`../api/08-architecture.md`](../api/08-architecture.md)

Amaç: SYSTEM motoru **kendisi dışında hiçbir şey durduramasın**. Yalnızca (1) meşru
güncelleme ve (2) kullanıcının PIN ile durdurması bir istisnadır. Bunun dışındaki her
sonlanma **tamper (saldırı)** sayılır → anında dirilir + cloud'a alarm gider.

> **Dürüst sınır:** Kullanıcı modunda mutlak öldürülemezlik yoktur. SeDebugPrivilege'li
> SYSTEM/admin bir süreç `TerminateProcess` çağırabilir (gerçek engel yalnız PPL veya
> imzalı kernel driver ile olur — kapsam dışı). Bu sözleşmenin garantisi:
> **"meşru sinyal olmadan durdurma → ≤5 sn içinde diriliş + kritik tamper alarmı".**

---

## 1) İki bağımsız bacak (cross-watchdog)

| Bacak | Process | Rol |
|-------|---------|-----|
| **A — Motor** | `CloudHoneypot-Background` scheduled task (`--mode=daemon`, SYSTEM Session 0) | Tam motor: threat, firewall, honeypot, RemoteCommands, control WS, RD, ransomware. `:58632` sahibi. |
| **B — Guardian** | **Windows Servisi** `CloudHoneypotGuardian` (LocalSystem, Automatic) | **Yalnız watchdog** — motoru değil, motorun *sağlığını* korur. Motor `motor_ok` değilse task'ı enable+run/spawn eder; silinmiş/disable task'ları yeniden kurar. |
| **C — Watchdog task** | `CloudHoneypot-Watchdog` (her 2 dk) | Üçüncü emniyet; A ve B ikisi de düşerse geri getirir. |

Karşılıklılık (mutual):
- **B → A:** Guardian servisi motor `:58632 motor_ok` kontrol eder (≤ **10 sn** aralık); düşükse Background task'ı enable+run, o da olmazsa daemon'ı doğrudan spawn eder.
- **A → B:** Motor, Guardian servisinin varlığını+çalışmasını kontrol eder (≤ **30 sn**); yoksa `sc create`/`sc start` ile yeniden kurar.
- **Tek motor kuralı korunur:** Guardian motoru **çalıştırmaz**, yalnız diriltir. Makinede yine tek SYSTEM motor.

Windows Servisi kurtarma (SCM native): `sc failure` → restart 1./2./sonraki hatada, sayaç günlük sıfırlanır. Bu, süreç çöküşünde SCM'nin kendiliğinden yeniden başlatmasını sağlar (bizim watchdog'a ek).

---

## 2) Durdurma politikası (yalnızca iki meşru yol)

Motor + Guardian **yalnızca** şu sinyallerle durabilir; her ikisi de bacakların hepsini
aynı anda "stand down" yapar:

| # | Meşru sinyal | Nasıl | Sonuç |
|---|--------------|-------|-------|
| 1 | **Güncelleme** | `update_in_progress.lock` (ProgramData) + `disarm_for_update` | Bacaklar durur; helper kurulum sonrası yeniden arm eder + yeni sürüm başlar |
| 2 | **PIN ile durdurma** | Tray → PIN doğrulama → imzalı `operator_stop.json` token | Bacaklar durur ve **kullanıcı elle başlatana kadar** durur (diriliş yok) |
| 3 | (Uninstall) | Installer tüm task+servis+flag siler | Kaldırma |

`operator_stop.json` (PIN token) alanları:
```json
{ "reason": "operator_pin", "pin_verified": true, "issued_at": "…", "host": "…", "sig": "<hmac>" }
```
- `sig` = HMAC (aynı `security.command_signing` anahtarı) — bir saldırganın sahte stop-token yazması engellenir.
- Guardian ve Watchdog bu token'ı **imza doğrulayarak** kabul eder; imza geçersiz → token yok sayılır + tamper alarmı.

**Kritik kural:** PIN stop tray/GUI'yi değil, **motoru** durdurur (mevcut davranış: PIN yalnız GUI'yi kapatıyordu → düzeltilecek). Update-lock veya geçerli PIN-token yoksa hiçbir bacak durmaz.

---

## 3) Tamper tespiti

| Katman | Mekanizma | Tetik |
|--------|-----------|-------|
| **DACL** | Process nesnesinde Everyone `PROCESS_TERMINATE` **deny** | Yetkisiz `taskkill` başarısız (mevcut) |
| **SACL audit** | Motor + Guardian process nesnesine audit ACE → Security log **4656/4663** | `PROCESS_TERMINATE` / `PROCESS_VM_WRITE` / `PROCESS_VM_OPERATION` erişim talebi → talep eden PID çözülür (image/path/sha256/user) |
| **Beklenmedik çıkış** | Meşru sinyal (update-lock / PIN-token) yokken çıkış | Hayatta kalan bacak diriltir + tamper alarmı |
| **Task/servis sabotajı** | Respawner task veya Guardian servisi update-lock/PIN-token olmadan disable/delete edilirse | Yeniden arm + tamper alarmı |

Diriliş hedefi: **≤ 5 sn**. Diriliş süresi (`resurrect_ms`) alarmda raporlanır.

---

## 4) Tamper alarm wire — `POST /api/alerts/urgent`

Kanary urgent ile aynı taşıyıcı ([`ransomware-shield.md`](ransomware-shield.md) "Wire"). Ek `system_context.tamper` bloğu:

```json
{
  "severity": "critical",
  "threat_type": "agent_tamper",
  "title": "🔴 AGENT TAMPER — motor durdurulmaya çalışıldı",
  "description": "…",
  "threat_score": 100,
  "target_service": "SYSTEM",
  "recommended_action": "isolate_host",
  "system_context": {
    "tamper": {
      "reason": "unexpected_exit | terminate_attempt | task_disabled | service_stopped | dacl_modified",
      "leg": "daemon | service",
      "legitimate": false,
      "resurrected": true,
      "resurrect_ms": 1840,
      "offender": {
        "pid": 6644,
        "image": "evil.exe",
        "path": "C:\\Temp\\evil.exe",
        "sha256": "…",
        "user": "NT AUTHORITY\\SYSTEM",
        "cmdline": "…"
      },
      "ts": "2026-07-21T08:00:00Z"
    }
  },
  "raw_events": [
    { "kind": "agent_tamper", "reason": "terminate_attempt", "offender_pid": 6644, "image": "evil.exe" }
  ]
}
```

- `offender` yalnızca SACL/eventlog'dan çözülebildiyse dolar; çözülemezse yok/boş.
- Meşru durdurmada (update / PIN) alarm gönderilmez; bunun yerine bilgi amaçlı lifecycle event
  (`CLIENT_GRACEFUL_STOP` — [`../api/07-lifecycle-sessions.md`](../api/07-lifecycle-sessions.md)).

**Cloud davranış (✅ implemented):**
- Kritik popup (kanary ile aynı detay motoru; `system_context.tamper.offender` + `raw_events[].offender_pid`/`image` → süreç/PID; `helpers.parse_alert_process_info`). `agent_tamper` popup listesinde (`helpers._POPUP_THREAT_TYPES`).
- **Dead-man (cloud fallback):** tamper urgent gelmeden motor sessizce ölürse — health `snapshot.persistence` `daemon_ok=false`/`service_ok=false` (operator_stop yok) → sentetik `agent_persistence_degraded` alert (high/70, 30 dk dedupe). Detay: [`threat-engine.md`](threat-engine.md) “Dead-man”.
- Opsiyonel: `offender` bilinirse otomatik `block_ip` yerine host'u "under_attack" işaretle + operatöre isolate öner.
- Dedupe: aynı `leg` + `reason` için kısa pencere (öneri 60 sn).

---

## 5) STATUS / health genişletmesi

Motor STATUS (`:58632`) ve `POST /api/health/report` snapshot'ına eklenir:

```json
"persistence": {
  "service_ok": true,
  "service_installed": true,
  "daemon_ok": true,
  "tasks_armed": true,
  "self_protection": true,
  "operator_stop": false,
  "last_tamper_ts": null,
  "tamper_count_24h": 0
}
```

Dashboard bunu "koruma sağlığı" rozeti olarak gösterebilir.
Cloud dead-man `service_installed` + `daemon_ok`/`service_ok` + `operator_stop` alanlarını kullanır (yukarı bkz).

---

## 6) Acceptance

- [ ] Guardian servisi kurulu + Automatic + LocalSystem + SCM restart-on-failure
- [ ] Motor öldürülünce Guardian ≤5 sn diriltir; Guardian öldürülünce motor ≤30 sn yeniden kurar
- [ ] Update-lock ve geçerli PIN-token dışında hiçbir sinyal bacakları durdurmaz
- [ ] PIN stop → **motor** durur ve kullanıcı başlatana kadar durur (diriliş yok)
- [ ] Sahte/imzasız `operator_stop.json` yok sayılır + tamper alarmı
- [ ] Yetkisiz `taskkill` DACL ile başarısız
- [ ] SACL: terminate talebi → offender PID çözülür → tamper alarmı
- [ ] Beklenmedik çıkış → `agent_tamper` urgent (dashboard popup detaylı)
- [ ] STATUS/health `persistence` bloğu dolu
