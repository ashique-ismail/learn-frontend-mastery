# Block vs Inline vs Inline-Block vs Replaced Elements

## Learning Objectives
- Understand the four main display types for HTML elements
- Learn how each type affects layout and box model behavior
- Master when to use each type and how to change defaults
- Understand replaced elements and their special behavior

---

## Element Display Types

Every HTML element has a **default display type** that determines how it participates in layout.

**Four main categories:**
1. **Block-level** - Takes full width, stacks vertically
2. **Inline** - Flows with text, only as wide as content
3. **Inline-block** - Hybrid: flows inline but behaves like block
4. **Replaced** - Content from external source, special sizing rules

---

## Block-Level Elements

### Characteristics

**Block elements:**
- Start on a **new line**
- Take up the **full width** available (by default)
- **Height** determined by content
- Can have `width`, `height`, `margin`, `padding` on all sides
- Stack vertically (normal flow)

### Default Block Elements

```html
<div>, <p>, <h1>-<h6>, <ul>, <ol>, <li>
<section>, <article>, <nav>, <header>, <footer>, <main>, <aside>
<form>, <fieldset>, <table>, <blockquote>, <pre>, <hr>
<address>, <figure>, <figcaption>, <details>, <summary>
```

### Visual Example

```html
<style>
  .block {
    background: lightblue;
    padding: 10px;
    margin: 10px 0;
  }
</style>

<div class="block">Block Element 1</div>
<div class="block">Block Element 2</div>
<div class="block">Block Element 3</div>
```

**Result:**
```
┌────────────────────────────────┐
│ Block Element 1                │  ← Full width
└────────────────────────────────┘
┌────────────────────────────────┐
│ Block Element 2                │  ← New line
└────────────────────────────────┘
┌────────────────────────────────┐
│ Block Element 3                │  ← New line
└────────────────────────────────┘
```

---

### Box Model for Block Elements

```html
<div style="
  width: 200px;
  height: 100px;
  padding: 20px;
  border: 5px solid;
  margin: 15px;
">
  Content
</div>
```

**Visual breakdown:**
```
Margin: 15px (transparent)
  ┌─────────────────────────────────┐
  │ Border: 5px (visible)           │
  │  ┌──────────────────────────┐   │
  │  │ Padding: 20px (inside)   │   │
  │  │  ┌──────────────────┐    │   │
  │  │  │ Content          │    │   │
  │  │  │ 200px × 100px    │    │   │
  │  │  └──────────────────┘    │   │
  │  └──────────────────────────┘   │
  └─────────────────────────────────┘

Total width: 15 + 5 + 20 + 200 + 20 + 5 + 15 = 280px
Total height: 15 + 5 + 20 + 100 + 20 + 5 + 15 = 180px
```

**With `box-sizing: border-box`:**
```css
div {
  box-sizing: border-box;
  width: 200px;  /* Now includes padding and border */
  padding: 20px;
  border: 5px solid;
}
/* Total width = 200px (padding + border included) */
```

---

## Inline Elements

### Characteristics

