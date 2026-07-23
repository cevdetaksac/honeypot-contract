# Remote Desktop — input path

> **Canonical:** `honeypot-contract/agent/remote-input.md`
> **Min client (input protocol 2 + persistent session helper):** ≥ **4.9.0**
> **Smoothness (move budget / short critical ACK / DC preferred):** ≥ **4.9.20** — see [`../cloud/REMOTE_DESKTOP_SMOOTHNESS.md`](../cloud/REMOTE_DESKTOP_SMOOTHNESS.md)
> **Legacy floor (frame-ACK `inputs[]` piggyback):** ≥ **4.5.55**
> **Related:** [`../api/05-remote-desktop.md`](../api/05-remote-desktop.md) (canonical Remote Desktop v2)

---

## Input delivery (client ≥ 4.9.0)

The client accepts remote input on whichever channel is live, all funneling into
one validator/injector:

1. **WS text** (primary while streaming): `{"t":"input","protocol":2,…}` on
   `/ws/remote/agent`.
2. **WebRTC data channel** (when WebRTC is active): same input-v2 envelope.
3. **Frame-ACK `inputs[]`** (primary while the stream is on HTTP): the response
   to `POST /api/remote/frame` / `POST /api/remote/frame-json` may carry
   `inputs[]`, which the client drains and applies.
4. **`GET /api/remote/inputs`** (compatibility backup drain only): polled
   ~**2.0 s** while WS is healthy, ~**0.30 s** while WS is down. Never the sole
   path.

> **Corrected (was wrong in ≤ v1.3.x):** the client does **not** POST an HTTP
> frame on every WS frame just to drain input. On the healthy path frames go out
> **WS-only**; HTTP frame upload (and its `inputs[]` ACK) is used **only while WS
> is down**. Cloud must not require duplicate per-frame HTTP posts.

### Frame-ACK `inputs[]` (HTTP fallback path)

```http
POST /api/remote/frame-json
→ { "status":"ok", "width":…, "height":…, "inputs":[…], "input_count":N }

POST /api/remote/frame   (multipart, file field "file")
→ same inputs[] shape
```

`inputs[]` example (normalized `x,y ∈ [0,1]`, mapped against the **native**
rectangle honoring `origin_x/y` — see api/05 §2):

```json
[
  {"event":"mousedown","x":0.52,"y":0.31,"button":"left","ts":"…Z"},
  {"event":"mouseup","x":0.52,"y":0.31,"button":"left","ts":"…Z"},
  {"event":"move","x":0.55,"y":0.33,"ts":"…Z"},
  {"event":"wheel","x":0.5,"y":0.5,"deltaY":120,"ts":"…Z"},
  {"event":"key","key":"a","code":"KeyA","ts":"…Z"}
]
```

The cloud drains the queue in the frame ACK; if the agent ignored it, clicks
would be lost. The agent keeps `mousedown`+`mouseup` separate (no synthetic
`click`).

---

## Input protocol 2 — envelope

Protocol 2 wraps the event so each input carries an id for acknowledgement,
while **legacy flat (protocol 1) events keep working unchanged**.

**v2 envelope (WS / data channel):**

```json
{ "t": "input", "protocol": 2, "id": "evt-7", "ts": 12345,
  "input": { "event": "tap", "x": 0.25, "y": 0.5 } }
```

- The inner payload may be under `input` **or** `payload`. `id` and `ts` are
  hoisted from the envelope if not already inside.
- `event` may be given as `event`, `gesture`, `type`, `name` or `action`.
- Event names are normalized: lower-cased, `-` → `_`, and aliases applied
  (`doubletap→double_tap`, `longpress→long_press`, `rightclick→right_click`,
  `dragstart→drag_start`, `dragmove→drag_move`, `dragend→drag_end`,
  `twofingerscroll→two_finger_scroll`, `trackpadmove→trackpad_move`).

**Legacy flat (still accepted):**

```json
{ "t": "input", "event": "mousedown", "x": 0.5, "y": 0.5, "button": "left", "ts": "…Z" }
```

Servers may batch under `inputs[]` or `events[]`; the client coalesces + applies
the batch (see edge ordering below).

### ACK schema (protocol 2)

For every protocol-2 input the client emits a best-effort ACK over the **WS text
channel**:

```json
{ "t": "input_ack", "protocol": 2, "id": "evt-7", "success": true }
```

On failure it adds `"error":"<reason, ≤200 chars>"`. When consecutive moves are
coalesced, **every** original `id` in the fold is still acknowledged (one ACK per
id). Legacy protocol-1 events are not ACK'd.

---

## Semantic events & fields

| `event` | Fields | Injection |
|---------|--------|-----------|
| `tap` | `x,y` | left click (down `0x0002` / up `0x0004`) |
| `double_tap` | `x,y` | two left clicks |
| `long_press` / `right_click` | `x,y` | right click (down `0x0008` / up `0x0010`) |
| `drag_start` | `x,y` **or** `dx,dy`, `button?`, `mode?` | press + (direct: move to `x,y`; relative/trackpad: relative move then press) |
| `drag_move` | `x,y` **or** `dx,dy` | move (mode inherited from drag_start) |
| `drag_end` | `x,y` **or** `dx,dy` | final move + release |
| `two_finger_scroll` / `scroll` | `deltaX`, `deltaY` (or `dx,dy`), `x,y` | vertical→wheel, horizontal→hwheel |
| `wheel` | `deltaX/deltaY` or legacy `delta`/`key`, `x,y` | wheel / hwheel |
| `move` / `mousemove` / `pointer` | `x,y` (or `pointer` `mode:"relative"` `dx,dy`) | absolute cursor (or relative) |
| `move_relative` / `mousemove_relative` / `rmove` / `trackpad_move` | `dx,dy` | **relative** move (SendInput `MOUSEEVENTF_MOVE`) |
| `mousedown` / `mouseup` | `x,y`, `button` | button edge |
| `click` / `dblclick` | `x,y`, `button` | click / double |
| `key` | `key`, `code?` | single char → Unicode `KEYEVENTF_UNICODE`; named (`enter`,`escape`,…) / `ctrl+…` → VK/shortcut |
| `type_text` | `text` | Unicode string, char-by-char |

