# Changelog — honeypot-contract

## 1.4.9 — 2026-07-22 (Threat intel HP-INTEL + bare-success no-block)

- Normative client **≥ 4.9.7** behavior (production floor stays **≥ 4.9.0**):
  - [`api/09-threat-intel.md`](api/09-threat-intel.md): `firewall_blocks` →
    **`HP-INTEL-*`** (in+out), never AutoResponse `HP-BLOCK-*`; policy /
    severity / allowlist / orphan reconcile; ACK `firewall_added` /
    `firewall_skipped` / `firewall_removed`.
  - [`api/06-firewall-blocks.md`](api/06-firewall-blocks.md): whitelist never
    blocks; `clear_firewall` wipe includes `HP-INTEL-*`; sync may surface both
    prefixes.
  - [`agent/threat-engine.md`](agent/threat-engine.md): `should_auto_block()` —
    bare `successful_logon*` → alert/challenge only (score cap ≤70 / silent ≤80,
    never 100 alone); silent hours no FW block; whitelist enforce unblock.
  - [`agent/gui-control-center.md`](agent/gui-control-center.md): whitelist
    SoT + immediate clear of matching `HP-BLOCK` / `HP-INTEL`.
  - [`cloud/threat-intel-ingest.md`](cloud/threat-intel-ingest.md): ACK stats
    aligned with client 4.9.7.
- INDEX / FLEET matrix rows for intel apply + auto-block policy.
- [`api/07-lifecycle-sessions.md`](api/07-lifecycle-sessions.md): uninstall
  PIN gate lifecycle events (`uninstall_requested` / `pin_failed` /
  `aborted` / `authorized`).

## 1.4.8 — 2026-07-22 (Server Management — dashboard ↔ client)

- New normative [`agent/server-management.md`](agent/server-management.md):
  Dashboard **Sunucu Yönetimi** (users / processes / services) inventory +
  mutate contract. Client must return full current lists and execute catalog
  actions.
- Additive **`list_services`** result shape (`name`, `display_name`, `status`,
  `start_type`, `pid?`); cloud caches `windows_services`.
- Service mutates accept **`name` or `service_name`**.
- `list_local_users` for Server Management uses `include_disabled: true`.
- Production floor unchanged (**client ≥ 4.9.0**); target ship **≥ 4.9.4**.
- INDEX / FLEET / CLIENT / api/03 catalog cross-links updated.

## 1.4.7 — 2026-07-22 (OOB-501 promoted to api/)


- Promoted offline urgent queue from design to normative
  [`api/10-offline-urgent-queue.md`](api/10-offline-urgent-queue.md).
- Wire: `POST /api/alerts/urgent/batch` ACK
  (`acked` / `duplicate` / `rejected`, cap 500) plus soft idempotency on
  `POST /api/alerts/urgent` via `event_id` / `idempotency_key`.
- Reject reasons: `schema` | `too_large` | `expired` (queued_at > 7d) |
  `transient`. Production floor stays **client ≥ 4.9.0**.
- Client flag `security.offline_urgent_queue` remains **default off**; cloud
  endpoint is live for pilot drain after heartbeat/WS.
- Cloud E2E: double-delivery of the same `event_id` yields one dashboard
  incident (`duplicate` on replay). Operator-key GET and TPM enrollment stay
  design-only.

## 1.4.6 — 2026-07-22 (dry-run confirm exception + design pointers)

- `agent/network-guard.md` / README: mutating `network_restore` remains
  confirm-gated; `dry_run:true` is plan-only without destructive confirm.
- README pin note points at live `VERSION` instead of a stale tag.

## 1.4.5 — 2026-07-22 (P1 observe schemas promoted additive)

- Promoted landed client P1 observe field shapes into canonical API/agent
  docs without raising the production floor (**client ≥ 4.9.0**) or enabling
  enforcement.
- `api/08-architecture.md`: filled `etw_shadow` (source/fallback/correlation),
  `event_log_health` (+ nested `password_burst`), `command_signing` counters
  (`ok`/`no_token`/`disabled`), and added `access_integrity`,
  `device_identity`, path-free `canary_coverage`, optional `deception_health[]`.
  `binary_integrity` observe semantics clarified (`unknown` = unsigned/dev).
- `api/01-auth.md`: optional `system_context.heartbeat_proof` (HMAC v1
  candidate) with soft-accept cloud behavior.
- `api/03-control-websocket.md`: additive hello `caps.command_envelope_v2`
  (`off`|`observe` only); `network_restore` params `dry_run` /
  `rollback_version` + dry-run result plan (mutate restore remains confirm-
  gated; dry-run-only must not require destructive confirm).
- `agent/network-guard.md`, `agent/ransomware-shield.md`,
  `agent/threat-engine.md`: dry-run/rollback, canary coverage, 4723/4724 +
  password-burst aggregates aligned.
- Wire key SoT for identity burst is **`password_burst`** (not
  `identity_burst`). Offline urgent ingest, operator key GET + verify, TPM
  enrollment, and all enforce toggles remain **not promoted**.
- **Cloud follow-up implemented:** health ingest persists
  `access_integrity` / `device_identity` / `canary_coverage` /
  `deception_health` (missing = legacy); heartbeat soft-stores
  `heartbeat_proof` without rejecting presence; control-WS hello stores
  `caps.command_envelope_v2` (`off|observe` only, no v2 wire emit);
  `network_restore` with `dry_run:true` skips destructive confirm (server +
  dashboard) and is not treated as high-impact for unsigned-dispatch
  blocking; Security & Resilience card shows the new observe badges;
  recommended client **4.9.1**, production floor stays **4.9.0**.
- **Cloud follow-up (design scaffolds):** `POST /api/alerts/urgent` soft
  idempotency (`event_id`/`idempotency_key`); `POST /api/alerts/urgent/batch`
  ACK shape (`acked`/`duplicate`/`rejected`, cap 500, mode=observe);
  `GET /api/agent/operator-keys` observe stub (`verify_enabled:false`, no
  private material). Client offline-queue / verify flags stay **off**.
  Dashboard adds network_restore **dry-run plan** button (no confirm).
  TPM enrollment remains design-only.

