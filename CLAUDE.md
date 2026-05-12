# Audio-to-QR Kiosk — Technical Plan

## Project Summary

A two-station art exhibit installation where visitors record a short voice message at a kiosk,
receive a QR code encoding the compressed audio, and can later scan that QR code at an interpreter
station to hear the message played back.

**Primary constraint:** All audio data must fit within a single QR code (~2,953 bytes max).  
**Primary optimization target:** Voice clarity — words must be intelligible on playback.

---

## Phase 1 Status — COMPLETE ✓

A fully working browser prototype (`audio-qr-phase1.html`) was built and validated. Key findings
from Phase 1 that update the original plan are documented throughout this file.

### What was validated
- End-to-end record → encode → QR → scan → decode → playback works in-browser
- Opus via `MediaRecorder` at 8kbps / 16kHz mono produces intelligible voice
- WebM container overhead is real (~260 bytes) and must be budgeted for
- Byte-budget-driven auto-stop is the correct approach (not fixed time)
- jsQR works universally across browsers including iOS Safari
- iOS requires a direct user tap to trigger audio playback (autoplay blocked)

---

## Capacity & Codec Decision

### Why Codec Choice Is Everything

QR codes top out at **2,953 bytes** of binary data (Version 40, error correction level L).
At standard audio quality this holds nothing useful. At voice-optimized settings it holds ~3 seconds.

### Codec: Opus via MediaRecorder (browser-native)

In the browser prototype we use the browser's built-in Opus encoder via `MediaRecorder`.
No WASM encoder is needed — the native implementation is sufficient and has zero load time.

Opus is the right choice:
- Designed specifically for voice at low bitrates
- Outperforms MP3, AAC, and Speex at equivalent bitrates
- Built into Chrome, Edge, Firefox natively
- For Electron: same API works without modification

### Confirmed Audio Parameters

| Parameter | Value | Notes |
|---|---|---|
| Codec | Opus (via MediaRecorder) | Browser-native, no WASM needed |
| Bitrate | 8 kbps (`audioBitsPerSecond: 8000`) | Confirmed working |
| Sample rate | 16 kHz | Wideband voice; captures consonants |
| Channels | Mono (`channelCount: 1`) | Required to fit in QR |
| Recording stop | Byte-budget driven | See below — NOT fixed time |
| Max duration | ~3 seconds | Depends on voice; byte budget stops it |

### Critical: WebM Container Overhead

The browser wraps Opus audio in a WebM container. This overhead is approximately **260 bytes**
and is unavoidable. It means:

- Raw budget: 2,953 bytes
- Usable for audio: **~2,650 bytes** (leaving ~300 bytes margin for container)
- Do NOT target 2,953 — the file will always be larger than the raw audio data

### Byte-Budget Auto-Stop (Confirmed Approach)

**Do not use a fixed recording duration.** Use byte-budget monitoring instead:

```js
// In MediaRecorder.ondataavailable:
bytesAccumulated += e.data.size;
if (bytesAccumulated >= 2650 && isRecording) stopRecording();
```

This approach:
- Maximizes duration for every speaker (quiet speech gets more time; loud/complex gets less)
- Eliminates the over-budget error entirely
- Adapts automatically if bitrate fluctuates
- Still has a hard 5-second `setTimeout` as an absolute safety ceiling

In practice this yields approximately **2.8–3.2 seconds** of usable recording.

---

## Voice Quality Pipeline

Voice clarity requires more than just a good codec. The full signal chain matters.

### Pre-Encoding Processing (Web Audio API)

These nodes run before audio reaches `MediaRecorder`:

1. **High-pass filter** — `BiquadFilterNode`, type `highpass`, cutoff 80 Hz, Q 0.7
   Removes low-frequency rumble, HVAC noise, footsteps.

2. **Dynamics compressor** — `DynamicsCompressorNode`
   threshold: -18 dBFS, knee: 6, ratio: 4:1, attack: 5ms, release: 200ms
   Tames dynamic range so quiet and loud speech both encode efficiently.

3. **Gain boost** — `GainNode`, value: 1.4
   Normalizes level upward before encoding.

4. **Analyser** — `AnalyserNode`, fftSize 128
   Feeds the real-time waveform visualizer only; does not affect audio.

### Signal Chain

```
MediaStreamSource
  -> BiquadFilter (HPF 80Hz)
  -> DynamicsCompressor
  -> GainNode (1.4x)
  -> AnalyserNode
  -> MediaStreamDestination -> MediaRecorder
```

