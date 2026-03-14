# PageLift — Book Scanner

A single-file web application (`index.html`) that turns physical books into digital PDFs using a phone camera. No build step, no server — open it in a browser and go.

---

## Overview

PageLift runs entirely in the browser. It uses the device camera to capture book pages one at a time, automatically detects when the page is still, processes the image to look like a flatbed scan, and exports the result as a PDF.

The app detects whether it's running on a mobile or desktop device and adjusts the UI accordingly.

---

## Screens

### Desktop View
Shown when the app is opened on a non-mobile device (screen width ≥ 768px or non-mobile user agent). Displays the current URL so the user can scan the QR code or copy the link to open it on their phone. A "Start scanning" button is also available to use the camera directly on the desktop if desired.

Clicking the URL box copies it to the clipboard and briefly shows a "Copied!" confirmation.

### Onboarding (Mobile)
Shown on first load on a mobile device. Explains the four-step workflow: mount phone above book, hold still for auto-capture, turn page, export PDF. Has a "Begin Scanning" button to proceed to the scanner.

### Scanner View
The main scanning interface. Shows the live camera feed with a frame overlay, status indicators, and controls. Only shown after the user taps "Begin Scanning" or "Start scanning".

### Settings Panel
A slide-up sheet over the scanner view. Contains four toggles (see Settings section). Dismisses with a "Close" button.

### Review View
A full-screen panel showing thumbnails of all captured pages. Accessible via the "Done" button. Pages can be deleted or the user can export to PDF. A "Continue" button returns to the scanner.

### Generating Overlay
A modal overlay with a spinner and status text shown during PDF generation. Covers the whole screen and disappears when the PDF has been saved.

---

## Scanning Workflow

### 1. Camera Startup
`startScanner()` requests the rear camera (`facingMode: 'environment'`) via `getUserMedia`. Resolution defaults to 1920×1080; high-resolution mode requests 3840×2160. Once the stream is live, the scan loop starts.

### 2. Scan Loop (`scanLoop`)
Runs via `requestAnimationFrame`, throttled to approximately 12 fps (one comparison every 83 ms). On each tick:

1. Draws the live video to a hidden 64×64 canvas.
2. Compares the current downsampled frame against the previous one using `getFrameDiff`.
3. Advances through the phase state machine based on the result.

### 3. Phase State Machine

| Phase | Meaning |
|-------|---------|
| `idle` | Scanner not started |
| `searching` | Looking for a stable frame; motion detected |
| `stabilizing` | Frame has been still for >3 ticks; counting down to capture |
| `captured` | Page was just captured; waiting for page-turn motion |
| `turning` | Motion detected after capture; returning to searching |

**Transitions:**
- `searching` → `stabilizing`: motion drops below threshold for >3 consecutive frames
- `stabilizing` → `captured`: stable for the required number of frames (default 18, ~1.5 s) and auto-capture is on
- `stabilizing` → `searching`: motion spikes back above threshold
- `captured` → `turning`: motion exceeds 2× the threshold (page being turned)
- `turning` → `stabilizing`: motion drops again

### 4. Frame Difference (`getFrameDiff`)
Samples every 4th pixel (every 16th byte, R channel only) from the 64×64 comparison frame. Returns the mean absolute difference per sampled pixel. Threshold is 12 by default.

### 5. Stability Ring
During the `stabilizing` phase, a circular SVG progress ring animates around the centre of the frame showing how close to capture the app is. It disappears once a capture happens or motion restarts.

### 6. Capture (`capturePage`)
When stability is confirmed:
1. Draws the full-resolution video frame to the capture canvas.
2. Encodes it as a JPEG (quality 0.92) data URL.
3. Pushes `{ dataUrl, timestamp, processing }` onto `state.pages`.
4. Triggers UI feedback: flash overlay, green frame border, page counter update.
5. Plays a synthesized shutter sound if sound is enabled.
6. Sets a 500 ms cooldown to prevent accidental double-captures.
7. If Scanner Mode is on, fires off background image processing (see Scanner Mode below).

### 7. Manual Capture
Tapping the large circular shutter button calls `manualCapturePage()`, which triggers `capturePage()` immediately as long as scanning is active and no page has just been captured.

---

## Settings

Accessed via the gear icon in the scanner controls. All settings are stored in the `state` object and persist for the duration of the session only (no localStorage).

| Setting | Default | Effect |
|---------|---------|--------|
| Auto-capture | On | Captures automatically after the stability countdown. When off, the user must tap the shutter button manually. |
| Capture sound | On | Plays a synthesized descending tone (1200 Hz → 600 Hz, 100 ms) on each capture using the Web Audio API. |
| High resolution | Off | Switches the camera request from 1920×1080 to 3840×2160. Takes effect on the next scanner start. |
| Scanner mode | On | Runs perspective correction and scanner enhancement on each captured image (see Scanner Mode). |

---

## Scanner Mode (Image Processing)

