# Pointer Events — Advanced (setPointerCapture, Gestures)

## The Idea

**In plain English:** Pointer Events are a way for your webpage to listen and respond to any kind of touch or click — whether the user is using a mouse, a finger on a touchscreen, or a stylus pen. Advanced pointer events let you "lock on" to a dragging finger or cursor so the webpage never loses track of it, even if it moves outside a button or off to the side.

**Real-world analogy:** Imagine you are a bouncer at a nightclub door holding a wristband scanner. Once you scan a guest's wristband, you keep tracking that exact guest no matter where they move in the crowd — even if they step outside momentarily.

- The wristband scanner = the webpage element that calls `setPointerCapture`
- The guest's unique wristband ID = the `pointerId` (a number that identifies one specific finger or cursor)
- Tracking the guest no matter where they go = receiving all `pointermove` and `pointerup` events for that pointer, even outside the element

---

## Overview

The Pointer Events API unifies mouse, touch, and stylus input into a single event model. The basic events (`pointerdown`, `pointermove`, `pointerup`) are straightforward, but building robust draggable sliders, carousels, drawing apps, and custom drag-drop requires mastering pointer capture, coalesced events, predicted events, and multi-touch handling.

---

## Why Pointer Events Over Mouse/Touch Events

```javascript
// Bad: separate handlers for mouse and touch (common legacy pattern)
el.addEventListener('mousedown', startDrag);
el.addEventListener('touchstart', startDrag);
el.addEventListener('mousemove', onDrag);
el.addEventListener('touchmove', onDrag);
el.addEventListener('mouseup', endDrag);
el.addEventListener('touchend', endDrag);

// Good: one unified handler
el.addEventListener('pointerdown', startDrag);
el.addEventListener('pointermove', onDrag);
el.addEventListener('pointerup', endDrag);
el.addEventListener('pointercancel', endDrag); // e.g. scroll interrupts drag
```

Pointer Events also carry additional data unavailable from mouse/touch events: `pressure`, `tiltX`, `tiltY`, `twist`, `width`, `height` (contact area), and `tangentialPressure` — valuable for stylus and drawing apps.

---

## PointerEvent Properties

```javascript
element.addEventListener('pointermove', (e) => {
  e.pointerId;       // unique ID per active pointer (multi-touch: different IDs)
  e.pointerType;     // 'mouse' | 'touch' | 'pen'
  e.isPrimary;       // true for the first touch / mouse button

  // Position
  e.clientX; e.clientY;   // viewport-relative
  e.pageX;   e.pageY;     // document-relative
  e.offsetX; e.offsetY;   // element-relative
  e.screenX; e.screenY;   // screen-relative

  // Stylus / pen properties
  e.pressure;             // 0.0–1.0 (0 when not in contact)
  e.tiltX; e.tiltY;       // degrees: pen tilt from perpendicular (-90 to 90)
  e.twist;                // rotation of stylus (0–359 degrees)
  e.tangentialPressure;   // barrel-button pressure (-1 to 1)
  e.width; e.height;      // contact ellipse size (in CSS pixels)

  // Buttons
  e.button;   // which button triggered the event (same as MouseEvent)
  e.buttons;  // bitmask of currently pressed buttons
});
```

---

## setPointerCapture — The Critical API

`setPointerCapture` redirects all pointer events from a specific pointer ID to a target element, even when the pointer moves outside that element or the browser window. Without it, `pointermove` stops firing as soon as the cursor leaves the element.

This is what makes sliders, resizable panels, and drawing canvases work correctly.

```javascript
// Problem without capture: drag breaks when cursor leaves the element
slider.addEventListener('pointerdown', (e) => {
  // drag starts...
});
// If user moves mouse outside slider, pointermove fires on a different element
// or stops entirely — drag is broken.

// Solution: capture the pointer
slider.addEventListener('pointerdown', (e) => {
  slider.setPointerCapture(e.pointerId);
  // Now ALL pointermove/pointerup events for this pointerId go to `slider`
  // regardless of where the pointer is on screen.
});
```

### Complete Slider Implementation

