# reflection — project handoff

A single-file browser app (`index.html`, vanilla JS + WebGL2 + Web Audio, no build step). The user turns on their webcam and mic and sees themselves rebuilt as a slow, weightless, **datamoshed** 3D surface, while their voice runs through a **VHS-style voiceover audio chain**. The processed audio plus the visuals can be recorded and played back in-app.

This file is the full context for continuing in a new chat. Read "What the user wants" and "Rejected ideas" first. The user has rejected several directions and is sensitive to over-engineering.

---

## 1. Running it

```bash
cd C:\Users\hakan\reflection-audioreactive
python -m http.server 8000      # then open http://localhost:8000
```

- Camera and mic need a secure context, so use `localhost` (not `file://`).
- `.claude/launch.json` already defines a `reflection` server (python http.server on port 8000) for the Claude desktop preview pane.
- Windows quirks seen in this environment: in the Bash tool use `python`, not `python3`. A Windows `python` cannot see MSYS `/tmp` paths; use `cygpath -w` or the scratchpad directory.
- Buttons: **start** (camera + mic) and **demo** (no permissions needed: synthetic "person" video, a generative music track, and a fake face for the voice effect).
- Keys: `H` hide panel, `U` clean view (hides all UI, for streaming/capture), `F` fullscreen, `S` save PNG, `R` record, `T` takes, `P` listen, `1`–`5` looks, `M` mirror. Drag to orbit, wheel to zoom, double-click to reset the view.

---

## 2. What the user wants (in their words, condensed)

- Minimal UI. A webcam "reflection" that is **aesthetic data-mesh / datamosh**.
- **Everything slow.** "We like slow LFOs." Sound should modulate gently, with lag.
- **Zero gravity everywhere.** No falling, no wind direction, nothing blown away.
- **No beats.** "I don't want the stupid beating." There must be no beat detection or beat-driven pulses, kicks, shocks or flicker.
- **Datamosh is the main idea**, as follows. When they move, **the old pixels should stay**, and the movement replaces them **slowly and driftily**. Motion glow should be *really subtle, or even absent*.
- When they talk, the effect on the picture is **not decided yet**. Two attempts failed (see §6). The current attempt ("voice push", §5.4) is untested by the user.
- A **processed, VHS-ish voiceover sound**, not raw audio. They record and want to stream. "Streaming" is currently only supported by capturing the window (OBS) with `U` clean view.
- Strip back and rebuild **one thing at a time**. They called the stacked-effects versions "a mess".
- Their audio setup is a **PreSonus Studio 24c** with the mic in input 1. Their mic is left-only, and the interface's own direct-monitor knob sends a raw, left-panned signal to their speakers. They need to turn it toward "playback" to hear only the processed sound.

Saved to Claude's memory as `feedback_reflection_aesthetic.md`. The memory folder is `C:\Users\hakan\.claude\projects\C--Users-hakan-reflection-audioreactive\memory\`.

---

## 3. File layout

```
reflection-audioreactive/
├── index.html        # the whole app (CSS + HTML + JS, ~1700 lines)
├── HANDOFF.md        # this file
└── .claude/launch.json
```

`index.html` is sectioned by comment banners:

| Section | Contents |
|---|---|
| 0 | helpers (`$`, `clamp`, `lerp`, `toast`) |
| 1 | `SLIDERS`, `PALETTES`, `SOUND_PRESETS`, `VISUAL_BASE`, `LOOKS`, params, persistence |
| 2 | audio engine: worklets, graph builder, demo track, sources/devices, analysis, `updateFx` |
| 3a | face detection (MediaPipe) and `voiceDrive` (slow voice integrator) |
| 3b | WebGL: shaders, render targets, camera upload, geometry, passes, `render()` |
| 4 | input sources (camera/mic/fallbacks), `start()` |
| 5 | recording + takes playback |
| 6 | main loop `frame()` + fps governor |
| 7 | UI (sliders, tabs, looks, presets, keyboard, idle fade) |

---

## 4. Audio engine

```
mic (L+R summed to mono) ─┐
file / demo track ────────┴→ input gain → 85Hz high-pass → noise gate (worklet)
   → [voice analyser tap] → compressor (+makeup) → mud cut (320Hz) → presence (3.2kHz)
   → bitcrusher (worklet) → tape (dry/wet: saturation, 2× low-pass, high-pass, wow/flutter delay, hiss)
   → room (pre-delay → convolver, generated IR) → limiter → master
        master → analyser (visuals) | listen gain → speakers | MediaStreamDestination (recorder)
```

