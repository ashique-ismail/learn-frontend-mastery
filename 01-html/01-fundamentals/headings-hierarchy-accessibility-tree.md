# Headings Hierarchy and Accessibility Tree

## Learning Objectives
- Understand the purpose and importance of heading hierarchy
- Learn how headings create document outlines for assistive technologies
- Master the accessibility tree and how headings contribute to it
- Avoid common heading mistakes that break accessibility

---

## Why Headings Matter

Headings (`<h1>` through `<h6>`) are the **most important semantic elements** for:

1. **Accessibility** - Screen readers use headings to navigate
2. **SEO** - Search engines use headings to understand content structure
3. **Usability** - Users scan headings to find information quickly
4. **Maintainability** - Developers understand code structure

**Statistic:** WebAIM study found that 68% of screen reader users navigate by headings.

---

## The Six Heading Levels

```html
<h1>Main Title (Level 1)</h1>
<h2>Section Title (Level 2)</h2>
<h3>Subsection Title (Level 3)</h3>
<h4>Sub-subsection Title (Level 4)</h4>
<h5>Fifth Level (Level 5)</h5>
<h6>Sixth Level (Level 6)</h6>
```

**Visual hierarchy (default browser styles):**
- `<h1>`: 2em (32px) - Largest
- `<h2>`: 1.5em (24px)
- `<h3>`: 1.17em (18.72px)
- `<h4>`: 1em (16px) - Same as body text
- `<h5>`: 0.83em (13.28px)
- `<h6>`: 0.67em (10.72px) - Smallest

**Important:** Default styles don't matter - you can restyle headings. **Semantic hierarchy** is what counts.

---

## Heading Hierarchy Rules

### Rule 1: One `<h1>` Per Page

**The `<h1>` is the main page title** - it should describe the primary purpose of the page.

```html
<!-- ✅ Good: One h1 -->
<h1>Complete Guide to React Hooks</h1>

<h2>Introduction</h2>
<p>...</p>

<h2>useState</h2>
<p>...</p>

<!-- ❌ Bad: Multiple h1s -->
<h1>Welcome</h1>
<h1>About Us</h1>
<h1>Contact</h1>
```

**Exception:** HTML5 spec technically allows multiple `<h1>` elements (one per `<section>`), but **screen readers don't implement this correctly**. Stick to one `<h1>`.

---

### Rule 2: Don't Skip Levels

Headings should **nest logically**. Going from `<h2>` to `<h4>` skips `<h3>`, which breaks the outline.

```html
<!-- ✅ Good: Logical progression -->
<h1>Main Title</h1>
  <h2>Section</h2>
    <h3>Subsection</h3>
      <h4>Detail</h4>
    <h3>Another Subsection</h3>
  <h2>Another Section</h2>

<!-- ❌ Bad: Skipped h3 -->
<h1>Main Title</h1>
  <h2>Section</h2>
    <h4>Subsection</h4>  <!-- Where's h3? -->
```

**Why it's bad:**
- Screen reader users think they missed content
- Document outline tools show broken structure
- Creates confusion about content hierarchy

**Visualizing the skip:**
```
h1: Main Title
  h2: Section A
    h4: Subsection  ← User wonders: "What happened to h3?"
```

---

### Rule 3: You Can Go Back Up

It's fine to go from `<h4>` back to `<h2>` - this closes the subsection and starts a new section.

```html
<h1>Complete CSS Guide</h1>

  <h2>Selectors</h2>
    <h3>Type Selectors</h3>
      <h4>Examples</h4>
    <h3>Class Selectors</h3>
      <h4>Examples</h4>
      
  <h2>Box Model</h2>  <!-- ✅ Back up to h2 -->
    <h3>Margin</h3>
    <h3>Padding</h3>
```

---

## Document Outline

Headings create a **document outline** (table of contents) that assistive technologies and browsers can expose.