## 1.4.4 — 2026-07-22 (client 4.9.1 WebRTC smoothness)

- Client 4.9.1 implements strict JPEG suppression while ICE+DTLS media is
  connected, clears pre-connect pending JPEGs, and resumes the existing
  JPEG-WS/HTTP fallback when media disconnects/fails/closes.
- WebRTC capture pacing is decoupled from JPEG-era controls (30 fps/Q78
  starting profile; helper accepts up to 60 fps), stale frames remain
  latest-only, and the WebRTC build optionally prefers DXGI Desktop Duplication
  (`dxcam`) before GDI/ImageGrab/MSS fallback.
- Added optional media telemetry: `encoder`, `effective_capture_fps`,
  `capture_quality`, `target_bitrate_bps`. Current encoder is honestly reported
  as aiortc; verified NVENC/QSV/AMF integration remains a later measured task.
- Production floor remains **4.9.0**; 4.9.1 is recommended for smoother WebRTC.

## 1.4.3 — 2026-07-22 (P1 client observe package planning)

- Recorded the landed client P1 observe/default-off primitives in
  `SECURITY_RESILIENCE_VNEXT.md`: signed-heartbeat candidate, path-free ACL
  drift, ETW correlation, deception/canary health, network restore dry-run and
  retained-version rollback, DPAPI offline queue, password burst aggregate,
  operator public-key metadata scaffold and read-only TPM capability.
- Added the exact cloud/dashboard promotion checklist. No candidate schema is
  normative yet; production wire and client floor **4.9.0** remain unchanged.
- Reaffirmed that destructive restore remains confirm-gated, identity
  correlation never auto-locks, ETW never auto-contains, TPM absence is
  unsupported rather than failure, and v2 operator verification remains off
  until canonical algorithms/key distribution/test vectors are promoted.

## 1.4.2 — 2026-07-22 (cloud-first resilience observe compatibility)

- Promoted the first v1.4.1 Lane-B/P0 schemas into canonical contracts without
  changing the production floor (**client 4.9.0**) or enabling enforcement.
- `api/03-control-websocket.md`: every new command creation path now receives
  one persisted v1 HMAC envelope; WS and pending poll deliver identical
  `issued_at`/`signature`. Cloud exposes signed/unsigned/failure coverage.
  Signature generation failure blocks destructive dispatch, while legacy
  low-impact soft-allow remains during observe compatibility.
- Added deterministic HMAC vectors covering timestamp serialization, Unicode
  input and machine-name casing. Existing terminal rows are excluded from the
  rollout denominator; pending legacy rows are backfilled.
- `api/08-architecture.md`: promoted optional `resilience`,
  `event_log_health`, `etw_shadow` sensor-health and `command_signing` health
  blocks. Missing means legacy. Two consecutive degraded resilience samples
  create a deduped observe alert; no automatic containment.
- `api/04-self-update.md`: promoted additive release trust storage
  (SHA-256, signer/publisher/thumbprints, timestamp policy, SBOM/provenance and
  rollout/error state). Unknown stays `null`; current release wire is not
  blocked during observe mode.
- Dashboard now shows command-signing coverage/state, Guardian/restart storm,
  binary integrity and release trust metadata before any enforcement toggle.
  Network Guard remains alert-only and destructive commands confirm-gated.
- Added `cloud/command-envelope-v2-design.md` for the ZT-601 design gate:
  canonical envelope candidate, WebAuthn/managed-signer custody decision,
  approval/replay/key lifecycle, optional HPKE separation and TPM
  proof-of-possession/re-enrollment. It explicitly cannot emit production v2
  wire before normative promotion.
- Follow-up from cloud Lane-B mapping: `kill_process` is now server
  confirm-gated; unsigned high-impact creates return HTTP 400 instead of
  queuing; legacy `tunnel_commands` carry additive `issued_at`/`signature`
  while `PendingCommand` remains the authoritative signed path.
- `api/05-remote-desktop.md`: added **“Client TODO — WebRTC smoothness”**
  (production observation 2026-07-22). Client should: (1) **remove the
  internal 10 fps request clamp** — cloud sent `fps: 30.0` in
  `remote_stream_start` and the agent result/meta reported
  `requested: {fps: 10.0}`; a 10 Hz source cannot look fluid on any
  transport, (2) stop JPEG WS frames while the peer connection is
  `connected` and resume them on fallback (measured `send_ms_ewma ≈ 12 s`
  and 67 degrades while both paths ran concurrently), (3) decouple WebRTC
  capture/encode from the JPEG `fps`/`quality` knobs — 30–60 fps
  change-driven capture (DXGI dirty regions), hardware encoder when
  available, bitrate governed by WebRTC bandwidth estimation instead of a
  fixed Q constant, (4) prefer frame dropping over queueing (latest-frame
  pacing), (5) after a stream restart the re-offer must complete or
  `webrtc_reject` — observed stuck `connecting` / `peer setup failed`,
  (6) optionally report `media.encoder` + effective encode fps / target
  bitrate (additive). No wire change. Cloud viewer now defaults to 24 fps
  and offers a 30 fps option (server clamp stays ≤ 30 for the JPEG path).
- Remote Desktop v2 viewer fixes (cloud dashboard): when WebRTC video is
  connected the JPEG `img` surface is force-hidden (no more double-desktop
  render), late `img.onload` cannot re-show it, status polling no longer
  downgrades a live WebRTC session to "waiting/empty frame", and the stats
  badge measures decoded video FPS via `getVideoPlaybackQuality()`.
  `GET /api/remote/status` now reports `live=true` / `diag=live` while a
  viewer WebRTC peer is connected even though the agent stops posting JPEG
  frames in that mode.

## 1.4.1 — 2026-07-22 (Security & Resilience vNext delivery plan)

- Added `SECURITY_RESILIENCE_VNEXT.md` as the shared implementation plan for
  Client, Cloud/API, Dashboard and QA. It is explicitly **planning /
  non-normative**: current wire behavior and the client **4.9.0** production
  floor do not change until each schema is promoted to its canonical contract.