**Inline elements:**
- Flow **within text** (don't start new line)
- Width determined by **content** (can't set width)
- Height determined by **line-height** and font-size (can't set height)
- **Horizontal** `margin`/`padding` work, **vertical** do NOT affect layout
- Cannot contain block-level elements (with rare exceptions)

### Default Inline Elements

```html
<span>, <a>, <strong>, <em>, <b>, <i>, <u>, <small>
<abbr>, <cite>, <code>, <mark>, <time>, <sub>, <sup>
<label>, <button>, <textarea>, <select>, <input>
<img> (special: replaced element)
```

### Visual Example

```html
<p>
  This is <strong>bold text</strong> and this is
  <em>italic text</em> and here's a
  <span style="background: yellow;">highlighted span</span>
  all flowing together.
</p>
```

**Result:**
```
This is bold text and this is italic text and here's a
highlighted span all flowing together.
  ↑ All on the same line, wraps naturally
```

---

### Box Model Limitations

```html
<style>
  .inline {
    background: lightblue;
    width: 200px;        /* ❌ Ignored! */
    height: 100px;       /* ❌ Ignored! */
    margin: 20px;        /* ⚠️ Only left/right work */
    padding: 20px;       /* ⚠️ Top/bottom don't push other elements */
  }
</style>

<span class="inline">Inline Element</span>
```

**Vertical margin/padding behavior:**
```
            ┌─────────────────────┐
Text before │ Inline Element      │ Text after
            └─────────────────────┘
              ↑ padding-top/bottom visually appear
              but don't push surrounding elements
```

**Example demonstrating the issue:**
```html
<style>
  span {
    background: lightblue;
    padding: 30px 10px;  /* Vertical padding */
    border: 2px solid blue;
  }
</style>

<p>Line 1 with <span>inline element</span> here.</p>
<p>Line 2 overlaps with the padding above.</p>
```

The second `<p>` will overlap the blue background because vertical padding on inline elements **doesn't affect line height**.

---

## Inline-Block Elements

### Characteristics

**Inline-block is a hybrid:**
- Flows **inline** like `<span>` (doesn't start new line)
- Respects box model like `<div>` (width, height, vertical margin/padding work)
- Perfect for **flowing layouts** where you need box model control

### No Default Inline-Block Elements

You create them with CSS:
```css
.inline-block {
  display: inline-block;
}
```

### Visual Example

```html
<style>
  .box {
    display: inline-block;
    width: 100px;
    height: 100px;
    margin: 10px;
    padding: 10px;
    background: lightblue;
    border: 2px solid blue;
  }
</style>

<div class="box">Box 1</div>
<div class="box">Box 2</div>
<div class="box">Box 3</div>
```

**Result:**
```
┌──────┐ ┌──────┐ ┌──────┐
│Box 1 │ │Box 2 │ │Box 3 │  ← Flow horizontally
└──────┘ └──────┘ └──────┘   ← All have fixed size
```

---

### Common Use Cases

**1. Horizontal navigation:**
```html
<nav>
  <a href="/" style="display: inline-block; padding: 10px 20px;">Home</a>
  <a href="/about" style="display: inline-block; padding: 10px 20px;">About</a>
  <a href="/contact" style="display: inline-block; padding: 10px 20px;">Contact</a>
</nav>
```

**2. Image gallery:**
```html
<style>
  .gallery-item {
    display: inline-block;
    width: 200px;
    height: 200px;
    margin: 10px;
  }
</style>

<div class="gallery-item"><img src="1.jpg" alt=""></div>
<div class="gallery-item"><img src="2.jpg" alt=""></div>
<div class="gallery-item"><img src="3.jpg" alt=""></div>
```

**3. Badge or tag components:**
```html
<style>
  .tag {
    display: inline-block;
    padding: 5px 10px;
    margin: 5px;
    background: #e0e0e0;
    border-radius: 15px;
  }
</style>

<span class="tag">React</span>
<span class="tag">TypeScript</span>
<span class="tag">CSS</span>
```

---

### The Whitespace Problem

**Inline-block elements preserve whitespace:**

```html
<!-- ❌ Unwanted gap -->
<div style="display: inline-block;">Box 1</div>
<div style="display: inline-block;">Box 2</div>
<!--          ↑ This whitespace creates a 4px gap -->
```

**Solutions:**

**1. Remove whitespace (ugly but works):**
```html
<div style="display: inline-block;">Box 1</div><div style="display: inline-block;">Box 2</div>
```

**2. HTML comments:**
```html
<div style="display: inline-block;">Box 1</div><!--
--><div style="display: inline-block;">Box 2</div>
```

**3. Negative margin:**
```css
.inline-block {
  display: inline-block;
  margin-right: -4px;  /* Cancel the whitespace */
}
```

**4. Parent font-size trick:**
```css
.parent {
  font-size: 0;  /* Removes whitespace */
}
.child {
  display: inline-block;
  font-size: 16px;  /* Restore font size */
}
```

**5. Use Flexbox instead (modern approach):**
```css
.parent {
  display: flex;
  gap: 10px;  /* Explicit gap, no whitespace issues */
}
```

---

## Replaced Elements

### What Are Replaced Elements?

**Replaced elements** have their content controlled by an **external resource**, not by CSS.

**Common replaced elements:**
```html
<img>, <video>, <audio>, <iframe>, <embed>, <object>
<input>, <textarea>, <select>, <button> (partially replaced)
<canvas>, <svg> (special cases)
```

### Special Characteristics

**1. Intrinsic dimensions**
```html
<!-- Image has intrinsic 800×600 size -->
<img src="photo.jpg" alt="">
<!-- Displays at 800×600 unless you set width/height -->
```

**2. Can set width and height:**
```css
img {
  width: 200px;  /* ✅ Works on replaced elements */
  height: 150px;
}
```

**3. Behave like inline-block:**
```html
<!-- Images flow inline by default -->
<img src="1.jpg" alt=""> <img src="2.jpg" alt="">
```

**4. Object-fit and object-position:**
```css
img {
  width: 300px;
  height: 300px;
  object-fit: cover;  /* Crop to fill */
  object-position: center;
}
```

---

### The Baseline Problem

Replaced elements sit on the **text baseline** by default, which can create unexpected gaps:

```html
<div>
  <img src="image.jpg" alt="">
</div>
```

**Visual:**
```
┌──────────────┐
│              │
│    Image     │
│              │
└──────────────┘
  ↑ 4px gap here (descender space)
```

**Why:** Images are inline by default and sit on the text baseline, leaving room for descenders (like the tail on 'g', 'y', 'p').

**Solutions:**

**1. Make it block:**
```css
img {
  display: block;  /* Removes baseline gap */
}
```

**2. Vertical align:**
```css
img {
  vertical-align: top;  /* or bottom, or middle */
}
```

**3. Parent font-size 0:**
```css
.image-wrapper {
  font-size: 0;  /* Removes baseline space */
}
```

---

## Comparison Table

| Feature | Block | Inline | Inline-Block | Replaced |
|---------|-------|--------|--------------|----------|
| **New line** | Yes | No | No | No (default inline) |
| **Set width/height** | ✅ Yes | ❌ No | ✅ Yes | ✅ Yes |
| **Vertical margin** | ✅ Yes | ❌ No | ✅ Yes | ✅ Yes |
| **Vertical padding** | ✅ Yes | ⚠️ Visual only | ✅ Yes | ✅ Yes |
| **Horizontal margin/padding** | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes |
| **Contains block elements** | ✅ Yes | ❌ No (usually) | ✅ Yes | N/A |
| **Default width** | 100% of parent | Content width | Content width | Intrinsic |
| **Box model** | Full | Partial | Full | Full |

---

## Changing Display Type

```css
/* Make inline element behave like block */
span {
  display: block;
}

/* Make block element behave like inline */
div {
  display: inline;
}

/* Make block element behave like inline-block */
div {
  display: inline-block;
}

/* Hide element completely (removes from layout) */
div {
  display: none;
}

/* Other modern display values */
div {
  display: flex;         /* Flexbox container */
  display: grid;         /* Grid container */
  display: inline-flex;  /* Inline flex container */
  display: inline-grid;  /* Inline grid container */
}
```

---

## Practical Examples

### Example 1: Button Group

```html
<style>
  .btn-group {
    font-size: 0;  /* Remove whitespace */
  }
  
  .btn {
    display: inline-block;
    padding: 10px 20px;
    font-size: 16px;  /* Restore */
    background: #007bff;
    color: white;
    border: none;
    cursor: pointer;
  }
  
  .btn:hover {
    background: #0056b3;
  }
  
  .btn:not(:last-child) {
    border-right: 1px solid #0056b3;
  }
</style>

<div class="btn-group">
  <button class="btn">Save</button>
  <button class="btn">Cancel</button>
  <button class="btn">Delete</button>
</div>
```

---

### Example 2: Card Grid

```html
<style>
  .card-container {
    font-size: 0;  /* Remove whitespace */
  }
  
  .card {
    display: inline-block;
    width: 300px;
    margin: 15px;
    padding: 20px;
    font-size: 16px;  /* Restore */
    background: white;
    border: 1px solid #ddd;
    border-radius: 8px;
    vertical-align: top;  /* Align tops */
  }
</style>

<div class="card-container">
  <div class="card">
    <h2>Card 1</h2>
    <p>Content here...</p>
  </div>
  <div class="card">
    <h2>Card 2</h2>
    <p>Different height content...</p>
    <p>More paragraphs...</p>
  </div>
  <div class="card">
    <h2>Card 3</h2>
    <p>Content here...</p>
  </div>
</div>
```

---

### Example 3: Image with Text (Baseline Issue)

```html
<!-- ❌ Problem: gap below image -->
<div style="background: #eee;">
  <img src="icon.png" alt="" style="width: 100px;">
</div>
<!-- 4px gap visible -->

<!-- ✅ Solution 1: Block display -->
<div style="background: #eee;">
  <img src="icon.png" alt="" style="width: 100px; display: block;">
</div>

<!-- ✅ Solution 2: Vertical align -->
<div style="background: #eee; line-height: 0;">
  <img src="icon.png" alt="" style="width: 100px; vertical-align: top;">
</div>
```

---

## Best Practices

✅ **Use semantic HTML first** - let elements use their default display

✅ **Block for layout containers** - `<div>`, `<section>`, etc.

✅ **Inline for text-level semantics** - `<span>`, `<strong>`, `<em>`

✅ **Inline-block for flowing layouts** - buttons, badges, image galleries

✅ **Fix baseline gaps on images** - use `display: block` or `vertical-align`

✅ **Prefer Flexbox/Grid over inline-block** - for modern layouts (easier, no whitespace issues)

✅ **Don't fight the defaults** - if you're constantly changing display types, reconsider your HTML structure

---

## Common Mistakes

❌ **Using `<div>` for inline text:**
```html
<!-- Bad -->
<p>Hello <div>world</div></p>  <!-- Breaks paragraph flow -->

<!-- Good -->
<p>Hello <span>world</span></p>
```

❌ **Setting width on inline elements:**
```css
/* Doesn't work */
span {
  width: 200px;  /* Ignored */
}

/* Use inline-block */
span {
  display: inline-block;
  width: 200px;  /* Works */
}
```

❌ **Not accounting for whitespace in inline-block:**
```html
<!-- Unexpected gaps -->
<div style="display: inline-block;">Box 1</div>
<div style="display: inline-block;">Box 2</div>
```

---

## Resources

- [MDN: Block-level elements](https://developer.mozilla.org/en-US/docs/Web/HTML/Block-level_elements)
- [MDN: Inline elements](https://developer.mozilla.org/en-US/docs/Web/HTML/Inline_elements)
- [MDN: display property](https://developer.mozilla.org/en-US/docs/Web/CSS/display)
- [CSS Tricks: display](https://css-tricks.com/almanac/properties/d/display/)