### Example Document

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <title>React Tutorial</title>
</head>
<body>
  <h1>React Tutorial</h1>
  
  <h2>Getting Started</h2>
  <p>Introduction content...</p>
  
    <h3>Installation</h3>
    <p>How to install...</p>
    
    <h3>Your First Component</h3>
    <p>Creating components...</p>
  
  <h2>Core Concepts</h2>
  <p>Core concepts intro...</p>
  
    <h3>JSX</h3>
    <p>JSX explanation...</p>
    
    <h3>Components</h3>
    <p>Component details...</p>
    
      <h4>Function Components</h4>
      <p>...</p>
      
      <h4>Class Components</h4>
      <p>...</p>
  
  <h2>Hooks</h2>
  <p>Hooks introduction...</p>
</body>
</html>
```

### Generated Outline

```
React Tutorial (h1)
├─ Getting Started (h2)
│  ├─ Installation (h3)
│  └─ Your First Component (h3)
├─ Core Concepts (h2)
│  ├─ JSX (h3)
│  └─ Components (h3)
│     ├─ Function Components (h4)
│     └─ Class Components (h4)
└─ Hooks (h2)
```

**How users access this:**
- **NVDA/JAWS:** Insert + F6 (heading list)
- **VoiceOver:** Rotor menu (headings)
- **Chrome:** HeadingsMap extension
- **Browser Reader Mode:** Generates TOC from headings

---

## The Accessibility Tree

The **accessibility tree** is a parallel structure to the DOM that assistive technologies use.

### What is the Accessibility Tree?

A browser-generated tree that represents the **semantic structure** of your page:
- Roles (button, navigation, heading)
- States (checked, expanded, disabled)
- Properties (label, level, description)
- Relationships (owns, labelledby)

**Example:**
```html
<button aria-label="Close dialog" aria-pressed="false">
  <svg>...</svg>
</button>
```

**In accessibility tree:**
```
Button
  role: button
  label: "Close dialog"
  state: not pressed
```

---

### Headings in the Accessibility Tree

```html
<h2 id="section-title">User Settings</h2>
```

**In accessibility tree:**
```
Heading (level 2)
  name: "User Settings"
  role: heading
  level: 2
```

**Screen reader announcement:**
> "Heading level 2, User Settings"

---

### Viewing the Accessibility Tree

**Chrome DevTools:**
1. Open DevTools (F12)
2. Elements tab
3. Select an element
4. Accessibility pane (right sidebar)
5. Click "Enable full-page accessibility tree"

**Firefox DevTools:**
1. Open DevTools (F12)
2. Accessibility tab
3. Enable accessibility features if prompted
4. View the tree structure

**Example tree view:**
```
document
├─ banner (header)
│  └─ navigation (nav)
│     ├─ link "Home"
│     └─ link "About"
├─ main (main)
│  ├─ heading (h1) "Page Title"
│  ├─ region (section)
│  │  ├─ heading (h2) "Section Title"
│  │  └─ paragraph "Content..."
│  └─ article
│     ├─ heading (h2) "Article Title"
│     └─ paragraph "Article content..."
└─ contentinfo (footer)
   └─ paragraph "Copyright..."
```

---

## Screen Reader Navigation

### How Screen Reader Users Navigate by Headings

**NVDA (Windows):**
- `H` - Next heading
- `Shift+H` - Previous heading
- `1-6` - Jump to heading level (e.g., `2` for next h2)
- `Insert+F7` - Elements list (shows all headings)

**JAWS (Windows):**
- `H` - Next heading
- `Shift+H` - Previous heading
- `1-6` - Jump by level
- `Insert+F6` - Heading list

**VoiceOver (Mac/iOS):**
- Rotor → Headings
- Flick up/down to navigate

**Example user flow:**
```
1. User presses H → Jumps to "Getting Started" (h2)
2. User presses H → Jumps to "Installation" (h3)
3. User presses 2 → Jumps to next h2 ("Core Concepts")
4. User presses H → Jumps to "JSX" (h3)
```

**Why this matters:** Users with visual impairments often "skim" pages by jumping through headings, similar to how sighted users scan visually.

---

## Common Mistakes

### Mistake 1: Using Headings for Styling

```html
<!-- ❌ Bad: Using h5 because it's smaller -->
<h1>Page Title</h1>
<h5>A small note</h5>  <!-- Should be <p> or <small> -->
<h2>Section Title</h2>
```

**Problem:** Screen reader announces "Heading level 5" between level 1 and 2, which is confusing.

**Fix: Separate semantics from styling:**
```html
<!-- ✅ Good -->
<h1>Page Title</h1>
<p class="text-sm">A small note</p>
<h2>Section Title</h2>
```

```css
.text-sm {
  font-size: 0.875rem;
  color: #666;
}
```

---

### Mistake 2: Multiple H1s per Page

```html
<!-- ❌ Bad: Multiple main titles -->
<header>
  <h1>Site Logo Text</h1>