Key facts:
- **Left-only mic fix:** the mic goes through a channel splitter, both channels are summed into a mono gain node, and the room reverb then spreads it to both speakers. Verified: a left-only test signal comes out with equal L and R levels.
- **Worklets** are built from function source via a Blob URL. `crusher` quantizes with soft, tanh-rounded steps, uses a sample-and-hold downsampler with a smoothing low-pass, and smooths its own parameters per sample. `gate` is a downward expander that learns the noise floor by itself.
- **Tape** has a 10 ms base delay (`TAPE_BASE_DELAY`) for wow/flutter. Wobble gains are kept small so the delay never goes negative.
- **The analyser sits after the effects**, so the visuals follow the processed sound.
- **Voice detection** uses a second analyser that taps after the gate: 220–3600 Hz energy above a learned noise floor, followed with attack 0.12 and release 0.04 per frame.
- **Slow level** (`audio.slow`) is the overall loudness followed over about 1–2 seconds. It is the only audio value that drives the picture (depth breathing, float amount, bloom). `audio.bass/mid/treble` are used only for the meter bars.
- **Beat detection is gone**. `react` and the beat effects were removed. Effects are held at the preset values, except tape wobble, which leans slightly with `audio.slow`.
- **Voice presets** (`SOUND_PRESETS`): `vhs voiceover` (default), `clean voice`, `radio`, `dream tape`, `crushed`. They set gate, compression, presence, tape mix, wobble, age, room, crush mix, bits and downsample. `gain` and `volume` are personal and never overwritten.
- **Measured in the preview pane** with full-band noise: `vhs voiceover` is about 10 dB duller above 6 kHz than `clean voice`, and `radio` is about 15 dB duller, with the lows cut. L/R levels are identical. Levels across presets are within about 1 dB.
- **Devices:** the sound tab has input and output selects. The input is chosen with `getUserMedia({deviceId})`, the output with `audioCtx.setSinkId`. Choices are saved in localStorage key `reflection.dev`. Raw mic constraints: echo cancellation, noise suppression and auto gain are all off.
- **Latency:** `listenLatencyMs()` estimates the delay (mic track latency + `baseLatency` + `outputLatency` + chain). It is shown in the status line, with a warning above 45 ms. Recording, analysis and visuals never pass through the listen path. The mic starts with listen off so there is no feedback or delayed-voice problem.

---

## 5. Visual pipeline

```
camera frame → camTex[2] (current + previous, mipmapped)
   → flowPass: Lucas-Kanade optical flow, 160px wide, ping-pong
        rg = motion vector (prev→cur, uv units), a = decaying "motion energy"
   → moshPass: 640px wide ping-pong with mipmaps  ← the datamosh memory
   → scenePass: a smooth grid surface wearing the mosh texture (+ flow energy for the subtle glow)
   → bloomPass: bright-pass → blur chain (half and quarter res)
   → compositePass: bloom, rgb split, slow tape sway, scanlines, grain, bit-crushed colour + pixel size, soft tone-map
```

### 5.1 Surface (`MESH_VS` / `MESH_FS`)
- A grid of `density` (40–200) columns, with 3 draw modes: `solid` (depth-tested, lit with derivative normals), `x-ray` (dimmed solid + additive lines), `wire`.
- Depth comes from the luminance of the **mosh texture** (LOD 2, so it is smooth despite the grain). Old pixels therefore keep their depth.
- **Zero-g float:** two value-noise fields drift in opposite directions at very slow speeds. They are scaled by `drift` and by `audio.slow`. There is no wind and no gravity.
- **Orbit:** the camera turns slowly on three axes (yaw, pitch, roll), each on its own long cycle, scaled by `spin` and `speed`.

