# WCAG conformance

## Overview

Web Content Accessibility Guidelines (WCAG) are international standards for web accessibility. WCAG 2.1 is the current standard, with WCAG 2.2 adding new criteria and WCAG 3.0 in development.

## Conformance levels

### Level A (Minimum)

Essential accessibility features. Severe barriers if not met.

### Level AA (Mid-range)

Recommended target for most websites. Required by many laws.

### Level AAA (Highest)

Enhanced accessibility. Not possible to satisfy for all content.

**Target:** Most websites should aim for **Level AA conformance**.

## Four principles (POUR)

### 1. Perceivable

Information and UI must be presentable to users in ways they can perceive.

### 2. Operable

UI components and navigation must be operable.

### 3. Understandable

Information and UI operation must be understandable.

### 4. Robust

Content must be robust enough to work with current and future technologies.

## WCAG 2.1 guidelines summary

### Perceivable

#### 1.1 Text alternatives

**1.1.1 Non-text Content (A)**

```html
<!-- Images -->
<img src="chart.jpg" alt="Sales increased 25% in Q4">

<!-- Decorative images -->
<img src="decoration.svg" alt="" role="presentation">

<!-- Functional images -->
<button aria-label="Close">
  <img src="close.svg" alt="">
</button>

<!-- Form controls -->
<input type="text" aria-label="Search">
```

#### 1.2 Time-based media

**1.2.1 Audio-only and Video-only (A)**

```html
<!-- Audio with transcript -->
<audio controls src="podcast.mp3"></audio>
<details>
  <summary>Transcript</summary>
  <p>Full transcript text...</p>
</details>
```

**1.2.2 Captions (A)**

```html
<video controls>
  <source src="video.mp4" type="video/mp4">
  <track kind="captions" src="captions-en.vtt" srclang="en" label="English">
</video>
```

**1.2.3 Audio Description or Media Alternative (A)**

**1.2.4 Captions (Live) (AA)**

**1.2.5 Audio Description (AA)**

```html
<video controls>
  <source src="video.mp4" type="video/mp4">
  <track kind="descriptions" src="descriptions.vtt" srclang="en">
  <track kind="captions" src="captions.vtt" srclang="en">
</video>
```

#### 1.3 Adaptable

**1.3.1 Info and Relationships (A)**

```html
<!-- Proper semantic structure -->
<nav aria-label="Main navigation">
  <ul>
    <li><a href="/">Home</a></li>
  </ul>
</nav>

<!-- Proper form labels -->
<label for="email">Email</label>
<input type="email" id="email">

<!-- Data tables -->
<table>
  <caption>Sales Data</caption>
  <thead>
    <tr>
      <th scope="col">Month</th>
      <th scope="col">Sales</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>January</td>
      <td>$10,000</td>
    </tr>
  </tbody>
</table>
```

**1.3.2 Meaningful Sequence (A)**

```html
<!-- DOM order matches visual order -->
<article>
  <h1>Article Title</h1>
  <p>Introduction...</p>
  <h2>Section 1</h2>
  <p>Content...</p>
</article>
```

**1.3.3 Sensory Characteristics (A)**

```html
<!-- Bad: relies only on visual -->
<p>Click the green button on the right</p>

<!-- Good: multiple cues -->
<p>Click the "Submit" button (green, on the right) to continue</p>
```

**1.3.4 Orientation (AA) - WCAG 2.1**

Don't lock content to single orientation unless essential.

```css
/* Don't prevent orientation changes */
/* Bad: */
@media (orientation: portrait) {
  body { transform: rotate(90deg); }
}
```

**1.3.5 Identify Input Purpose (AA) - WCAG 2.1**

```html
<input type="text" name="name" autocomplete="name">
<input type="email" name="email" autocomplete="email">
<input type="tel" name="phone" autocomplete="tel">
<input type="text" name="address" autocomplete="street-address">
```

#### 1.4 Distinguishable

**1.4.1 Use of Color (A)**

```html
<!-- Bad: color only -->
<p style="color: red;">Required field</p>

<!-- Good: color + text/icon -->
<p style="color: red;">* Required field</p>
```

**1.4.2 Audio Control (A)**

```html
<audio src="background.mp3" autoplay>
  <!-- Must provide pause/stop control -->
</audio>
<button>Pause background audio</button>
```

**1.4.3 Contrast (Minimum) (AA)**

- Normal text: 4.5:1
- Large text (18pt+): 3:1
- UI components: 3:1

```css
/* Good contrast examples */
.text {
  color: #333333; /* Dark gray */
  background: #FFFFFF; /* White */
  /* Contrast: 12.6:1 */
}

.button {
  color: #FFFFFF;
  background: #0066CC; /* Blue */
  /* Contrast: 6.9:1 */
}
```