</header>

<main>
  <h1>Page Title</h1>
  
  <article>
    <h1>Article Title</h1>
  </article>
</main>
```

**Problem:** Confuses the main topic of the page.

**Fix:**
```html
<!-- ✅ Good -->
<header>
  <div class="logo" aria-label="Site Name">Site Logo Text</div>
  <!-- Or <p> if it's just branding -->
</header>

<main>
  <h1>Page Title</h1>  <!-- One h1 -->
  
  <article>
    <h2>Article Title</h2>  <!-- Use h2 for article -->
  </article>
</main>
```

---

### Mistake 3: Skipping Levels for Visual Effect

```html
<!-- ❌ Bad: Skipped h2 for larger text -->
<h1>Main Title</h1>
<h3>Subtitle</h3>  <!-- Wanted it slightly smaller, so skipped h2 -->
```

**Fix: Use correct level + CSS:**
```html
<!-- ✅ Good -->
<h1>Main Title</h1>
<h2 class="subtitle">Subtitle</h2>
```

```css
.subtitle {
  font-size: 1.25rem;  /* Smaller than default h2 */
  font-weight: 400;
  color: #666;
}
```

---

### Mistake 4: Empty Headings or Icon-Only Headings

```html
<!-- ❌ Bad: No text content -->
<h2>
  <svg>...</svg>  <!-- Screen reader can't announce this -->
</h2>

<!-- ❌ Bad: Empty -->
<h2></h2>
```

**Fix: Add accessible text:**
```html
<!-- ✅ Good: Visible text -->
<h2>
  <svg aria-hidden="true">...</svg>
  Settings
</h2>

<!-- ✅ Good: Visually hidden text -->
<h2>
  <svg aria-hidden="true">...</svg>
  <span class="sr-only">Settings</span>
</h2>
```

```css
.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}
```

---

### Mistake 5: Headings Inside Interactive Elements

```html
<!-- ❌ Bad: Button containing heading -->
<button>
  <h3>Click Me</h3>
</button>

<!-- ❌ Bad: Link containing heading -->
<a href="/about">
  <h2>About Us</h2>
</a>
```

**Problem:** Creates confusing accessibility tree (is it a heading or a button?).

**Fix:**
```html
<!-- ✅ Good: Heading wraps interactive element -->
<h3>
  <button>Click Me</button>
</h3>

<h2>
  <a href="/about">About Us</a>
</h2>

<!-- ✅ Or: Separate them -->
<h3>Section Title</h3>
<button>Click Me</button>
```

---

## Advanced: Sectioning and Heading Algorithms

### HTML5 Sectioning Elements

These elements create **implicit sections**:
- `<article>`
- `<aside>`
- `<nav>`
- `<section>`

**The (broken) theory:** Each sectioning element resets heading levels.

```html
<!-- Theoretical HTML5 outline -->
<h1>Site Title</h1>

<article>
  <h1>Article Title</h1>  <!-- Should be treated as h2 -->
  
  <section>
    <h1>Section Title</h1>  <!-- Should be treated as h3 -->
  </section>
</article>
```

**The reality:** No browser or screen reader implements this correctly. **Don't rely on it.**

**Best practice: Use explicit heading levels:**
```html
<!-- ✅ Good: Explicit levels -->
<h1>Site Title</h1>

<article>
  <h2>Article Title</h2>
  
  <section>
    <h3>Section Title</h3>
  </section>
