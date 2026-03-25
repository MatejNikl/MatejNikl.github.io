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
- **Glare**: Shown only when `getCapabilities()` reports `exposureMode`
  including `manual` and an `exposureTime` range. **On by default** at the
  shortest ET (the ET floor, ~250 µs), which is enough to show realistic glare
  in normally-lit and bright conditions. Toggles manual exposure mode; if the
  camera exposes `iso` in capabilities, **ISO is fixed at 400** (clamped to
  the hardware min/max) on every `applyConstraints` call — there is no ISO slider.
- **Settings** (gear): Opens a panel below the gear with:
  - **Grayscale type**: Dropdown — **Science-based** (scotopic / rod-weighted
    luminance in the shader) vs **Plain B&W** (Rec. 601) when grayscale is on.
  - **Language**: Dropdown — **System default**, **English**, or **Czech** (see
    **Language** paragraph above).
  - **Shutter**: Logarithmic ET slider, shown when Glare is on. Adjusts the
    manual exposure time from the ET floor (~250 µs) up to the camera's maximum.
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
— setting an explicit manual ET value. The **Glare** control does this via a
direct manual shutter (ET) slider. It cannot emulate the gradual rod saturation
curve, but it lets the user force the camera to over-expose, which is useful for
demonstrating what happens in bright light.

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
blur is simulated with a Gaussian via CSS `filter: blur(σpx)` on the canvas
element.

The blur sigma formula is `(1/VA - 1) * K` where `K = 0.3`. This maps VA 1.0 to
zero blur and increases as VA decreases. The constant K was calibrated considering
that convolved blur appears ~26% more degraded than equivalent natural optical
blur (Artal et al.), so K was reduced from 0.4 to 0.3 to compensate.

### Why CSS filter instead of ctx.filter?

`CanvasRenderingContext2D.filter` is **disabled by default** in Safari/WebKit on
all iOS versions (including the latest 26.x), despite the feature being
implemented. Apple keeps it behind a feature flag. Since all iOS browsers use
WebKit, `ctx.filter` is silently ignored on every iOS browser.

CSS `filter: blur()` on the `<canvas>` HTML element works on all browsers since
Safari 6 (2012). It applies after canvas rendering, which is fine since the blur
doesn't need to feed back into the pixel processing.

## Camera resolution

`getUserMedia` is called with `width: { ideal: 1920 }, height: { ideal: 1080 }`
to request 1080p from the camera. Using `ideal` (not `exact`) ensures the request
never fails — the browser negotiates the closest supported resolution. Without
these constraints, most mobile browsers default to 640×480.

The canvas is sized to `video.videoWidth` × `video.videoHeight` from the active
stream; that size is not displayed on the page.

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
`manual` and a defined `exposureTime` range. It is **on by default** at the
shortest ET (the ET floor), so the simulator shows realistic glare out of the
box in normally-lit environments. A logarithmic **Shutter** slider in
**Settings** lets the user increase the ET for more wash-out.

**Why fixed ISO 400:** It matches the empirical calibration on the Samsung S22
(f/1.8) used to derive `C_emp ≈ 30.86`. It is a sensible mid-gain default for
phone cameras — not universal, but stable and easy to reason about.
**Semi-automatic ISO** (manual shutter while the ISP freely adjusts gain) is not
relied on: hybrid constraint behavior is inconsistent across devices, and
reading real-time ISO from `getSettings()` is unreliable on many Android
browsers (often dummy values), so the app does not try to track or follow auto
ISO in software.

The **Shutter** slider's minimum is an **ET floor** derived so that, at the fixed
ISO above, lengthening exposure from that point loosely aligns with "wash-out
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

ET is clamped to hardware min/max; if `ET_floor` exceeds the hardware maximum,
the slider range collapses toward the long-exposure end as a safety net.

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
