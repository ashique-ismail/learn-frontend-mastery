# Reduced Motion Preferences

## The Idea

**In plain English:** Some people get dizzy, nauseous, or can even have seizures when they see things moving on a screen. Reduced motion preferences let a user tell their computer "please show me fewer animations," and websites can listen to that request and tone down or remove their moving effects.

**Real-world analogy:** Imagine a movie theater that offers two versions of a film — the full version with all the shaky-cam action scenes, and a "calm cut" where those intense sequences are replaced with simple still shots. The theater asks each guest at the door which version they prefer, and puts them in the right screening room automatically.

- The guest's preference card at the door = the `prefers-reduced-motion` OS setting
- The theater checking which room to send the guest to = the CSS `@media (prefers-reduced-motion: reduce)` query
- Swapping the shaky-cam scenes for still shots = replacing or removing CSS/JS animations in code

---

## Overview

Vestibular disorders, epilepsy, and motion sensitivity affect a significant portion of users — WCAG estimates around 35% of people have some sensitivity to motion. Animations that feel delightful on a desktop can trigger vertigo, nausea, migraines, or seizures for these users. The `prefers-reduced-motion` media query lets users signal their preference through OS settings, and CSS/JavaScript can honor it. This guide covers the WCAG criteria, implementation patterns, and the difference between "remove animation" and "provide a safe alternative."

## Understanding Motion Sensitivity

```
Who is affected by motion:

Vestibular disorders:
  - BPPV (benign paroxysmal positional vertigo)
  - Labyrinthitis, Ménière's disease
  - Symptoms: vertigo, nausea, disorientation triggered by visual motion

Photosensitive epilepsy:
  - Flashing content > 3Hz can trigger seizures
  - WCAG 2.3.1 Success Criterion (Level A)
  - No exemption for prefers-reduced-motion — flashing must be avoided

Vestibular migraine:
  - Large-scale movement, parallax, zoom effects trigger episodes

Cognitive effects:
  - Attention disorders — animation distracts from content
  - Anxiety disorders — unexpected movement increases distress
```

## The prefers-reduced-motion Media Query

Users set their preference in OS accessibility settings:
- macOS: System Preferences → Accessibility → Display → Reduce Motion
- Windows: Settings → Ease of Access → Display → Show animations
- iOS: Settings → Accessibility → Motion → Reduce Motion
- Android: Settings → Accessibility → Remove Animations

```css
/* Default: animations ON */
.animated-element {
  transition: transform 0.4s ease;
  animation: fadeSlideIn 0.6s ease;
}

/* Reduced motion: remove or simplify */
@media (prefers-reduced-motion: reduce) {
  .animated-element {
    transition: none;
    animation: none;
  }
}
```

## Values

```css
/* prefers-reduced-motion: no-preference
   User has not expressed a preference, or system doesn't report one
   → Play animations normally */

/* prefers-reduced-motion: reduce
   User has requested minimal motion
   → Remove or substantially reduce animation */
```

## CSS Implementation Patterns

### Pattern 1: Disable All Animations Globally

```css
/* Blunt but safe — applies to everything */
@media (prefers-reduced-motion: reduce) {
  *,
  *::before,
  *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}
```

Note: Use `0.01ms` instead of `0` — some animations use `animationend` events that may not fire if duration is exactly 0.

### Pattern 2: Opt-In Animations (Mobile-First Reduced Motion)

```css
/* Default: no animations (opt-in approach) */
.slide-in {
  opacity: 1;
  transform: none;
}

/* Animation only for users who haven't opted out */
@media (prefers-reduced-motion: no-preference) {
  .slide-in {
    animation: slideIn 0.5s ease forwards;
  }
}

/* This pattern is more intentional — you choose which animations to enable */
```

### Pattern 3: Replace Motion with Opacity

```css
/* Preserve the feedback, remove the motion */
.button:active {
  transform: scale(0.95);  /* provides tactile feedback */
  transition: transform 0.1s;
}

@media (prefers-reduced-motion: reduce) {
  .button:active {
    transform: none;
    /* Replace motion feedback with opacity change */
    opacity: 0.7;
    transition: opacity 0.1s;
  }
}
```

