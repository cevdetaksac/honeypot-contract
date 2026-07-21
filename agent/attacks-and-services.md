# Attacks & honeypot services

> **Contract VERSION:** root `VERSION`  
> **API base:** `https://honeypot.yesnext.com.tr`  
> **Auth:** Bearer (register hariç)  
> **Kaynak:** canlı `routes_agent.py` + `schemas.py`

Client kendi içinde bait servisler çalıştırır (RDP/SSH/FTP/MYSQL/MSSQL); credential yakalar → cloud’a POST. Gerçek auth **yapılmaz**.

---

## POST /api/attack

Alias’lar (aynı handler ailesi): `/api/honeypot-attack`, legacy `/attacks/` (varsa).

```json
{
  "token": "<optional if Bearer>",
  "attacker_ip": "203.0.113.10",
  "target_ip": "198.51.100.5",
  "username": "administrator",
  "password": "Passw0rd!",
  "service": "RDP",
  "port": 3389
}
```

| Alan | Not |
|------|-----|
| `service` | `RDP` / `SSH` / `FTP` / `MSSQL` / `MYSQL` / `Network` — port’tan normalize edilebilir |
| `attacker_ip` | Yoksa `ip` / peer IP |
| Şifre | Cloud log/audit’te maskelenebilir; attack kaydında saklanır (dashboard) |

**200:** `{ "status": "ok", … }` — cloud `attacks` satırı + geoenrich + notification_rules eşik sayacı.

---

## Honeypot servis modeli (eski “tunnel”)

Dashboard hâlâ “tunnel” API isimleri kullanır; anlam = **bait listen desired/state**.

| Endpoint | Kim | Ne |
|----------|-----|-----|
| `GET /api/premium/tunnel-status` | Agent poll | Desired + reported state |
| `POST /api/agent/tunnel-status` | Agent | Gerçek listen durumu raporu |
| `POST /api/premium/tunnel-set` | Dashboard | Desired start/stop / port |
| Komut `tunnel_start` / `tunnel_stop` | Control WS / poll | Agent bait servisi aç/kapa |

Varsayılan listen portları (şablon): RDP 3389, MSSQL 1433, MYSQL 3306, SSH 22, FTP 21.

### `pending_tunnel_commands` — TTL + dedupe (geri uyumluluk kuyruğu)

`tunnel-set` iki yol yazar: **hızlı yol** `pending_commands` (`tunnel_start`/`tunnel_stop`, agent zaten poll/WS ile alır) + **legacy kuyruk** `settings_json.tunnel_commands` (eski agent’lar için `tunnel-status` yanıtındaki `pending_tunnel_commands`).

Legacy kuyruk artık **TTL + dedupe** uygular (cloud, `routes_agent._prune_tunnel_commands`):

- **Dedupe:** servis başına yalnız **en yeni** komut tutulur (start→stop art arda gelirse en son desired kazanır); `tunnel-set` aynı servisin eski girdilerini düşürür.
- **TTL:** `created_at` üzerinden **30 dk**’dan eski girdiler düşer. Hem yazma (`tunnel-set`) hem okuma (`tunnel-status`) yolunda uygulanır; okuma sırasında değişiklik olursa kalıcılaşır.
- Agent gerçek listen durumunu `POST /api/agent/tunnel-status` ile raporladığında desired’e ulaşan girdiler zaten düşürülür (mevcut davranış korunur).

**RDP port taşıma:** Gerçek RDP güvenli porta alınır; bait 3389’da kalabilir. Onay akışı client’ta korunur — cloud sadece desired/state görür.

---

## POST /api/agent/open-ports

```json
{
  "ports": [
    { "port": 3389, "protocol": "tcp", "process": "svchost.exe", "state": "LISTEN" }
  ]
}
```

Cadence ~**5 dk**. Cloud `settings_json.open_ports` yazar; dashboard honeypot işaretleri bilinen port setine göre.

---

## Client kuralları

1. Bait protokol handshake + credential capture; başarılı gerçek login **yok**.
2. Her yakalamada `/api/attack` (veya batch politikası varsa).
3. Desired tunnel state’e uy; rapor `tunnel-status` ile.
4. `protection.block_rules` / local fail sayacı **gerçek** auth fail event’leri içindir — bait capture ayrı kanal (`agent/register-protection.md`, `agent/threat-engine.md`).

---

## Acceptance

- [ ] Bait RDP/SSH fail credential → `/api/attack` 200, dashboard Attacks’te görünür
- [ ] Tunnel-set start → agent listen + tunnel-status `running`
- [ ] open-ports periyodik 200