- Defined parallel delivery lanes, ownership, dependency order, compatibility
  rollout and joint acceptance gates for command integrity, release trust,
  resilience, identity events, ETW ransomware telemetry, binary integrity,
  deception, device identity and future E2E authorization/encryption.
- P0 cloud instructions make existing command HMAC coverage observable before
  enforcement, require deterministic Client/Cloud crypto test vectors and a
  cloud-first dual-compatible rollout. Missing signatures are not rejected
  fleet-wide until cloud signing reaches 100% and the observing client release
  proves compatibility.
- Added coordinated Authenticode/WinVerifyTrust release metadata, Guardian
  health/SLO, Event IDs 4723/4724, ETW shadow-mode aggregation, SBOM/provenance
  and canonical asymmetric command-envelope design packages.
- Reaffirmed safety boundaries: no automatic process/network containment,
  automatic admin lockout, process-wide realtime priority, covert DNS/ICMP
  tunnel or irreversible hardware fingerprint lock.
- `VERSION`/`INDEX.md`/`README.md` updated to **1.4.1**. This planning-only
  release does not modify `FLEET.md`.

## 1.4.0 — 2026-07-21 (client **4.9.0** — Remote Desktop v2)

- `api/05-remote-desktop.md` is now **canonical** with an early, clearly
  delimited **"Remote Desktop v2 (client ≥ 4.9.0)"** section for cloud/dashboard
  implementers. Legacy prompt-sourced material is preserved but marked
  *(superseded by v2)* where it conflicts.
- **WS `hello`/`meta` exact additive schemas** documented (protocol 2):
  truthful `capabilities` (`codecs` always starts `["jpeg"]`; `h264`/`vp8` and
  `"webrtc"` transport only when the aiortc/av runtime is really present;
  `webrtc.signaling=1`, `ice="non-trickle"`, `fallback="jpeg-ws"`), and `meta`
  carrying `native_*` vs encoded `width/height`, `origin_x/y` (may be negative),
  requested-vs-effective adaptive fields and monotonic capture/send stamps.
- **Healthy path is WS + JPEG only** — corrected the long-standing wrong
  guidance that the agent must POST an HTTP frame on every WS frame. HTTP frame
  upload (and its `inputs[]` ACK, the primary input path while on HTTP) is used
  **only while WS is down**. Latest-frame / coalescing outbound semantics and
  input backup poll cadence (2.0 s WS-healthy / 0.30 s WS-down) specified.
- **Input protocol 2** exact envelope + legacy protocol-1 compatibility, WS
  `input_ack` schema (one ACK per coalesced move id), semantic mobile events
  (tap/double_tap/long_press, drag_start/move/end, trackpad/direct modes,
  two-finger vertical+horizontal scroll, relative movement), critical-edge
  ordering/non-drop + move budget, and the **no key/text logging** privacy rule.
- `agent/remote-input.md` rewritten: no longer requires duplicate per-frame HTTP
  posts; documents the **persistent same-session helper** (SYSTEM daemon loopback
  listener + one `CreateProcessAsUser` helper, HMAC-framed `RDH1` full-duplex
  F/I/A/C/S protocol, live config, fire-and-forget moves + short-ACK critical
  edges) and the legacy one-shot fallback.
- **WebRTC signaling over the existing agent/view WS relay** (optional, client
  ≥ 4.9.0): exact `webrtc_offer`/`answer`/`reject` schema with **protocol=1**,
  strict `stream_id`/`session_id` matching, **non-trickle ICE** (client explicitly
  **rejects standalone trickle ICE** with `reason=non_trickle_ice`), H264
  preference, data-channel input, automatic JPEG-WS fallback. Cloud requirements:
  viewer-created offer, answer/reject relay, runtime capability gating, session
  cleanup / stale rejection. No new endpoints invented; existing WS/HTTP
  endpoints kept.
- **Cloud-supplied ICE configuration implemented (client ≥ 4.9.0):** the
  protocol-1 **offer** may carry optional `ice_servers` (≤8 servers, ≤8 URLs
  each, `stun:`/`turn:`/`turns:` only, URL ≤512 / username ≤256 / credential
  ≤512 chars, no whitespace/control chars, only `urls`/`username`/`credential`
  keys); the client validates and applies it to its `RTCPeerConnection`,
  rejecting out-of-bounds configs **without echoing values** (generic errors;
  no credential in logs/reject payloads/status). Truthful capability flag
  `webrtc.ice_server_config` advertised in `hello`/`meta`. Cloud flow over the
  **existing viewer/agent WS only**: cloud pushes short-lived
  `{"t":"webrtc_ice_config","protocol":1,"stream_id",…,"ice_servers":[…]}` to
  the viewer before offer creation; the viewer builds its RTCPeerConnection
  and forwards the same bounded `ice_servers` in the offer; cloud must redact
  credentials from logs/audits/status. TURN credential issuance/config
  delivery is now an **actionable Cloud TODO** (client consumer shipped), not
  an open question.
- Added an ordered **Cloud TODO / acceptance checklist** (dashboard mobile
  pointer UX, `<video>`/WebRTC fallback, telemetry/status badges, browser/mobile
  test matrix).
- `VERSION` → **1.4.0**; `INDEX.md`/`FLEET.md`/`README.md`/`agent/CLIENT.md`
  version + production-floor references aligned to **client ≥ 4.9.0** (README's
  stale 4.5.68 floor fixed).