### 5.2 Datamosh (`MOSH_FS`), the heart of the project
The picture is a **memory**, not a window. Per camera frame:
1. `drift = mv * mosh` (memory dragged along the real motion vectors) `+ dn * .0022 * mosh * (.12 + energy)` (slow zero-g noise drift, mostly where you moved so still areas stay crisp) `+ push * .004` (voice, see §5.4).
2. `pred = prevMosh(uv - drift)`.
3. `rate = exp(-4.2 * mosh)`: how readily the live picture replaces the memory (mosh 0 → 1, mosh 0.7 → about 0.05, mosh 1 → about 0.015). It is multiplied by `(1 - .6*energy*mosh)` so the old pixels hold on longer where you moved, and by `(1 - .5*voice*faceOn*exp(-1.5*dist))` so they hold on longer around you while you talk.
4. **Pixel-by-pixel replacement:** a per-pixel hash (cells of 320×180, new each frame) decides whether the pixel takes the live colour (`r < rate`) or keeps the memory (blended slightly toward live by `rate*.25`).

Tuning knobs: the `4.2` exponent, `.0022`, `.004`, `.6`, `.5`, `.25`, and the 320×180 grain cell size.

### 5.3 Motion glow
`en² * 2 * mglow` is added to the colour, and `en * mglow * .3` to depth. The default is now `mglow: .15` and the slider only goes 0–1. The user said "really subtle, or maybe not at all", so consider defaulting it to 0.

### 5.4 Voice → picture ("voice push", the current, unconfirmed attempt)
`voiceDrive.v` is a slow integrator: target `smoothstep(.03, .22, audio.voice) * params.dissolve`, rising with τ = 2.5 s and falling with τ = 4 s. It is therefore slow, and reaches full strength at ordinary speaking volume (the user complained that earlier versions were too fast and their upper threshold too high).
The face's mouth position (`uMouth`, from face detection) is the origin. The voice does two things in the mosh pass:
- pushes old pixels gently outward from the mouth (`push`, falling off as `exp(-2.2*dist)`);
- makes the memory hold on longer around the speaker.

Checked only with the synthetic demo face, forcing `voiceDrive.v = 1`: the face region smears and grains outward from the mouth while the still background stays crisp. **The user has not seen this with their own face and voice.**

### 5.5 Face detection (`initFace` / `updateFace`)
- MediaPipe `FaceDetector` (BlazeFace short-range), loaded at runtime from jsDelivr (`@mediapipe/tasks-vision@0.10.14`, WASM from the same path) with the model from `storage.googleapis.com`. It runs locally on the video; nothing is uploaded.
- It only produces: face present or not, box, and the mouth keypoint (keypoint index 3). All values are eased so nothing jumps, and `present` fades over about 1.5 s.
- If it can't load (offline): `face.mode = 'off'` and a centred "assumed face" is used. In demo mode the demo canvas supplies a fake face.
- **Verified only** that the model loads and returns zero detections on a blank frame. **It has never run on a real face.**

### 5.6 Looks (visual only, keys 1–5)
`clear` (default: true colour, mono palette), `neon` (x-ray), `ember`, `aurora`, `wire`. Looks only set visual keys, and sound presets only set sound keys. Transitions ease over about 0.8 s. Unrelated settings are persisted in localStorage `reflection.v4`.

---

## 6. Rejected ideas (do not bring these back without asking)

1. **Point/dot rendering** and the **spectrogram-over-wind "sound stream"**: "too cheap", "just a waveform".
2. **Circular wobble** driven by bass.
3. **Tile/macroblock 3D mesh with datamosh blocks, beat shocks, wind gusts, gravity wells**: "not how I want". "Everything is too fast".
4. **Voice dissolve into flying tiles**: "way too fast, upper threshold way too high".
5. **Voice melt (wet paint, from the mouth)**: the user explicitly chose this style when asked, then said "it's sooooooo terrible, let's try something else". Do not assume the cause. It may have been the look, the speed, or something about how it behaved with their real face and mic.
6. **Beat detection and all beat-driven effects**: "stupid beating".
7. **mp4 recording first**: MediaRecorder's H.264 path silently dropped the video track in the preview pane, so webm is preferred.

---

## 7. Recording and playback

- `R` records `canvas.captureStream(60)` plus the processed audio (from `fx.recDest`, tapped before the listen volume, so it records whether or not you are listening).
- MIME order: webm vp8+opus, webm vp9+opus, webm, then mp4 (Safari fallback). 10 Mbps video and 192 kbps audio.
- **Empty-take guard:** a canvas only produces frames while the tab is painting. A tiny take shows a "keep this tab visible" warning.
- **Takes viewer** (`T`): plays, downloads and deletes takes. The live listen sound is muted while a take plays. A webm duration workaround seeks to `1e7` and back.
- Canvas dimensions are kept divisible by 4 (encoders reject odd sizes and the bloom chain halves twice).
- A benign console error `ERR_REQUEST_RANGE_NOT_SATISFIABLE` comes from that duration seek on the blob URL.
- **Streaming** is not built. The intended path is `U` (clean view) + fullscreen + OBS window capture, with listen on if the processed audio is needed from the desktop.