### Pattern 4: Parallax and Scroll Effects

```css
/* Parallax scroll effect */
.hero-image {
  transform: translateY(var(--scroll-offset));
  transition: transform 0.1s linear;
}

@media (prefers-reduced-motion: reduce) {
  .hero-image {
    transform: none;          /* Static — no parallax */
    transition: none;
  }
}

/* Large-scale page transitions */
.page-enter {
  opacity: 0;
  transform: translateY(30px);
}

.page-enter-active {
  opacity: 1;
  transform: translateY(0);
  transition: opacity 0.4s, transform 0.4s;
}

@media (prefers-reduced-motion: reduce) {
  .page-enter {
    opacity: 0;
    transform: none;        /* No slide — just fade */
  }

  .page-enter-active {
    opacity: 1;
    transition: opacity 0.2s;   /* Fade only — no movement */
  }
}
```

## WCAG Success Criteria

```
WCAG 2.1 — Animation from Interactions (2.3.3) [Level AAA]:
  Motion animation triggered by interaction can be disabled
  unless the animation is essential

WCAG 2.3.1 — Three Flashes or Below Threshold [Level A]:
  No content flashes more than 3 times per second
  OR flashing is below the general flash threshold
  This criterion has NO reduced-motion exception — always required

WCAG 1.4.5 — Images of Text [Level AA]:
  (Related: animated text may fail this at certain speeds)

WCAG 2.1.1 — Keyboard [Level A]:
  All animation-triggered content must be keyboard accessible
```

### What "Essential" Means

```
Essential animation = no substitute conveys the same information:

ESSENTIAL:
  ✓ Loading spinner (communicates that work is happening)
  ✓ Video content (the animation IS the content)
  ✓ Progress bar filling (conveys completion state)

NOT ESSENTIAL:
  ✗ Decorative background animations
  ✗ Page transition effects (the page loads regardless)
  ✗ Hover animation feedback (state change can be communicated via color/opacity)
  ✗ Auto-playing decorative GIFs
  ✗ Parallax scrolling
```

## JavaScript Implementation

### Reading the Preference in JavaScript

```typescript
// Check current preference
const prefersReducedMotion = window.matchMedia('(prefers-reduced-motion: reduce)');

if (prefersReducedMotion.matches) {
  // Disable JavaScript-driven animation
  disableAnimations();
} else {
  initAnimations();
}

// Listen for preference changes (user toggles during session)
prefersReducedMotion.addEventListener('change', (e) => {
  if (e.matches) {
    disableAnimations();
  } else {
    initAnimations();
  }
});
```

### React Hook

```typescript
import { useState, useEffect } from 'react';

function useReducedMotion(): boolean {
  const [prefersReducedMotion, setPrefersReducedMotion] = useState<boolean>(
    // SSR-safe initial value
    typeof window !== 'undefined'
      ? window.matchMedia('(prefers-reduced-motion: reduce)').matches
      : false
  );

  useEffect(() => {
    const mediaQuery = window.matchMedia('(prefers-reduced-motion: reduce)');

    const handleChange = (event: MediaQueryListEvent) => {
      setPrefersReducedMotion(event.matches);
    };

    mediaQuery.addEventListener('change', handleChange);
    setPrefersReducedMotion(mediaQuery.matches); // in case it changed before mount

    return () => mediaQuery.removeEventListener('change', handleChange);
  }, []);

  return prefersReducedMotion;
}

// Usage
function AnimatedHero() {
  const reducedMotion = useReducedMotion();

  return (
    <motion.div
      initial={{ opacity: 0, y: reducedMotion ? 0 : 30 }}
      animate={{ opacity: 1, y: 0 }}
      transition={{
        duration: reducedMotion ? 0.01 : 0.5,
        ease: 'easeOut',
      }}
    >
      <h1>Welcome</h1>
    </motion.div>
  );
}
```

