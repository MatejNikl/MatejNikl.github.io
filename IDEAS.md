# Feature Ideas & Improvements

Open ideas for the Achromatopsia Simulator. Each idea has a rough
**impact** × **effort** rating to help triage. Items already shipped
have been pruned; the remaining list reflects the current state of
`index.html`.

Ratings are a rough hint, not gospel:

- **Impact** — how much it improves the *primary* use case (a parent
  showing teachers / therapists / family what the child sees).
- **Effort** — relative coding work, including capability checks and
  cross-browser quirks.

| Symbol | Meaning |
|--------|---------|
| ★★★    | high    |
| ★★     | medium  |
| ★      | low     |

---

## Recommended next batch

These have the best impact-to-effort ratio and are concrete enough to
pick up immediately.

### Nystagmus simulation — impact ★★★ · effort ★

Almost every complete achromat has involuntary eye oscillations
(nystagmus) that destabilise fixation and degrade effective acuity.
A subtle sinusoidal CSS `translate()` on the canvas (~2–4 Hz, small
amplitude, ~2–4 px) would approximate the instability without any
shader changes — add as a toggle pill alongside Grayscale/Blur. This
is the single most-requested missing piece of feedback from people
who have been shown the current simulator: "is it really this
*still*?" Respect `prefers-reduced-motion`.

### Hide controls on inactivity — impact ★★ · effort ★

Auto-fade the pill row and slider after ~3 s of no interaction,
revealing a clean edge-to-edge viewfinder. Any tap (or
`pointermove`) brings them back. Especially valuable when the phone
is held up for sustained viewing or recording.

### Safe-area insets for notched phones — impact ★★ · effort ★

Controls at `top: 10px; right: 10px` can sit under the camera cutout
or Dynamic Island in landscape on recent iPhones / Pixels. Add
`viewport-fit=cover` to the viewport meta and offset controls with
`env(safe-area-inset-*)`. Trivial CSS-only change.

### Snapshot / freeze-frame — impact ★★★ · effort ★★

Tap a "Freeze" pill (or the canvas itself) to stop the rAF loop and
hold the current frame. Lets the demonstrator say "look at *this*
specific scene" without scene motion confusing the comparison. The
existing `processFrame()` already cancels its prior `rafId` on
re-entry, so wiring a paused state is mostly UI work. Pair with a
"Save image" affordance via `canvas.toBlob()` for a free shareable
still.

### Info / about overlay — impact ★★ · effort ★★

A small `(i)` pill (or a tap on the title) that opens a dismissible
overlay explaining, in plain language, what achromatopsia is and what
each toggle simulates. Localised to EN/CS like the rest of the UI.
Target audience (teachers, therapists, extended family) often does
not know the terminology — this is the difference between "the screen
looks weird" and "ah, *that's* what he sees".

### Retry on camera-error toast — impact ★★ · effort ★

Currently `errCameraFailed` is a static toast and the only escape is
a full reload. Make the toast itself tappable to re-run `start()`,
and (when relevant) show a localized "Tap to retry" suffix. Cheap
polish that prevents a dead-end on transient camera-busy errors,
which are common on Android when another app held the camera.

---

## Simulation accuracy

### Contrast sensitivity loss — impact ★★★ · effort ★★

Achromats have reduced contrast sensitivity, not just reduced acuity.
The current Gaussian blur captures acuity loss but not the washed-out
contrast perception. Add a fragment-shader contrast reduction (pull
pixels toward mid-gray by a tunable factor) inside the existing
grayscale shader path so it's free per-pixel. Could either ride along
the Blur toggle or sit as its own slider in **Settings**. This is
arguably the most physiologically meaningful gap left in the
simulation.

### Scotopic adaptation latency — impact ★★ · effort ★

The software auto-exposure now in place (commit `c6df039`) makes this
easy: when the measured luminance drops sharply (i.e. moving from a
bright outdoor scene into a dim room), lengthen `AE_DAMPING` for a
few seconds so the glare wash-out *lingers* the way real rod
adaptation does. Asymmetric damping — fast on darken, slow on
brighten — would make abrupt scene changes feel much more authentic.

### White-balance lock — impact ★ · effort ★★

Phone cameras run continuous WB by default; under strong tinted
lighting (sodium street lamps, warm tungsten) the WB shift muddies
the rod-weighted grayscale. If `whiteBalanceMode: 'continuous'` /
`'manual'` are reported in `getCapabilities()`, expose a small toggle
or default to a fixed daylight-ish preset. Capability-gated like the
Glare pill. Probably only matters for power users.

