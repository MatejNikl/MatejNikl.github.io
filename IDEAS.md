# Feature Ideas & Improvements

Collected ideas for the Achromatopsia Simulator, roughly ordered by
expected impact relative to effort.

---

## Simulation accuracy

### Nystagmus simulation

Nearly all complete achromats have involuntary eye oscillations (nystagmus)
that degrade effective acuity and make fixation unstable. A subtle
sinusoidal CSS `translate()` on the canvas (~2–4 Hz, small amplitude)
would approximate the visual instability without any shader changes.
Add as a toggle pill like Grayscale/Blur. This would make the simulator
meaningfully more complete — teachers and family often don't realize the
world isn't just blurry and gray, it's also shaking.

### Contrast sensitivity loss

Achromats have reduced contrast sensitivity, not just reduced acuity.
The current Gaussian blur simulates acuity loss but not the washed-out
contrast perception. A shader-based contrast reduction (pull pixel values
toward mid-gray by a tunable factor) would add another dimension. Could
be tied to the Blur toggle or a separate control.

### Scotopic adaptation latency

Rods adapt much more slowly than cones when moving between light levels.
When the scene brightness changes abruptly (walking indoors → outdoors),
there's a prolonged period of poor vision. This could be simulated by
slowing the auto-exposure convergence when brightness increases sharply
(lengthening the damping period), making the glare wash-out linger.

---

## UX & usability

### Fullscreen re-entry

Currently fullscreen is requested only once on the first tap. If the user
exits fullscreen (system gesture), there's no way back in without
reloading. Options:
- Remove the one-shot flag and request on every tap (skip if exited
  < 2 seconds ago to respect intentional exits).
- Add a small fullscreen button to the controls.
- Re-request on any pill tap but not on canvas taps.

### Snapshot / compare mode

Tap to freeze the current frame and show it alongside the original color
version (side-by-side or toggle). Useful for demonstrations: "here's
what you see, here's what he sees." The raw camera texture is already
available before the grayscale shader — just render both to split
viewports or toggle the `uGrayscale` uniform.

### Torch / flashlight toggle

`MediaStreamTrack.applyConstraints({ advanced: [{ torch: true }] })` works
on Android Chrome. In dim indoor environments (classrooms), turning on the
flashlight lets the user see the glare effect without going outside. Check
`getCapabilities().torch`, hide if absent — same pattern as the Glare pill.

### Safe area insets for notched phones

Controls at `top: 10px; right: 10px` can overlap the camera cutout or
Dynamic Island. Adding `viewport-fit=cover` to the meta viewport and
offsetting controls by `env(safe-area-inset-*)` would handle this.

### PWA / Add to Home Screen

A minimal `manifest.json` with icons would let users add the app to their
home screen so it launches fullscreen without the URL bar. Good for the
"hand the phone to the teacher" use case.

### Info / about overlay

A small `(i)` button that briefly explains what achromatopsia is and what
each toggle simulates. The target audience (teachers, therapists, family)
may not know the terminology. A few sentences in a dismissible overlay,
localized to both languages.

### Hide controls on inactivity

Auto-fade the pill buttons and sliders after a few seconds of no
interaction, revealing a clean full-screen viewfinder. Tap anywhere to
show them again. Useful when the phone is being held up for extended
viewing.

### Landscape orientation hint

On first visit in portrait mode, briefly show a hint suggesting landscape
for a wider field of view. The camera stream is 16:9 and portrait crops
the top/bottom significantly due to `object-fit: cover`.

---

## Technical / robustness

### WebGL2 upgrade path

The app currently uses WebGL 1. WebGL 2 is supported everywhere WebGL 1
is (Android Chrome, iOS Safari 15+). Benefits: `texStorage2D` for
immutable textures, cleaner FBO setup, GLSL 300 es with integer types.
Not urgent — the current WebGL 1 code works — but would simplify the
downsample pipeline.

### Zoom (pinch-to-zoom)

Achromats often need to get very close to see details. A pinch-to-zoom
on the canvas (CSS `transform: scale()`) would let users magnify the
view. Alternatively, switch to a telephoto lens on multi-camera phones
when zoom exceeds 2×.

### Video recording / export

Let users record a short clip of the simulated view and share it. Use
`MediaRecorder` on a `canvas.captureStream()`. Useful for sending to
someone who isn't physically present — "this is what my child sees in
your classroom."

### Accessibility: VoiceOver / TalkBack labels

The pill buttons are `<div>` elements without ARIA roles or labels.
Adding `role="button"`, `aria-pressed`, and `aria-label` attributes
would make the app usable with screen readers (relevant for sighted
assistants helping a visually impaired user navigate the UI).