**1.4.4 Resize Text (AA)**

```html
<!-- Text must be resizable to 200% without loss of function -->
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<!-- Don't use user-scalable=no -->
```

**1.4.5 Images of Text (AA)**

```html
<!-- Bad: text in image -->
<img src="heading.png" alt="Welcome">

<!-- Good: real text -->
<h1>Welcome</h1>
```

**1.4.10 Reflow (AA) - WCAG 2.1**

Content reflows to 320px width without horizontal scrolling.

```css
/* Responsive design */
.container {
  max-width: 100%;
  padding: 1rem;
}
```

**1.4.11 Non-text Contrast (AA) - WCAG 2.1**

UI components and graphics have 3:1 contrast.

```css
/* Form input borders */
input {
  border: 2px solid #767676; /* 4.5:1 contrast */
}

/* Focus indicators */
button:focus {
  outline: 2px solid #0066CC; /* Sufficient contrast */
}
```

**1.4.12 Text Spacing (AA) - WCAG 2.1**

Content adapts to text spacing changes:

```css
* {
  line-height: 1.5;
  letter-spacing: 0.12em;
  word-spacing: 0.16em;
}

p {
  margin-bottom: 2em;
}
```

**1.4.13 Content on Hover or Focus (AA) - WCAG 2.1**

```html
<!-- Dismissible tooltip -->
<button aria-describedby="tooltip">Help</button>
<div role="tooltip" id="tooltip">
  Help text
  <button aria-label="Close">×</button>
</div>
```

### Operable

#### 2.1 Keyboard accessible

**2.1.1 Keyboard (A)**

```html
<!-- All functionality available via keyboard -->
<button onclick="submit()">Submit</button>

<!-- Custom controls keyboard accessible -->
<div 
  role="button" 
  tabindex="0"
  onclick="handleClick()"
  onkeydown="handleKeyDown(event)">
  Custom Button
</div>
```

**2.1.2 No Keyboard Trap (A)**

```javascript
// Modal focus trap with escape
document.addEventListener('keydown', (e) => {
  if (e.key === 'Escape' && modalOpen) {
    closeModal();
  }
});
```

**2.1.4 Character Key Shortcuts (A) - WCAG 2.1**

Single character shortcuts must be able to be turned off, remapped, or only active on focus.

#### 2.2 Enough time

**2.2.1 Timing Adjustable (A)**

```html
<!-- Session timeout warning -->
<div role="alert">
  <p>Your session will expire in 2 minutes.</p>
  <button>Extend session</button>
</div>
```

**2.2.2 Pause, Stop, Hide (A)**

```html
<!-- Auto-playing carousel with controls -->
<div class="carousel">
  <button aria-label="Pause carousel">⏸</button>
  <button aria-label="Play carousel">▶</button>
</div>
```

#### 2.3 Seizures and physical reactions

**2.3.1 Three Flashes or Below Threshold (A)**

Avoid content that flashes more than 3 times per second.

#### 2.4 Navigable

**2.4.1 Bypass Blocks (A)**

```html
<a href="#main-content" class="skip-link">Skip to main content</a>

<header>
  <nav><!-- Navigation --></nav>
</header>

<main id="main-content">
  <!-- Main content -->
</main>
```

**2.4.2 Page Titled (A)**

```html
<title>About Us - Company Name</title>
```

**2.4.3 Focus Order (A)**

Logical tab order matching visual order.

**2.4.4 Link Purpose (A)**

```html
<!-- Bad: ambiguous -->
<a href="/article">Click here</a>

<!-- Good: descriptive -->
<a href="/article">Read our accessibility guide</a>
```

**2.4.5 Multiple Ways (AA)**

Provide multiple navigation methods (menu, search, sitemap).

**2.4.6 Headings and Labels (AA)**

```html
<form>
  <h2>Contact Form</h2>
  
  <label for="name">Full Name</label>
  <input type="text" id="name">
  
  <label for="email">Email Address</label>
  <input type="email" id="email">
</form>
```

**2.4.7 Focus Visible (AA)**

```css
button:focus-visible {
  outline: 2px solid #0066CC;
  outline-offset: 2px;
}
```

#### 2.5 Input Modalities - WCAG 2.1

**2.5.1 Pointer Gestures (A)**

Don't require complex gestures without simpler alternative.

**2.5.2 Pointer Cancellation (A)**

Activation occurs on up-event, not down-event.

**2.5.3 Label in Name (A)**

```html
<!-- Visual label matches accessible name -->
<button aria-label="Search products">Search</button>
```

**2.5.4 Motion Actuation (A)**

Provide alternative to device motion/shaking gestures.

### Understandable

#### 3.1 Readable

**3.1.1 Language of Page (A)**