- **Cloud implemented (honeypot.yesnext.com.tr, 2026-07-22):** agent
  `hello`/per-frame `meta` protocol-2 alanları forward-compatible parse edilip
  canlı room/status durumuna eklendi; sağlıklı JPEG akışı WS-only çalışır ve
  HTTP yalnız fallback/input-drain yoludur. Viewer input-v2 envelope/ACK,
  legacy fallback, direct + trackpad mobil pointer, tap/double-tap/long-press,
  relative drag ve iki eksenli two-finger scroll uygular. Mevcut agent/view WS
  üzerinde stream/session eşlemeli non-trickle WebRTC offer/answer/reject relay,
  `<video>` → JPEG otomatik fallback ve stream restart/stop cleanup eklendi.
  `webrtc_ice_config` bounded validation ile viewer'a iletilir; opsiyonel
  coturn REST secret üzerinden stream-bazlı 60–900 sn kimlik bilgisi üretilir
  ve credential hiçbir status/audit/log yüzeyine konmaz. Dashboard transport,
  effective→requested kalite/FPS, coalesced/degrade/recover/WS failure ve
  WebRTC state/codec rozetlerini gösterir. Production floor **4.9.0**'a çekildi.
- **Cloud TURN/STUN devrede (honeypot.yesnext.com.tr, 2026-07-22):** cloud
  sunucusunda coturn (`194.5.236.181:3478` STUN+TURN udp/tcp, relay
  49160–49300/udp, iç ağlara relay kapalı) kuruldu; `REMOTE_STUN_URLS` /
  `REMOTE_TURN_URLS` / `REMOTE_TURN_SECRET` ile stream-bazlı kısa ömürlü REST
  kimlikleri artık gerçek `webrtc_ice_config` push'unda gönderiliyor. Ayrıca
  viewer'a **15 sn WebRTC bağlanma zaman aşımı** eklendi: ICE `connected`
  olmazsa peer kapatılır, aynı stream için tekrar offer denenmez ve görüntü
  JPEG-WS'te kalır; `webrtc_viewer_state:closed` hub'a elle bildirilir.
  `remote_stream_start` sonrası hub yeni stream için `webrtc_ice_config`'i
  tüm viewer'lara kendiliğinden iter (agent `hello`'su beklenmez).
- **CLIENT TODO (4.9.0 sahada gözlenen bug — öncelik yüksek):** agent, viewer
  offer'ını yanıtladıktan sonra ICE hiç tamamlanmadan (`ice_state:"checking"`,
  `error:"peer setup failed"`, `codec:""`) medya oturumunu **aktif** sayıp
  `media.connection_state:"connected"` raporladı ve tüm kareleri media
  mailbox'a yönlendirdi (`mailbox_coalesced` 1160+); JPEG WS kareleri kesildi
  ve viewer donuk kaldı (`diag: agent_ws_no_frames`). §"JPEG WS fallback from
  WebRTC" gereği: (1) kareler medya yoluna **yalnız ICE/DTLS gerçekten
  bağlandıktan sonra** yönlendirilmeli, (2) `checking` durumu makul bir zaman
  aşımıyla (örn. 15 sn) başarısız sayılıp JPEG-WS'e otomatik dönülmeli,
  (3) `media.connection_state` ICE tamamlanmadan `connected` raporlanmamalı.

## 1.3.13 — 2026-07-21 (client **4.8.5** — block-removed ACK ips + updated)

- **Cloud data retention (honeypot.yesnext.com.tr, 2026-07-21):** saldırı
  geçmişine client-bazlı 1/3/6 aydan eski kayıtları batch silme aksiyonları
  ve dashboard-auth korumalı `POST /api/dashboard/attacks/prune` eklendi.
  Otomatik `Attack` retention 180 günden **365 güne** çıkarıldı;
  büyük `raw_events/system_context` alanları taşıyan `ThreatAlert` kayıtlarına
  da 365 günlük üst sınır getirildi. Takip güncellemesinde Tehdit Merkezi'ne
  uyarı ve bildirim geçmişi için ayrı 1/3/6 ay silme menüleri, görünür
  `NotificationLog` tablosu ve dashboard-auth korumalı ortak
  `POST /api/dashboard/history/prune` (`attacks|alerts|notifications`) eklendi.
  Uyarı ve bildirim menülerinde ayrıca güçlü onay metinli `clear_all` seçeneği
  vardır; saldırı geçmişi toplu `clear_all` kapsamına bilinçli olarak alınmaz.
  `NotificationLog` ve pasif `AutoBlock` için 365 gün; terminal durumdaki
  `PendingCommand` ve kapanmış `LogonChallenge` için 90 gün otomatik retention
  uygulanır. Açık komut/challenge, aktif bloklar ve silinmez `CommandAuditLog`
  kapsam dışıdır. `SecurityEvent` 90 günlük mevcut politikayı korur. Saatlik
  temizleyici, binlerce client'ta backlog oluşmaması için tablo başına
  kontrollü olarak en fazla 10 batch boşaltır.
- `api/06-firewall-blocks.md`: `POST /api/agent/block-removed` ACK body now
  documents `block_ids` **and** `ips`/`ip`. Live 4.8.4: ids-only often returned
  `updated:0` while dashboard stayed on "Kaldırılıyor…"; ip ACK returns
  `updated>0`. Cloud TODO: match remove_pending by block_ids too; keep queue
  items until ACK (not on GET). Production floor 4.8.5.
- **Cloud implemented (honeypot.yesnext.com.tr, 2026-07-21):** `block-removed`
  artık `block_ids` **ve** `ips`/`ip`'yi birlikte işler (4.8.4'teki
  `if not block_ids` erken-çıkış hatası giderildi); satırlar `id` ile dedupe,
  `updated` = kapatılan benzersiz satır sayısı, `pending` durumu da kapatılır,
  yanıt `removed_ips[]` döndürür. `pending-unblocks` GET salt-okunur (öğeler ACK
  gelene kadar kuyrukta kalır). `AgentBlockRemoved` şemasına `ips: List[str]`
  eklendi.
- **Cloud dashboard (honeypot.yesnext.com.tr, 2026-07-21):** yeni
  `POST /api/dashboard/whitelist-remove` (dashboard-internal) — whitelist
  IP/subnet satır silme; başarılı silmede agent'a
  `{"v":1,"t":"threat_config_updated"}` WS push yollanır (whitelist-ip ile
  simetrik, v1.3.12 SoT invariantı ile uyumlu). Bloklama sayfası Whitelist
  sekmesi tag-input yerine satır bazlı tabloya çevrildi (ekle/sil anında
  kaydedilir); ana sayfa canlı bloklama mini tablolarına satır aksiyonları
  eklendi (bekleyen: whitelist+sil, uygulanan: whitelist+kaldır, oto:
  whitelist+engel kaldır).