</article>
```

---

## Testing Your Heading Structure

### Automated Tools

**1. HeadingsMap Browser Extension**
- Chrome/Edge/Firefox extension
- Shows heading outline in sidebar
- Highlights skipped levels

**2. axe DevTools**
- Free Chrome/Firefox extension
- Checks for heading level issues
- Part of broader accessibility testing

**3. WAVE**
- [https://wave.webaim.org/](https://wave.webaim.org/)
- Shows heading structure visually on page
- Identifies missing or skipped headings

### Manual Testing

**1. Outline test:**
```javascript
// In console: list all headings
Array.from(document.querySelectorAll('h1, h2, h3, h4, h5, h6'))
  .map(h => `${'  '.repeat(h.tagName[1] - 1)}${h.tagName}: ${h.textContent.trim()}`)
  .join('\n');
```

**Example output:**
```
H1: React Tutorial
  H2: Getting Started
    H3: Installation
    H3: Your First Component
  H2: Core Concepts
    H3: JSX
```

**2. Screen reader test:**
- Use NVDA (free), VoiceOver (Mac/iOS), or JAWS
- Navigate by headings (H key)
- Verify outline makes sense

---

## Practical Examples

### Blog Post

```html
<article>
  <header>
    <h1>Understanding React Hooks</h1>
    <p>By Jane Doe • May 26, 2025</p>
  </header>
  
  <h2>Introduction</h2>
  <p>Hooks changed how we write React...</p>
  
  <h2>useState Hook</h2>
  <p>The most basic hook...</p>
  
    <h3>Example: Counter</h3>
    <pre><code>...</code></pre>
    
    <h3>Functional Updates</h3>
    <p>When your new state depends on old state...</p>
  
  <h2>useEffect Hook</h2>
  <p>For side effects...</p>
  
    <h3>Dependency Array</h3>
    <p>Control when effects run...</p>
    
    <h3>Cleanup Functions</h3>
    <p>Prevent memory leaks...</p>
  
  <h2>Conclusion</h2>
  <p>Hooks make React simpler...</p>
</article>
```

**Outline:**
```
Understanding React Hooks (h1)
├─ Introduction (h2)
├─ useState Hook (h2)
│  ├─ Example: Counter (h3)
│  └─ Functional Updates (h3)
├─ useEffect Hook (h2)
│  ├─ Dependency Array (h3)
│  └─ Cleanup Functions (h3)
└─ Conclusion (h2)
```

---

### Documentation Site

```html
<body>
  <header>
    <div>Site Logo</div>
    <nav aria-label="Main">...</nav>
  </header>
  
  <main>
    <h1>API Reference</h1>
    
    <section id="authentication">
      <h2>Authentication</h2>
      <p>API uses OAuth 2.0...</p>
      
      <h3>Getting Access Token</h3>
      <p>...</p>
      
      <h3>Refreshing Token</h3>
      <p>...</p>
    </section>
    
    <section id="endpoints">
      <h2>Endpoints</h2>
      
      <h3>GET /users</h3>
      <p>Retrieve users...</p>
      
        <h4>Parameters</h4>
        <ul>...</ul>
        
        <h4>Response</h4>
        <pre>...</pre>
      
      <h3>POST /users</h3>
      <p>Create user...</p>
      
        <h4>Parameters</h4>
        <ul>...</ul>
        
        <h4>Response</h4>
        <pre>...</pre>
    </section>
  </main>
</body>
```

---

## Best Practices Summary

✅ **One `<h1>` per page** - the main page title

✅ **Don't skip levels** - h1 → h2 → h3, not h1 → h3

✅ **Use headings for structure, not styling** - style with CSS

✅ **Every section should have a heading** - helps navigation

✅ **Headings should be descriptive** - "Getting Started" not "Introduction"

✅ **Test with keyboard and screen reader** - navigate by headings

✅ **Use headings even if visually hidden** - for screen reader users

✅ **Validate with tools** - HeadingsMap, axe, WAVE

---

## Resources

- [MDN: Heading elements](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/Heading_Elements)
- [W3C: Headings and Labels](https://www.w3.org/WAI/WCAG22/Understanding/headings-and-labels.html)
- [WebAIM: Semantic Structure](https://webaim.org/techniques/semanticstructure/)
- [HeadingsMap extension](https://chrome.google.com/webstore/detail/headingsmap/flbjommegcjonpdmenkdiocclhjacmbi)
- [How screen readers navigate](https://www.youtube.com/watch?v=OUDV1gqs9GA)