```javascript
class RangeSlider {
  #el;
  #track;
  #thumb;
  #isDragging = false;
  #value = 0;

  constructor(container, { min = 0, max = 100, value = 0 } = {}) {
    this.#el = container;
    this.min = min;
    this.max = max;
    this.#value = value;
    this.#render();
    this.#attachEvents();
  }

  #render() {
    this.#el.innerHTML = `
      <div class="track" role="slider" 
           aria-valuemin="${this.min}" 
           aria-valuemax="${this.max}" 
           aria-valuenow="${this.#value}"
           tabindex="0">
        <div class="fill"></div>
        <div class="thumb"></div>
      </div>`;
    this.#track = this.#el.querySelector('.track');
    this.#thumb = this.#el.querySelector('.thumb');
    this.#updateUI();
  }

  #attachEvents() {
    this.#thumb.addEventListener('pointerdown', this.#onPointerDown);
    this.#thumb.addEventListener('pointermove', this.#onPointerMove);
    this.#thumb.addEventListener('pointerup',   this.#onPointerUp);
    this.#thumb.addEventListener('pointercancel', this.#onPointerUp);

    // Keyboard support
    this.#track.addEventListener('keydown', this.#onKeyDown);
  }

  #onPointerDown = (e) => {
    e.preventDefault(); // prevent text selection during drag
    this.#isDragging = true;
    this.#thumb.setPointerCapture(e.pointerId); // ← key line
  };

  #onPointerMove = (e) => {
    if (!this.#isDragging) return;

    const rect = this.#track.getBoundingClientRect();
    const ratio = Math.max(0, Math.min(1, (e.clientX - rect.left) / rect.width));
    this.#value = this.min + ratio * (this.max - this.min);
    this.#updateUI();
    this.#el.dispatchEvent(new CustomEvent('change', { detail: this.#value }));
  };

  #onPointerUp = (e) => {
    this.#isDragging = false;
    this.#thumb.releasePointerCapture(e.pointerId);
  };

  #onKeyDown = (e) => {
    const step = (this.max - this.min) / 100;
    if (e.key === 'ArrowRight' || e.key === 'ArrowUp')
      this.#value = Math.min(this.max, this.#value + step);
    else if (e.key === 'ArrowLeft' || e.key === 'ArrowDown')
      this.#value = Math.max(this.min, this.#value - step);
    else return;
    this.#updateUI();
  };

  #updateUI() {
    const pct = ((this.#value - this.min) / (this.max - this.min)) * 100;
    this.#thumb.style.left = `${pct}%`;
    this.#el.querySelector('.fill').style.width = `${pct}%`;
    this.#track.setAttribute('aria-valuenow', Math.round(this.#value));
  }
}
```

---

## Resizable Panel

```javascript
function makeResizable(panel, handle) {
  let startX, startWidth;

  handle.addEventListener('pointerdown', (e) => {
    startX     = e.clientX;
    startWidth = panel.offsetWidth;
    handle.setPointerCapture(e.pointerId); // capture so resize works outside panel
    panel.style.userSelect = 'none'; // prevent text selection during drag
  });

  handle.addEventListener('pointermove', (e) => {
    if (!handle.hasPointerCapture(e.pointerId)) return;
    const delta = e.clientX - startX;
    panel.style.width = `${Math.max(200, startWidth + delta)}px`;
  });

  handle.addEventListener('pointerup', (e) => {
    handle.releasePointerCapture(e.pointerId);
    panel.style.userSelect = '';
  });
}
```

---

## Coalesced and Predicted Events

At 60fps, `pointermove` fires up to 60 times per second. But a stylus or high-DPI touch screen can generate 250+ position samples per second. The browser coalesces intermediate positions into a single event to avoid flooding the main thread.

`getCoalescedEvents()` lets you access all intermediate positions for high-fidelity drawing:

```javascript
canvas.addEventListener('pointermove', (e) => {
  const ctx = canvas.getContext('2d');

  // Standard: only the final position (may skip intermediate points)
  // drawLine(lastX, lastY, e.clientX, e.clientY);

  // High-fidelity: all intermediate positions since last event
  const events = e.getCoalescedEvents();
  for (const coalesced of events) {
    drawLine(lastX, lastY, coalesced.clientX, coalesced.clientY);
    lastX = coalesced.clientX;
    lastY = coalesced.clientY;
  }
});
```

