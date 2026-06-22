# Analogue Timer Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a single self-contained `index.html` analogue timer with a minimalist clock face, winding interaction (drag + scroll), typed h/m input, and smooth countdown with a quiet expiry signal.

**Architecture:** A `<canvas>` element redraws every animation frame via `requestAnimationFrame`. Countdown uses `endTime = Date.now() + remainingSeconds * 1000` so the renderer computes the display angle directly from wall-clock time — no `setInterval` jitter, naturally smooth hand motion. State is a single plain object; all UI syncs from it on each draw.

**Tech Stack:** Vanilla HTML/CSS/JS, no dependencies, no build tools. Opens directly in a browser.

## Global Constraints

- Single file: `index.html` — inline `<style>` and `<script>` only
- No external fonts, libraries, or network requests
- No seconds displayed anywhere in the UI
- No audio
- Works by double-clicking `index.html` — no server required
- h input range: 0–99 · m input range: 0–59

---

## File Structure

```
index.html          ← entire application
docs/
  superpowers/
    specs/2026-06-23-analogue-timer-design.md
    plans/2026-06-23-analogue-timer.md
```

`index.html` is divided into clearly-commented sections in this order:

```
<style>  Layout · Face · Controls · Expiry animation
<script>
  // === Constants ===
  // === State ===
  // === Canvas setup ===
  // === Renderer ===
  // === RAF loop ===
  // === Wind interaction ===
  // === Input sync ===
  // === Control buttons ===
  // === Expiry handler ===
  // === Bootstrap ===
```

---

## Task 1: HTML scaffold + CSS

**Files:**
- Create: `index.html`

**Interfaces:**
- Produces: styled page shell with `#clock-wrapper > canvas#clock`, `#inp-h`, `#inp-m`, `#btn-start`, `#btn-pause`, `#btn-reset`

- [ ] **Step 1: Create `index.html` with full HTML structure and CSS**

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Timer</title>
  <style>
    *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

    body {
      min-height: 100vh;
      display: flex;
      align-items: center;
      justify-content: center;
      background: #f5f5f2;
      font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
    }

    #app {
      display: flex;
      flex-direction: column;
      align-items: center;
      gap: 28px;
    }

    #clock-wrapper {
      border-radius: 50%;
      box-shadow: 0 4px 24px rgba(0,0,0,0.08), 0 1px 4px rgba(0,0,0,0.05);
    }

    @keyframes pulse {
      0%   { opacity: 1; }
      40%  { opacity: 0.35; }
      100% { opacity: 1; }
    }

    #clock-wrapper.pulse {
      animation: pulse 0.9s ease-in-out 1;
    }

    canvas {
      display: block;
      border-radius: 50%;
      cursor: grab;
    }

    canvas:active { cursor: grabbing; }

    #inputs {
      display: flex;
      align-items: baseline;
      gap: 6px;
      color: #aaa;
      font-size: 13px;
      letter-spacing: 0.02em;
    }

    #inputs input {
      width: 52px;
      text-align: center;
      border: none;
      border-bottom: 1px solid #ddd;
      background: transparent;
      font-size: 22px;
      font-family: inherit;
      color: #333;
      padding: 2px 4px;
      outline: none;
      -moz-appearance: textfield;
    }

    #inputs input::-webkit-inner-spin-button,
    #inputs input::-webkit-outer-spin-button { -webkit-appearance: none; }

    #inputs input:focus { border-bottom-color: #aaa; }
    #inputs input:disabled { color: #bbb; border-bottom-color: #eee; }

    #controls {
      display: flex;
      gap: 16px;
    }

    #controls button {
      background: none;
      border: none;
      font-family: inherit;
      font-size: 12px;
      letter-spacing: 0.08em;
      text-transform: uppercase;
      color: #666;
      cursor: pointer;
      padding: 6px 14px;
      border-radius: 4px;
      transition: color 0.15s, background 0.15s;
    }

    #controls button:hover { color: #111; background: rgba(0,0,0,0.05); }
    #btn-start { color: #333; font-weight: 500; }
  </style>
</head>
<body>
  <div id="app">
    <div id="clock-wrapper">
      <canvas id="clock"></canvas>
    </div>
    <div id="inputs">
      <input type="number" id="inp-h" min="0" max="99" value="0">
      <span>h</span>
      <input type="number" id="inp-m" min="0" max="59" value="0">
      <span>m</span>
    </div>
    <div id="controls">
      <button id="btn-start">Start</button>
      <button id="btn-pause" hidden>Pause</button>
      <button id="btn-reset" hidden>Reset</button>
    </div>
  </div>
  <script>
    // === Constants ===
    // === State ===
    // === Canvas setup ===
    // === Renderer ===
    // === RAF loop ===
    // === Wind interaction ===
    // === Input sync ===
    // === Control buttons ===
    // === Expiry handler ===
    // === Bootstrap ===
  </script>