```html
<html lang="en">
```

**3.1.2 Language of Parts (AA)**

```html
<p>The French word for hello is <span lang="fr">bonjour</span>.</p>
```

#### 3.2 Predictable

**3.2.1 On Focus (A)**

Focus doesn't trigger unexpected changes.

**3.2.2 On Input (A)**

```html
<!-- Bad: auto-submit on select -->
<select onchange="this.form.submit()">

<!-- Good: explicit submit -->
<select id="options"></select>
<button type="submit">Submit</button>
```

**3.2.3 Consistent Navigation (AA)**

Navigation in same order across pages.

**3.2.4 Consistent Identification (AA)**

Same functionality labeled consistently.

#### 3.3 Input Assistance

**3.3.1 Error Identification (A)**

```html
<input 
  type="email" 
  aria-invalid="true"
  aria-describedby="email-error">
<div id="email-error" role="alert">
  Error: Please enter a valid email address
</div>
```

**3.3.2 Labels or Instructions (A)**

```html
<label for="password">
  Password (minimum 8 characters)
</label>
<input type="password" id="password" aria-describedby="password-help">
<div id="password-help">
  Must include uppercase, lowercase, and number
</div>
```

**3.3.3 Error Suggestion (AA)**

```html
<div role="alert">
  Error: Invalid date format. Please use MM/DD/YYYY (e.g., 01/15/2025)
</div>
```

**3.3.4 Error Prevention (AA)**

Provide confirmation for legal/financial transactions.

```html
<form>
  <!-- Transaction details -->
  <button type="submit">Submit</button>
</form>

<!-- Confirmation dialog -->
<div role="dialog" aria-labelledby="confirm-title">
  <h2 id="confirm-title">Confirm Purchase</h2>
  <p>Total: $99.99</p>
  <button>Cancel</button>
  <button>Confirm</button>
</div>
```

### Robust

#### 4.1 Compatible

**4.1.1 Parsing (A)**

Valid HTML (deprecated in WCAG 2.2).

**4.1.2 Name, Role, Value (A)**

```html
<!-- All UI components have accessible name and role -->
<button aria-label="Close dialog">×</button>

<div 
  role="checkbox" 
  aria-checked="false"
  aria-label="Accept terms">
</div>
```

**4.1.3 Status Messages (AA) - WCAG 2.1**

```html
<div role="status" aria-live="polite">
  Changes saved successfully
</div>
```

## WCAG 2.2 new criteria

### 2.4.11 Focus Not Obscured (Minimum) (AA)

Focused element not entirely hidden by other content.

### 2.4.12 Focus Not Obscured (Enhanced) (AAA)

No part of focused element hidden.

### 2.4.13 Focus Appearance (AAA)

Enhanced focus indicator requirements.

### 2.5.7 Dragging Movements (AA)

Provide alternative to drag-and-drop.

```html
<div draggable="true">Item</div>
<button>Move up</button>
<button>Move down</button>
```

### 2.5.8 Target Size (Minimum) (AA)

Click targets at least 24x24 CSS pixels.

```css
button {
  min-width: 24px;
  min-height: 24px;
  padding: 8px 16px; /* Usually larger */
}
```

### 3.2.6 Consistent Help (A)

Help mechanism in consistent location.

### 3.3.7 Redundant Entry (A)

Don't ask for same information twice.

```html
<!-- Bad: re-enter email -->
<input type="email" name="email">
<input type="email" name="confirm-email">

<!-- Good: auto-fill -->
<input type="email" name="email" autocomplete="email">
```

### 3.3.8 Accessible Authentication (Minimum) (AA)

Don't require cognitive function tests for authentication.

```html
<!-- Bad: complex CAPTCHA -->
<!-- Good: simple checkbox or alternative method -->
```

## Testing for WCAG compliance

### Automated tools

- axe DevTools
- WAVE
- Lighthouse
- Pa11y

### Manual testing

- Keyboard navigation
- Screen reader testing (NVDA, JAWS, VoiceOver)
- Color contrast checker
- Text resize to 200%
- Reflow at 320px width

### Screen readers

- NVDA (Windows, free)
- JAWS (Windows, commercial)
- VoiceOver (macOS/iOS, built-in)
- TalkBack (Android, built-in)

## Key takeaways

- Target WCAG 2.1 Level AA for most websites
- Four principles: Perceivable, Operable, Understandable, Robust
- All images need text alternatives
- Ensure 4.5:1 text contrast (AA)
- All functionality keyboard accessible
- Provide visible focus indicators
- Use semantic HTML
- Label all form controls
- Provide text alternatives for time-based media
- Test with automated tools AND manual testing
- Test with actual screen readers
- WCAG 2.2 adds new criteria, especially for mobile
