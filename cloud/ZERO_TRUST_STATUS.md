# Zero Trust — durum (1.4.16)

> Asimetrik envelope / operator keyset **hâlâ Design-only**.  
> Bu dosya cloud’a “şimdi ne yapmalı / ne yapmamalı” netliği verir.

## Gemini iddiası

> “SaaS çıkmadan önce asymmetric envelope production’a alınmalı.”

## Gerçek durum

| Paket | Statü | Dosya |
|-------|--------|--------|
| v1 HMAC command signing | Normative (soft-allow missing sig) | `api/03-control-websocket.md` |
| `caps.command_envelope_v2` | Observe only (`off`\|`observe`) | `api/03` |
| ZT-601 envelope v2 | **Design-only** | `cloud/command-envelope-v2-design.md` |
| ZT-602/603 operator keys | **Design-only** (+ cloud observe stub) | `cloud/operator-keyset-design.md` |
| TPM PoP / enrollment | **Design-only** | design gate + `device_identity` observe |

Client scaffold (`command_envelope`, `operator_keys`, `device_identity`) **imza doğrulamaz**; `verify_enabled` zorla false.

## Neden hemen “production enforce” yok

Design gate açık: serialization (JCS/CBOR), algo (Ed25519/WebAuthn), signer custody, replay, key lifecycle, test vectors.  
Bunlar kapanmadan `version:2` emit etmek = kırık filo + false sense of security.

## Cloud’un şimdi yapması (1.4.16)

1. **Yapma:** fleet’e `version:2` asymmetric envelope emit / enforce  
2. **Yap:** v1 HMAC coverage ölçümü (ZT-600) — imzasız komut oranını dashboard’da göster  
3. **Yap (observe):** `GET /api/agent/operator-keys` public-only stub (`verify_enabled:false`) zaten varsa koru  
4. **Sonra (ayrı sprint):** design gate karar notu + test vectors → OOB-501 gibi promote

## “Hacklenirseniz biz de hackleniriz”

Bugün savunma: Bearer token + v1 HMAC soft-allow + confirm-gated mutates + Network Guard / System Recovery.  
Asimetrik ZT bunu **güçlendirir** ama **tek başına SaaS kapısı değil**. Önce gate kapat, sonra observe, sonra pilot, sonra enforce.

Detay: [`SECURITY_RESILIENCE_VNEXT.md`](../SECURITY_RESILIENCE_VNEXT.md).