### Framer Motion Integration

```typescript
import { motion, useReducedMotion } from 'framer-motion';

// Framer Motion provides useReducedMotion built-in
function Card({ children }: { children: React.ReactNode }) {
  const shouldReduceMotion = useReducedMotion();

  const variants = {
    hidden: {
      opacity: 0,
      // Only use y transform if motion is acceptable
      y: shouldReduceMotion ? 0 : 20,
    },
    visible: {
      opacity: 1,
      y: 0,
      transition: {
        duration: shouldReduceMotion ? 0.1 : 0.4,
      },
    },
  };

  return (
    <motion.div
      variants={variants}
      initial="hidden"
      animate="visible"
    >
      {children}
    </motion.div>
  );
}
```

### GSAP Integration

```typescript
import gsap from 'gsap';
import ScrollTrigger from 'gsap/ScrollTrigger';

const prefersReducedMotion = window.matchMedia('(prefers-reduced-motion: reduce)').matches;

if (!prefersReducedMotion) {
  // Complex scroll-driven animations
  gsap.registerPlugin(ScrollTrigger);

  gsap.fromTo('.hero-text', {
    opacity: 0,
    y: 60,
  }, {
    opacity: 1,
    y: 0,
    duration: 0.8,
    scrollTrigger: {
      trigger: '.hero',
      start: 'top 80%',
    },
  });
} else {
  // Just make content visible — no animation
  gsap.set('.hero-text', { opacity: 1, y: 0 });
}
```

## Safe Animation Alternatives

```
What to replace specific animations with:

SCALE/ZOOM (triggers vestibular):
  → opacity transition only
  → none (no animation needed for decorative zoom)

PARALLAX (large-scale movement):
  → static background
  → none

SPINNING/ROTATION:
  → opacity pulse (if loading indicator)
  → static icon with text label

FLY-IN / SLIDE-IN (large displacement):
  → fade in (opacity only)
  → instant appearance

AUTO-PLAYING VIDEO/GIF:
  → paused by default, play button
  → static image

BOUNCE/ELASTIC:
  → simple linear transition or none
  → immediate state change

RIPPLE EFFECTS:
  → color/background-color change on click
  → outline change
```

## CSS Custom Property Approach

```css
/* Define animation duration via custom property — easy global control */
:root {
  --animation-duration: 0.4s;
  --transition-duration: 0.2s;
}

@media (prefers-reduced-motion: reduce) {
  :root {
    --animation-duration: 0.01ms;
    --transition-duration: 0.01ms;
  }
}

/* Components use the custom property */
.button {
  transition: background-color var(--transition-duration) ease;
}

.modal {
  animation: fadeIn var(--animation-duration) ease;
}

/* Can also be controlled via JavaScript for user preference UI */
document.documentElement.style.setProperty('--animation-duration', '0.01ms');
```

## Testing Reduced Motion

### Chrome DevTools

```
Chrome DevTools → Rendering panel → Emulate CSS media feature:
  prefers-reduced-motion: reduce
```

### Automated Testing

```typescript
// Playwright — test that reduced motion preference is honored
import { test, expect } from '@playwright/test';

test('hero animation respects reduced motion', async ({ page }) => {
  // Emulate prefers-reduced-motion: reduce
  await page.emulateMedia({ reducedMotion: 'reduce' });
  await page.goto('/');

  const heroText = page.locator('.hero-text');

  // With reduced motion, element should be immediately visible (not animated in)
  await expect(heroText).toBeVisible();

  // Verify no transform animation is applied
  const transform = await heroText.evaluate(
    (el) => window.getComputedStyle(el).animationDuration
  );
  expect(parseFloat(transform)).toBeLessThanOrEqual(0.01);
});

test('loading spinner still present with reduced motion', async ({ page }) => {
  await page.emulateMedia({ reducedMotion: 'reduce' });
  await page.goto('/heavy-page');

  // Spinner should still be visible (essential animation)
  const spinner = page.locator('[role="status"]');
  await expect(spinner).toBeVisible();
});
```

