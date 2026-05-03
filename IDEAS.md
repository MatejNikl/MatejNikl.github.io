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

### Info / about overlay — impact ★★★ · effort ★★

*Up next.* A small `(i)` pill (or a tap on the title) that opens a
dismissible overlay explaining, in plain language, what
achromatopsia is and what the app does — including what each toggle
simulates and the (deliberate) limits of the simulation (no
contrast-sensitivity loss yet, no nystagmus, etc.). Localised to
EN/CS like the rest of the UI. Target audience (teachers,
therapists, extended family) often does not know the terminology —
this is the difference between "the screen looks weird" and "ah,
*that's* what he sees", which is the whole point of the app. Worth
re-rating from ★★ to ★★★ impact precisely because it's the bridge
between the technical simulator and the people the simulator is
*for*. Pair with the reproducibility URL: the overlay is the natural
place to surface a Share button so a teacher can send the current
view back to a parent.

### Hide controls on inactivity — impact ★★ · effort ★

Auto-fade the pill row and slider after ~3 s of no interaction,
revealing a clean edge-to-edge viewfinder. Any tap (or
`pointermove`) brings them back. Especially valuable when the phone
is held up for sustained viewing.

### Safe-area insets for notched phones — impact ★★ · effort ★

Controls at `top: 10px; right: 10px` can sit under the camera cutout
or Dynamic Island in landscape on recent iPhones / Pixels. Add
`viewport-fit=cover` to the viewport meta and offset controls with
`env(safe-area-inset-*)`. Trivial CSS-only change.

### Retry on camera-error toast — impact ★★ · effort ★

Currently `errCameraFailed` is a static toast and the only escape is
a full reload. Make the toast itself tappable to re-run `start()`,
and (when relevant) show a localized "Tap to retry" suffix. Cheap
polish that prevents a dead-end on transient camera-busy errors,
which are common on Android when another app held the camera.

### Share-current-view button — impact ★★ · effort ★

The reproducibility URL is already kept in `location.hash` on every
change, but nothing surfaces it to the user — they have to know to
copy from the address bar. A small share button (using `navigator.share`
where available, falling back to `navigator.clipboard.writeText`)
turns the existing infrastructure into a feature. Natural home is
the Info / about overlay or next to the gear.

### VA preset chips — impact ★ · effort ★

The Sharpness slider is currently a bare 0.05–0.30 range; the value
0.10 is meaningful (≈20/200, the typical achromat acuity) but
nothing in the UI says so. Three small preset chips next to the
slider (e.g. "20/200 · typical", "20/100 · mild", "20/40 · cone-ish")
would make the control legible to non-clinicians without expanding
the UI much. Could live in the Info overlay rather than the main
viewfinder if space is a concern.

---

## Simulation accuracy

### Nystagmus contribution to acuity (NOT a shaky picture) — impact ★★ · effort ★★

**Counter-intuitive but well established:** people with infantile
(congenital) nystagmus — which includes virtually all complete
achromats — **do not perceive the world as shaking.** The visual
cortex develops alongside the oscillating retinal image and never
forms the perceptual "stable world" reference that acquired-nystagmus
patients lose. The two standard sources both spell this out:

- Straube et al., *Nystagmus and oscillopsia*, Eur. J. Neurol., 2012:
  "Congenital nystagmus is a fixational nystagmus... Typically, the
  patients report **little or no oscillopsia or visual blurring**,
  compared with the fast nystagmus velocities seen."
- Biousse & Newman, *Neuro-ophthalmology Illustrated*, 2nd ed.,
  Thieme, 2012, §16.1.2: "**There is no oscillopsia**, but there is
  decreased visual acuity (related to associated afferent conditions
  and to the nystagmus present in primary gaze)."

So a literal "shake the canvas" toggle would actively misrepresent
the experience. What the nystagmus *does* contribute is reduced
**foveation time** (the fovea is on-target less of the day) plus a
real retinal smear during the slow phases — both of which present
subjectively as further acuity loss, not motion. Two ways to model
that honestly:

- **Anisotropic motion blur along the nystagmus axis.** Achromat
  nystagmus is predominantly horizontal pendular (~2–8 Hz, ~1–10°
  amplitude). A horizontal-only directional blur, layered on top of
  the existing isotropic Gaussian, captures the slow-phase smear
  without implying perceived shake. Implement either as a separable
  shader pass or with `filter: blur()` plus a horizontal-axis CSS
  motion-blur trick.
- **Foveation penalty bump.** Offer a "with nystagmus" preset that
  multiplies the configured VA by ~0.7 to reflect the foveation-time
  cost on top of the cone deficit. Cheap, no shader work.

Whichever route is taken, the UI copy must **not** suggest that the
child sees the world moving — it should explain that nystagmus
further blurs vision rather than destabilising it. Worth pairing
with the Info / about overlay so the explanation lives next to the
control.

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

### Landscape orientation hint — impact ★ · effort ★

The 16:9 stream gets cropped top/bottom by `object-fit: cover` in
portrait. On first visit in portrait, briefly show a hint suggesting
landscape for a wider field of view. Dismissable, locale-aware,
remembered in prefs.

---

## Technical / robustness

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
integer types and built-in `precision highp` defaults (which would
remove the `FRAG_PRECISION` macro dance), and `SRGB8_ALPHA8` as a
core internal format (which would remove the `EXT_sRGB` runtime
extension lookup and the RGBA8 fallback path in `_allocFbo`). Not
urgent — the current code works on every GPU we care about — but
the rewrite would meaningfully shrink the shader-init code path
and the precision/sRGB explainer comments around it.

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
- **Snapshot / freeze-frame** — Freeze pill cancels the rAF loop and
  redraws the held frame on Grayscale / Blur / Sharpness changes.
- **Save photo** — Photo pill uses `canvas.toBlob` and the Web Share
  API (with `<a download>` fallback). The blur is rendered into the
  WebGL drawing buffer so it shows up in the saved PNG.
- **Move blur to a shader pass** — replaced the CSS `filter: blur()`
  with a two-pass separable Gaussian shader. Required prerequisite
  for the photo / video items above.
- **Linear-light blur** — the separable Gaussian now convolves
  linear-light values rather than sRGB-encoded ones, with sRGB ↔
  linear flips at the pipeline bookends. The 8-bit precision concern
  that originally deferred this is solved by allocating the blur FBO
  chain with `EXT_sRGB`-typed storage where supported (every modern
  WebGL 1 device) and falling back to RGBA8 with linear bytes
  otherwise. Documented in CONTEXT.md.
- **Diagnostics overlay (`?debug=1`)** — top-left monospace HUD shows
  canvas vs viewport sizes (with DPR), blur sigma / downsample level /
  per-level sigma, FPS, current ET / ISO, last-measured luminance, and
  active camera. Read once at boot, updates at 1 Hz, English-only.
- **Reproducibility URL** — current settings (grayscale + mode, blur +
  VA, glare, locale) are encoded in `location.hash` and rewritten via
  `history.replaceState` on every change. URL wins over saved prefs at
  load. Camera selection deliberately excluded (deviceId not portable).