</body>
</html>
```

- [ ] **Step 2: Verify in browser**

Open `index.html`. Expected:
- Light warm-grey page background (`#f5f5f2`)
- A blank white circle centered on the page with a subtle shadow
- Below it: two empty number inputs labelled `h` and `m`, and a "Start" button
- Canvas has a grab cursor on hover

---

## Task 2: State + clock face renderer

**Files:**
- Modify: `index.html` — fill in the `// === Constants ===` through `// === RAF loop ===` sections

**Interfaces:**
- Consumes: `canvas#clock` DOM element
- Produces: `state` object · `draw()` function · `startRaf()` / `stopRaf()` functions · `getDisplaySeconds()` function

- [ ] **Step 1: Add Constants, State, Canvas setup, Renderer, RAF loop**

Replace the comment placeholders inside `<script>` with:

```js
// === Constants ===
const SIZE = 400;
const DPR  = Math.min(window.devicePixelRatio || 1, 2);
const CX   = SIZE / 2;
const CY   = SIZE / 2;

const FACE_R     = 170;
const TICK_OUTER = 160;
const TICK_SHORT =   5;
const TICK_LONG  =  11;
const HAND_LEN   = 132;
const HAND_TAIL  =  22;
const DOT_R      =   5;
const LABEL_R    = 138;

const C_FACE         = '#ffffff';
const C_TICK         = '#d4d4d4';
const C_TICK_5       = '#b0b0b0';
const C_LABEL        = '#999999';
const C_HAND         = '#1c1c1c';
const C_DOT_IDLE     = '#1c1c1c';
const C_DOT_EXPIRED  = '#c97a4a';

// === State ===
const state = {
  totalMinutes:     0,
  remainingSeconds: 0,
  status:           'idle',  // 'idle' | 'running' | 'paused' | 'expired'
  endTime:          null,    // Date.now() + remainingSeconds*1000 while running
};

// === Canvas setup ===
const canvas = document.getElementById('clock');
const ctx    = canvas.getContext('2d');
canvas.width  = SIZE * DPR;
canvas.height = SIZE * DPR;
canvas.style.width  = SIZE + 'px';
canvas.style.height = SIZE + 'px';
ctx.scale(DPR, DPR);

// === Renderer ===
function getDisplaySeconds() {
  if (state.status === 'running' && state.endTime !== null) {
    return Math.max(0, (state.endTime - Date.now()) / 1000);
  }
  return state.remainingSeconds;
}

function draw() {
  const dispSecs = getDisplaySeconds();
  const dispMins = dispSecs / 60;          // fractional minutes for smooth hand

  // Expiry detection (called from rAF loop while running)
  if (state.status === 'running' && dispSecs <= 0) {
    state.remainingSeconds = 0;
    state.status = 'expired';
    onExpiry();
  }

  ctx.clearRect(0, 0, SIZE, SIZE);

  // — Face —
  ctx.beginPath();
  ctx.arc(CX, CY, FACE_R, 0, Math.PI * 2);
  ctx.fillStyle = C_FACE;
  ctx.fill();

  // — Ticks + labels —
  for (let i = 0; i < 60; i++) {
    const ang    = (i / 60) * Math.PI * 2 - Math.PI / 2;  // 0 = 12 o'clock, CW
    const isFive = i % 5 === 0;
    const len    = isFive ? TICK_LONG : TICK_SHORT;
    const ox     = CX + Math.cos(ang) * TICK_OUTER;
    const oy     = CY + Math.sin(ang) * TICK_OUTER;
    const ix     = CX + Math.cos(ang) * (TICK_OUTER - len);
    const iy     = CY + Math.sin(ang) * (TICK_OUTER - len);

    ctx.beginPath();
    ctx.moveTo(ox, oy);
    ctx.lineTo(ix, iy);
    ctx.strokeStyle = isFive ? C_TICK_5 : C_TICK;
    ctx.lineWidth   = isFive ? 1.5 : 0.8;
    ctx.stroke();

    if (isFive) {
      // Label: 12 o'clock position (i=0) → "60", i=5 → "5", ..., i=55 → "55"
      const label = i === 0 ? '60' : String(i);
      const lx    = CX + Math.cos(ang) * LABEL_R;
      const ly    = CY + Math.sin(ang) * LABEL_R;
      ctx.font         = "10px -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif";
      ctx.fillStyle    = C_LABEL;
      ctx.textAlign    = 'center';
      ctx.textBaseline = 'middle';
      ctx.fillText(label, lx, ly);
    }
  }

  // — Hand —
  const handAng = (dispMins % 60) / 60 * Math.PI * 2 - Math.PI / 2;
  const tipX  = CX + Math.cos(handAng) * HAND_LEN;
  const tipY  = CY + Math.sin(handAng) * HAND_LEN;
  const tailX = CX - Math.cos(handAng) * HAND_TAIL;
  const tailY = CY - Math.sin(handAng) * HAND_TAIL;

  ctx.beginPath();
  ctx.moveTo(tailX, tailY);
  ctx.lineTo(tipX, tipY);
  ctx.strokeStyle = C_HAND;
  ctx.lineWidth   = 1.5;
  ctx.lineCap     = 'round';
  ctx.stroke();

  // — Centre dot —
  ctx.beginPath();
  ctx.arc(CX, CY, DOT_R, 0, Math.PI * 2);
  ctx.fillStyle = state.status === 'expired' ? C_DOT_EXPIRED : C_DOT_IDLE;
  ctx.fill();

  // — Hour label (only when ≥ 60 minutes remain) —
  const wholeHours = Math.floor(dispMins / 60);
  if (wholeHours >= 1) {
    ctx.font         = "bold 13px -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif";
    ctx.fillStyle    = '#c0c0c0';
    ctx.textAlign    = 'center';
    ctx.textBaseline = 'middle';
    ctx.fillText(wholeHours + 'h', CX, CY - 32);
  }
}

// === RAF loop ===
let rafId = null;

function startRaf() {
  if (rafId) return;
  (function loop() {
    draw();
    rafId = requestAnimationFrame(loop);
  }());
}

function stopRaf() {
  if (rafId) { cancelAnimationFrame(rafId); rafId = null; }
}
```