## 1.3.12 — 2026-07-21 (client **4.8.4** — whitelist cloud SoT invariantı)

- `agent/gui-control-center.md`: yeni **whitelist SoT invariantı** — whitelist
  tek kaynağı cloud `threats/config.whitelist_ips`. Persist merge-only (cloud
  seti + yerel engine setleri + açık add/remove deltası; kör overwrite yasak);
  tablo render'ı engine setleri ∪ cloud seti okur.
- Canlı 4.8.3 bulgusu: frontend-only GUI'de engine nesneleri `None` olduğundan
  hızlı whitelist ekleme buluta boş liste yolluyordu ("eklendi" toast'ı ama
  Whitelist (0) ve cloud wipe riski). Production floor 4.8.4.
- **Cloud implemented (honeypot.yesnext.com.tr, 2026-07-21):** dashboard
  production floor 4.8.2 → **4.8.4** güncellendi (sürüm uyarı rozeti + download
  kartı). `POST /api/dashboard/whitelist-ip` başarılı eklemede artık agent'a
  `{"v":1,"t":"threat_config_updated"}` WS push da yollar — agent 60 sn
  cache'ini beklemeden cloud whitelist setini yeniden çeker (SoT invariantı ile
  uyumlu).

## 1.3.11 — 2026-07-21 (client **4.8.3** — dashboard PIN yönetimi + IP hızlı aksiyonları)

- `api/03-control-websocket.md`: yeni komutlar **`set_gui_pin`** (`pin` 4-12
  hane; confirm gate; result/audit'te PIN maskeli — cloud `scrub_command_params`
  listesine `pin` eklenmeli) ve **`clear_gui_pin`** (confirm gate). Cloud TODO:
  `VALID_COMMAND_TYPES` + `DESTRUCTIVE_COMMAND_TYPES` + dashboard "GUI PIN
  tanımla/sıfırla" aksiyonu.
- GUI PIN store dış-değişiklik algılama: daemon `gui_lock.json`'ı yazınca GUI
  süreci mtime'dan yeniden yükler ve oturum kilidini düşürür (restart yok).
- `agent/gui-control-center.md`: hesap bağlıysa PIN diyaloglarında
  "dashboard'dan tanımla/sıfırla" ipucu (`pin_dashboard_hint`).
- IP Listeleri başlığına **＋ IP Engelle** / **＋ Whitelist'e Ekle** hızlı
  aksiyon butonları (modal IP girişi + `ipaddress` doğrulaması + PIN gate;
  satır aksiyonlarıyla aynı IPC/cloud yolu). Production floor 4.8.3.
- **Cloud implemented (honeypot.yesnext.com.tr, 2026-07-21):** `set_gui_pin` /
  `clear_gui_pin` whitelist (42 tip) + destructive confirm gate; `pin` alanı
  `scrub_command_params` ile audit/GET/result'ta `***` (komut kapanınca DB'de de
  maskeli); server-side `pin` format doğrulaması (4-12 hane, yalnız rakam → 400);
  dashboard Tehdit Merkezi → Remote Commands → **GUI PIN** modalı (tanımla /
  sıfırla, confirm dialoglu, TR/EN i18n).
- **Cloud dashboard IP hızlı aksiyonları (2026-07-21):** Bloklama sayfası
  başlığına **IP Engelle** (kırmızı, mevcut `#modalIPBlock`) ve **Whitelist'e
  Ekle** (yeşil, yeni `#modalWhitelistAdd`) butonları eklendi. Whitelist modalı
  IP veya CIDR subnet kabul eder; `POST /api/dashboard/whitelist-ip` artık
  subnet'i de `ipaddress.ip_network(strict=False)` ile doğrulayıp
  `whitelist_subnets` alanına yazar (IP → `whitelist_ips`), geçersiz girişte
  400 `invalid_ip`. Modal "Mevcut IP'mi kullan" hızlı doldurmasını içerir.

## 1.3.10 — 2026-07-21 (client **4.8.2** — Settings config surface documented)

- `agent/threat-engine.md`: `GET /api/threats/config` field list now documents
  the fields the GUI Settings tab writes — email prefs (`alert_email_enabled`,
  `instant_email_for_critical`, `min_severity_for_email`, `daily_digest_enabled`,
  **cloud-consumed**) and the webhook fields.
- Webhook semantics clarified: `webhook_enabled`/`webhook_url` are now written to
  **cloud** threats/config (was local-only ≤4.8.1); the daemon bridges them to
  local `notifications.*` and still forwards alerts client-side. Cloud does not
  send webhooks itself. Production floor raised to 4.8.2.
- **Cloud implemented (honeypot.yesnext.com.tr, 2026-07-21):**
  `POST /api/threats/config` deep-merge accepts GUI Settings fields
  (`alert_email_*`, `instant_email_for_critical`, `min_severity_for_email`,
  `daily_digest_enabled`, `auto_block_*`, `silent_hours{…}`, `webhook_*`) and
  returns effective config + `{ "v":1, "t":"threat_config_updated" }` WS push.
  Cloud consumes email prefs (severity gate + instant critical + daily digest
  tick); webhook is store/return only (no cloud forward). Health/STATUS
  `ransomware_running` + `network_guard{}` mirrored to client_status /
  dashboard-live. Dashboard production floor warn → **4.8.2**. Mirror published.

## 1.3.9 — 2026-07-21 (client **4.8.1** — protection popup data-source fix)

- `agent/gui-control-center.md`: new **detay popup veri-kaynağı invariantı** —
  a card/chip and its detail popup must show the same truth. Frontend-only GUI
  holds no local engine objects (`process_protection`, `ransomware_shield`,
  `network_guard`, `memory_guard` are always `None`), so protection popups must
  read daemon `STATUS` (IPC), never `if local_obj:`. Fixes 4.8.0 "Koruma Motoru
  chip AKTİF but Koruma popup OFF".