---

## 8. Development and testing notes (very useful)

- **The preview pane throttles `requestAnimationFrame`** when it is not painting, so the app appears frozen and audio-derived values stop updating. Step frames manually from the JS tool:

  ```js
  window.__step = async (n, ms=40) => { const raf=window.requestAnimationFrame; window.requestAnimationFrame=()=>0;
    for (let i=0;i<n;i++){ perf.acc=-1e9; await new Promise(r=>setTimeout(r,ms)); frame(performance.now()); }
    window.requestAnimationFrame=raf; };
  ```
  `perf.acc=-1e9` stops the fps governor from lowering the grid detail during manual stepping.
- **Persisted state interferes with tests.** `applyLook()` and `persist()` write localStorage after 300 ms. Clear `localStorage` before testing defaults, and note that a running look transition (`target`) will pull params you set by hand back to the look values.
- **To test the mosh with sharp content**, swap `source` for a canvas that has a `.tick(t)` function and a `.face` object (`{cx,cy,w,h,mx,my}`), set `lastVT=-1; vt.moshInit=true`. The default demo person is too soft to show anything.
- **To test the audio chain** without a mic, stop the demo (`stopDemo()`), feed a noise buffer into `fx.micSplit` (left channel only via a `ChannelMerger`), tap `fx.master` with a splitter + analysers, and read band energies with `getFloatFrequencyData`.
- The preview pane renders in software (about 20 fps), so frame-rate-dependent behaviour is not representative.

---

## 9. What has NOT been tested (real-world risks)

1. The **whole app on a real camera and the real mic through the Studio 24c** by me. The user has only seen earlier versions.
2. **Face detection on a real face**, and whether keypoint 3 really is the mouth.
3. **LK optical flow under real camera noise.** The thresholds (`td/9 - .02`, `*7`, `lam = .002`) were chosen on synthetic content.
4. **The look of the per-pixel grain** at `mosh` 0.7–1.0 (at 1.0 the picture can get muddy). The grain cell (320×180) may be too coarse or too fine.
5. **Voice detection thresholds with a real mic** (`vstate`, `smoothstep(.03,.22)`), and the gate behaviour on their room noise.
6. **Performance on a real GPU** (the governor lowers mesh detail below 40 fps).
7. By ear: whether `vhs voiceover` actually sounds like the character the user wants. It was only verified spectrally.
8. Output device switching (`setSinkId`) and input switching on real hardware.

---

## 10. Suggested next steps

1. **Ask the user to try it on their camera and mic and report**, specifically about: the datamosh feel (does old-pixel memory + slow replacement feel right?), whether motion glow should be 0, the grain, and **what voice should do** now that dissolve, tile-scatter and melt all failed. Offer concrete alternatives rather than guessing a fourth time, for example: voice only slows the replacement rate (the image "freezes" slightly while talking), the voice injects its own drift vectors, or the voice does nothing visual at all and only the sound effects react.
2. Tune the mosh against real camera noise (flow thresholds, grain cell size, `drift`, `push`).
3. Consider `mglow` default 0 and removing the glow code if unwanted.
4. Real streaming support (WebRTC / RTMP) is not built. If the user wants it, discuss whether OBS capture is enough.
5. Cleanup candidates: `GL_COMMON` still contains the unused `oratorMask()` and `uFace` uniform (only `uMouth` and `uFaceOn` are used now); `audio.bass/mid/treble` only feed the meter; the look `wire`/`x-ray` modes have had little real-camera testing.

---

## 11. Key defaults (for reference)

`VISUAL_BASE`: mode 0 (solid), palette 4 (mono), mirror on, density 120, depth .8, glow .6, hue 0, spin .4, speed .5, drift .6, sens .8, smooth .8, mosh .7, mglow .15, dissolve (voice push) 1, video 1, split .1.
`vhs voiceover`: gate .5, comp .7, presence .55, tape .85, wobble .3, age .6, room .25, crush .1, bits 11, rate 1.5. `gain` 1, `volume` .8.