- [ ] **Step 2: Add bootstrap call at the end of `<script>` (replace the `// === Bootstrap ===` comment)**

```js
// === Bootstrap ===
draw();
```

- [ ] **Step 3: Verify in browser**

Open `index.html`. Expected:
- White clock face with 60 tick marks; 5-minute ticks are visibly longer
- Labels 5, 10, 15 … 55 around the face, with "60" at 12 o'clock
- A thin needle pointing straight up (12 o'clock = 0 minutes)
- Small dark dot at the centre
- No "Xh" label visible (total is 0 minutes)

Set `state.totalMinutes = 90` in the browser console, then call `draw()`. Expected:
- The centre shows "1h"
- The hand points to the 30-minute position (6 o'clock)

---

## Task 3: Wind interaction

**Files:**
- Modify: `index.html` — fill in `// === Wind interaction ===` section

**Interfaces:**
- Consumes: `state`, `startRaf()`, `draw()`
- Produces: `setTotalMinutes(n)` function (used by Task 4 input sync)

- [ ] **Step 1: Add wind interaction inside `<script>`, replacing `// === Wind interaction ===`**

```js
// === Wind interaction ===
let dragging        = false;
let dragPrevAngle   = 0;
let dragStartMins   = 0;
let dragCumDelta    = 0;

function clockAngle(e) {
  // Returns angle 0..2π with 0 at 12 o'clock, positive clockwise
  const rect = canvas.getBoundingClientRect();
  const dx   = e.clientX - rect.left - rect.width  / 2;
  const dy   = e.clientY - rect.top  - rect.height / 2;
  return (Math.atan2(dx, -dy) + Math.PI * 2) % (Math.PI * 2);
}

function shortestDelta(from, to) {
  // Smallest signed angle difference between two clock angles
  let d = to - from;
  if (d >  Math.PI) d -= Math.PI * 2;
  if (d < -Math.PI) d += Math.PI * 2;
  return d;
}

function setTotalMinutes(mins) {
  const clamped           = Math.max(0, Math.round(mins));
  state.totalMinutes      = clamped;
  state.remainingSeconds  = clamped * 60;
  syncInputsFromState();   // defined in Task 4; safe to call before then (noop until defined)
}

canvas.addEventListener('mousedown', e => {
  if (state.status === 'running' || state.status === 'expired') return;
  dragging      = true;
  dragPrevAngle = clockAngle(e);
  dragStartMins = state.totalMinutes;
  dragCumDelta  = 0;
  startRaf();
});

document.addEventListener('mousemove', e => {
  if (!dragging) return;
  const ang    = clockAngle(e);
  const delta  = shortestDelta(dragPrevAngle, ang);
  dragCumDelta += delta;
  dragPrevAngle = ang;
  setTotalMinutes(dragStartMins + dragCumDelta / (Math.PI * 2) * 60);
});

document.addEventListener('mouseup', () => { dragging = false; });

canvas.addEventListener('wheel', e => {
  if (state.status === 'running' || state.status === 'expired') return;
  e.preventDefault();
  const step = e.shiftKey ? 5 : 1;             // Shift+scroll = 5-min jumps
  setTotalMinutes(state.totalMinutes + (e.deltaY < 0 ? step : -step));
  startRaf();
}, { passive: false });
```

- [ ] **Step 2: Verify in browser**

Open `index.html`.

Drag test:
- Click-drag clockwise from 12 o'clock — hand sweeps clockwise, minutes increase
- Drag counter-clockwise — hand returns, minutes decrease, stops at 0
- Drag past one full rotation — hand loops around; e.g. drag 1.5 rotations → hand at 6 o'clock, should show ≈90 total minutes (hand at 6 o'clock + "1h" label in centre)

Scroll test:
- Hover over clock and scroll up → hand moves clockwise 1 minute per scroll step
- Scroll down → decreases, stops at 0
- Shift+scroll → moves 5 minutes per step

---

## Task 4: Input sync

**Files:**
- Modify: `index.html` — fill in `// === Input sync ===` section

**Interfaces:**
- Consumes: `state`, `setTotalMinutes()`, `startRaf()`, `draw()`
- Produces: `syncInputsFromState()` (Task 3 calls this stub; now it becomes real)

- [ ] **Step 1: Add input sync replacing `// === Input sync ===`**

```js
// === Input sync ===
const inpH = document.getElementById('inp-h');
const inpM = document.getElementById('inp-m');

function syncInputsFromState() {
  inpH.value = Math.floor(state.totalMinutes / 60);
  inpM.value = state.totalMinutes % 60;
}

function syncStateFromInputs() {
  const h    = Math.max(0, Math.min(99, parseInt(inpH.value,  10) || 0));
  const m    = Math.max(0, Math.min(59, parseInt(inpM.value, 10) || 0));
  inpH.value = h;
  inpM.value = m;
  setTotalMinutes(h * 60 + m);
}

inpH.addEventListener('input', () => {
  if (state.status === 'running') return;
  syncStateFromInputs();
  startRaf();
});

inpM.addEventListener('input', () => {
  if (state.status === 'running') return;
  syncStateFromInputs();
  startRaf();
});
```

- [ ] **Step 2: Verify in browser**

Open `index.html`.

Input → clock:
- Type `1` in the `h` field → hand moves to 12 o'clock, "1h" appears in centre
- Type `30` in the `m` field → hand moves to 6 o'clock, centre still shows "1h" (total 90 min)
- Clear both to 0 → hand back to 12 o'clock, no "Xh" label

Clock → input:
- Wind the hand to some position (e.g. ~45 min) → `h` field shows `0`, `m` field shows `45`
- Wind past 60 minutes → `h` field shows `1`, `m` field shows the remaining minutes

---

## Task 5: Countdown engine + control buttons

**Files:**
- Modify: `index.html` — fill in `// === Control buttons ===` section; update `// === Bootstrap ===`

**Interfaces:**
- Consumes: `state`, `startRaf()`, `stopRaf()`, `draw()`, `syncInputsFromState()`, `onExpiry()` (defined Task 6 — safe forward ref)
- Produces: `updateButtons()` · `setInputsDisabled(bool)` (used by Task 6)

- [ ] **Step 1: Add control button logic replacing `// === Control buttons ===`**

```js
// === Control buttons ===
const btnStart = document.getElementById('btn-start');
const btnPause = document.getElementById('btn-pause');
const btnReset = document.getElementById('btn-reset');

function updateButtons() {
  const s = state.status;
  btnStart.hidden  = s === 'running' || s === 'expired';
  btnPause.hidden  = s !== 'running';
  btnReset.hidden  = s === 'idle';
  btnStart.textContent = s === 'paused' ? 'Resume' : 'Start';
}

function setInputsDisabled(disabled) {
  inpH.disabled = disabled;
  inpM.disabled = disabled;
}

btnStart.addEventListener('click', () => {
  if (state.remainingSeconds <= 0) return;         // nothing set
  state.status  = 'running';
  state.endTime = Date.now() + state.remainingSeconds * 1000;
  updateButtons();
  setInputsDisabled(true);
  startRaf();
});

btnPause.addEventListener('click', () => {
  state.remainingSeconds = Math.max(0, (state.endTime - Date.now()) / 1000);
  state.status  = 'paused';
  state.endTime = null;
  updateButtons();
  setInputsDisabled(false);
});

btnReset.addEventListener('click', () => {
  state.totalMinutes     = 0;
  state.remainingSeconds = 0;
  state.status           = 'idle';
  state.endTime          = null;
  document.getElementById('clock-wrapper').classList.remove('pulse');
  syncInputsFromState();
  updateButtons();
  setInputsDisabled(false);
  stopRaf();
  draw();
});
```

- [ ] **Step 2: Update Bootstrap to call `updateButtons()`**

Replace `// === Bootstrap ===` with:

```js
// === Bootstrap ===
updateButtons();
draw();
```

- [ ] **Step 3: Verify in browser**

Open `index.html`.

Start/Pause/Reset flow:
- Set 2 minutes via inputs or winding → "Start" button visible
- Click Start → button changes to "Pause", "Reset" appears, inputs grey out, hand begins sweeping counter-clockwise
- Click Pause → hand freezes, inputs re-enable, button shows "Resume"
- Adjust the time while paused → hand updates
- Click Resume → counts down from adjusted value
- Click Reset → hand returns to 12 o'clock, inputs clear to 0, only "Start" visible

Edge case: click Start with nothing set (0 minutes) → nothing happens (button stays, no countdown).

---

## Task 6: Expiry behaviour

**Files:**
- Modify: `index.html` — fill in `// === Expiry handler ===` section

**Interfaces:**
- Consumes: `state`, `stopRaf()`, `draw()`, `updateButtons()`, `setInputsDisabled()`
- Produces: `onExpiry()` (called by `draw()` in Task 2)

- [ ] **Step 1: Add expiry handler replacing `// === Expiry handler ===`**

```js
// === Expiry handler ===
function onExpiry() {
  state.endTime = null;
  updateButtons();
  setInputsDisabled(false);
  stopRaf();
  draw();                                   // final draw — dot is now amber

  const wrapper = document.getElementById('clock-wrapper');
  wrapper.classList.remove('pulse');
  void wrapper.offsetWidth;                 // force reflow so animation re-triggers
  wrapper.classList.add('pulse');
}
```

- [ ] **Step 2: Verify in browser**

Open `index.html`.

Set 1 minute, click Start, wait for the hand to reach 12 o'clock. Expected:
- The clock face dims briefly (single pulse, ~0.9s) then returns to full opacity
- The centre dot changes from dark (`#1c1c1c`) to warm amber (`#c97a4a`)
- "Pause" button disappears, only "Reset" remains
- Inputs re-enable
- Hand rests exactly at 12 o'clock

Reset and verify:
- Click Reset → dot returns to dark, "Reset" button hides, inputs clear, hand at 12 o'clock

---

## Self-Review Checklist

### Spec coverage

| Spec requirement | Task |
|---|---|
| Single HTML file, no deps | Task 1 |
| Minimalist white face, thin hand, subtle shadow | Task 1–2 |
| Tick marks, longer at 5-min, labels 5…60 | Task 2 |
| Winding via drag (CW = more, CCW = less) | Task 3 |
| Winding via scroll | Task 3 |
| Past-12 rotation increments/decrements hour counter | Task 3 (`dragCumDelta` accumulates across rotations) |
| Centre hour label when ≥ 60 min | Task 2 |
| h/m text inputs, live-update hand | Task 4 |
| Inputs sync from winding | Task 3 → `syncInputsFromState()` |
| h range 0–99, m range 0–59 | Task 4 |
| Idle/Running/Paused/Expired states | Task 5 |
| Start/Pause/Reset buttons, correct visibility | Task 5 |
| Winding/inputs disabled while running | Task 5 |
| Smooth countdown, no second ticks visible | Task 2 (`getDisplaySeconds` from `endTime`) |
| Single pulse animation on expiry | Task 6 (CSS keyframe, one iteration) |
| Centre dot turns amber on expiry | Task 2 (`C_DOT_EXPIRED` when `status === 'expired'`) |
| No audio | — (never added) |
| No seconds displayed | — (hand position derived from minutes fraction only) |
