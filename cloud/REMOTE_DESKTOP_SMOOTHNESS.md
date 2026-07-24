# Remote Desktop smoothness — Cloud/Dashboard requirements

> **Contract VERSION:** root `VERSION` (**1.4.20**)  
> **Client target:** ≥ **4.9.20** (WebRTC raw + HW H.264 + input fluidity)  
> **Canonical RD API:** [`../api/05-remote-desktop.md`](../api/05-remote-desktop.md)  
> **Input:** [`../agent/remote-input.md`](../agent/remote-input.md)

Bu belge **cloud + dashboard viewer** tarafının yapması gerekenleri listeler.  
Client 4.9.20+ ham RGB → H.264 (NVENC/QSV/AMF/x264-zerolatency) üretir; JPEG-WS yalnız fallback’tır.

---

## 0. Hedef deneyim

Chrome Remote Desktop seviyesine yaklaşmak:

| Katman | Hedef |
|--------|--------|
| Video | WebRTC H.264, 30–60 capture, dirty/skip idle, HW encode tercih |
| Input | Data-channel (veya WS) üzerinde fare/klavye **düşük gecikme**; move coalesce OK, key/button asla geciktirme |
| Fallback | ICE/PC fail → JPEG-WS; kullanıcıya tek surface |

---

## 1. Cloud / viewer zorunlulukları (C-RD-*)

| ID | İş |
|----|-----|
| C-RD-1 | WebRTC offer **yalnız** `capabilities.webrtc.available` + `"webrtc"∈transports` iken |
| C-RD-2 | Viewer: WebRTC `connected` iken JPEG `<img>` **gönderme/çizme** (bandwidth yarışı yok) |
| C-RD-3 | Input tercihen **RTC data channel** (aynı input-v2 envelope); WS yedek |
| C-RD-4 | Pointer move coalesce viewer’da serbest; **keydown/keyup/mousedown/mouseup/wheel** anında flush (batch’leme ≤1 frame) |
| C-RD-5 | `webrtc_ice_config` + short-lived TURN; reject/`webrtc_reject` → anında JPEG fallback UI |
| C-RD-6 | `meta.media.encoder` (`nvenc\|qsv\|amf\|x264\|aiortc`) + `effective_capture_fps` rozeti |
| C-RD-7 | Stream restart: eski peer drop; yeni `stream_id`; stale reject honor |
| C-RD-8 | Latans telemetrisi (opsiyonel): `capture_mono_ms` / RTT badge |

---

## 2. Viewer input akıcılığı (normative UX)

1. Mouse move: rAF veya ≤8–16 ms coalesce; son pozisyon kazanır.  
2. Click / key: **coalesce etme** — hemen data-channel `send`.  
3. Relative + absolute karıştırma: mevcut protocol 2 kuralları.  
4. Ack beklemeden sonraki move’a izin (fire-and-forget moves).  
5. WS-only fallback’ta aynı kurallar; HTTP input poll **kullanma** eğer WS/DC ayaktaysa.

---

## 3. Acceptance (cloud + lab)

- [ ] WebRTC connect → video `<video>` akıcı; JPEG frame ignore  
- [ ] `media.encoder` dolu (mümkünse `nvenc`/`qsv`/`amf`)  
- [ ] Typing / click hissedilir gecikme düşük (lab: yan yana CRD karşılaştırması)  
- [ ] ICE fail → JPEG-WS ≤ 2s  
- [ ] TURN credential log’larda redact  

---

## 4. Client özeti (referans)

≥4.9.20:

- WebRTC path: **no JPEG double-encode** (raw RGB mailbox → VideoFrame)  
- HW H.264 patch when FFmpeg codec present; else libx264 ultrafast+zerolatency  
- Static desktop: unchanged frame skip (encoder near-idle)  
- Input: higher move rate, shorter critical ACK, DC preferred  
- Adaptive JPEG knobs **do not** thrash helper while WebRTC connected  

Detay: [`../api/05-remote-desktop.md`](../api/05-remote-desktop.md) § Client WebRTC smoothness (promoted).
