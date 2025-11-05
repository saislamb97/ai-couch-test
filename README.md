# WS Chat — 3D Viseme Debug (ARKIT-15)

A standalone web viewer that connects to your backend at `/ws/chat/`, streams model output, and drives a **Ready Player Me** (or any GLB with ARKit-style morphs) avatar using **ARKIT-15** visemes, perfectly time-synced to server-generated audio.

> This file is a Django template (note the `{% load static %}` tag). You can also serve it as plain HTML—see **Running**.

---

## Features

* **WebSocket client** to `wss://<host>/ws/chat/` (or ws:// over HTTP).
* **3D avatar** powered by Three.js + GLTFLoader + OrbitControls.
* **Morph target auto-detection** for ARKit synonyms (incl. `mouthLeft`/`mouthRight`).
* **Audio + viseme sync**: uses server’s authoritative `viseme_times` timeline; client does only linear interpolation.
* **Emotion-first gating**: buffers text tokens until the first `emotion` event (or a 600 ms timeout) to avoid mismatch between tone & text.
* **Push-to-talk** (PTT) with MediaRecorder (WebM/Opus preferred; MP4/OGG fallbacks).
* **Live controls**:

  * Viseme **gain** (global)
  * Viseme **smoothing** (client-side low-pass)
  * **Jaw gain** (extra gain on `jawOpen`)
* **Debug panels** for text tokens, slides JSON, audio/viseme info, and events.
* **Mute/Stop** controls with matching WS messages.

---

## Running

### Option A — via Django (recommended)

1. Drop `test.html` into a Django templates folder and render it from a view.
2. Make sure `{% load static %}` works and `favicon.ico` is available.
3. Serve your backend and expose **`/ws/chat/`** on the **same host** as the page (for cookies to apply easily).

### Option B — as plain HTML

If you serve this statically (no Django), remove the `{% load static %}` line and change the favicon link to a direct path or remove it.

> **HTTPS required** for microphone (except on `http://localhost`).
> The page loads ESM modules from UNPKG; ensure outbound network access and CSP allowlists if you lock CSP down.

---

## Browser Support

* Latest Chrome/Edge/Firefox (ES modules + `importmap`, `MediaRecorder`, `getUserMedia`).
* Safari works with recent versions, but MediaRecorder/Opus support may differ; the code falls back to other mime types.

---

## UI Overview

### Connect section

* **Agent bot_id**: UUID from your backend.
* **Visitor thread_id**: automatically normalized to `user_<16 hex>` if you paste a bare suffix.
* **Website language**: `en`, `es`, `fr` (passed to WS as `website_language`).
* **API Key**: stored as cookie `api_key` (`SameSite=Lax`; `Secure` on HTTPS).
* **Connect** builds:

  ```
  ws(s)://<current-host>/ws/chat/?bot_id=<uuid>&thread_id=<user_...>&website_language=<en|es|fr>&api_key=<key>
  ```
* Badges show WS, audio mute, and emotion status; **Ping**, **Mute**, **Stop Audio**, **Disconnect** are available when connected.

### Avatar section

* Paste a **GLB URL** (e.g. Ready Player Me). Click **Load Avatar**.
* **Viseme gain**: 0.5×–2.0×
* **Smoothing (lerp)**: 0–0.8 (0.05 default) — increases inertia client-side.
* **Jaw gain**: 0.5×–2.2× (defaults to 1.2×) — extra boost on `jawOpen` only.
* **Test Mouth**: sets `jawOpen`=1 for 600 ms (quick sanity check).

### Send section

* **Text input** → `text_query` on Enter/Send.
* **🎤 Hold to Talk**: press/hold to record; release to send an `audio_query`.

  * If mic is blocked, use **Grant mic** button.
  * Recording uses the best supported mime (`audio/webm;codecs=opus` preferred, then MP4/OGG).
* **Start this run muted**: prompts backend to start muted (also toggles local player).

### Logs

* **Text tokens**: raw `text_token` stream (with emotion-first gating).
* **Slides**: pretty-printed slides JSON (`slides` or `slides_raw`).
* **Audio + visemes**: queue/play info + payload heads; client drains an internal queue.
* **Events**: all other messages & lifecycle notes.

---

## Backend Contract (authoritative)

When the server responds with audio:

```jsonc
{
  "type": "audio_response",
  "audio": "data:audio/mpeg;base64,<base64_mp3>", // or just "<base64>"
  "viseme": [[0.0, 0.1, ... 15 items] , ... N frames],   // ARKIT-15
  "viseme_times": [0.000, 0.033, 0.066, ...],            // seconds, length N
  "viseme_format": "arkit15"                             // string (ignored if not ARKIT-15)
}
```

**Rules:**

* `viseme` and `viseme_times` **must** be same length **N ≥ 2**.
* Times span **0 … duration** (inclusive), matching the MP3’s duration.
* Client **does not** resample or time-stretch; it linearly interpolates between the two nearest frames for the *current audio time*.
* Audio is expected to be **MP3** (the player is set to `audio/mpeg`).

Other messages the page understands:

* `connected`: `{ bot_id, thread_id }`
* `response_start`
* `text_token`: `{ token }` (buffered until first `emotion` or 600 ms)
* `emotion`: `{ emotion: { name, intensity } }`
* `slides_response`: `{ slides: [...] }` **or** `{ slides_raw: {...} }`
* `slides_done`
* `audio_muted`: `{ muted: true|false }`
* `stop_audio`
* `response_done`: `{ timings: {...} }`
* `response_ended`
* `pong`
* `error`: `{ message }`

Client-to-server messages:

* `text_query`: `{ text, local_time, muteAudio }`
* `audio_query`: `{ audio: "data:<mime>;base64,...", format: "webm"|"m4a"|"ogg", muteAudio }`
* `mute_audio` / `unmute_audio`
* `stop_audio`
* `ping`

---

## ARKit-15 mapping (client-side)

The viewer expects **15-element frames** in this order (matching your backend):

```
0  jawOpen
1  mouthFunnel
2  mouthClose
3  mouthPucker
4  mouthSmileLeft
5  mouthSmileRight
6  mouthLeft          // NEW
7  mouthRight         // NEW
8  mouthFrownLeft
9  mouthFrownRight
10 mouthDimpleLeft
11 mouthDimpleRight
12 mouthStretchLeft
13 mouthStretchRight
14 tongueOut
```

The loader scans each mesh’s `morphTargetDictionary` for these synonyms (case-insensitive):

* `jawOpen` → `jawOpen`, `mouthOpen`
* `mouthFunnel` → `mouthFunnel`, `lipsFunnel`
* `mouthPucker` → `mouthPucker`, `lipsPucker`
* `mouthSmileLeft` → `mouthSmileLeft`, `mouthSmile`
* `mouthSmileRight` → `mouthSmileRight`
* `mouthLeft` → `mouthLeft`
* `mouthRight` → `mouthRight`
* …and the rest are exact: `mouthClose`, `mouthFrown{Left,Right}`, `mouthDimple{Left,Right}`, `mouthStretch{Left,Right}`, `tongueOut`.

> If your avatar doesn’t expose these morphs, the page will log
> “**No ARKit morph targets were found on this model.**”

---

## Audio & Viseme Sync

* The MP3 is turned into an object URL and played via `<audio>`.
* A requestAnimationFrame loop reads `audio.currentTime` and finds the enclosing indices in `viseme_times`.
* The two frames are **lerped**; then UI **smoothing** (low-pass) and **gains** (global + jaw) are applied before writing influences.
* When a clip ends, influences reset to zero and the next queued item plays.

---

## PTT (Push-to-Talk)

* On press/hold, the page requests mic permission and starts a `MediaRecorder` at ~64 kbps.
* On release, it sends `{ type: "audio_query", audio: <dataURL>, format: <webm|m4a|ogg> }`.
* If your backend expects bytes, it must handle the data URL form or decode base64.

**Mime order (first supported is used):**

1. `audio/webm;codecs=opus`
2. `audio/webm`
3. `audio/mp4;codecs=mp4a.40.2`
4. `audio/mp4`
5. `audio/ogg;codecs=opus`
6. `audio/ogg`

---

## Security & Cookies

* The **API Key** input is also written to `document.cookie` as `api_key=<value>; SameSite=Lax; Secure?` (Secure on HTTPS).
* Because WS URL uses query params **and** you have a cookie, you can authenticate server-side via either/both.
* Serve the page from the **same origin** as `/ws/chat/` to avoid CORS/cookie surprises.

---

## Troubleshooting

* **Avatar doesn’t move**

  * Check the Events log for “No ARKit morph targets”. Your GLB may use different morph names or omit them.
* **Audio plays but lips don’t**

  * Verify `viseme_times.length === viseme.length` and that times span 0..duration (inclusive). The viewer rejects mismatches.
* **Autoplay error**

  * Browsers may require a user gesture. Click anything first; the code already catches `play()` errors and logs them.
* **Mic blocked**

  * Use **Grant mic**. On HTTPS or `localhost` only. Check browser settings/site permissions.
* **Nothing connects**

  * Confirm the WS URL in Events log matches your server; check that your server accepts `bot_id`, `thread_id`, `website_language`, and `api_key`.
* **Slides panel empty**

  * Ensure the server emits `slides_response` with either `slides` or `slides_raw`.

---

## Quick Server Checklist

Your `/ws/chat/` implementation should:

* Accept `bot_id`, `thread_id`, `website_language`, `api_key`.
* Emit the event types above.
* For audio responses, return **MP3** base64 and aligned **ARKIT-15** frames with exact `viseme_times`.
* Optionally honor `muteAudio` on `text_query`/`audio_query`.
* Support `mute_audio`, `unmute_audio`, and `stop_audio` commands.