- Protection/engine popup now derives state from `STATUS.motor_ok` +
  `STATUS.persistence.self_protection`; local object is a fallback, not the sole
  source. Production floor raised to 4.8.1.

## 1.3.8 — 2026-07-21 (client **4.8.0** — GUI control center v2)

- `agent/gui-control-center.md`: new **Protection Status strip** on the landing
  page (motor / ransomware / network guard / guardian / honeypots / quarantine
  chips fed from daemon `STATUS`, each a shortcut to detail or tab) and new
  **Settings tab** managing email alerts, auto-block limits, silent hours and
  webhook via partial `POST /api/threats/config` deep-merge.
- `api/08-architecture.md`: daemon `STATUS` payload extended with
  `ransomware_running` and `network_guard{present,enabled,running,
  suspended_processes,baseline_age_sec,internet_ok}` so the frontend GUI never
  assumes local engines for layer state.
- Toggle render invariant documented: CTk switches must be enabled **before**
  select/deselect — fixes 4.7.x "shield ACTIVE but layer toggles look OFF".
- Layers/Settings tabs re-sync from cloud on every visit (no stale UI).

## 1.3.7 — 2026-07-21 (client **4.7.6** — daily log retention)

- New `agent/log-retention.md`: date-named client/threat/lifecycle logs,
  7-calendar-day retention and daemon/Guardian rename-race avoidance.
- `update-install.log` keeps its fixed active name for liveness compatibility;
  old dated lines are partitioned into 7-day archives.
- Threat logs now use the canonical APPDATA/ProgramData root instead of cwd.
- Production floor raised to 4.7.6.

## 1.3.6 — 2026-07-21 (client **4.7.5** — clean update/tamper handoff)

- Update authorization lock must remain active until the replacement daemon
  completes its boot-time previous-session check and reports motor ready.
- Fixes planned installer restarts being falsely emitted as
  `unexpected_exit` / `agent_tamper`.
- Production floor raised to 4.7.5.

## 1.3.5 — 2026-07-21 (client **4.7.4** — daemon STATUS IPC health)

- Live 4.7.3 smoke test found a recursive health dependency:
  daemon `STATUS` → persistence summary → `is_motor_healthy()` → daemon `STATUS`.
  The single-thread control server accumulated recursive requests/CLOSE_WAIT and
  GUI/Guardian probes timed out although protection and cloud health continued.
- Client ≥4.7.4 passes local daemon health into persistence summary while serving
  STATUS; it never probes its own STATUS socket. External health callers still
  probe normally. Regression test added.
- Production floor raised to 4.7.4. Old empty quarantine state left by the
  4.7.0/4.7.1 false-positive was cleared on the validated host.

## 1.3.4 — 2026-07-21 (client **4.7.3** — operator-approved containment + GUI)

- Network Guard safety invariant tightened: detection is **always alert-only**;
  cloud config cannot enable automatic suspend/kill/network restore.
- New confirm-gated `suspend_process` and non-destructive `resume_process`.
  Commands require exact identity (`pid`, image/path, `process_start_time`) to
  prevent delayed approval from acting on a reused PID.
- New `agent/gui-control-center.md`: Security Layers tab, immediate
  `POST /api/threats/config` + rollback, `threat_config_updated` WS push,
  canonical card/popup count rules and row-action requirements.
- Client tracked-IP card/detail use the same blocked∪watching snapshot.
- Cloud follow-up: add `_suspect` popup Suspend action, command whitelist/gate,
  threat-config deep-merge/effective response/WS broadcast; `_suspect` must not
  set `under_attack`.
- **Cloud implemented (honeypot.yesnext.com.tr, 2026-07-21):**
  `suspend_process`/`resume_process` whitelist + suspend confirm-gate +
  exact-identity validation (`pid`+`expected_image`+`process_start_time`);
  `POST /api/threats/config` deep-merge → effective config response +
  `{ "v":1, "t":"threat_config_updated" }` control-WS push;
  `ransomware_offline_suspect` warning→high normalize + trigger/pid dedupe +
  dashboard popup **Suspend/Resume** buttons; `under_attack` only on `_bomb`
  (`_suspect`/warning never sets it). Network Guard safe-defaults mirrored
  (`auto_contain/auto_kill/auto_restore=false`, `require_strong_signal=true`).
  v1.3.5/1.3.6 are client-internal (daemon STATUS IPC, update/tamper handoff) —
  no cloud API change required.

## 1.3.3 — 2026-07-21 (client **4.7.2** — KRİTİK güvenlik düzeltmesi)

- **Network Guard güvenli-varsayılan (client ≥4.7.2):** 4.7.0/4.7.1 canlı bir
  geliştirme makinesinde normal ağır-I/O uygulamalarını (Chrome/Firefox/Cursor/
  GameLoop/EdgeWebView) "offline fidye bombası" sanıp **otomatik suspend edip PC'yi
  kilitledi.** Sözleşme buna göre sıkılaştırıldı:
  - **`net_cut` yalnız gerçek internet erişim kaybında** (`internet_lost`) True olur;
    adapter down / VPN-Wi-Fi churn tek başına net_cut sayılmaz. (Kök hata: güncel
    adapter listesi boşken tüm baseline adapterları "down" sanılıyordu.)
  - **Otomatik containment varsayılan KAPALI:** `auto_contain=false`, `auto_kill=false`,
    `auto_restore=false`, `require_strong_signal=true`. Network Guard varsayılanda
    yalnız **alarm** üretir (`ransomware_offline_suspect`, severity **warning**);
    süreç dondurma/ağ değiştirme yapmaz.
  - Otomatik containment yalnız operatör `auto_contain=true` yaptıysa **ve** yüksek
    güvenli imza (aktif canary/VSS quarantine) varsa çalışır; o zaman
    `ransomware_offline_bomb` (critical) üretir. Ham yazma hızı tek başına asla süreç dondurmaz.
  - `threats/config` `protection.network_guard{}` alanları + güvenli defaultlar,
    STATUS/health `network_guard` bloğuna `auto_contain`, 60 sn trigger debounce.
