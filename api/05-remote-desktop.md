# Remote Desktop

> **Canonical** cloud / dashboard contract for the Windows client Remote
> Desktop feature. This file is the single source of truth; the prompt-sourced
> material below the v2 section is **legacy** and retained for history only.
> API: `https://honeypot.yesnext.com.tr`
> Base client: **≥ 4.5.48** (`list_local_users` / `remote_session_prepare`).
> **Remote Desktop v2: client ≥ 4.9.0** (see next section).
> Companion: [`../agent/remote-input.md`](../agent/remote-input.md) (input path).

---

# ▶ Remote Desktop v2 (client ≥ 4.9.0)

> **Audience:** cloud + dashboard implementers.
> Everything in this delimited section reflects the **actual** client 4.9.0
> implementation (`client_remote_desktop.py`, `client_rd_session_helper.py`,
> `client_rd_adaptive.py`, `client_rd_media.py` and their tests). Fields are
> exact. Where a v2 statement conflicts with the legacy prompt material further
> down, **v2 wins** and the legacy line is marked *(superseded by v2)*.

## v2 at a glance

- **Transports, in priority order:** WebRTC (optional, only if the client
  advertises it) → **JPEG over WebSocket** (healthy default) → JPEG over HTTP
  (fallback). The healthy path is **WS + JPEG only** — a healthy stream does
  **not** also POST every frame over HTTP.
- **Endpoints unchanged.** Still `wss://…/ws/remote/agent` (agent) and
  `wss://…/ws/remote/view` (viewer), plus HTTP `POST /api/remote/frame`,
  `POST /api/remote/frame-json`, `GET /api/remote/inputs`, `POST /api/remote/input`,
  `POST /api/remote/session`, `GET /api/remote/status`, `POST /api/remote/cad`.
  No new endpoint is required for v2. WebRTC signaling is **relayed over the
  existing agent/view WS**. Any genuinely new endpoint is flagged **Cloud TODO**.
- **Two protocol numbers, do not confuse them:**
  - `protocol: 2` → WS **application** envelope (`hello`, `meta`, `input`,
    `input_ack`).
  - `protocol: 1` → **WebRTC signaling** envelope (`webrtc_offer` / answer /
    reject / ice). The client rejects signaling with any other protocol value.
- **Honesty invariants (unchanged from v1):** `streaming:true` is never faked;
  no interactive desktop → `NO_INTERACTIVE_SESSION`; 0×0 / black capture →
  `CAPTURE_NO_DESKTOP`; frames `< 1500 B` are rejected as too small.

## 1. Agent WS `hello` (agent → server, once on connect)

Sent as the first text frame after `wss://…/ws/remote/agent` connects
(Authorization: `Bearer <token>`; legacy `?token=` only if the client has
`api.legacy_token_query` enabled). **Exact, additive schema:**

```json
{
  "t": "hello",
  "role": "agent",
  "protocol": 2,
  "stream_id": "6f1c…hex",
  "capabilities": {
    "input_protocols": [1, 2],
    "input_v2": true,
    "transports": ["jpeg-ws", "jpeg-http"],
    "fallback": "jpeg-ws",
    "codecs": ["jpeg"],
    "webrtc": {
      "available": false,
      "signaling": 1,
      "ice": "non-trickle",
      "ice_server_config": false
    }
  }
}
```

**Truthful capabilities (do not assume more than advertised):**

- `codecs` always begins with `"jpeg"`. Extra codecs (`"h264"`, `"vp8"`) are
  appended **only** when the optional `aiortc`/`av` runtime is actually present
  and a real `RTCPeerConnection` was constructed successfully at probe time.
- `transports` is `["jpeg-ws", "jpeg-http"]` by default; `"webrtc"` is
  **prepended** (→ `["webrtc", "jpeg-ws", "jpeg-http"]`) only when the runtime
  is available.
- `webrtc.available` is `false` on the default build (no aiortc). `webrtc.signaling`
  is always `1`; `webrtc.ice` is always `"non-trickle"`.
- `webrtc.ice_server_config` is `true` **only** when the WebRTC runtime is
  available — it advertises that the client accepts cloud-supplied
  `ice_servers` inside the protocol-1 offer (see §5). On the default build it
  is `false`; a viewer must not attach `ice_servers` to an offer unless this
  flag is `true`.
- `fallback` is always `"jpeg-ws"`.

Cloud **must** gate any WebRTC offer on `capabilities.webrtc.available === true`
**and** `"webrtc" ∈ transports`. Otherwise use JPEG-over-WS.

## 2. Agent WS `meta` (agent → server, before every binary frame)

A `meta` text frame is enqueued **immediately before each binary JPEG** (and at
minimum every 5th frame). It gives the viewer the geometry and adaptive state
for the frame that follows. **Exact schema:**

```json
{
  "t": "meta",
  "protocol": 2,
  "stream_id": "6f1c…hex",
  "capabilities": { "...": "same object as hello" },
  "width": 1280,
  "height": 720,
  "native_width": 1920,
  "native_height": 1080,
  "origin_x": -1920,
  "origin_y": 0,
  "seq": 42,
  "fps": 6.0,
  "quality": 35,
  "max_width": 1280,
  "requested_fps": 8.0,
  "requested_quality": 40,
  "requested_max_width": 1600,
  "capture_mono_ms": 123456,
  "last_send_mono_ms": 123450,
  "session_id": 2,
  "username": "Administrator",
  "media": { "...": "media transport status, see §6" }
}
```

**Dimension semantics (critical for correct pointer mapping):**

- `width` / `height` = **encoded** (downscaled) JPEG dimensions actually sent.
- `native_width` / `native_height` = the **source monitor** resolution.
- `origin_x` / `origin_y` = the captured monitor's **virtual-desktop origin**
  and **can be negative** (secondary monitor left of / above primary).
- Normalized input `x,y ∈ [0,1]` maps to a device pixel as
  `px = origin + x·(native−1)` (see §4). The viewer must send normalized
  coordinates against the **native** rectangle, not the encoded one.

**Adaptive telemetry** exposes both the client-**requested** ceiling
(`requested_fps/quality/max_width`) and the current **effective** values
(`fps/quality/max_width`), so the dashboard can render "8 fps requested → 6 fps
effective" style badges. `capture_mono_ms` / `last_send_mono_ms` are monotonic
millisecond stamps for latency/liveness display.

## 3. Binary frames + latest-frame / coalescing semantics

- Each video frame is a **raw JPEG** binary WS message (`FF D8 … FF D9`),
  preceded by its `meta` text frame.
- The outbound queue keeps **control/meta messages in order** but retains **only
  the newest JPEG**. If a new frame is produced before the previous one is sent,
  the stale frame is dropped (coalesced) and counted in `stats.frames_coalesced`
  — the viewer never receives a backlog of stale frames.
