# Analogue Timer (No Seconds) — Design Spec

**Date:** 2026-06-23  
**Status:** Approved

---

## Overview

A single-file HTML analogue timer. No build tools, no dependencies. Opens directly in a browser. The timer displays remaining time as a clock hand on a minimalist dial — no seconds shown. Time is set either by winding the hand with mouse/scroll, or by typing hours and minutes directly.

---

## Visual Design

- **Page:** Light near-white background, single centred clock face.
- **Face:** Clean circle, subtle drop shadow, no bezel texture.
- **Tick marks:** 60 thin marks around the inner edge, slightly longer at every 5-minute interval. Minute labels (5, 10, 15 … 60) in a light system sans-serif.
- **Hand:** Single slender tapered needle from the centre pivot to the tick ring. Points to remaining minutes.
- **Centre element:** A small typographic label showing remaining whole hours (e.g. `2h`) when ≥ 60 minutes remain. Hidden when under 60 minutes. A small centre dot whose colour changes on expiry.
- **Controls:** "Start", "Pause", and "Reset" text buttons below the face — minimal styling. `h` and `m` numeric input fields also below the face, clearly labelled.
- **Typography:** System sans-serif throughout. No external font dependency.

---

## Interaction

### Setting the timer

Two methods, both update the hand and input fields in real time:

**Winding (mouse drag):**
- Click-drag anywhere on the clock face.
- Drag clockwise → increases time; counter-clockwise → decreases time.
- Crossing the 12 o'clock boundary clockwise increments the hour counter; crossing counter-clockwise decrements it.
- Minimum is 0, no enforced maximum (supports many hours).

**Winding (scroll wheel):**
- Scroll up over the clock face → increases time (1 minute per scroll step).
- Scroll down → decreases time, minimum 0.

**Text input:**
- Two fields: `h` (hours, 0–99) and `m` (minutes, 0–59).
- Changing either field immediately updates the hand position and hour counter.
- Winding also updates these fields to stay in sync.

### Timer states

| State    | Hand       | Inputs/Winding | Controls shown         |
|----------|------------|----------------|------------------------|
| Idle     | Movable    | Enabled        | Start, Reset           |
| Running  | Sweeping   | Disabled       | Pause, Reset           |
| Paused   | Frozen     | Enabled        | Start (resume), Reset  |
| Expired  | At 12 o'clock | Disabled    | Reset                  |

### Countdown behaviour

- Driven by `setInterval` at 1-second resolution internally.
- The hand position is derived from remaining minutes only (no sub-minute hand movement visible to the user).
- Hand moves smoothly between minute positions using linear interpolation within each minute interval, so it doesn't snap once per minute.

### Expiry

- Hand reaches 12 o'clock (zero).
- Clock face wrapper plays a single soft opacity pulse (CSS keyframe, one iteration, no repeat).
- Centre dot transitions to a muted accent colour (e.g. warm amber).
- No audio. No repeated animation.

---

## Architecture

Single `index.html` file. Inline `<style>` and `<script>` blocks. Clock rendered on a `<canvas>` element redrawn each animation frame via `requestAnimationFrame`.

### State object

```
{
  totalMinutes: number,       // time set by user (hours * 60 + minutes)
  remainingSeconds: number,   // countdown in seconds
  status: 'idle' | 'running' | 'paused' | 'expired'
}
```

### Modules (comment-delimited sections in the JS block)

- **Clock renderer** — draws face, ticks, labels, hand, centre text each frame.
- **Wind interaction** — `mousedown`/`mousemove`/`mouseup`/`wheel` on canvas; converts pointer angle + rotation count to total minutes; syncs to state and input fields.
- **Input sync** — `input` event on h/m fields updates `totalMinutes` and `remainingSeconds` (when idle/paused); triggers redraw.
- **Countdown loop** — `setInterval` at 1000ms decrements `remainingSeconds`; fires expiry handler at 0.
- **Expiry handler** — sets status to `expired`, triggers CSS animation, updates centre dot.
- **Control buttons** — Start/Pause/Reset wire to state transitions and loop management.

---

## File Structure

```
index.html          ← entire app (HTML + CSS + JS, no external deps)
docs/
  superpowers/
    specs/
      2026-06-23-analogue-timer-design.md
```

---

## Non-goals

- No seconds display anywhere.
- No audio.
- No server required.
- No mobile-specific gesture handling (touch events are out of scope).
- No persistence (timer resets on page refresh).