### Pointer modes: trackpad / direct

- **direct** (default): `x,y ∈ [0,1]` absolute against the native rectangle.
- **relative** / **trackpad**: `dx,dy` device deltas via relative `SendInput`.
  Applies to `pointer` (`mode:"relative"`), `trackpad_move`, and drags started
  with `mode:"relative"|"trackpad"`.

### Two-finger scroll axes

Browser/mobile `deltaX`/`deltaY` are positive down/right, so **both axes are
inverted** to Windows wheel orientation: `two_finger_scroll {deltaX:30, deltaY:50}`
→ vertical wheel `-50`, horizontal wheel `-30`. Legacy `delta`/`key` values are
already Windows-oriented and passed through.

### Relative movement

Relative moves accumulate: a run of `move_relative` (or trackpad) events folds
into a single summed `dx,dy` before injection.

---

## Critical edge ordering, non-drop & rate limits

- **Move budget:** pointer moves (absolute + relative) draw from a dedicated
  budget (**60 moves / 1 s**). Excess moves are rejected (`"move rate limited"`)
  — this **never** blocks other events.
- **Critical edges never dropped:** button/wheel/key/tap/drag edges bypass the
  move budget entirely (tracked for stats only, up to 240/s, never rejected).
- **Coalescing preserves edge ordering:** consecutive absolute moves collapse to
  the **last** position; consecutive relative moves **accumulate** dx/dy; but any
  button/wheel/key/gesture edge **flushes** the pending move first, so the cursor
  is at the correct position at the instant of the edge. Example:
  `move,move,mousedown,move_relative,mouseup` → `move(last),mousedown,move_relative,mouseup`.
- **Anti-stuck:** held buttons and an active drag are force-released on stop /
  helper disconnect, so a drag can never leave a button stuck down.

---

## Privacy — no key/text logging

The client logs **one line per non-move input** for self-check, but **never logs
key or text content**: it records only `event`, `key_present` (bool),
`text_len` (int) and the target `session`. Cloud/dashboard implementations must
likewise avoid persisting keystroke or typed-text content. Moves are not logged.

```
[remote-input] t=input event=key key_present=True text_len=0 session=2
```

---

## Persistent same-session helper (client ≥ 4.9.0)

Capture and input for a **different WTS session** (agent runs as SYSTEM /
Session 0, target is an interactive RDP/console session) go through a
**persistent helper process**, not a per-frame `CreateProcessAsUser`.

- The SYSTEM daemon owns a **loopback TCP listener** and launches **exactly one**
  helper in the selected session (`CreateProcessAsUser`). The helper stays alive
  for the whole stream (persistent), replacing the expensive one-shot-per-frame
  path.
- The channel is a small **length-framed binary protocol** (`magic RDH1`),
  **HMAC-SHA256** authenticated with a random 32-byte per-session secret, with
  strictly **ordered sequence numbers**; only `127.0.0.1`/`::1` peers accepted.
- **Full-duplex message kinds:** `H` hello, `C` config, `F` frame (JPEG +
  geometry: `width/height` encoded, `native_width/native_height`, `origin_x/y`,
  `capture_ms`, `method`), `I` input, `A` input-ack, `E` error, `S` stop.
- **Runtime config** (`fps`, `quality`, `max_width`, `monitor`) is pushed live
  over `C` — including adaptive-controller changes — without relaunching.
- **Input semantics over the helper:** moves are **fire-and-forget** (no wait);
  critical edges use a **short ACK** (≤ 0.2 s) but still count as delivered if
  the ACK is slow, so a slow ACK never turns a real keypress into a failure.
- **Fallback:** if the persistent helper fails to start/probe, the client tries
  the legacy one-shot `CreateProcessAsUser` capture (throttled to ≤ 2 fps). If a
  cross-session helper is unavailable, the client refuses to inject from
  Session 0 (which would hit the wrong desktop) and reports failure honestly.
- **Same-session streams** (agent already in the target session) inject directly
  with `SendInput`/`SetCursorPos` — no helper needed.

---

## CAD (Ctrl+Alt+Del)

A plain `key=ctrl+alt+delete` cannot produce the Secure Attention Sequence. The
dashboard uses `POST /api/remote/cad` → pending `remote_send_sas` command
(`params.session_id`); the agent runs `SendSAS` in an elevated service and posts
an honest `ok` / error result.

---

## Acceptance

1. Dashboard click → remote cursor click within ~1 s (WS or, on HTTP, via frame
   ACK). Drag-and-drop, right-click, wheel work.
2. Protocol-2 input receives a matching `input_ack` (coalesced moves ACK every
   id); legacy flat input still applies.
3. Mobile gestures (tap / double-tap / long-press / drag / two-finger scroll /
   trackpad) map correctly, including negative-origin multi-monitor.
4. Cross-session (Session 0 → RDP) input reaches the target desktop via the
   persistent helper; no Session-0 misfire.
5. No key/text content appears in logs.
6. Cloud queue (`pending_inputs`) stays near 0 during a stream (no duplicate
   per-frame HTTP posts on the healthy WS path).

Min client: **4.9.0** (input protocol 2 + persistent helper). Frame-ACK
`inputs[]` piggyback baseline: **4.5.55**.
