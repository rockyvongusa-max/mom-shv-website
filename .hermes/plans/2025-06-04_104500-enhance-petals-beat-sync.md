# Plan: Cat Developer Vibes — Enhanced Petals + Beat-Synced Splash

**Date:** 2025-06-04
**Project:** `~/AppData/Local/hermes/cat-developer-vibes/`
**Status:** PLANNING ONLY — do not execute

---

## Goal

Redesign two visual effects in the HyperFrames video:

1. **Sakura petals** — 2200 petals, fast falling, bright pink/yellow/white mix, subtle/invisible individually
2. **Beat-synced splash** — audio-reactive particle bursts that splash in sync with the music

---

## Current Context

- Project at: `C:\Users\DARA-PC\AppData\Local\hermes\cat-developer-vibes\`
- File: `index.html` (current v2 with 22 petals)
- Audio: `assets/audio.mp3`
- Output: `output/cozy-code-sakura-v2.mp4` (22 MB, 60s)
- HyperFrames lint/inspect passes, render uses `--quality draft` · 30fps

---

## Proposed Approach

### 1. Sakura Petals — 2200 petals, fast, bright, subtle

**Scaling strategy:** 2200 DOM elements will be slow to render and may overload the browser compositor at 30fps. Instead of 2200 true DOM nodes, use a **batched canvas approach** or **wave of staggered GSAP timelines** with 3批次 of ~730 petals each.

**Fallback (simpler):** Instead of 2200 individual DOM nodes, use CSS-only multiple `box-shadow` petal trick + GSAP timeline cycling a small set of pre-created petal elements at high spawn rate.

**Petal color palette:**
- Pink: `hsla(340, 100%, 85%, 0.9)` — hot pink/white
- Yellow: `hsla(50, 100%, 80%, 0.9)` — warm yellow
- White: `hsla(0, 0%, 100%, 0.8)`

**Fast fall animation:**
- Duration: 2–4s (not 9–15s)
- Scale: 0.25–0.5 (smaller so color is less visible)
- Opacity: 0.3–0.5 max (nearly invisible individually)
- Spawn rate: 2200 ÷ 60s ≈ **37 petals/second** split across 3 staggered batches

**GSAP implementation:**
- Create 3 independent petal sets of ~730 each with staggered `from()` calls
- Each petal: `gsap.from(el, { y: -100, opacity: 0, duration: 3–4s, delay: staggered } )`
- Use `clearProps` to reset after each batch completes → reuse pool

### 2. Beat-Synced Splash

**Strategy:** Web Audio API → AnalyserNode → amplitude analysis → drive splash particle bursts.

The existing sound wave already has 64 bars driven by Web Audio. Reuse the same analyser to also drive splash particles.

**Implementation:**
1. Create a `<canvas>` overlay positioned above everything
2. On each animation frame, sample `analyser.getByteFrequencyData()`
3. Measure bass amplitude (bins 0–8 of 64)
4. When bass exceeds a threshold (e.g. >0.65) → emit a burst of splash particles from a random point on screen
5. Each splash: 8–15 small circles radiate outward, fade out over 0.4s with `ease: "power2.out"`

**Splash colors:** Match the music's frequency band colors — deep pink → orange → gold

---

## Step-by-Step Plan

### Step 1 — Update petal CSS `.petal`
- Shrink size: `width: 16px; height: 16px` (smaller → less color)
- Update color range: rotate through pink/yellow/white via `gsap.utils.random` hue 20–360
- Lower opacity: `0.3–0.5` via `gsap.utils.random`

### Step 2 — Rewrite JS to use 3-batch petal system
- Create 3 independent `createPetalBatch(count=730)` functions
- Each batch: loop + create div, `scene.appendChild()`, `gsap.set()` opacity:0, `gsap.from()` y:-120 → y:1180
- Stagger delay: `index * 0.0015` (ultra-fast spawn: 37/s across batches)
- Total petals: **2190** (3 × 730, close to 2200)

### Step 3 — Add beat-sensing canvas
- Insert `<canvas id="splash-canvas" width="1920" height="1080"></canvas>` before `#wave-container`
- CSS: `position: absolute; inset: 0; pointer-events: none; z-index: 10`
- JS: create `AudioContext` + `AnalyserNode`
- `getByteFrequencyData()` sampled at 30fps via `requestAnimationFrame`
- Bass threshold detection: if avg(bins 0–8) > 160 (of 255) → `emitSplash()`
- `emitSplash(x, y, color)`: draws 10–15 circles, each moves outward (`x += dx * 60`, `y += dy * 60`), fades opacity to 0 over 0.35s

### Step 4 — Add MP3 audio playback via Web Audio API
- Load `assets/audio.mp3` via `fetch()` → `arrayBuffer` → `decodeAudioData()`
- Connect: `source → analyser → gain → destination`
- Analyser config: `fftSize: 128`, `smoothingTimeConstant: 0.75`
- Start playback: `source.start(0)`

### Step 5 — Integrate beat detection into sound wave
- The existing `tl.call(...)` frame-by-frame wave animation is incompatible with real audio analysis
- Replace with `requestAnimationFrame` loop that simultaneously drives both the wave DOM bars and the splash canvas
- Pause/disable the GSAP timeline wave updates; control bar heights directly from analyser data

### Step 6 — Lint, inspect, render
```bash
cd ~/AppData/Local/hermes/cat-developer-vibes
npx hyperframes lint
npx hyperframes inspect
npx hyperframes render --output output/cozy-code-sakura-v3.mp4 --fps 30 --quality draft
```

### Step 7 — Verify + send to Telegram
- Check file size (expect ~22–25 MB)
- Send preview text + video to `-1003973934522:2061`

---

## Files Likely to Change

| File | Change |
|------|--------|
| `index.html` | Full rewrite: petal system (3 batches), canvas splash, Web Audio API, beat detection |
| `output/cozy-code-sakura-v3.mp4` | New render output |

---

## Tests / Validation

- HyperFrames lint: `0 error(s)` (warnings OK)
- HyperFrames inspect: `0 layout issues`
- Video plays 60s without crash or freeze
- Petals visibly falling (≠ static)
- Splash particles visibly reactive to audio beats (≥1 burst per 3s on average)
- Color of petals: pink/yellow/white mix visible against background
- Audio plays throughout full 60s

---

## Risks & Tradeoffs

| Risk | Mitigation |
|------|-----------|
| 2200 DOM nodes causes browser crash during render | Use 3-batch system (730 DOM elements total, staggered via delay not nodes) |
| Audio playback in headless render environment | Wrap in `try/catch`; if audio fails, video still renders with visual fallback |
| Web Audio API conflicts with GSAP timeline | Disconnect GSAP wave updates; control via rAF only |
| Steam animation broken after JS change | Retain existing steam GSAP code unchanged |

## Open Questions

1. Steam positions (y=662) were set for the previous mug location — should steam be repositioned or removed?
2. Should the title float animation be kept, or disabled during the enhanced effects to reduce visual noise?
3. Quality: `--quality draft` vs `best` for v3? (draft = faster, ~5 min render vs ~10 min for best)