## Common Mistakes

### 1. Only Testing on Fast Machines/Networks

```
Animations that look fine at 60fps on a MacBook Pro can cause
issues on a slow Android phone that drops frames.
Animation lag can feel like motion to users with vestibular disorders.

Test on real devices at reduced performance.
```

### 2. Using 0ms Duration Instead of 0.01ms

```css
/* ❌ Exact 0 — animationend may not fire, causing logic bugs */
@media (prefers-reduced-motion: reduce) {
  .animated { animation-duration: 0ms; }
}

/* ✅ Near-zero — fires events, visually instant */
@media (prefers-reduced-motion: reduce) {
  .animated { animation-duration: 0.01ms; }
}
```

### 3. Removing Animations That Communicate State

```css
/* ❌ Removing a loading spinner entirely */
@media (prefers-reduced-motion: reduce) {
  .spinner { display: none; }
  /* Now users with reduced motion can't tell if the page is loading */
}

/* ✅ Replace motion with static indicator */
@media (prefers-reduced-motion: reduce) {
  .spinner {
    animation: none;     /* no spin */
    opacity: 0.6;        /* visible but static */
  }
}
```

### 4. Forgetting JavaScript Animations

```typescript
// ❌ CSS reduced motion applied, but JS animation ignores it
class ParticleSystem {
  start() {
    this.raf = requestAnimationFrame(() => this.update());
  }
}

// ✅ Check preference before starting JS animations
class ParticleSystem {
  start() {
    if (window.matchMedia('(prefers-reduced-motion: reduce)').matches) {
      this.showStaticFallback();
      return;
    }
    this.raf = requestAnimationFrame(() => this.update());
  }
}
```

## Interview Questions

### 1. What is prefers-reduced-motion and which users benefit from it?

**Answer:** `prefers-reduced-motion` is a CSS media query that reflects the user's OS-level accessibility setting for reduced animation. Users who benefit include those with vestibular disorders (BPPV, labyrinthitis, Ménière's disease) where visual motion triggers vertigo and nausea; photosensitive epilepsy (though flashing above 3Hz must be avoided regardless of the preference); vestibular migraine, where parallax and zoom trigger episodes; and cognitive/attention conditions where motion is distracting. It's set in OS accessibility settings and propagates through the browser's media query system.

### 2. What is the difference between WCAG 2.3.1 and 2.3.3 regarding motion?

**Answer:** WCAG 2.3.1 (Three Flashes or Below Threshold) is Level A — the minimum — and prohibits content that flashes more than 3 times per second. This applies unconditionally regardless of any user preference; no reduced-motion exception exists. WCAG 2.3.3 (Animation from Interactions) is Level AAA and requires that motion animation triggered by user interaction can be disabled, unless the animation is essential. "Essential" means no non-animated alternative conveys the same information. Meeting AAA isn't required by most standards, but 2.3.1 (no flashing) is always required.

### 3. Should you remove animations entirely or provide alternatives?

**Answer:** Provide alternatives where the animation serves a purpose, remove where it's purely decorative. A loading spinner communicates "work is happening" — removing it entirely leaves users uncertain; replace the rotation with a static visual indicator. A page slide-in transition is purely decorative — remove it, show the page immediately. A progress bar animation communicates completion state — keep the bar, remove the smooth fill transition (jump to final value). The guiding question: does the animation communicate information, or is it decoration? Information must be preserved; decoration can be removed.

### 4. How would you implement a user-facing "reduce motion" toggle in the UI?

**Answer:** Respect `prefers-reduced-motion` as the default, then allow in-app override stored in `localStorage`. Apply by toggling a class on `<html>` or updating a CSS custom property (`--animation-duration`). Sync the UI toggle to the current OS preference on first visit. Read preference on page load via both `localStorage` and `matchMedia` — use localStorage override if set, otherwise respect the OS preference. The in-app toggle is a progressive enhancement — users who didn't know about the OS setting or who want different behavior for your specific app get explicit control.