- Frame accounting (`stats.frames_sent`, `bytes_sent`) increments **only on the
  actual socket send**, never on enqueue.
- On (re)connect the client re-sends `hello`, a forced `meta`, and the last good
  frame so the viewer is not blank while waiting for the next capture.

## 4. Healthy path vs HTTP fallback (transport decision)

Per frame the client decides (`_dispatch_frame`):

1. If a WebRTC session is **active**, the frame is published to the media track
   and `transport = "webrtc"`. No WS/HTTP frame is sent for that frame.
2. Else the frame is buffered for the WS thread. If the WS is healthy,
   `transport = "websocket"` and the WS thread performs the send. **No duplicate
   HTTP POST occurs on the healthy path** *(supersedes the legacy "POST HTTP
   frame on every WS frame" guidance below and in the old remote-input.md)*.
3. Else (WS down/unhealthy) the client uploads over HTTP and `transport = "http"`:
   - `POST /api/remote/frame` (multipart, file field **`file`**) **or**
   - `POST /api/remote/frame-json` `{ token, image_base64, width, height, seq }`.
   - The **frame ACK may carry `inputs[]`**, which the client drains and applies
     — this is the **primary input path while on HTTP**.
- `MIN_JPEG_BYTES = 1500`: the client never sends a smaller frame and expects
  the server to keep rejecting `< ~1500 B` ("Frame too small"). Nearly-black
  captures are skipped, not sent.

### Input backup cadence semantics

- **Primary input** is delivered to the agent over the **WS text channel**
  (`{"t":"input",…}`), or via **frame-ACK `inputs[]`** while the stream is on
  HTTP, or over the **WebRTC data channel** when WebRTC is active.
- `GET /api/remote/inputs?token=…&limit=80` is a **compatibility backup drain**
  that runs at **two cadences**: ~**2.0 s** while the WS is healthy (slow, to
  avoid redundant round-trips) and ~**0.30 s** while the WS is down. It is never
  the sole path and is safe to leave enabled.

## 5. WebRTC signaling over the agent/view WS relay (client ≥ 4.9.0, optional)

WebRTC is **optional** and only reachable when the client advertised it in
`hello`. Signaling is **relayed by the cloud over the existing WS endpoints**
(`/ws/remote/view` ⇄ relay ⇄ `/ws/remote/agent`); there is **no dedicated
signaling endpoint**. All signaling envelopes use **`protocol: 1`**.

**The viewer is the offerer; the client only answers.** The client never
initiates an offer.

### Offer (viewer → cloud relay → agent)

```json
{
  "t": "webrtc_offer",
  "protocol": 1,
  "stream_id": "6f1c…hex",
  "session_id": "peer-abc",
  "sdp": "v=0…complete offer SDP…",
  "ice_servers": [
    { "urls": "stun:stun.example.test:3478" },
    {
      "urls": [
        "turn:turn.example.test:3478?transport=udp",
        "turns:turn.example.test:5349?transport=tcp"
      ],
      "username": "short-lived-user",
      "credential": "short-lived-secret"
    }
  ]
}
```

`ice_servers` is **optional** and only valid on the **offer** (never on
`answer`/`ice`); when omitted, the client builds its peer connection with the
aiortc default configuration. The client also accepts
`{"t":"webrtc_signal","action":"offer",…}`. Strict validation before the offer
is handed to the media thread — the client rejects (with `webrtc_reject`)
unless **all** hold:

- `protocol === 1` (missing or `2` → rejected);
- the stream is running and has a `stream_id`;
- `stream_id` **exactly** equals the current stream's id (stale → rejected);
- `session_id` is present; once a peer is bound, later signals with a different
  `session_id` are rejected;
- `action ∈ { offer, answer, ice }`;
- the client's WebRTC runtime is available.

The first valid `offer` binds `session_id` as the media peer.

### Cloud-supplied ICE configuration (`ice_servers`, client ≥ 4.9.0)

The client **consumes** `ice_servers` from the offer and applies it to its
`RTCPeerConnection` (`RTCConfiguration`/`RTCIceServer`). Identity validation
(`protocol`/`stream_id`/`session_id`, §5 above) runs **first**; a stale offer is
rejected before its `ice_servers` are even parsed.

**Exact client validation bounds** (any violation → offer rejected via
`webrtc_reject`; error text never echoes supplied values):

- `ice_servers` must be a **list** of at most **8** server objects.
- Each server object allows **only** the keys `urls`, `username`, `credential`
  — any unknown field rejects the offer.
- `urls` is a single string or a non-empty list of at most **8** URLs;
  each URL is ≤ **512** chars, must match scheme **`stun:` / `turn:` / `turns:`**
  (case-insensitive; anything else, e.g. `https:`, is rejected), must have a
  non-empty endpoint after the scheme, and must contain no whitespace or
  control characters.
- `username` ≤ **256** chars, `credential` ≤ **512** chars (may be empty
  strings); both must be strings without whitespace/control characters.
- If `ice_servers` is absent or `null`/empty, the aiortc **default**
  configuration is used.

**Credential privacy (client-enforced, cloud must match):** validation and
peer-setup errors are deliberately generic — the client never places supplied
URLs, usernames or credentials in error messages, `webrtc_reject` payloads,
logs or status. If the runtime constructor itself fails, the viewer sees only
`"invalid ice server configuration"` / `"peer setup failed"`.

### Answer (agent → cloud relay → viewer)

```json
{
  "t": "webrtc_signal",
  "action": "answer",
  "protocol": 1,
  "session_id": "peer-abc",
  "stream_id": "6f1c…hex",
  "sdp": "v=0…complete answer SDP…",
  "type": "answer",
  "ice": "non-trickle"
}
```

### Reject (agent → cloud relay → viewer)

```json
{
  "t": "webrtc_reject",
  "protocol": 1,
  "stream_id": "6f1c…hex",
  "session_id": "peer-abc",
  "error": "stale or mismatched stream_id"
}
```

### ICE / non-trickle (important)

- Signaling is **non-trickle**: offers and answers contain the **complete SDP
  after ICE gathering**. There are no incremental candidates on the happy path.
- **The client currently does NOT accept standalone trickle ICE.** A standalone
  `ice` signal is validated but rejected with `{"reason":"non_trickle_ice"}`
  (surfaced to the viewer as a `webrtc_reject`). Cloud/dashboard must therefore
  send full-SDP offers/answers and must not rely on trickle ICE.

### Codec preference & data-channel input

- The client prefers **H264** (`setCodecPreferences` H264-first) but retains
  negotiated fallback codecs; the actually-selected codec is reported in
  `media.codec` (see §6).
- Remote input can flow over the **WebRTC data channel** using the **same
  input-v2 envelope** as WS (`{"t":"input","protocol":2,"id":…,"input":{…}}`);
  it is routed through the identical validator.

### JPEG WS fallback from WebRTC

- If the peer connection fails / disconnects / closes (or ICE fails), the client
  automatically falls back: it clears the media session and resumes JPEG frames
  over WS (`transport → "websocket"`, or `"http"` if WS is also down). The viewer
  should present a `<video>` element for WebRTC and seamlessly fall back to the
  JPEG `<img>` renderer.

### Cloud requirements for WebRTC

1. **Viewer-created offer:** dashboard builds the offer; cloud relays it to the
   agent with `protocol:1` + `stream_id` (current) + `session_id` (peer id).
2. **Answer/reject relay:** relay the agent's `answer` / `webrtc_reject`
   verbatim back to the originating viewer, keyed by `stream_id` + `session_id`.
3. **Runtime capability gating:** offer WebRTC **only** when the agent `hello`
   advertised `capabilities.webrtc.available` + `"webrtc" ∈ transports` (and,
   for H264, `"h264" ∈ codecs`). Attach `ice_servers` to the offer only when
   `capabilities.webrtc.ice_server_config` is `true`. Otherwise stay on
   JPEG-over-WS.
4. **Session cleanup / stale rejection:** each `remote_stream_start` creates a
   **new `stream_id`**; cloud must drop peers bound to a previous `stream_id`,
   tear down the RTCPeerConnection on stream stop, and honor client
   `webrtc_reject` for stale `stream_id`/`session_id`.
5. **TURN/STUN configuration delivery & security (client consumer implemented,
   ≥ 4.9.0):** the delivery flow rides the **existing viewer/agent WS relay
   only** — no HTTP endpoint is added:
   1. Cloud issues **short-lived** TURN credentials (e.g. per stream, TTL
      minutes not hours) and sends the ICE config to the **viewer** over the
      existing `/ws/remote/view` connection **before** the viewer creates its
      offer. The viewer-bound message is **cloud-side** (the agent never sees
      or needs it); designated shape:

      ```json
      { "t": "webrtc_ice_config", "protocol": 1, "stream_id": "6f1c…hex",
        "ice_servers": [ { "urls": "…", "username": "…", "credential": "…" } ] }
      ```

   2. The viewer constructs its own `RTCPeerConnection` with those servers
      **and** forwards the **same** `ice_servers` array inside the
      `webrtc_offer` so the agent side uses matching TURN/STUN.
   3. Cloud keeps the forwarded `ice_servers` within the client bounds above
      (≤ 8 servers, ≤ 8 URLs each, `stun|turn|turns` only, length limits) —
      an out-of-bounds config makes the agent reject the whole offer.
   4. **Credential redaction:** cloud must redact `ice_servers[].username`/
      `credential` from request/relay logs, audit trails and any status/debug
      surfaces (mirror of `scrub_command_params` practice); the client already
      guarantees it never echoes them back.
   5. STUN-only configs need no credentials; `ice_servers` stays optional and
      JPEG-WS remains the guaranteed fallback path.

## 6. `media` status object (in `hello`/`meta` and `remote_stream_start` result)

`media` mirrors the transport's live state, e.g.:

```json
{
  "available": true,
  "active": false,
  "connection_state": "new",
  "ice_state": "new",
  "codec": "",
  "preferred_codec": "h264",
  "error": "",
  "mailbox_coalesced": 0,
  "session_id": "peer-abc",
  "stream_id": "6f1c…hex"
}
```

When the runtime is absent, `available:false` and `connection_state:"unavailable"`.

## 7. `remote_stream_start` result (honest, rich)

`remote_stream_start` returns `data = get_status()` — a superset of the legacy
shape. Notable fields: `transport` (`idle|websocket|http|webrtc`), `websocket`
(bool), `requested{fps,quality,max_width}`, `effective{fps,quality,max_width}`,
`screen{x,y,w,h}` (origin + native), `capture{w,h}` (encoded), `session_id`,
`stream_id`, `username`, `monitor`, `capture_method`, `telemetry{…}` (see §8),
`media{…}` (§6), `capabilities{…}` (§1) and `stats{…}`. `screen`/`capture` `0`
still yields `success:false` + `CAPTURE_NO_DESKTOP`.

## 8. Adaptive stream telemetry (requested vs effective)

The client runs a conservative local adaptive controller
(`client_rd_adaptive.py`) bounded by the caller-requested ceilings:

- **Degrade** at most once per **5 s cooldown** under backpressure (coalesced
  frames, slow/failed sends, WS failures): `fps ×0.8`, `quality −5`,
  `max_width ×0.85`, floored at `1 fps / 20 q / 640 px`.
- **Recover** one small step (`fps +0.5`, `quality +2`, `max_width +128`, capped
  at the requested ceiling) only after a **20 s stable window**.
- `telemetry` (also under `get_status().telemetry`) exposes:
  `capture_ms_ewma`, `send_ms_ewma`, `http_ms_ewma`, `capture_samples`,
  `send_samples`, `http_failures`, `ws_failures`, `coalesced_frames`,
  `degrades`, `recovers`, `last_change_mono_ms`, plus `last_capture_mono_ms`,
  `last_send_mono_ms`.

Dashboards should show effective vs requested and a "degraded" badge when
`effective < requested`.

## 9. Cloud TODO / acceptance checklist (hand this to the cloud agent)

Ordered, concrete work items for the cloud + dashboard implementation. Existing
WS/HTTP endpoints stay; only items explicitly marked **(new)** add surface.

1. **Parse the additive `hello`/`meta` v2 fields** (`protocol`, `capabilities`,
   `requested_*`, `native_*`, `origin_*`, `capture_mono_ms`, `media`) without
   breaking legacy viewers. Ignore unknown fields forward-compatibly.
2. **Healthy-path frame handling:** accept WS-only JPEG streams; do **not**
   require or expect a duplicate HTTP frame POST per WS frame. Keep the
   `< 1500 B` "Frame too small" rejection.
3. **Pointer mapping / mobile UX:** map dashboard pointer + touch input to
   normalized `x,y` against the **native** rectangle honoring `origin_x/y`
   (support negative origins / multi-monitor). Implement the mobile pointer UX:
   tap, double-tap, long-press (→ right-click), drag (direct + trackpad/relative),
   two-finger vertical **and** horizontal scroll, and a trackpad mode. Send them
   as **input-v2** envelopes (§ remote-input.md).
4. **Input-v2 envelope + ACK:** send `{"t":"input","protocol":2,"id",…,"input":{…}}`
   over WS (and data channel when WebRTC); consume `{"t":"input_ack","protocol":2,
   "id","success"[,"error"]}`; keep accepting legacy flat protocol-1 input.
5. **WebRTC signaling relay (new plumbing over existing WS):** viewer builds the
   offer; relay `webrtc_offer`/`answer`/`reject` (`protocol:1`) between view and
   agent keyed by `stream_id`+`session_id`; gate on advertised capabilities
   (incl. `webrtc.ice_server_config` before attaching `ice_servers`);
   **do not** rely on trickle ICE (client rejects standalone ICE). Add a
   `<video>` element with automatic fallback to the JPEG renderer.
6. **Session lifecycle:** treat each `remote_stream_start` `stream_id` as
   authoritative; drop/cleanup stale peers and stream state on stop / restart;
   honor `webrtc_reject`.
7. **TURN/STUN delivery (client consumer shipped in 4.9.0 — cloud work is
   actionable now):** stand up/configure TURN+STUN; mint **short-lived**
   per-stream credentials; push `{"t":"webrtc_ice_config","protocol":1,
   "stream_id",…,"ice_servers":[…]}` to the viewer over `/ws/remote/view`
   before offer creation; viewer forwards the identical bounded `ice_servers`
   in the `webrtc_offer`; enforce the client bounds (≤8 servers / ≤8 URLs /
   `stun|turn|turns` / length limits) server-side; **redact credentials** from
   logs, audits and status surfaces. No new HTTP endpoint.
8. **Telemetry / status badges:** surface `transport`, effective-vs-requested
   fps/quality/width, `frames_coalesced`, `degrades/recovers`, `ws_failures`,
   WebRTC `connection_state`/`codec`, and a live "WS / WebRTC / HTTP" transport
   badge on `GET /api/remote/status`.
9. **Browser / mobile test matrix:** verify on desktop Chrome/Edge/Firefox/Safari
   and mobile Safari (iOS) + Chrome (Android): WS-JPEG happy path, WebRTC happy
   path + fallback, drag/scroll gestures, keyboard/Unicode input, negative-origin
   multi-monitor mapping, and reconnect after WS drop.

---

> **Legacy material below** (prompt-sourced, pre-v2). Retained for history. Where
> it disagrees with the v2 section above, the v2 section is authoritative;
> superseded lines are annotated inline.

---

## Akış: kullanıcı seç → prepare → yayın

```
list_local_users  →  list_sessions (can_capture)
        ↓
remote_session_prepare { username, password?, session_id?, timeout_sec }
        ↓ ready_for_stream + session_id
remote_stream_start { session_id, username, … }
        ↓
remote_stream_stop  /  remote_session_logoff
```

### `list_local_users`
`params`: `{ "include_disabled": true }`  
`data.users[]`: username, sid, enabled, is_admin, last_logon, has_session, session_id, session_status — **şifre yok**.

### `list_sessions`
Her oturumda `can_capture: true` yalnızca **Active** + interactive (`session_id > 0`).

### `remote_session_prepare`
Dashboard “Bağlan” — one-shot `password` (RAM only, loglanmaz).

| Durum | Davranış |
|--------|----------|
| Active + desktop | `ready_for_stream: true` |
| Disconnected | `WTSConnectSession` / `tscon` → Active + JPEG probe |
| Oturum yok | `UNSUPPORTED` (Session 0’dan fresh logon yok; bir kez RDP/console gerekir) |
| Yanlış şifre | `AUTH_FAILED` / `ACCOUNT_LOCKED` / `ACCOUNT_DISABLED` |

Başarı: `data.ready_for_stream`, `session_id`, `screen.w/h`, `method`.  
Sonra cloud **aynı `session_id`** ile `remote_stream_start` gönderir.

### Güvenlik
- Password disk/config/autologon’a yazılmaz; history’de `***`.
- Session 0’dan sahte siyah JPEG yok — probe fail → `NO_INTERACTIVE_DESKTOP` / `CAPTURE_NO_DESKTOP`.

---

## Kaynak: `AGENT_REMOTE_DESKTOP_PROMPT.md`

# Agent Prompt: Uzak Masaüstü — Akıcı WebSocket + HTTP fallback

> **Kime:** Windows tray / honeypot client  
> **API:** `https://honeypot.yesnext.com.tr`  
> **Tarih:** 2026-07-18  

## Öncelikli kanal: WebSocket (akıcı)

Dashboard `/ws/remote/view` kullanıyor. Agent **mutlaka** şunu açmalı:

```
wss://honeypot.yesnext.com.tr/ws/remote/agent?token=CLIENT_TOKEN
```

(HTTP test: `ws://...`)

### Agent → Server
1. text: `{"t":"meta","width":1280,"height":720,"seq":12,"fps":5}` — *(superseded
   by v2 §2: `meta` is now sent before **every** binary frame and carries
   `protocol`, `native_*`, `origin_*`, requested/effective and telemetry fields)*
2. binary: ham JPEG bytes (`FF D8 … FF D9`)

Hedef: **5–10 fps**, max genişlik 1280, JPEG quality ~30–40, kare ≤ 200–350 KB.

### Server → Agent (input)
Text JSON:

```json
{"t":"input","event":"mousedown","x":0.42,"y":0.61,"button":"left"}
```

| event | Uygula |
|--------|--------|
| mousedown / move / mouseup | Sürükle-bırak |
| click / dblclick | Tık |
| wheel | `key` = delta |
| type_text / key | Klavye |

`x,y` = 0..1 → ekran pikseli.

Yayın start/stop hâlâ: `remote_stream_start` / `remote_stream_stop` via `GET /api/commands/pending`.

---

## HTTP fallback (WS yoksa)

- Frame: `POST /api/remote/frame` (multipart JPEG)
- Input: `GET /api/remote/inputs?token=...&limit=80` her **200–500 ms**

WS varsa HTTP input poll şart değil.

---

## Acceptance

- [ ] Agent WS bağlanıyor (`{"t":"hello","role":"agent"}`) — *(superseded by v2
  §1: `hello` now carries `protocol:2` + full `capabilities{}`)*
- [ ] Dashboard “WebSocket” rozeti yeşil
- [ ] Sürükle-bırak gecikmesi düşük
- [ ] 5+ fps görünür
- [ ] WS kopunca HTTP fallback çalışıyor

---

## Kaynak: `AGENT_REMOTE_SESSION_SELECT_PROMPT.md`

# Agent Prompt: Remote Desktop — Oturum Seçimi (`session_id`)

> **Kime:** Windows `honeypot-client`  
> **API:** `https://honeypot.yesnext.com.tr`  
> **Tarih:** 2026-07-18  
> **Dashboard:** Oturum dropdown + “oturum yok → RDP öner” UI hazır

---

## Ürün kuralları

| Durum | Dashboard | Agent |
|--------|-----------|--------|
| **0** interaktif oturum | Uyarı: açık masaüstü yok → RDP ile oturum açın. Start **blocked**. | Stream başlatma; `NO_INTERACTIVE_SESSION` |
| **1** oturum | Otomatik seç + start | O `session_id` desktop’ını capture + input |
| **2+** oturum | Dropdown (Console > Active > diğer). Kullanıcı değiştirebilir. | Seçilen `session_id` |

Varsayılan öncelik (API de aynı): **Console Active → Console → Active RDP → ilk kayıt**.

---

## Komut sözleşmesi

### `remote_stream_start` (GET `/api/commands/pending`)

```json
{
  "command_type": "remote_stream_start",
  "params": {
    "fps": 8.0,
    "quality": 32,
    "max_width": 1280,
    "monitor": 0,
    "session_id": 2,
    "username": "Administrator"
  }
}
```

| Alan | Zorunlu | Açıklama |
|------|---------|----------|
| `session_id` | Evet (mümkünse) | WTS session id — **bu session’ın desktop’ını** yakala |
| `username` | Hayır | Doğrulama / log |
| `monitor` | Hayır | Multi-monitor index (0 = primary) |
| `fps` / `quality` / `max_width` | Evet | Capture ayarı |

### `remote_stream_stop`

Önceki gibi; aktif stream’i kapat.

---

## Agent zorunlu davranışlar

### 1) `active_sessions` düzenli raporla

`POST /api/health/report` (veya mevcut health) içinde:

```json
"active_sessions": [
  {
    "username": "Administrator",
    "session_id": 2,
    "session_name": "RDP-Tcp#3",
    "status": "Active",
    "protocol": "RDP",
    "client_ip": "1.2.3.4"
  },
  {
    "username": "User1",
    "session_id": 1,
    "session_name": "Console",
    "status": "Active",
    "protocol": "Console"
  }
]
```

- Kaynak: `WTSEnumerateSessions` / `query user`
- En az **30–60 sn**’de bir güncelle (health zaten gidiyorsa yeterli)
- `Disconnected` oturumları da listele (`status: "Disconnected"`)

Dashboard `GET /api/remote/status` → `sessions[]` buradan beslenir.

### 2) Start gelince doğru session’ı aç

1. `params.session_id` var → **sadece o WTS session** desktop capture
2. Yoksa agent kendi default’unu seç (Console > Active) ve result’ta `session_id` döndür
3. Session listesi boş / id yok →

```json
{
  "success": false,
  "error": "NO_INTERACTIVE_SESSION",
  "message": "No interactive desktop to mirror"
}
```

`streaming: true` yalan söyleme.

### 3) Capture + input aynı session

- JPEG stream: seçilen session’ın WinSta/Desktop’ı
- Mouse/keyboard (`/ws/remote/agent` input veya `/api/remote/inputs`): **aynı session**’a inject
- Session 0 servisten BitBlt yetmez → active session’a inject / helper process

### 4) WebSocket

```
wss://honeypot.yesnext.com.tr/ws/remote/agent?token=CLIENT_TOKEN
```

Start sonrası ≤ 2 sn bağlan; binary JPEG + opsiyonel meta (`width`, `height`, `seq`, `fps`, `session_id`).

### 5) Result dürüst olsun

```json
{
  "success": true,
  "message": "remote stream started",
  "data": {
    "streaming": true,
    "transport": "websocket",
    "session_id": 2,
    "username": "Administrator",
    "screen": {"w": 1920, "h": 1080},
    "capture": {"w": 1280, "h": 720},
    "stats": {"frames_sent": 0, "bytes_sent": 0}
  }
}
```

`screen/capture` 0 → `success: false`, `error: "CAPTURE_NO_DESKTOP"`.

---

## Dashboard API (referans — zaten canlı)

### `GET /api/remote/status?token=…` (DASH_AUTH)

```json
{
  "sessions": [ ... ],
  "session_count": 2,
  "suggested_session_id": 1,
  "diag": "waiting_start | no_interactive_session | live | ..."
}
```

### `POST /api/remote/session`

```json
{
  "token": "...",
  "action": "start",
  "fps": 8,
  "quality": 32,
  "session_id": 2,
  "username": "Administrator"
}
```

- Oturum yoksa: `{ "status": "blocked", "reason": "no_interactive_session" }` — komut kuyruğa **yazılmaz**
- OK: kuyruğa `remote_stream_start` + `params.session_id`

---

## Kabul kriterleri

1. Health’de `active_sessions` dolu → Remote sayfasında dropdown görünür.
2. 0 session → Start disabled / blocked + RDP mesajı; agent’a boş start yağmuru yok.
3. 2 session → kullanıcı SID değiştirince yeni `remote_stream_start` `session_id` ile gelir; görüntü o kullanıcıya geçer.
4. Input tıklamaları seçili session masaüstüne gider.
5. Disconnected seçilirse ya görüntü gelir ya dürüst `CAPTURE_NO_DESKTOP` / uyarı.

---

## Özet

**Mirror = belirli bir WTS `session_id` masaüstü.** Agent `active_sessions` raporlasın; start’ta `session_id` dinlesin; yoksa `NO_INTERACTIVE_SESSION`. Dashboard tarafı hazır.

---

## Kaynak: `AGENT_REMOTE_KEYBOARD_PROMPT.md`

# Agent Prompt: Uzak Masaüstü — Klavye Self-Check (acil)

> **Kime:** Windows tray / honeypot client (remote input)  
> **API:** `https://honeypot.yesnext.com.tr`  
> **Belirti:** Dashboard’da mouse tıklama çalışıyor, “Metin gönderildi” / tuş toast geliyor ama **ekranda yazı yok**.  
> **Tarih:** 2026-07-18  

Dashboard tarafı input’u agent’a iletiyor. Sorun büyük ihtimalle **agent’ın `key` / `type_text` uygulamaması** veya yanlış session’a inject.

---

## 1) Input nereden gelir? (ikisinden biri)

### A) WebSocket (tercih — akıcı)

```
wss://honeypot.yesnext.com.tr/ws/remote/agent?token=CLIENT_TOKEN
```

Server → agent **text** JSON:

```json
{"t":"input","event":"key","key":"a","ts":"…"}
{"t":"input","event":"key","key":"escape","ts":"…"}
{"t":"input","event":"key","key":"enter","ts":"…"}
{"t":"input","event":"mousedown","x":0.5,"y":0.5,"button":"left","ts":"…"}
```

Eski / nadir:

```json
{"t":"input","event":"type_text","text":"hello","ts":"…"}
```

> **Not (2026-07-18):** Dashboard artık yazılabilir metni çoğunlukla **karakter başına `event=key`** olarak gönderiyor. `type_text` gelirse de işle; yoksa en azından `key` şart.

### B) HTTP fallback (agent WS yoksa)

```
GET /api/remote/inputs?token=CLIENT_TOKEN&limit=80
```

Her **200–500 ms** poll. Cevap:

```json
{"events":[{"event":"key","key":"a","ts":"…"}],"count":1}
```

Her event’i uygula, kuyruk sunucuda silinir (drain).

---

## 2) Zorunlu event tablosu

| `event` | Alanlar | Agent ne yapmalı |
|---------|---------|------------------|
| `mousedown` / `move` / `mouseup` | `x,y` 0..1, `button` | Mouse (zaten çalışıyor deniyor) |
| `wheel` | `x,y`, `key`=delta | Scroll |
| **`key`** | **`key` string** | **Klavye — KRİTİK** |
| `type_text` | `text` string | Unicode string yaz (opsiyonel ama destekle) |

### `key` değerleri (case-insensitive)

| `key` | Anlam |
|-------|--------|
| tek karakter (`a`, `A`, `1`, `@`, `ğ`, `İ` …) | O karakteri yaz (Unicode OK) |
| `escape` / `esc` | Esc |
| `enter` / `return` | Enter |
| `tab` | Tab |
| `backspace` | Backspace |
| `delete` / `del` | Delete |
| `space` / ` ` | Space |
| `up` `down` `left` `right` | Oklar |
| `ctrl+c`, `alt+f4`, `shift+tab` … | Modifiers + `+` birleşik |
| `ctrl+alt+delete` | **CAD değil** — sadece key inject; gerçek CAD için aşağıdaki komut |

**Uygulama:** Seçili WTS `session_id` masaüstüne `SendInput` / eşdeğeri.  
Session 0 / service context’ten UI session’a inject etmiyorsan **mouse da gitmezdi** — mouse gidiyorsa aynı inject path’e klavyeyi ekle.

Önerilen: her `key` = keydown + keyup (veya Unicode `KEYEVENTF_UNICODE` çift).

---

## 2b) Türkçe klavye / AltGr / Unicode (sık kırılır)

Dashboard (güncel) tarayıcının **ürettiği karakteri** gönderir:

| Yerel tuş (TR Q) | Gönderilen `key` |
|------------------|------------------|
| ğ ü ş ı ö ç İ | aynı Unicode char |
| AltGr+Q → `@` | `"@"` — **`ctrl+alt+q` değil** |
| AltGr+E → `€` | `"€"` |
| AltGr+7/8/9/0 → `{[ ]}` | o karakterler |
| Ctrl+C | `"ctrl+c"` (gerçek kısayol) |

Opsiyonel alan: `"code":"KeyQ"` (fiziksel tuş).

**Agent kuralı (zorunlu):**

1. `key` uzunluğu **1** ise → **asla** QWERTY scancode map’leme.  
   `SendInput` ile **`KEYEVENTF_UNICODE`** (keydown+keyup) kullan.
2. `key` ∈ `escape|enter|tab|…` veya `ctrl+…` → sanal VK / shortcut.
3. `ctrl+alt+…` gelirse (eski dashboard) ve tek char üretilemiyorsa `code`’a bak; mümkünse Unicode tercih et.
4. Yerel makine TR, remote US (veya tersi) olsa bile: **Unicode inject layout’tan bağımsız çalışır**.

Yanlış: `key="ğ"` → VK_OEM_ something / ToUnicode yanlış layout.  
Doğru: `key="ğ"` → Unicode 0x011F SendInput.

Self-test: Notepad’te `üğişçö@{}€` yazılabilmeli.

---

## 3) CAD (Ctrl+Alt+Del) — ayrı komut

Düz `key=ctrl+alt+delete` Windows’ta **Secure Attention Sequence üretmez**.

Dashboard `POST /api/remote/cad` → pending command:

```json
{
  "command_type": "remote_send_sas",
  "params": { "session_id": 1 }
}
```

Agent:

1. `GET /api/commands/pending` ile al
2. Elevated service: **`SendSAS(0)`** (sas.dll / eşdeğeri)
3. `POST` result: `ok` / hata mesajı

`remote_send_sas` bilmiyorsan CAD butonu **bilerek** çalışmaz — log’a yaz.

---

## 4) Hemen self-check (log + test)

Kodda şunu ekle / doğrula:

```
[remote-input] t=input event=… key=… text=…
```

Her gelen input’ta bir satır.

### Checklist

- [ ] Agent WS bağlı mı? (`hello` aldın mı?)
- [ ] Dashboard’da mouse tıklayınca log’da `mousedown`/`mouseup` görünüyor mu?
- [ ] Masaüstüne focus + klavye `a` → log’da `event=key key=a` görünüyor mu?
- [ ] Görünüyorsa ama ekranda yok → **inject API eksik / yanlış session**
- [ ] Log’da hiç `key` yok → WS text dinlenmiyor veya sadece binary frame işleniyor
- [ ] Sadece HTTP kullanıyorsan `/api/remote/inputs` poll var mı?
- [ ] `remote_stream_start` ile gelen `session_id` input inject’te de kullanılıyor mu?
- [ ] `type_text` gelirse loop ile her char `key` gibi yazılıyor mu?
- [ ] `remote_send_sas` pending’de işleniyor mu?

### 5 dakikalık repro

1. Stream açık, mouse ile Start menüsüne tıkla (çalışıyor olmalı).
2. Arama kutusuna tıkla.
3. Dashboard’da görüntüye tıkla (“Klavye aktif”) → `test123` yaz.
4. Agent log: 7× `key` (`t`,`e`,`s`,`t`,`1`,`2`,`3`).
5. Windows arama kutusunda `test123` görünmeli.
6. Esc butonu → `key=escape` → arama kapanmalı.
7. CAD → pending `remote_send_sas` (SendSAS yoksa dürüst fail).

---

## 5) Sık hatalar

1. **Sadece mouse handler var**, `key`/`type_text` `default`/`ignore`.
2. WS’te yalnızca **binary JPEG** okunuyor; **text JSON input** parse edilmiyor.
3. Input **Session 0**’a gidiyor; capture başka session’dan — tıklar ofsetli/yanlış hedef.
4. `key` adları birebir (`Escape` vs `escape`) — **normalize et (ToLower)**.
5. Toast “gönderildi” = sunucu kuyruğa aldı; agent uygulamadan ekranda bir şey olmaz.

---

## 6) Acceptance (geçti sayılır)

- [ ] Görüntü focus iken canlı yazı remote’ta görünür
- [ ] Esc / Enter toolbar butonları çalışır
- [ ] Yaz (toplu metin) çalışır
- [ ] Log’da her tuş görünür
- [ ] CAD: ya Secure Desktop açılır ya `remote_send_sas` result’ta net hata

**Özet:** Mouse path’inin yanına **aynı session’a `event=key` inject** ekle; WS text + HTTP `/api/remote/inputs` ikisini de dinle; CAD için `SendSAS`.

---

## Kaynak: `AGENT_REMOTE_BLACK_SCREEN_PROMPT.md`

# Agent Prompt: Uzak Masaüstü SİYAH EKRAN — teşhis & fix

> **Kime:** Windows honeypot-client  
> **API:** `https://honeypot.yesnext.com.tr`  
> **Tarih:** 2026-07-18  
> **Kanıt (client_id=36, güncel exe):**

```
remote_stream_start → success
transport: websocket | streaming: true
frames_sent: 0 | bytes_sent: 0
screen: {w:0, h:0} | capture: {w:0, h:0}
```

Dashboard WS viewer bağlı (“Canlı · WS”) ama **hiç gerçek JPEG gelmiyor**.  
Eski çalışan oturumda: `screen 1920×1080`, `capture 1280×720`, `frames_sent: 161`.

---

## Kök neden (büyük ihtimal)

**Ekran yakalama başarısız** — agent “streaming” flag’ini açıyor ama desktop bitmap alamıyor.

Windows’ta klasik sebepler:

| Sebep | Belirti |
|--------|---------|
| **Session 0** (servis olarak çalışıyor, interaktif masaüstü yok) | `screen 0×0`, siyah/boş JPEG |
| Yanlış session / `WTSGetActiveConsoleSessionId` yok | RDP açıkken bile siyah |
| DXGI/BitBlt fail sessizce yutuluyor | `frames_sent` artmıyor veya çok küçük JPEG |
| WS “açık” sanılıyor ama sunucuya binary gitmiyor | API log’da `/ws/remote/agent` yok |
| HTTP fallback’te boş base64 | `frame-json` 200 ama minik/siyah görüntü |

---

## Zorunlu düzeltmeler

### 1) Capture gerçek desktop’tan olmalı
- Agent **interaktif kullanıcı session**’ında capture etsin (RDP/console).
- Servis (Session 0) ise: active session’a inject / helper process (`CreateProcessAsUser` + desktop capture) kullan.
- Start sonrası `screen.w/h` ve `capture.w/h` **> 0** olmalı; 0 ise `commands/result` → **failed** + net hata (`CAPTURE_NO_DESKTOP`), `streaming=true` yalan söyleme.

### 2) WebSocket gerçekten API’ye bağlanmalı
```
wss://honeypot.yesnext.com.tr/ws/remote/agent?token=CLIENT_TOKEN
```
- `hello` mesajını bekle: `{"t":"hello","role":"agent"}`
- Her kare: **binary** ham JPEG (`FF D8 … FF D9`), ideally ≥ 5–20 KB (1280 genişlik, q~30–40)
- Meta (opsiyonel text): `{"t":"meta","width":1280,"height":720,"seq":N,"fps":8}`

API log’da şunu görmeliyiz:
`WebSocket /ws/remote/agent?token=... [accepted]` — kaynak IP = agent host.

Bağlanamazsa **HTTP fallback**:
- `POST /api/remote/frame` multipart field adı **`file`** (JPEG) + Form `token`
- veya `POST /api/remote/frame-json` `{token, image_base64, width, height, seq}`
- Not: `POST /api/remote/frame` without `file` → **400** (eski log’da vardı)

### 3) Boş / siyah kare gönderme
- JPEG **< ~1500 byte** API artık reddediyor (`Frame too small`).
- Capture siyahsa gönderme; fail say + log.
- `frames_sent` / `bytes_sent` artmalı; 10 sn `0` ise stream’i failed işaretle.

### 4) `remote_stream_start` sonucu dürüst olsun
```json
{
  "success": true,
  "message": "remote stream started",
  "data": {
    "streaming": true,
    "transport": "websocket",
    "screen": {"w": 1920, "h": 1080},
    "capture": {"w": 1280, "h": 720},
    "stats": {"frames_sent": 0, "bytes_sent": 0}
  }
}
```
`screen/capture` 0 ise `success: false`, `error: "CAPTURE_NO_DESKTOP"`.

---

## Acceptance

- [ ] `remote_stream_start` sonrası 2 sn içinde API’de agent WS `accepted`
- [ ] `screen.w/h > 0` ve `capture.w/h > 0`
- [ ] 5 sn içinde `frames_sent ≥ 10`, her kare ≥ 5 KB
- [ ] Dashboard’da gerçek masaüstü (siyah değil)
- [ ] Capture fail → result `failed` + `CAPTURE_NO_DESKTOP` (streaming=true yalan yok)

---

## Hızlı debug (agent makinesi)

1. RDP ile oturum açık mı? (Session 0’dan capture deneme)
2. Local test: JPEG’i diske yaz — siyah mı?
3. Wireshark/log: `wss://…/ws/remote/agent` kuruluyor mu?
4. Fallback: tek `frame-json` ile ≥10KB JPEG at → dashboard’da görünmeli

---

## Kaynak: `AGENT_REMOTE_CLIENT_FIX_PROMPT.md`

# Agent Prompt: Uzak Masaüstü + Komut Poll — Client Teşhis & Güncelleme

> **Kime:** Windows `honeypot-client` geliştirme  
> **API:** `https://honeypot.yesnext.com.tr`  
> **Tarih:** 2026-07-18  
> **Öncelik:** 🔴 Kritik — dashboard Remote Desktop sayfası agent poll etmeyince çalışmıyor

---

## Üretim kanıtı (API log + DB)

Dashboard **WIN-CNCMGHODTA4** (`client_id=49`) üzerinde “Yayını Başlat” basıldı.

| Kontrol | Sonuç |
|--------|--------|
| `POST /api/remote/session` → `remote_stream_start` | ✅ Kuyruğa yazıldı |
| Viewer `GET /ws/remote/view?token=705a…` | ✅ Accept edildi |
| Agent `GET /api/commands/pending?token=705a…` | ❌ **Hiç yok** |
| Agent `GET /ws/remote/agent?token=705a…` | ❌ **Hiç yok** |
| Frame `data/remote_frames/49.jpg` | ❌ Yok |
| Komut durumu | `pending` kaldı → `dispatched`/`completed` olmadı |

Karşılaştırma:

| Sunucu | IP | Token (prefix) | Agent | Poll | Remote |
|--------|-----|----------------|-------|------|--------|
| WIN-4VTPQJINJTU | `194.5.236.238` | `0ea8836b-…` | **4.4.38** | ✅ sürekli `commands/pending` | Daha önce frame üretmiş (`36.jpg`); siyah ekran geçmişi var |
| WIN-CNCMGHODTA4 | `194.5.236.239` | `705a7746-…` | **4.4.37** | ❌ poll yok | Stream start stuck |

**Kök neden (49):** Agent bu token ile komut poll etmiyor (servis ölü / yanlış config token / process hang). Dashboard tarafı sağlıklı.

**İkincil risk (36):** Poll var ama geçmişte `screen 0×0` / `frames_sent:0` (Session 0 capture) görüldü — ayrı fix (aşağıda §3).

---

## Client’ta kontrol listesi (sırayla)

### 1) Süreç gerçekten ayakta mı?

Hedef makine: **`194.5.236.239` / WIN-CNCMGHODTA4**

- [ ] `honeypot-client.exe` çalışıyor mu? (Services + Task Manager)
- [ ] Kurulum yolu: `C:\Program Files\YesNext\Cloud Honeypot Client\honeypot-client.exe`
- [ ] Config’teki **token** API’deki ile birebir mi?

```
705a7746-a14e-4bda-911f-6711c6f72785
```

- [ ] Log’da son 5 dk içinde API’ye giden istek var mı?
  - `GET /api/commands/pending`
  - heartbeat / health / attack-count
- [ ] Yoksa: servisi **Restart**, sonra 30 sn izle. Hâlâ yoksa token/config/crash dump.

### 2) Komut poll zorunlu (Remote Desktop’un tetikleyicisi)

```
GET https://honeypot.yesnext.com.tr/api/commands/pending?token=CLIENT_TOKEN
```

**Gereksinimler:**

- [ ] Poll aralığı **≤ 2 sn** (IR / remote için; 10 sn kabul edilmez)
- [ ] `remote_stream_start` / `remote_stream_stop` dispatch edilsin
- [ ] Alındığında hemen `POST /api/commands/result` (`dispatched` → `completed`/`failed`)
- [ ] Aynı anda birden fazla `remote_stream_start` varsa **en sonuncuyu** uygula; eskileri ignore + result `cancelled`/`failed`

Örnek start params:

```json
{
  "command_type": "remote_stream_start",
  "params": { "fps": 8.0, "quality": 32, "max_width": 1280 }
}
```

Start sonrası **2 sn içinde** API log’da şunu görmeliyiz:

```
WebSocket /ws/remote/agent?token=705a7746-… [accepted]
```

kaynak IP = `194.5.236.239`.

### 3) Uzak masaüstü kanalı (WS öncelikli)

```
wss://honeypot.yesnext.com.tr/ws/remote/agent?token=CLIENT_TOKEN
```

- [ ] Bağlan → text `{"t":"hello","role":"agent"}` bekle
- [ ] Her kare: **binary** JPEG (`FF D8…FF D9`), ideally ≥ 5–20 KB
- [ ] Opsiyonel meta text: `{"t":"meta","width":1280,"height":720,"seq":N,"fps":8}`
- [ ] Input text: `{"t":"input","event":"mousedown","x":0.4,"y":0.5,"button":"left"}` → gerçek input uygula

**HTTP fallback** (WS yoksa):

- Frame: `POST /api/remote/frame` multipart field adı **`file`** + Form `token`
- veya `POST /api/remote/frame-json` `{token, image_base64, width, height, seq}`
- Input: `GET /api/remote/inputs?token=…&limit=80` her **200–500 ms**

API reddeder: JPEG **&lt; ~1500 byte** → `400 Frame too small` (siyah stub gönderme).

### 4) Ekran yakalama (siyah ekran / 0×0)

Geçmiş fail (client 36):

```
streaming: true, frames_sent: 0, screen: {w:0,h:0}, capture: {w:0,h:0}
```

- [ ] Capture **interaktif kullanıcı session**’ından (RDP/console) — Session 0 servisten BitBlt/DXGI yetmez
- [ ] Session 0 ise: active session’a helper (`CreateProcessAsUser` + desktop capture)
- [ ] `remote_stream_start` result dürüst olsun:

```json
{
  "success": true,
  "data": {
    "streaming": true,
    "transport": "websocket",
    "screen": {"w": 1920, "h": 1080},
    "capture": {"w": 1280, "h": 720},
    "stats": {"frames_sent": 0, "bytes_sent": 0}
  }
}
```

`screen/capture` 0 ise → `success: false`, `error: "CAPTURE_NO_DESKTOP"` — `streaming=true` yalan söyleme.

- [ ] 10 sn `frames_sent==0` ise stream’i failed işaretle + log

### 5) Sürüm / build

- [ ] **49** makineyi en az **4.4.38+** yap (şu an 4.4.37, poll yok)
- [ ] **36** zaten 4.4.38 — capture fix’i bu build’e veya üstüne koy
- [ ] `settings` / heartbeat ile `agent_version` raporlamaya devam

---

## Kabul kriterleri (QA)

1. **239 (49):** Servis restart sonrası 30 sn içinde API log’da `commands/pending?token=705a…` görünür.
2. Dashboard Remote → Start → komut **≤ 5 sn** içinde `completed` (veya dürüst `failed` + CAPTURE hatası).
3. Aynı pencerede **≤ 5 sn** canlı JPEG (WS veya HTTP).
4. Mouse click dashboard’dan agent masaüstüne yansır.
5. Siyah/0×0 capture’da API’ye mini JPEG gitmez; result `failed`.
6. Agent offline iken dashboard artık “Starting…” diye dönmez — teşhis: `diag=cmds_pending_not_acked` / `agent_offline` (API tarafı hazır).

---

## Hızlı manuel test (client makinede)

```powershell
# Token doğru mu?
$token = "705a7746-a14e-4bda-911f-6711c6f72785"
Invoke-RestMethod "https://honeypot.yesnext.com.tr/api/commands/pending?token=$token"

# Start komutu dashboard'dan basıldıktan sonra burada remote_stream_start görünmeli
# Sonra WS veya:
# curl.exe -F "token=$token" -F "file=@capture.jpg" https://honeypot.yesnext.com.tr/api/remote/frame
```

---

## Referans (mevcut API promptları)

- `AGENT_REMOTE_DESKTOP_PROMPT.md` — WS + HTTP sözleşme  
- `AGENT_REMOTE_BLACK_SCREEN_PROMPT.md` — Session 0 / 0×0 capture  
- Dashboard status teşhisi: `GET /api/remote/status` → `diag`, `pending_stream_cmds`, `agent_ws`, `agent_presence`

---

## Özet (tek cümle)

**49’da agent `commands/pending` poll etmiyor → remote start hiç işlenmiyor; önce token/servis/poll düzelt, sonra WS frame + interaktif session capture’ı doğrula; 36 için siyah ekran (0×0) ayrı capture fix.**

