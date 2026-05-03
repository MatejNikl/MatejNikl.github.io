# Achromatopsia Simulator — Design Context

A single-file web app (`index.html`) that uses the phone's back camera to show
approximately what a person with complete achromatopsia (rod monochromacy) sees.

## User interface

**Language:** Under **Settings → Language**, the user can choose **System default**
(follow the browser's preferred languages), **English**, or **Czech**. **System
default** uses the same rule as before: if `navigator.languages` / `navigator.language`
includes any tag whose primary subtag is `cs`, the UI is Czech; otherwise English.
A choice of English or Czech is stored in `localStorage` under the key
`achromatopsia-simulator-locale` so it persists across visits; **System default**
clears that override.

Controls are anchored top-right unless noted.

- **Grayscale**: Toggles the scotopic grayscale shader on or off (full color when off).
- **Blur**: Toggles the acuity simulation. When on, a **Sharpness** slider appears at
  the bottom (still driven by visual acuity internally, e.g. 0.10 ≈ 20/200 — higher
  value means sharper / less blur).
- **Freeze**: Pauses the render loop on the current frame and reveals the
  **Photo** pill (which is otherwise hidden — saving a still only makes sense
  on a held frame the user has chosen to keep). Toggling Grayscale, Grayscale
  type, Blur, or the Sharpness slider while frozen redraws the held frame
  immediately — Glare changes still talk to the camera but do not affect the
  held image. Tap again to resume the live feed; the **Photo** pill hides
  again.
- **Photo**: Visible only while frozen. Captures the current canvas to a PNG
  and offers it via the Web Share API where available, falling back to a
  hidden `<a download>` click. Filename is `achromat-YYYY-MM-DD_HH-MM-SS.png`.
- **Glare**: Shown only when `getCapabilities()` reports `exposureMode`
  including `manual` and an `exposureTime` range. **On by default** with
  **software auto-exposure** enabled — the app measures average frame brightness
  on the GPU and adjusts the manual ET to keep dark scenes properly exposed
  while preserving glare in bright conditions. Toggles manual exposure mode;
  if the camera exposes `iso` in capabilities, **ISO is fixed at 400** (clamped
  to the hardware min/max) on every `applyConstraints` call — there is no ISO
  slider.
- **Fullscreen**: Shown only when the browser reports the Fullscreen API
  as enabled (`document.fullscreenEnabled || document.webkitFullscreenEnabled`).
  Toggles browser fullscreen mode, hiding the URL bar and system navigation
  for a full edge-to-edge viewfinder. Syncs state via `fullscreenchange`
  events, so the button reflects the actual state even when the user exits
  fullscreen via system gesture. iOS Safari does not enable element
  fullscreen on the `<html>` element (it only supports fullscreen on
  `<video>` elements), so the capability check returns false there and the
  pill stays hidden — no explicit iOS UA sniff needed.
- **About** (ⓘ): Opens a modal overlay with a plain-language explanation
  of achromatopsia, what each control simulates, the original motivation,
  and links to authoritative external sources (NIH GARD, Achromatopsia
  Network). Aimed at the people the simulator is *for* — teachers,
  therapists, family — rather than the parent who already knows the
  story; the prose is deliberately short and skips clinical terminology.
  Dismissable three ways: the close (×) button, a tap on the dim
  backdrop, or the Escape key. Focus management restores focus to the
  pill on close so keyboard / screen-reader users don't lose their
  place. The overlay also hosts a **Share this view** button that
  prefers `navigator.share` (so mobile users get the native share
  sheet) and falls back to `navigator.clipboard.writeText` (with the
  button label flipping to "Link copied" for ~1.5 s as inline
  confirmation) — both leverage the reproducibility URL hash that
  `syncURLHash` already keeps current.
- **Settings** (gear): Opens a panel below the gear with:
  - **Grayscale type**: Dropdown — **Science-based** (scotopic / rod-weighted
    luminance in the shader) vs **Plain B&W** (Rec. 601) when grayscale is on.
  - **Language**: Dropdown — **System default**, **English**, or **Czech** (see
    **Language** paragraph above).
  - **Camera**: Dropdown listing rear cameras, only when more than one was
    detected after enumeration.

## Original motivation

The simulator was built by a parent whose young son has achromatopsia. The child
is too young to describe how he sees, so the goal is to approximate his visual
experience for family, teachers, and therapists.

The initial idea was straightforward: grayscale plus photophobia simulation. For
photophobia, the hypothesis was that the camera's auto-exposure could be exploited
— in bright conditions the camera shortens its shutter speed and lowers ISO to
compensate, but rods cannot adapt like that. If the camera were prevented from
lowering its exposure settings below a certain range, bright scenes would
naturally blow out to white, mimicking rod saturation.

## Photophobia via camera exposure — what was tried and why it failed

### Attempt 1: Fixed manual exposure

The first approach locked `exposureTime` and `iso` to high fixed values via
`MediaStreamTrack.applyConstraints({ exposureMode: 'manual', exposureTime: ...,
iso: ... })`. This made everything too bright — even a dimly lit living room
washed out, because real rods *do* adapt within a limited range. A fully fixed
exposure is like a retina that is permanently saturated.

### Attempt 2: Clamped auto-exposure

The next idea was to let the camera auto-adjust within a *restricted* range —
auto-exposure with a floor on ET, so it can't shorten the shutter below a
threshold. Two strategies were tried:

- **Strategy A (range-constrained continuous)**: Set `exposureMode: 'continuous'`
  with `min`/`max` range constraints on ET via `applyConstraints`. The constraint
  was accepted without error on Samsung S22, but the camera silently ignored it.
- **Strategy B (software polling loop)**: Read `track.getSettings()` every 500ms,
  and if the auto-exposure chose an ET below the floor, switch to manual mode and
  clamp it. This failed because `getSettings()` on the S22 (and likely most
  Android Chrome) always returns a static dummy value (`exposureTime: 300`,
  `iso: 0`), regardless of actual lighting conditions. The camera does not expose
  real-time exposure feedback through the web API.

### Outcome

Both strategies depend on reading the camera's *current* exposure parameters,
which `getSettings()` does not reliably provide on Android. The clamped
auto-exposure concept is physiologically sound but technically infeasible with
current web APIs on most Android devices.

What *does* work is `applyConstraints({ exposureMode: 'manual', exposureTime: X })`
— setting an explicit manual ET value. The **Glare** control does this, with
the ET driven by a software auto-exposure controller (see "Software
auto-exposure metering loop" below). It cannot emulate the gradual rod
saturation curve, but the AE controller seeds at the ET floor and only
lengthens ET in dim scenes, so bright scenes naturally over-expose — which is
the wash-out we want.

## Scotopic grayscale conversion

Rod monochromats have no functioning cone cells and rely entirely on rods. Rods
have a spectral sensitivity peaked at ~498-507 nm (blue-green), with near-zero
sensitivity to deep red — the Purkinje shift.

The scotopic coefficients for linear sRGB are **0.007 R + 0.519 G + 0.474 B**,
sourced from Maksimainen, Kurkela, Bhusal, and Hyyppa, "Calculation of Mesopic
Luminance Using per Pixel S/P Ratios Measured with Digital Imaging" (2018). These
are also the coefficients used by the ixora.io ColorBlindness library for rod
monochromacy simulation.

The conversion pipeline:

1. Linearize sRGB values using the standard sRGB transfer function.
2. Compute scotopic luminance: `0.007 * linR + 0.519 * linG + 0.474 * linB`.
3. Re-apply sRGB gamma for display.

All pixel processing runs in a WebGL fragment shader (GPU-accelerated). The video
feed is uploaded as a texture each frame via `gl.texImage2D`. On iOS Safari, where
direct video-to-texture upload may fail (WebKit bugs #133511, #223294, #230617),
the code auto-detects the failure and falls back to drawing the video to an
offscreen 2D canvas first, then uploading that canvas as the texture. This avoids
the expensive `getImageData`/`putImageData` roundtrip that the pre-WebGL version
used.

A Rec. 601 mode (`0.299 R + 0.587 G + 0.114 B` on gamma-encoded values) is
selectable under **Settings → Grayscale type** as **Plain B&W** — this is the standard
"B&W filter" that camera apps typically use. It represents photopic (cone-based)
luminance and is *not* accurate for achromatopsia.

### Why not the earlier coefficients?

An earlier version used `-0.0908 R + 0.7408 G + 0.3500 B`. These have a negative
red weight, which means pure red maps to zero (clamped). The Maksimainen
coefficients avoid negative weights and are better supported by research.

## Visual acuity blur

Complete achromatopsia typically gives visual acuity of ~20/200 (VA 0.10). The
blur is simulated with a Gaussian implemented as a two-pass separable shader
on the GPU.

The blur sigma formula is `(1/VA - 1) * K` where `K = 0.3`. This maps VA 1.0 to
zero blur and increases as VA decreases. The constant K was calibrated considering
that convolved blur appears ~26% more degraded than equivalent natural optical
blur (Artal et al.), so K was reduced from 0.4 to 0.3 to compensate.

K = 0.3 was originally calibrated against `filter: blur(N px)` on the
`<canvas>` element, where N is in CSS-display pixels, so the natural unit of
the formula is display pixels. The shader operates in **internal-canvas
pixels**, and `getBlurSigma()` does the conversion in one step: the canvas is
laid out at `100vw / 100vh` with `object-fit: cover`, so one internal pixel
renders as `max(clientW/w, clientH/h)` display pixels (cover takes the larger
ratio so content fully fills the box); the function divides the display-space
sigma by that scale factor — equivalently multiplies by `min(w/clientW,
h/clientH)` — to keep blur strength visually equivalent across desktop,
tablet, and phone layouts. Without this correction the shader looked 1.5–2x
weaker than the historical CSS reference on typical desktop viewports.

### Why a shader-based separable Gaussian?

The earlier implementation applied `filter: blur()` to the `<canvas>` element
in CSS. That works fine on screen but lives purely in the browser's
compositor — `canvas.toBlob` sees only the WebGL drawing buffer, so any saved
photo would be missing the blur. The blur runs in the GL pipeline so the
captured PNG carries it, and so the output stays consistent across browsers
(CSS `blur()` is implementation-defined, especially at the lower end of the
radius range — two phones could produce visibly different blur for the same
sigma, which is bad for a *simulator*).

The pipeline is **downsample → small Gaussian → bilinear upsample**, the same
strategy Skia uses on the GPU for non-trivial blur radii in `filter: blur()`.
A small fixed-size kernel at unit pixel spacing is a true Gaussian, but its
support (`±~4σ`) caps how large σ can be before the tails are truncated.
Downsampling first lets us keep that same small kernel and still cover any
radius — at half resolution one full-res sigma costs half a kernel pixel.

```
videoTex --(prep LINEARISE: sRGB->linear + flipY)-> srcFbo[0]    full res
srcFbo[0] --(passthrough, LINEAR filter)----------> srcFbo[1]    half res
srcFbo[1] --(passthrough, LINEAR filter)----------> srcFbo[2]    quarter res
srcFbo[L] --(blur H, σ_at_level)------------------> pingFbo[L]   L-level res
pingFbo[L] --(blur V, σ_at_level)-----------------> srcFbo[L]    L-level res
srcFbo[L] --(prep FINISH: gray + linear->sRGB)----> default framebuffer (screen)
```

The pipeline is **linear-light throughout the blur**. The prep
shader has three modes selected by a `uMode` uniform, with an
extra `uInputIsLinear` flag that tells `FINISH` what colour space
its input is in:

- `LINEARISE` — sRGB in, linearise via the IEC 61966-2-1 transfer
  function, write linear. Entry point of the blur pipeline.
- `PASSTHROUGH` — copy with optional flipY, no encoding flips.
  Used between FBOs in the downsample chain so they stay linear.
- `FINISH` — optional grayscale, sRGB out. Used both as the last
  step of the blur pipeline (`uInputIsLinear = true`, sample from
  `srcFbo[L]`) and as the only pass when blur is off
  (`uInputIsLinear = false`, sample directly from the sRGB video
  texture). Grayscale is applied in the reference space of each
  weighting: `scotopic` is colorimetric and operates on linear
  light Y; `plain` is the naive Rec. 601 luma on sRGB-encoded
  values (Y' rather than Y). The shader linearises or
  re-encodes only when the input doesn't already match the
  weighting's reference space, so the blur-off + grayscale-off
  combination collapses to a one-shader straight texture copy.

Doing the convolution on linear values is what makes glare halos
and bright-against-dark transitions look right. Averaging
sRGB-encoded values darkens the average artificially because sRGB
is a perceptual encoding, not a physical-light one. CSS
`filter: blur()` averages sRGB; we don't, and that's the main
reason this is a custom shader rather than a CSS filter.

**FBO storage uses `EXT_sRGB` when available** so the intermediate
buffers hold sRGB-encoded values (perceptually uniform 8-bit
quantisation) while the shader still operates on linear values.
The hardware does the encode/decode at the framebuffer boundary —
shader writes linear, GPU stores sRGB; shader reads, GPU returns
linear. This also makes the bilinear `MIN_FILTER` /
`MAG_FILTER` blends linear-correct (the spec says decoding happens
before filtering on sRGB-typed textures), so the box-filter
downsample and the kernel's bilinear taps are also colorimetrically
right. Without `EXT_sRGB` the FBOs are plain RGBA8 holding linear
values — same algorithm, but smooth dark gradients band more
because linear 8-bit wastes precision in the dark region. The
extension is supported on essentially every WebGL 1 device since
2013.

**Design discipline (taken from Skia):** the blur shader is
*single-resolution*. Both H and V passes have src and dst at the
same L-level resolution, so `vTexCoord` lands on integer texel
centres and the shader never multiplies tiny sub-texel deltas by
large per-fragment coordinates. Resolution changes happen in
separate passthrough passes that have nothing else going on. An
earlier revision tried to fold the final upsample into the V pass —
the V pass would shade at full canvas resolution while sampling from
the L-level H-blurred buffer — under the theory that the bilinear
taps would do high-quality reconstruction "for free." That caused
visible horizontal banding on Mali GPUs because GLSL ES 1.00
defaults `varying`s to whatever default precision the receiving
stage declares, which on most mobile GPUs is mediump (16-bit
half-float, ~10 mantissa bits). When `vTexCoord` lands on integer
texel centres the quantization is invisible; when adjacent fragments
need sub-source-texel deltas, those deltas underflow the half-float
mantissa and a run of adjacent rows samples the same source row.
Skia avoids the whole class of bug by simply never crossing
resolutions inside the blur shader — and does its resize as a
separate generic pass — and we follow suit.

The downsample step is just the prep shader rebound (in
`PASSTHROUGH` mode) to render an FBO into a half-size FBO; with
`MIN_FILTER = LINEAR`, sampling at the centre of every 2×2 source
block averages four pixels for free, which is the box filter we
want to suppress aliasing of any high-frequency detail beyond the
per-level Nyquist limit.

The final upsample is the prep shader bound in `FINISH` mode
(`uInputIsLinear = true`), sampling `srcFbo[L]` with hardware
bilinear `MAG_FILTER` into the default framebuffer. That's a naive
bilinear stretch combined with the grayscale + sRGB-encode work —
cheap and usually fine for `L ≤ 2`. Browsers do something similar;
iOS Safari and Skia both rely on their downsample factor staying
small so a plain bilinear upsample is invisible.

The blur shader (`BLUR_FRAG_SRC`) does a separable Gaussian using
**bilinear-tap sampling**: each off-centre fetch is a bilinear
sample positioned between two adjacent unit-pixel offsets, weighted
by the sum of the two underlying Gaussian weights. The hardware
bilinear filter blends the two adjacent texels in exactly the
proportion the unit-spaced Gaussian would have summed them at, for
the cost of *one* fetch. `MAX_BLUR_TAPS_HALF = 6` paired off-centre
taps reach **12 unit pixels** out from the centre (each side) while
costing only 13 texture fetches per pixel per pass (1 centre + 12
off-centre). That doubles the kernel's reach vs naive unit-spaced
sampling at the same shader cost. This is exactly the algorithm
Skia's `Compute1DBlurLinearKernel` builds (`W' = Wi + Wj`,
`offset = Wj/(Wi+Wj)`); we verified our output is bit-equivalent
(<1e-15 max error vs a naive 25-tap kernel) before relying on it.

Weights and offsets are baked in JS each frame from the per-level
`σ_kernel = σ / 2^L`, where the host picks a **downsample level**
so that `σ_kernel` stays under `sigmaCap = 3 px`. Underlying
Gaussian weights at unit offsets beyond `radius = ceil(3·σ_kernel)`
from the centre are zeroed (Skia's `SkBlurEngine::SigmaToRadius`
rule, capturing ~99.7% of a true Gaussian's mass). If both members
of a pair are zeroed the paired weight is zero too and the offset
is set to the pair's nominal centre to keep the value finite. The
shader always runs the full unrolled 6-iteration loop — the wasted
work for zero-weight pairs is well below the noise floor and
avoiding a dynamic loop bound keeps both the shader source and the
host call site simpler.

**Precision plumbing.** `vTexCoord` is declared `varying highp vec2`
in *both* the vertex shader and every fragment shader that uses it
(belt-and-suspenders), and each fragment shader's default precision
is `highp` if the GPU advertises it via the
`GL_FRAGMENT_PRECISION_HIGH` preprocessor macro — otherwise
mediump. We rely on the macro instead of
`gl.getShaderPrecisionFormat()` because Safari has had a long-
standing bug where the runtime query returns wrong values
regardless of actual hardware support. Devices without
fragment-shader highp will fall back gracefully to mediump; on
those we may see banding when the kernel evaluates near sub-texel
boundaries, but the app still runs.

When **Blur is off** the entire FBO chain is skipped and the prep
shader draws straight to the default framebuffer. That keeps the
blur-off cost identical to the non-blur path.

> **Earlier attempt that didn't work.** Before the downsample
> chain, the blur shader scaled tap *spacing* by σ at full
> resolution (a 9-tap kernel with samples at offsets `±k·σ`). It
> correctly sums to 1, but at large σ the taps land 5–22 pixels
> apart and skip everything between, so high-frequency detail
> leaked through (the blur looked too weak) and the image ghosted
> at offsets of `σ` (visible banding around VA 0.05). A true
> Gaussian needs dense sampling within `±3σ`, which the
> downsample-first approach recovers cheaply.

> **Earlier attempt that didn't work, take two.** A later revision
> folded the final bilinear upsample into the V pass: V would shade
> at full canvas resolution while sampling from `pingFbo[L]`,
> theoretically replacing the naive bilinear MAG_FILTER upsample
> with a high-quality Gaussian reconstruction. That caused
> horizontal banding on Mali (and presumably other mobile GPUs)
> because cross-resolution sampling exposes mediump quantization in
> the varying interpolation, and `precision highp float` in the
> fragment shader alone wasn't enough — the varying needed to be
> declared highp explicitly. Even after fixing the precision, this
> approach diverges from how Skia and the browser's `filter:
> blur()` are structured (their blur shaders are
> single-resolution; resize is a separate pass), so we reverted to
> the same discipline.

### Why not `ctx.filter`?

`CanvasRenderingContext2D.filter` would have been the obvious 2D-canvas
alternative to a shader pass, but it is **disabled by default** in
Safari/WebKit on all iOS versions (including the latest 26.x), despite the
feature being implemented. Apple keeps it behind a feature flag, and since
all iOS browsers use WebKit, `ctx.filter` is silently ignored everywhere on
iOS — ruling it out as a portable alternative.

## Saving photos

The **Photo** pill produces a `Blob` and hands it to a single
`shareBlob(blob, filename)` helper that:

1. Tries `navigator.canShare({ files: [...] })` and, if true, calls
   `navigator.share({ files: [...] })`. User cancellation (`AbortError`) is
   treated as success.
2. Otherwise creates an object URL, programmatically clicks a hidden
   `<a download>`, and revokes the URL on the next tick.

This works on iOS Safari 15+ and Android Chrome (Web Share with files),
desktop Safari/Edge (most cases), and falls back to a download on Firefox
desktop.

### Photo: re-render-then-capture

The WebGL context is created **without** `preserveDrawingBuffer`, so the back
buffer is not guaranteed to survive past the current task — by the time a
click handler fires, the live render loop's last draw may already have been
swapped out. `savePhoto()` therefore calls `renderFrame()` synchronously
before `canvas.toBlob`, which guarantees fresh contents for the encoder
without paying the per-frame copy cost of `preserveDrawingBuffer` for the
99 % of the time when nobody is capturing. The output is PNG (lossless) since
the grayscale + blur combination is dominated by smooth gradients that JPEG
would mangle.

### Visibility-hidden interaction

`suspend()` (called on `visibilitychange` when the page is hidden) cancels the
render loop and tears the camera stream down. The freeze state is left as the
user set it; on `resume()` the next `processFrame()` draws a fresh frame and
the user can re-freeze if they want.

## Camera resolution

`getUserMedia` is called with `width: { ideal: 1920 }, height: { ideal: 1080 }`
to request 1080p from the camera. Using `ideal` (not `exact`) ensures the request
never fails — the browser negotiates the closest supported resolution. Without
these constraints, most mobile browsers default to 640×480.

The canvas is sized to `video.videoWidth` × `video.videoHeight` from the active
stream; that size is not displayed on the page.

## Diagnostics overlay

Adding `?debug=1` to the URL turns on a small monospace HUD at the top
left of the viewport. It is a developer affordance, not a user feature:
no UI exists to toggle it, the strings are English-only, and the flag is
read once at boot (toggling it requires a reload). With the flag off the
overlay is completely inert — no hidden DOM updates, no `setInterval`.

What it shows, terse one line each:

- `canvas WxH / css cwxch (DPR d)` — internal canvas resolution vs the
  CSS layout box the browser is painting it into. The ratio drives the
  cover-scale used by `getBlurSigma`.
- `blur VA=v σ=s.spx L=n (lwxlh) σ@L=k.kk` — only when blur is on.
  Internal-pixel sigma, the chosen downsample level, and the per-level
  blur sigma. Together these explain how aggressively the FBO chain is
  shrinking before the Gaussian, which is the main lever on visible
  upsample-grid artifacts. `blur off` when the toggle is off.
- `fps n.n` — sliding-window FPS measured in `processFrame`. Useful for
  spotting throttling or thermal slowdown during long demos.
- `glare on/off · ET <µs> · ISO <iso>` — only when the camera supports
  manual exposure. Replaces the role the old Shutter slider used to
  serve (showing the current ET).
- `lum <measured> target <target>` — only when Glare is on. Lets you
  sanity-check that the AE controller is converging toward the target.
- `cam <label>` — active camera label. Matters on multi-camera phones
  to confirm the right rear lens was picked.

The HUD updates at ~1 Hz. FPS is computed from a frame counter that
`processFrame` increments unconditionally; the counter is reset every
update so it costs one integer increment per rAF when the flag is off
(measurable as zero).

## Shareable URL hash

Every shareable setting is mirrored into `location.hash` so the URL
itself reproduces the current view. Send a colleague the link, they
open it, they see exactly what you saw. The hash is rewritten on every
change via `history.replaceState` (no history entry per pill toggle)
and is read once on load — manual edits to the hash mid-session do
*not* snap the UI around.

Format is plain `key=value` separated by `&`, kept short:

| Key | Meaning              | Values                          |
|-----|----------------------|---------------------------------|
| `g` | Grayscale            | `1` / `0`                       |
| `gm`| Grayscale type       | `scotopic` / `rec601`           |
| `b` | Blur                 | `1` / `0`                       |
| `va`| Visual acuity        | `0.050`–`0.300`, three decimals |
| `et`| Glare                | `1` / `0`                       |
| `l` | Locale override      | `auto` / `en` / `cs`            |

Precedence at load is **defaults → localStorage → URL** so a shared
link wins over the recipient's saved prefs. Settings applied from the
URL only affect the in-memory state — the localStorage prefs blob is
not touched on load, so closing the link without further interaction
returns the recipient to their saved view on next visit. Toggling a
pill afterwards saves the new state as usual.

The **locale** is the deliberate exception: a URL `l=cs` (or `en`,
`auto`) is written through to `LOCALE_STORAGE_KEY` immediately so the
language dropdown reflects the URL's choice and so subsequent visits
remember it. Locale is the one setting where mid-session change is
disruptive (every UI label re-renders), and a recipient who opens a
Czech link almost certainly wants Czech going forward; if not, they
can change it back.

The camera selection is intentionally *not* in the URL: `deviceId` is
not portable across devices, so encoding it would actively mislead
the recipient. The freeze state is also excluded — it's transient
session state, not a configuration anyone wants to share.

## PWA / Add to Home Screen

A `manifest.json` with `"display": "fullscreen"` allows users to install
the app to their home screen. When launched from there, the browser chrome
(URL bar, navigation) is completely hidden — the app behaves like a native
camera app. The manifest includes 192×192 and 512×512 PNG icons.

`<meta name="mobile-web-app-capable">` and
`<meta name="apple-mobile-web-app-capable">` provide the same "Add to Home
Screen" fullscreen behavior on older Android / iOS versions that predate
full PWA manifest support.

For users who access the app via a normal browser tab, a **Fullscreen**
pill button provides an explicit opt-in to fullscreen mode (see UI section).

## Camera handling

### Multi-camera devices

`facingMode: 'environment'` requests any rear camera, but on multi-camera phones
(like Samsung S22 with main + wide-angle + telephoto) it often picks the
wide-angle lens.

The camera selection logic:

1. After the initial stream is acquired (granting label access), enumerate all
   video input devices.
2. Filter to rear-facing cameras using `InputDeviceInfo.getCapabilities().facingMode`
   (returns `["environment"]` for rear cameras). Falls back to label parsing
   (`/back|environment/i`) where the API isn't available.
3. Sort by camera ID extracted from labels (`/camera\s*(\d+)/i`). On Android,
   camera 0 is the primary rear camera per camera2 API conventions.
4. If the initially selected camera isn't the main one (index 0 after sorting),
   auto-switch to it.
5. When multiple rear cameras are available, **Settings → Camera** shows a
   dropdown to pick which rear device to use.

### Android camera switch race condition

Stopping one camera and immediately calling `getUserMedia` for another fails on
Android Chrome with "Could not start video source" — the hardware hasn't fully
released. Fix: clear `video.srcObject`, wait, then retry up to 3 times with
increasing delays (300ms, 600ms, 900ms).

### iOS Safari frozen frame

iOS Safari requires `muted` on the `<video>` element for autoplay to work with
media streams. Without it, the video freezes on the first frame. Switching to
another camera via **Settings → Camera** can also unfreeze the stream as a side
effect. Explicit `video.play()` calls after setting `srcObject` ensure playback
starts across all browsers.

## Glare (photophobia via manual exposure)

Manual exposure time via `MediaStreamTrack.applyConstraints`,
with **ISO held fixed at 400** when the camera supports setting `iso` (clamped
to `[iso.min, iso.max]` so odd hardware ranges still work). The **Glare**
pill is hidden until `getCapabilities()` reports `exposureMode` including
`manual` and a defined `exposureTime` range. It is **on by default** with
**software auto-exposure** active, seeded at the ET floor and adapting
upward in dim lighting (see subsection below). The pill is the only
control: there is no manual shutter override — Glare on means "fixed ISO
+ AE-driven ET", Glare off means "let the camera handle exposure".

**Why fixed ISO 400:** It matches the empirical calibration on the Samsung S22
(f/1.8) used to derive `C_emp ≈ 30.86`. It is a sensible mid-gain default for
phone cameras — not universal, but stable and easy to reason about.
**Semi-automatic ISO** (manual shutter while the ISP freely adjusts gain) is not
relied on: hybrid constraint behavior is inconsistent across devices, and
reading real-time ISO from `getSettings()` is unreliable on many Android
browsers (often dummy values), so the app does not try to track or follow auto
ISO in software.

The AE controller's lower clamp is the **ET floor**, derived so that, at
the fixed ISO above, exposure at the floor loosely aligns with "wash-out
territory" around **~1000 lux** — the illuminance at which rod saturation
overwhelms an achromat's vision in the model.

The floor uses the incident-light exposure formula `t = N² × C / (E × S)`,
rearranged with fixed `E_overwhelm = 1000` and `S =` fixed ISO:

```
ET_floor [µs] = N² × C_emp × 1e6 / (E_overwhelm × ISO)
```

The standard calibration constant is `C = 250`, but phone camera ISP pipelines
(tone mapping, auto-gain) make the image brighter than a raw sensor reading.
Empirical testing on the Samsung S22 showed that ISO 400 + 250 µs already
produces overwhelm at ~1000 lux, whereas the standard formula predicts ~2025 µs.
Solving backwards gives `C_emp ≈ 30.86`.

The aperture `N` is hardcoded at 1.8 (typical main camera on modern phones).
The Web API does not expose f-number, so per-lens adjustment is not possible.
On multi-camera phones the telephoto may have f/2.4, which would shift the ET
floor — but since we can't detect it, the calibration is slightly off for
non-primary lenses.

If the camera does not expose `iso` in capabilities, constraints send only
`exposureTime`; the ET floor math still assumes **ISO 400** for the same
calibration curve (the device's actual gain is then whatever the ISP applies).

ET is clamped to hardware min/max; if `ET_floor` exceeds the hardware
maximum, the seed and clamp collapse toward the long-exposure end as a
safety net.

### Software auto-exposure metering loop

With the camera fully in manual mode, a fixed ET at the floor (~250 µs) makes
bright scenes wash out correctly but leaves dark rooms severely underexposed.
Reading the camera's actual exposure via `getSettings()` is unreliable (see
"Attempt 2 / Strategy B" above), so the app implements its own auto-exposure
controller by measuring average frame brightness on the GPU.

**Metering pipeline (GPU-side):** After the main render pass, every 10th frame
the raw camera texture (before grayscale conversion) is downsampled through a
chain of framebuffer objects (FBOs). A dedicated GLSL shader averages 16×16
pixel cells per pass, so a
1920×1080 frame reaches 1×1 in ~3 passes (`ceil(1920/16) = 120`, then
`ceil(120/16) = 8`, then `ceil(8/16) = 1`). The final 1×1 pixel is read back
with `gl.readPixels()` — reading a single pixel keeps the GPU pipeline stall
negligible (~1–2 ms). Luminance uses photopic weights
(`0.2126 R + 0.7152 G + 0.0722 B`) since this is metering, not the
achromatopsia simulation.

**Feedback controller:** A proportional controller in log-space compares the
measured average brightness against a target (0.35, roughly mid-gray in
gamma-encoded space). The new ET is computed as:

```
ratio     = target / measured
logNew    = logCurrent + damping × (log(clamp(current × ratio)) - logCurrent)
newET     = clamp(round(exp(logNew)), etFloor, etMax)
```

Log-space interpolation matches the logarithmic nature of exposure (each
doubling of ET doubles brightness). The damping factor (0.15) smooths
convergence to ~1–2 seconds at the ~3 Hz metering rate, preventing
overshoot from the ~400–500 ms round-trip latency (metering interval +
camera HAL delay). A **luminance dead-band** (±0.02 around the target)
suppresses noise-driven oscillation at equilibrium: if the measured
brightness is within 0.02 of the target, the controller does not adjust.
This avoids flicker from chasing sensor noise while never stalling
convergence when the scene actually needs adaptation — far from the
target the error is large and the dead-band is irrelevant.

**Clamping preserves glare:** The ET floor remains the lower bound. In bright
scenes the controller settles at or near the floor, producing the same
wash-out as the original fixed-ET approach. The loop only raises ET above the
floor in dim scenes, preventing underexposure.

## Known issues

- **Manual exposure (Glare pill) is effectively Android Chrome only.** The
  `exposureTime` and `iso` capabilities in `MediaStreamTrack.getCapabilities()`
  are part of the W3C mediacapture-image spec but only implemented in Chrome on
  Android (since Chrome 72). iOS Safari does not support them — all iOS browsers
  use WebKit, which has no `exposureTime` or `iso` implementation. The Glare
  control stays hidden on iOS because capability detection does not find manual
  `exposureTime`. There is no web API workaround for manual shutter speed on iOS.
- iOS Safari (not Chrome) may still show a frozen first frame on initial load.
  Switching cameras via **Settings → Camera** works around this when multiple
  rear cameras exist.
- `InputDeviceInfo.getCapabilities()` is not available in Firefox. The label-based
  fallback handles this.