### getUserMedia Constraints

```js
{
  audio: {
    channelCount: 1,
    echoCancellation: true,
    noiseSuppression: false,  // handled in pipeline
    autoGainControl: false,   // handled in pipeline
    sampleRate: { ideal: 16000 }
  }
}
```

### Microphone Recommendation (Physical Kiosk)

Use a **cardioid or supercardioid condenser mic** positioned 15-20cm from the visitor's mouth.
- Rejects ambient gallery noise from the sides
- Rode NT-USB Mini is a practical choice for USB simplicity
- Avoid omnidirectional mics in a gallery setting

---

## Data Flow

```
[Visitor speaks]
      |
[Microphone -> 16kHz mono capture]
      |
[Web Audio API pipeline: HPF -> Compressor -> Gain -> Analyser]
      |
[MediaRecorder -> Opus, 8kbps]
      |  <- byte budget monitor stops at ~2,650 bytes
[WebM/Opus binary blob, ~2,700 bytes total]
      |
[qrcode-generator (byte mode) -> Version 35-40, Error Correction L]
      |
[QR displayed on screen AND sent to thermal printer simultaneously]
      |
[Visitor takes printed receipt to interpreter station]
      |
[Camera scans QR -- jsQR or BarcodeDetector]
      |
[Binary blob recovered as Uint8Array]
      |
[Blob URL -> HTMLAudioElement -> playback triggered by user tap (iOS) or auto (others)]
      |
[Audio plays through speaker or headphones]
```

---

## System Architecture

### Two Modes, One Web App

Both stations run the same application. Mode is selected manually via a toggle in the prototype;
in the final kiosk build, mode should be locked per-device via a URL parameter:
`?mode=record` / `?mode=interpret`

### Recorder Station

**Role:** Capture voice, encode, display QR, print automatically.

**Stack:**
- Electron (kiosk mode, full-screen, no browser chrome)
- `MediaRecorder` API + Web Audio API pipeline
- `qrcode-generator` npm package in byte mode for QR generation
- `node-escpos` or `escpos-usb` for thermal printer integration

**UX Flow:**
1. Idle/attract screen shown when no one is present
2. Visitor presses physical button (mapped to `keydown` spacebar or `pointerdown`)
3. Recording starts — ring fills as byte budget is consumed, elapsed time shown
4. Button released early stops immediately; budget exhausted stops automatically
5. ~0.5s processing: Opus encode + QR generation
6. QR displayed full-screen AND print job sent simultaneously (no visitor action needed)
7. "Take your receipt" prompt shown while printer spools (~2s on TM-T88)
8. Returns to idle after 15 seconds of inactivity

**Button Behaviour:**
- `pointerdown` to start, `pointerup` to stop early
- Do NOT use `pointerleave` as a stop trigger — causes false stops with physical buttons
- Minimum 0.5s debounce to prevent accidental taps

### Interpreter Station

**Role:** Scan printed QR, decode audio, play back.

**Stack:**
- Same app, `?mode=interpret`
- `jsQR` (universal) with `BarcodeDetector` fast path where available
- `HTMLAudioElement` with Blob URL for playback

**UX Flow:**
1. Live camera feed shown with targeting corners overlay
2. QR detected -> camera freezes, status updates
3. Auto-play attempted; if blocked (iOS) a "Tap to Play" button appears
4. Audio plays with progress bar
5. "Scan Again" appears after playback completes

---

## QR Code Technical Details

### Encoding

- **Mode:** Byte mode — Opus output is binary, must NOT be Base64 encoded
  (Base64 adds ~33% overhead, exceeding QR capacity)
- **Error Correction:** Level L (7%) — maximizes data capacity; appropriate for clean
  printed receipts in a controlled gallery environment
- **Version:** ~35-40 depending on recording length; auto-selected by library
- **Library:** `qrcode-generator` npm package

### Display & Print Specs

| Output | Minimum Size | Notes |
|---|---|---|
| Screen display | 400x400px | White background, pixelated rendering |
| Thermal print | 4cm x 4cm | Test scan reliability at kiosk distance |

Receipt paper: standard 80mm roll gives ample room for a 5-6cm QR.

---

## Cross-Platform Scanner

Two scanning paths are implemented. Both recover identical binary data:

| Platform | Method | Notes |
|---|---|---|
| Chrome desktop / Android | `BarcodeDetector` API | Fast, hardware-accelerated |
| iOS Safari, Firefox, all others | `jsQR` + canvas frame capture | Universal fallback, ~200ms poll |

