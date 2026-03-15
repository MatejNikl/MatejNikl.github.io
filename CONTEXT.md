# Achromatopsia Simulator — Design Context

A single-file web app (`index.html`) that uses the phone's back camera to show
approximately what a person with complete achromatopsia (rod monochromacy) sees.

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
— setting an explicit manual ET value. This is what the current E button does: a
direct manual ET slider. It cannot emulate the gradual rod saturation curve, but
it lets the user force the camera to over-expose, which is useful for
demonstrating what happens in bright light.

The sigmoid wash-out (L button) was added as a software fallback for photophobia
simulation — it is device-independent and always works, though it operates on
pixel brightness rather than actual scene illuminance.

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
available for comparison — this is the standard "B&W filter" that camera apps
typically use. It represents photopic (cone-based) luminance and is *not*
accurate for achromatopsia.

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

The actual resolution is shown in the info overlay (e.g. `1080×1920`).

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
5. A cycle button (C) appears when multiple rear cameras are available.

### Android camera switch race condition

Stopping one camera and immediately calling `getUserMedia` for another fails on
Android Chrome with "Could not start video source" — the hardware hasn't fully
released. Fix: clear `video.srcObject`, wait, then retry up to 3 times with
increasing delays (300ms, 600ms, 900ms).

### iOS Safari frozen frame

iOS Safari requires `muted` on the `<video>` element for autoplay to work with
media streams. Without it, the video freezes on the first frame. Cycling cameras
also unfreezes the stream as a side effect. Explicit `video.play()` calls after
setting `srcObject` ensure playback starts across all browsers.

## Other effects

- **Glare (L button)**: Sigmoid wash-out simulating photophobia / light
  sensitivity. Applies `1/(1+exp(-12*(t-mid)))` blend toward white. Adjustable
  midpoint threshold.
- **Exposure (E button)**: Manual exposure time control via
  `MediaStreamTrack.applyConstraints`. Only shown when the camera supports manual
  exposure mode. Logarithmic slider mapping.

  When manual ET is active, ISO is fixed at 400 (clamped to the hardware range).
  This models rod sensitivity: rods are highly sensitive receptors (high ISO
  equivalent) that work well in dim light but cannot reduce their sensitivity
  enough for daylight. Fixing a moderately high ISO means a dim room looks
  normal, while outdoor scenes blow out — matching the achromat's experience.

  Without fixing ISO, the camera auto-adjusts it when switching to manual ET,
  snapshotting whatever value auto-exposure was using. This caused inconsistent
  behavior: entering manual mode indoors (where auto-ISO is high) and then
  going outside produced an over-exposed image, but re-toggling ET outdoors
  (where auto-ISO resets lower) made the same ET value look normal. Fixing ISO
  eliminates this dependence on when the user toggles the mode.

  The ET slider has a floor of 250 µs — the slider range maps from
  `max(hardwareMin, 250)` to `hardwareMax`. This prevents the user from
  simulating "adapting down" more than rods physically can. The value was chosen
  empirically: on the Samsung S22, 1000 µs already over-exposes a dim room (due
  to auto-ISO compensation), so 250 µs corresponds roughly to the ~1000 lux
  threshold where rod saturation overwhelms an achromat's vision.

## Known issues

- iOS Safari (not Chrome) may still show a frozen first frame on initial load.
  Cycling cameras via the C button works around this.
- `InputDeviceInfo.getCapabilities()` is not available in Firefox. The label-based
  fallback handles this.
