# Achromatopsia Simulator — Design Context

A single-file web app (`index.html`) that uses the phone's back camera to show
approximately what a person with complete achromatopsia (rod monochromacy) sees.

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

## Known issues

- iOS Safari (not Chrome) may still show a frozen first frame on initial load.
  Cycling cameras via the C button works around this.
- `InputDeviceInfo.getCapabilities()` is not available in Firefox. The label-based
  fallback handles this.