When Scanner Mode is on, each captured image is processed in the background via `processPageImage()`. The scanner continues working immediately — processing does not block capture.

Processing is capped at 2500 px on the longest side. High-resolution captures are downscaled before processing and the downscaled version is stored.

Thumbnails in the review grid show a green spinner while their page is still being processed.

### Step 1 — Grayscale (`buildGray`)
Converts the image to a luminance map using the standard coefficients: `L = 0.299R + 0.587G + 0.114B`.

### Step 2 — Corner Detection (`detectPageCorners`)

**Threshold (Otsu's method):** Computes a histogram of grayscale values, then finds the threshold that maximises between-class variance. This cleanly separates bright page pixels from the darker desk or book cover.

**Corner finding:** Iterates over all bright pixels (sampled at 1/300th of the short dimension for speed) and finds four extreme points using diagonal projections:
- Top-left: minimise `x + y`
- Top-right: maximise `x − y`
- Bottom-left: minimise `x − y`
- Bottom-right: maximise `x + y`

**Validation:** The detected quadrilateral is rejected (no warp applied) if:
- It covers less than 15% of the image area — detection likely failed.
- All four corners are within 3% of the image edges — the page fills the frame and no meaningful warp is needed.

### Step 3 — Perspective Warp (`warpPerspective`)
If corners are valid:

1. Computes the output dimensions from the distances between opposite corners (max of top/bottom widths, max of left/right heights).
2. Builds a homography matrix mapping the 4 detected source corners to the 4 corners of a clean rectangle using the **Direct Linear Transform** (DLT). This yields an 8×8 linear system solved via Gaussian elimination with partial pivoting (`gaussElim`).
3. Inverts the homography (`invertMatrix3x3`) to get the reverse mapping.
4. Fills every destination pixel by sampling the source using the inverse homography + **bilinear interpolation**.

### Step 4 — Scanner Enhancement (`applyScannerEffect`)

**Background estimation:** Box-blurs the luminance channel with a radius of ~12% of the shorter image dimension (two-pass horizontal + vertical, O(N) regardless of radius). This captures the low-frequency illumination field — shadows, lamp gradients, uneven desk lighting — while ignoring text detail.

**Shadow removal:** Divides each pixel's luminance by its estimated background value, then scales so a perfectly-lit background maps to ~220. Shadows and gradients vanish; the paper appears uniformly lit.

**Auto-levels:** Builds a histogram of the normalised values and identifies the 2nd percentile (black point) and 98th percentile (white point). Stretches the result to fill the 0–255 range.

**S-curve:** Applies a contrast-boosting power curve (exponent 1.8 on each half):
- Values below 128 are pulled darker (text becomes crisper).
- Values above 128 are pushed lighter (paper becomes whiter).

**Output:** Written back to the canvas as grayscale (R = G = B), matching the neutral monochrome look of a physical flatbed scanner.

---

## Review

`openReview()` renders all captured pages as a grid of thumbnails. Each thumbnail has:
- The page image (processed if Scanner Mode was on and processing is complete, otherwise the raw capture).
- A page number badge.
- A delete button (×) revealed on hover/tap.
- A spinner overlay if the page is still being processed.

Pages are deleted from `state.pages` in place. The counter in the scanner header updates to reflect the new count.

---

## PDF Export (`generatePDF`)

1. Loads the first page image to determine the aspect ratio.
2. Creates a jsPDF document sized to A4 width (210 mm) with the matching height. Uses landscape orientation if the image is wider than tall.
3. Iterates through all pages, adding each as a JPEG image filling the page exactly.
4. Yields to the UI thread between pages (10 ms setTimeout) to keep the spinner responsive.
5. Saves the file as `PageLift-YYYY-MM-DD.pdf`.

Uses jsPDF 2.5.1 loaded from the Cloudflare CDN.

---

## State Object

```js
state = {
  pages: [],           // [{ dataUrl, timestamp, processing }]
  isScanning: false,
  autoCapture: true,
  sound: true,
  hires: false,
  scannerMode: true,
  stream: null,        // MediaStream from getUserMedia
  animFrame: null,     // requestAnimationFrame ID
  phase: 'idle',       // idle | searching | stabilizing | captured | turning
  stableFrames: 0,     // consecutive stable frame count
  lastFrameData: null, // Uint8ClampedArray from previous 64×64 frame
  captureTimeout: null,
  stabilityRequired: 18,  // frames needed to trigger auto-capture (~1.5 s at 12 fps)
  motionThreshold: 12,    // mean pixel diff to classify as motion
  cooldownUntil: 0,       // timestamp before which captures are suppressed
}
```

---

## Dependencies

| Library | Version | Source | Purpose |
|---------|---------|--------|---------|
| jsPDF | 2.5.1 | Cloudflare CDN | PDF generation |
| DM Mono | — | Google Fonts | UI monospace font |
| Instrument Serif | — | Google Fonts | Logo / heading font |

No build tools, no package manager, no server required. Everything runs from a single HTML file.