- **Not (cloud'a):** popup builder artık iki tip görmeli — `ransomware_offline_suspect`
  (warning, `recommended_action=review_suspects`, otomatik aksiyon YOK) ve
  `ransomware_offline_bomb` (critical, containment yapıldı). `under_attack` bayrağı
  yalnız `_bomb` (critical) için tetiklenmeli; `_suspect` host'u under_attack yapmamalı.

## 1.3.2 — 2026-07-21

- **Cloud gap-fill (network-guard + survival):**
  - `under_attack` bayrağı: `agent_tamper` / `ransomware_offline_bomb` / canary urgent → `settings`; `client_status` + `dashboard-live` expose; `network_restore`/`unlock_ransomware_quarantine` completed → clear; 24 saat TTL.
  - Health snapshot `persistence` + `network_guard` → settings persist → `client_status`/`dashboard-live`.
  - `GET/POST threats/config`: `protection.network_guard{enabled,auto_restore,auto_kill}` (defaults + update).
  - Dashboard confirm: `create_user` / `remote_logon` / `set_autologon` / `reboot` (+ mevcut `network_restore`).
  - Tunnel cloud acceptance maddeleri kapatıldı (TTL/dedupe/desired).

## 1.3.1 — 2026-07-21

- **Cloud implemented (network-guard / client ≥4.7.0):**
  - `VALID_COMMAND_TYPES` → **38 tip**: `network_snapshot`, `network_restore`, `list_network_baseline`.
  - `network_restore` destructive confirm gate (`confirm:true` yoksa 400).
  - `ransomware_offline_bomb` popup builder: `system_context.network_guard.suspects[]` + `raw_events[].suspect_pid`/`image` → süreç/PID; popup listesinde.
  - 60 sn trigger dedupe (`routes_v4._find_recent_duplicate_urgent`).
  - Dashboard: `network_restored` rozeti + `network_restore` onay dialogu + kritik modal butonu.
  - Canlı doğrulama (client 41): offline-bomb urgent → popup `cursor.exe`/PID + `network_restored=true` + restore_actions dolu.

## 1.3.0 — 2026-07-21 (spec; client **4.7.0** planlı)

- **Yeni:** [`agent/network-guard.md`](agent/network-guard.md) — **offline fidye bombası**
  savunması (fire-and-forget + internetsiz kütle şifreleme). Beş parça:
  **A** imzalı ağ baseline yedeği (mapped drive / shares / adapter / DNS / route / firewall),
  **B** internetsiz davranışsal tespit (ağ-kesme delta + FS yazma/rename fırtınası + fidye notu deseni skorlama; ağ-kesme+FS-fırtınası → canary beklemeden tetik),
  **C** agresif containment **suspend-first** (kill değil; acil VSS snapshot; operatör onayıyla kill/release; opsiyonel auto-kill),
  **D** ağ/bağlantı kurtarma (adapter/DNS/firewall/route/mapped-drive/shares baseline'dan restore → daemon yeniden bağlanır),
  **E** `ransomware_offline_bomb` urgent alarmı (`system_context.network_guard`, `restored` işaretli).
- Yeni komutlar: `network_snapshot`, `network_restore` (confirm), `list_network_baseline`.
- STATUS/health `network_guard{}` bloğu.
- Dürüst sınır: tam EDR/AV değil; davranışsal tespit ayarlanabilir eşik + güvenli
  (suspend-first) varsayılan; garanti = erken containment + kurtarılabilirlik.

## 1.2.0 — 2026-07-21 (client **4.6.0** implemented)

- **Yeni:** [`agent/persistence-and-tamper.md`](agent/persistence-and-tamper.md) — survival modeli:
  Windows Servisi `CloudHoneypotGuardian` (SCM restart-on-failure) + Session 0 motor
  **çapraz watchdog**; durdurma yalnız (1) update-lock (2) imzalı PIN `operator_stop.json`;
  başka her sonlanma → **tamper** → ≤5 sn diriliş + `agent_tamper` urgent (SACL offender PID).
- **Yeni:** [`agent/disaster-recovery.md`](agent/disaster-recovery.md) — felaket kurtarma:
  `create_user` (yeni/yeniden Administrator), `remote_logon` (var olan oturum reconnect →
  yoksa **autologon `AutoLogonCount=1` + reboot** break-glass, boot sonrası LSA secret temizlenir),
  `set_autologon`/`clear_autologon`/`reboot` primitifleri, uçtan uca kurtarma runbook.
- `api/03-control-websocket.md`: komut kataloğuna `create_user`, `remote_logon`,
  `set_autologon`/`clear_autologon`, `reboot`; hepsi HMAC + **destructive confirm** listesinde.
- `api/08-architecture.md`: Guardian servis bacağı + durdurma politikası.
- `agent/attacks-and-services.md`: cloud `pending_tunnel_commands` **TTL/expiry + dedupe**
  sözleşmesi (10 aylık bayat komut bulgusu); client zaten yok sayıyor, `desired` otorite.
- `FLEET.md` / `INDEX.md`: 4.6.0 hedef satırları; production floor **4.5.68** (değişmedi).
- **Cloud implemented (aynı gün):**
  - `VALID_COMMAND_TYPES` → **35 tip**: `create_user`, `remote_logon`, `set_autologon`, `clear_autologon`, `reboot` eklendi.
  - Destructive confirm gate genişletildi: `create_user`/`remote_logon`/`set_autologon`/`reboot` `confirm:true` olmadan 400 (`clear_autologon` non-destructive). Alan doğrulaması: create_user/remote_logon → username+password, set_autologon → username. `remote_logon`/`reboot` TTL 15 dk.
  - Break-glass şifre maskeleme (`helpers.scrub_command_params`): audit log + `GET /api/commands/{id}` → `***`; agent’a giden gerçek şifre `pending`/WS payload’ta; terminal durumda DB’de şifre ezilir.
  - `agent_tamper` popup builder: `system_context.tamper.offender` (+ `raw_events[].offender_pid`/`image`) → popup süreç/PID; `agent_tamper` popup listesinde.
  - `pending_tunnel_commands` TTL (**24 saat**, kontrat yaşam döngüsü bölümü) + servis-başına dedupe (`routes_agent._prune_tunnel_commands`) — tunnel-set (yazma) + tunnel-status (okuma).
  - Opsiyonel dead-man: health `snapshot.persistence` `daemon_ok=false`/`service_ok=false` (operator_stop yok) → sentetik `agent_persistence_degraded` (high/70, 30 dk dedupe). Tamper urgent’e ikincil ağ.
  - Doküman: `agent/threat-engine.md` (tamper + dead-man wire), `agent/attacks-and-services.md` (tunnel TTL/dedupe detay).

## 1.1.7 — 2026-07-21

- **Fix (cloud):** Komut zarfına `type` alias'ı eklendi (= `command_type`) — client `verify_command_signature` tipi `type`/`command` anahtarından okuduğu için alias'sız imza doğrulaması reject ediyordu ("Invalid command signature"). Canlı 4.5.68 ile signed `list_sessions` → completed doğrulandı.
- Threat-intel cloud checklist güncellendi: MVP + ack + WS push maddeleri implemented olarak işaretlendi (ingest bg_worker'da ~30 dk, 3 kaynak senkron, 30 bundle).
- api/03 acceptance: WS push + unknown-command-400 kapatıldı.

## 1.1.6 — 2026-07-21

- Client **4.5.68** hotfix: canary tetiğinde **tek** zengin urgent yolu —
  4.5.67'deki ince `handle_alert` + zengin `send_urgent` yarışı kaldırıldı
  (canlı smoke boş-alanlı payload'ı yakaladı).
- Kural: canary urgent **her zaman** `system_context.ransomware` + `raw_events` +
  `target_service=SYSTEM` + `recommended_action=isolate_host` taşır.
- Fleet production floor → **4.5.68**.
- Canlı doğrulama (DESKTOP-F5SCL3G): canary MODIFIED → quarantine arm →
  urgent 200 `received` (dashboard+email), `RS_UNLOCK` OK.

## 1.1.5 — 2026-07-21

- **Cloud implemented:** HMAC command signing canlı — WS `command` push + `GET /api/commands/pending` artık `issued_at` + `signature` içeriyor (key = raw SHA256 digest; hostname kaynağı: heartbeat `hostname`)
- **Cloud implemented:** Destructive IR server gate canlı — `POST /api/commands/send` `confirm: true` olmadan 400 (`reset_password`, `disable_account`, `disable_all_users`, `enable_lockdown`, `clear_firewall` wipe)
- **Cloud implemented:** `unlock_ransomware_quarantine` whitelist'te (30 tip); dashboard kritik ransomware popup'ına unlock butonu
- **Cloud implemented:** Canary popup detayı `system_context.ransomware` (≥4.5.67 wire) + `raw_events`'ten okunuyor; health-report canary fallback'i 30 dk dedupe ile sentetik alert üretmiyor (Wire kuralına uyum)

## 1.1.4 — 2026-07-21

- Client **4.5.67** implements enriched canary urgent payload:
  `system_context.ransomware`, process-compatible `raw_events`, SYSTEM target/action.
- Health snapshot implements `ransomware_quarantine` with persisted suspect entries.
- Fleet production floor moved to 4.5.67 for detailed cloud canary popup.

## 1.1.3 — 2026-07-21

- `agent/ransomware-shield.md`: **Wire** bölümü — canary alert SoT = `alerts/urgent`
  (score 100 + `Dosya:`/`Değişiklik:` description’da); health snapshot sadece
  `canary_files_intact` / `ransomware_shield_status` / `vss_shadow_count`
- Cloud kuralı: health-report’tan sentetik canary alert üretme (boş popup / dupe)
- ≥4.5.67 hedef şema: urgent `system_context.ransomware` (file, change_type, suspects[pid/cmdline/sha256], quarantine)
- `agent/threat-engine.md`: health/report ransomware alanları not
- Cloud tarafı (aynı gün): HMAC signing + destructive gate + unlock whitelist (detay 1.1.5)

## 1.1.2 — 2026-07-21

- `api/03-control-websocket.md`: HMAC command signing (`security.command_signing`) + result signature
- Destructive IR: dashboard confirmation gate (cloud) documented
- `agent/threat-engine.md`: note optional local `notifications.webhook_url` SIEM forward

## 1.1.1 — 2026-07-21

- `FLEET.md`: feature → min-client matrix; production floor **4.5.66**
- `agent/ransomware-shield.md`: canary UX, quarantine, unlock paths
- `api/08-architecture.md`: `RS_STATUS` / `RS_UNLOCK`, ProgramData, STATUS `rs_quarantine`
- `api/03-control-websocket.md`: `unlock_ransomware_quarantine`; WS intel acceptance closed
- INDEX / CLIENT: ransomware + fleet; sprint WS handler closed
- register-protection: unit acceptance for 3-fail threshold

## 1.1.0

- Mimari / core loop contract’a taşındı (client referans paketi).
- **Yeni:** `agent/attacks-and-services.md`, `agent/threat-engine.md`, `agent/polling.md`, `cloud/overview.md`
- **Yeniden yazıldı (canlı şema):** `api/01-auth.md`, `api/03-control-websocket.md` (29’lu komut kataloğu), `api/06-firewall-blocks.md`
- `agent/CLIENT.md` → indeks; `register-protection` merged with client ≥4.5.66 apply-closed notes
- `INDEX.md` güncellendi

## 1.0.1 — 2026-07-21

- `agent/register-protection.md`: cloud+client apply path closed (client ≥4.5.66); open questions resolved
- INDEX: note client 4.5.66 for register-protection + WS threat_intel_updated

## 1.0.0 — 2026-07-21

- İlk yayın: api/01–09, agent CLIENT / register-protection / remote-input, cloud threat-intel-ingest