jsQR must be the primary fallback. Do not show an error if `BarcodeDetector` is absent.
Fall through to jsQR silently.

---

## iOS Audio Playback

iOS Safari blocks `audio.play()` unless called from a direct user gesture. The `setInterval`
scan callback does not qualify. The confirmed working pattern:

```js
// After QR detected, decode blob and create Audio object, then:
try {
  await audio.play(); // succeeds on desktop/Android
} catch(e) {
  // iOS: show "Tap to Play" button
  // triggerPlay() calls audio.play() directly from onclick -- iOS allows this
  showTapToPlayButton();
}
```

This is transparent on desktop/Android (auto-plays) and gracefully degrades on iOS.

---

## Hardware Recommendations

### Recorder Kiosk

| Component | Recommendation |
|---|---|
| Computer | Mac Mini, Intel NUC, or Raspberry Pi 5 |
| Microphone | Rode NT-USB Mini (cardioid condenser, USB) |
| Display | 10-15" touchscreen or monitor + physical arcade button |
| Printer | Epson TM-T88 thermal receipt printer (required, USB) |
| Enclosure | Custom fabrication or modified kiosk shell |

### Interpreter Station

| Component | Recommendation |
|---|---|
| Device | iPad or Android tablet |
| Audio output | Built-in speaker or wired headphones |
| Mount | Adjustable arm for flexible aiming angle |

---

## Key Risks & Mitigations

| Risk | Status | Mitigation |
|---|---|---|
| WebM overhead causes over-budget blob | RESOLVED | Byte-budget auto-stop at 2,650 bytes |
| iOS autoplay blocked | RESOLVED | Tap-to-play button fallback |
| BarcodeDetector not on iOS | RESOLVED | jsQR canvas fallback, universal |
| Ambient gallery noise degrades voice | Open | HPF + compressor pipeline; directional mic |
| Printer goes offline or out of paper | Open | Show on-screen error; display QR on screen as fallback |
| Visitor speaks too quietly | Open | Compressor + 1.4x gain boost in pipeline |
| Physical button mapped wrong in Electron | Open | Map to `keydown` Space; test on target hardware early |

---

## Development Phases

### Phase 1 - Proof of Concept COMPLETE
- [x] Record audio in browser at 16kHz mono
- [x] Encode with Opus via MediaRecorder at 8kbps
- [x] Implement byte-budget auto-stop
- [x] Generate QR in binary/byte mode
- [x] Scan with phone using jsQR (iOS) and BarcodeDetector (Chrome)
- [x] Decode and play back audio end-to-end
- [x] Fix iOS autoplay restriction with tap-to-play pattern
- [x] Confirm words are intelligible end-to-end

### Phase 2 - Voice Quality
- [ ] Test pipeline in noisy environment vs quiet room
- [ ] Tune compressor and gate thresholds for gallery ambient levels
- [ ] A/B compare intelligibility with pipeline on vs off
- [ ] Evaluate whether a noise gate is needed (compressor may be sufficient)

### Phase 3 - Kiosk UX
- [ ] Build full-screen recorder UI in Electron kiosk mode
- [ ] Wire physical arcade button to keyboard input
- [ ] Implement automatic print-on-complete via node-escpos
- [ ] Handle printer errors gracefully (paper out, offline)
- [ ] Build interpreter UI locked to interpret mode
- [ ] Add idle/attract screen with 15s inactivity timeout
- [ ] Lock each device to its mode via ?mode= URL param or config file

### Phase 4 - Hardware Integration
- [ ] Install and test on actual kiosk hardware
- [ ] Validate thermal print quality and QR scan reliability at receipt size
- [ ] Test physical button debounce timing
- [ ] Verify mic placement and audio quality through real speakers

### Phase 5 - Gallery Testing
- [ ] Test in actual exhibit space with real ambient noise
- [ ] Tune audio pipeline based on environment
- [ ] Stress test: 50+ recordings back to back
- [ ] Validate printer paper consumption over a full day

---

## Open Questions

- **Headphones or open speaker** at the interpreter station?
  Headphones are more intimate and resistant to ambient noise.

- **Single kiosk or multiple?**
  Multiple kiosks need the same codebase, just multiple instances — no architectural changes needed.

- **Archive recordings?**
  QR is ephemeral by design. If preservation is needed, add optional server-side logging of
  audio blobs on record.

- **Minimum recording length?**
  Currently no minimum enforced beyond a 0.5s debounce. Very short recordings will still
  generate a QR — consider adding a 1s minimum with a UI prompt.