---

## UX & usability

### Pinch-to-zoom (with telephoto auto-switch) — impact ★★★ · effort ★★

Achromats habitually move very close to objects to compensate for
poor acuity, which is hard to demonstrate when the demonstrator is
holding a phone. A two-finger pinch on the canvas (CSS
`transform: scale()` is enough) would let the user lean in
virtually. On multi-camera phones, swap to the telephoto lens once
zoom exceeds ~2× — the existing camera switching plumbing already
handles the constraint dance.

### Torch / flashlight toggle — impact ★★ · effort ★

`applyConstraints({ advanced: [{ torch: true }] })` works on Android
Chrome. In dim indoor environments (classrooms, hospital rooms),
turning the flashlight on lets people see the **Glare** wash-out
without having to walk outside. Capability-detect with
`getCapabilities().torch` and reuse the same hidden-pill pattern as
Glare.

### Reproducibility URL — impact ★ · effort ★

Encode the current settings in `location.hash`
(e.g. `#gray=on&blur=0.10&et=auto&grayMode=scotopic`) so a colleague
can be sent a link that opens in exactly the same configuration.
Already trivial given the existing `persistPrefs()` shape — read on
load, write on change.

### Landscape orientation hint — impact ★ · effort ★

The 16:9 stream gets cropped top/bottom by `object-fit: cover` in
portrait. On first visit in portrait, briefly show a hint suggesting
landscape for a wider field of view. Dismissable, locale-aware,
remembered in prefs.

### Diagnostics overlay (`?debug=1`) — impact ★ · effort ★★

A small monospace HUD showing measured luminance, current ET, ISO
(when available), camera resolution, and FPS, gated behind a
URL parameter. Invaluable for calibrating `C_EMP` on a new device
without rebuilding, and for anyone debugging AE behaviour in the
field. Zero impact when the param isn't set.

---

## Technical / robustness

### Video recording / export — impact ★★ · effort ★★

`MediaRecorder` on `canvas.captureStream()` can produce a short clip
of the simulated view, which the parent can share with somebody who
isn't physically present ("this is what he sees in your
classroom"). Needs care to size the recording to a reasonable
duration/bitrate and to expose a clean download or Web Share Target
fallback.

### `display_override` and richer manifest — impact ★ · effort ★

Add `"display_override": ["fullscreen", "standalone", "minimum-ui"]`
to `manifest.json` so newer browsers fall back gracefully when
`fullscreen` isn't honoured. Also worth adding `categories`,
`screenshots`, and a proper `description` for richer install UIs.

### Persist camera by `groupId` rather than raw `deviceId` — impact ★ · effort ★★

`deviceId` is regenerated across sessions on some Android builds,
which silently invalidates the saved "preferred camera" pref.
`groupId` is more stable. Worth investigating whether matching on
`(groupId, label)` survives reboots better, then falling back to
`deviceId`. Low priority unless users actually complain.

### iOS frozen-first-frame on cold start — impact ★★ · effort ★★

Documented in CONTEXT.md as having no workaround. Worth one more
investigation pass: a delayed second `video.play()` call, a
post-attach `track.applyConstraints({})`, or briefly toggling
`srcObject` may unstick it without the user having to dive into
**Settings → Camera**. If a reliable trigger is found, this removes
the most common iOS first-impression papercut.

### WebGL2 upgrade path — impact ★ · effort ★★

The app uses WebGL 1. WebGL 2 is now everywhere WebGL 1 is (Android
Chrome forever, iOS Safari 15+). Benefits: `texStorage2D` for
immutable textures, cleaner FBO setup, GLSL 300 es with proper
integer types. Not urgent — the current code works — but would let
the downsample pipeline drop a couple of guards.

### Battery / thermal-aware metering — impact ★ · effort ★★

When the device reports low battery (`navigator.getBattery()`) or
the page is partially throttled, increase `METER_EVERY_N` from 10 to
~30 frames and consider lowering the requested resolution. Tiny
power win for a phone held up for a long demo.

---

## Pruned (already implemented)

For the historical record, these earlier ideas have shipped:

- **PWA / Add to Home Screen** — `manifest.json` with icons,
  `display: fullscreen`, and theme colors is in place.
- **Fullscreen re-entry** — replaced by an explicit Fullscreen pill
  whose state syncs via `fullscreenchange`.
- **Accessibility: ARIA on pills** — pills are now real `<button>`
  elements with `aria-pressed`, `aria-expanded`, locale-aware
  `aria-label` on the gear, and a `:focus-visible` outline.