`getPredictedEvents()` provides browser-predicted future positions to reduce perceived latency in drawing apps:

```javascript
canvas.addEventListener('pointermove', (e) => {
  // Draw actual coalesced path
  for (const c of e.getCoalescedEvents()) { /* ... */ }

  // Draw predicted path (lighter, will be overwritten on next event)
  ctx.globalAlpha = 0.3;
  for (const p of e.getPredictedEvents()) {
    drawLine(lastX, lastY, p.clientX, p.clientY);
  }
  ctx.globalAlpha = 1.0;
});
```

---

## Multi-Touch Handling

Each active touch point has a unique `pointerId`. Track them in a Map:

```javascript
const activePointers = new Map(); // pointerId → { x, y }

canvas.addEventListener('pointerdown', (e) => {
  canvas.setPointerCapture(e.pointerId);
  activePointers.set(e.pointerId, { x: e.clientX, y: e.clientY });
});

canvas.addEventListener('pointermove', (e) => {
  if (!activePointers.has(e.pointerId)) return;
  activePointers.set(e.pointerId, { x: e.clientX, y: e.clientY });

  if (activePointers.size === 2) {
    // Pinch-to-zoom: calculate distance between two touches
    const [p1, p2] = [...activePointers.values()];
    const distance = Math.hypot(p2.x - p1.x, p2.y - p1.y);
    applyZoom(distance);
  }
});

canvas.addEventListener('pointerup', (e) => {
  activePointers.delete(e.pointerId);
});
canvas.addEventListener('pointercancel', (e) => {
  activePointers.delete(e.pointerId);
});
```

---

## touch-action CSS — Delegating Scroll to Browser

When handling pointer events, tell the browser which gestures you're handling so it can still handle the rest natively (e.g., vertical scroll):

```css
/* Handle horizontal drag yourself; let browser handle vertical scroll */
.carousel {
  touch-action: pan-y;
}

/* Handle everything yourself (disable browser panning/zooming) */
.drawing-canvas {
  touch-action: none;
}

/* Disable pinch-zoom but allow panning */
.map {
  touch-action: pan-x pan-y;
}
```

If `touch-action` is not set, the browser may cancel the pointer sequence (firing `pointercancel`) when it detects a scroll gesture — breaking your drag logic.

---

## hasPointerCapture

Check if an element currently has capture for a given pointer:

```javascript
handle.addEventListener('pointermove', (e) => {
  // Guard against stale events delivered before capture is established
  if (!handle.hasPointerCapture(e.pointerId)) return;
  // ... resize logic
});
```

---

## Interview Questions

**Q: What is `setPointerCapture` and when do you need it?**
A: It routes all future pointer events for a specific pointer ID to the target element, even when the pointer moves outside it or over other elements. You need it whenever a drag operation must continue outside the originating element — sliders, resize handles, drawing canvases, custom drag-and-drop. Without it, `pointermove` stops firing once the cursor leaves the element.

**Q: Why use Pointer Events over separate mouse and touch events?**
A: Pointer Events unify all input types (mouse, touch, stylus) into one event model, eliminating duplicated handlers. They also carry richer data: pressure, tilt, twist, and contact size — unavailable from mouse events. `setPointerCapture` has no equivalent in the Touch Events API.

**Q: What are coalesced events and when do they matter?**
A: The browser batches rapid `pointermove` events (e.g., 250Hz stylus input) into one event per frame to avoid flooding the main thread. `getCoalescedEvents()` exposes all intermediate positions that were merged. For drawing applications, skipping intermediate positions creates jagged lines — coalesced events provide the full high-fidelity path.

**Q: What does `touch-action: none` do?**
A: It tells the browser you're handling all pointer gestures yourself — disabling native browser pan and zoom for that element. This prevents `pointercancel` from firing when the browser detects a scroll gesture, which would otherwise interrupt a drag sequence. For elements with custom scroll or pan behavior, set the most permissive `touch-action` that still allows the browser to handle what you don't.
