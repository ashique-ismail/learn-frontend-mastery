# Global Attributes

## The Idea

**In plain English:** Global attributes are special instructions you can attach to any HTML element on a page, like adding extra information or special powers to any piece of your content. Think of an "attribute" as a label or tag you stick on something to tell the browser how to treat it.

**Real-world analogy:** Imagine a school where every student wears a lanyard that can hold different badges — a name badge (so the office can find them uniquely), a house-color badge (so they can be grouped with others), and a sticky note with extra info like allergies. Every single student, whether in art class or gym, can wear these same lanyards.
- The student's unique name badge = the `id` attribute (one-of-a-kind identifier)
- The house-color badge shared with classmates = the `class` attribute (groups elements together)
- The sticky note with extra info = the `data-*` attribute (stores custom details about that element)

---

## Learning Objectives
- Understand what global attributes are and why they matter
- Master the most important global attributes (`id`, `class`, `data-*`, `tabindex`, `hidden`)
- Learn practical use cases for each attribute
- Understand accessibility implications

---

## What Are Global Attributes?

**Global attributes** can be used on **any HTML element**, regardless of type.

```html
<!-- These work on ANY element -->
<div id="main" class="container" data-user="123">...</div>
<span id="badge" class="badge" data-count="5">...</span>
<button id="submit" class="btn" data-action="save">...</button>
```

**Most common global attributes:**
- `id` - Unique identifier
- `class` - Class name(s) for CSS/JS
- `data-*` - Custom data attributes
- `title` - Tooltip text
- `style` - Inline CSS
- `hidden` - Hide element
- `tabindex` - Keyboard navigation order
- `contenteditable` - Make element editable
- `lang` - Language
- `dir` - Text direction (ltr, rtl)
- `draggable` - Enable drag-and-drop
- `accesskey` - Keyboard shortcut
- `aria-*` - Accessibility attributes

---

## The `id` Attribute

### Purpose

Uniquely identifies an element on the page.

```html
<div id="header">Site Header</div>
<nav id="main-nav">...</nav>
<main id="content">...</main>
```

### Rules

**1. Must be unique on the page:**
```html
<!-- ❌ Bad: duplicate IDs -->
<div id="box">Box 1</div>
<div id="box">Box 2</div>  <!-- Invalid! -->

<!-- ✅ Good: unique IDs -->
<div id="box-1">Box 1</div>
<div id="box-2">Box 2</div>
```

**2. Valid characters:**
- Must start with a letter (a-z, A-Z)
- Can contain letters, digits, hyphens, underscores, periods
- Case-sensitive

```html
<!-- ✅ Valid IDs -->
<div id="header">...</div>
<div id="user-profile">...</div>
<div id="item_123">...</div>
<div id="section.2">...</div>

<!-- ❌ Invalid IDs -->
<div id="123-box">...</div>  <!-- Starts with number -->
<div id="my box">...</div>   <!-- Contains space -->
<div id="special!char">...</div>  <!-- Special character -->
```

**3. No spaces:**
```html
<!-- ❌ Wrong -->
<div id="my header">...</div>

<!-- ✅ Correct -->
<div id="my-header">...</div>
<div id="myHeader">...</div>
<div id="my_header">...</div>
```

---

### Use Cases

**1. CSS targeting:**
```css
#header {
  background: blue;
}
```

**2. JavaScript selection:**
```javascript
const header = document.getElementById('header');
// Or with querySelector
const header = document.querySelector('#header');
```

**3. URL fragment (jump links):**
```html
<a href="#section-2">Jump to Section 2</a>

<h2 id="section-2">Section 2</h2>
<!-- Clicking link scrolls to this heading -->
```

**4. Form label association:**
```html
<label for="email">Email:</label>
<input type="email" id="email" name="email">
<!-- Clicking label focuses the input -->
```

**5. ARIA relationships:**
```html
<button aria-describedby="help-text">Submit</button>
<div id="help-text">Click to submit the form</div>
```

---

## The `class` Attribute

### Purpose

Assigns one or more class names for styling (CSS) or selection (JavaScript).

```html
<div class="container">...</div>
<button class="btn btn-primary btn-lg">Click Me</button>
```

### Multiple Classes

**Space-separated:**
```html
<div class="card featured highlighted">Content</div>
```

**Styling:**
```css
.card { border: 1px solid #ddd; }
.featured { background: gold; }
.highlighted { box-shadow: 0 0 10px blue; }
```

---

### Naming Conventions

**BEM (Block Element Modifier):**
```html
<div class="card">
  <div class="card__header">
    <h2 class="card__title">Title</h2>
  </div>
  <div class="card__body">
    <p class="card__text">Text</p>
  </div>
  <button class="card__button card__button--primary">Click</button>
</div>
```

**Utility classes:**
```html
<div class="flex items-center justify-between p-4 bg-blue-500 text-white">
  Content
</div>
```

**Semantic naming:**
```html
<!-- ❌ Presentational (avoid) -->
<div class="red-box big-text">Alert</div>

<!-- ✅ Semantic (better) -->
<div class="alert alert-danger">Alert</div>
```

---

### Use Cases

**1. CSS styling:**
```css
.btn {
  padding: 10px 20px;
  border: none;
  cursor: pointer;
}

.btn-primary {
  background: blue;
  color: white;
}
```

**2. JavaScript selection:**
```javascript
// Single element
const btn = document.querySelector('.btn-primary');

// Multiple elements
const btns = document.querySelectorAll('.btn');
btns.forEach(btn => {
  btn.addEventListener('click', handleClick);
});
```

**3. Conditional classes (React example):**
```jsx
function Button({ variant, disabled }) {
  return (
    <button 
      className={`btn btn-${variant} ${disabled ? 'disabled' : ''}`}
    >
      Click Me
    </button>
  );
}
```

---

## `id` vs `class`: When to Use Which

| Use `id` when... | Use `class` when... |
|------------------|---------------------|
| Element is unique on page | Multiple elements share styling |
| Need URL fragment target | Applying reusable styles |
| Form label association | Component-based styling |
| ARIA relationships | JavaScript event delegation |
| Anchor link target | Multiple class combinations |

**Example combining both:**
```html
<div id="user-profile" class="card card--featured">
  <h2 id="profile-name" class="card__title">John Doe</h2>
  <p class="card__description">Software Engineer</p>
  <button id="edit-btn" class="btn btn-primary">Edit</button>
</div>
```

---

## `data-*` Attributes

### Purpose

Store **custom data** on elements without using non-standard attributes.

```html
<div 
  data-user-id="12345" 
  data-role="admin" 
  data-created-at="2025-05-26"
>
  User Info
</div>
```

### Naming Rules

- Must start with `data-`
- After `data-`, can contain letters, numbers, hyphens, underscores
- No uppercase letters (use hyphens for word separation)

```html
<!-- ✅ Valid -->
<div data-user-id="123">...</div>
<div data-item-count="5">...</div>
<div data-is-active="true">...</div>

<!-- ❌ Invalid -->
<div data-userId="123">...</div>  <!-- Camel case not allowed -->
<div dataUserId="123">...</div>   <!-- Missing hyphen -->
```

---

### Accessing Data Attributes

**JavaScript (dataset API):**
```html
<button 
  id="save-btn"
  data-user-id="123" 
  data-action-type="save"
  data-confirm-required="true"
>
  Save
</button>

<script>
const btn = document.getElementById('save-btn');

// Read (camelCase conversion)
console.log(btn.dataset.userId);  // "123"
console.log(btn.dataset.actionType);  // "save"
console.log(btn.dataset.confirmRequired);  // "true"

// Write
btn.dataset.lastSaved = new Date().toISOString();
// Adds data-last-saved="2025-05-26T..."

// Delete
delete btn.dataset.confirmRequired;
// Removes data-confirm-required attribute
</script>
```

**CSS (attribute selectors):**
```css
/* Style based on data attribute */
[data-status="active"] {
  background: green;
}

[data-status="inactive"] {
  background: gray;
}

/* Partial match */
[data-role*="admin"] {
  font-weight: bold;
}
```

---

### Use Cases

**1. Storing metadata:**
```html
<article 
  data-article-id="456" 
  data-author="Jane Doe"
  data-published="2025-05-26"
  data-category="tech"
>
  Article content...
</article>
```

**2. JavaScript behavior:**
```html
<button 
  data-action="delete" 
  data-confirm="Are you sure?"
  data-target-id="post-123"
>
  Delete
</button>

<script>
document.addEventListener('click', (e) => {
  if (e.target.dataset.action === 'delete') {
    const confirmed = confirm(e.target.dataset.confirm);
    if (confirmed) {
      deleteItem(e.target.dataset.targetId);
    }
  }
});
</script>
```

**3. State management (React example):**
```jsx
function TodoItem({ todo }) {
  return (
    <li 
      data-todo-id={todo.id}
      data-completed={todo.completed}
      className={todo.completed ? 'completed' : ''}
    >
      {todo.text}
    </li>
  );
}
```

**4. Analytics tracking:**
```html
<button 
  data-track-event="button-click"
  data-track-category="cta"
  data-track-label="sign-up"
>
  Sign Up
</button>

<script>
document.addEventListener('click', (e) => {
  if (e.target.dataset.trackEvent) {
    analytics.track(e.target.dataset.trackEvent, {
      category: e.target.dataset.trackCategory,
      label: e.target.dataset.trackLabel
    });
  }
});
</script>
```

---

## The `tabindex` Attribute

### Purpose

Controls keyboard navigation order and focus behavior.

### Values

**`tabindex="0"`** - Natural tab order, focusable
```html
<div tabindex="0">Now keyboard focusable</div>
```

**`tabindex="-1"`** - Programmatically focusable only, skipped in tab order
```html
<div tabindex="-1" id="modal">
  Modal content (focused via JS)
</div>

<script>
document.getElementById('modal').focus();
</script>
```

**`tabindex="1+"`** - Custom tab order (anti-pattern, avoid!)
```html
<!-- ❌ Don't do this -->
<input tabindex="3">
<input tabindex="1">  <!-- Focused first -->
<input tabindex="2">  <!-- Then this -->
```

---

### Use Cases

**1. Making non-interactive elements focusable:**
```html
<div 
  tabindex="0" 
  role="button"
  onclick="handleClick()"
  onkeydown="if(event.key==='Enter') handleClick()"
>
  Custom Button
</div>
```

**2. Modal focus management:**
```html
<div id="modal" tabindex="-1" role="dialog" aria-labelledby="modal-title">
  <h2 id="modal-title">Modal Title</h2>
  <p>Content...</p>
  <button onclick="closeModal()">Close</button>
</div>

<script>
function openModal() {
  const modal = document.getElementById('modal');
  modal.style.display = 'block';
  modal.focus();  // Move focus into modal
}
</script>
```

**3. Skip links:**
```html
<a href="#main" class="skip-link">Skip to main content</a>
<!-- Keyboard users can skip nav -->

<nav>...</nav>

<main id="main" tabindex="-1">
  <!-- Content -->
</main>

<script>
// Focus main when skip link clicked
document.querySelector('.skip-link').addEventListener('click', (e) => {
  e.preventDefault();
  document.getElementById('main').focus();
});
</script>
```

---

### Best Practices

✅ **Use `tabindex="0"` for custom interactive elements**

✅ **Use `tabindex="-1"` for programmatic focus (modals, error messages)**

❌ **Never use positive integers** - breaks natural tab order

✅ **Prefer native interactive elements** - `<button>`, `<a>`, `<input>` are focusable by default

✅ **Always add keyboard handlers** if using `tabindex` on non-interactive elements

---

## The `hidden` Attribute

### Purpose

Hides an element from **all** rendering contexts (visual, screen readers, search engines).

```html
<div hidden>
  This content is completely hidden
</div>
```

**Equivalent CSS:**
```css
[hidden] {
  display: none;
}
```

---

### `hidden` vs CSS `display: none`

**Same effect, but:**
- `hidden` is **semantic** (indicates content is not currently relevant)
- CSS `display: none` is **presentational**

```html
<!-- Semantic: content not relevant right now -->
<div hidden>
  Success message (shown after form submit)
</div>

<!-- Presentational: hiding for layout reasons -->
<div style="display: none;">
  Mobile menu (hidden on desktop)
</div>
```

---

### Toggle with JavaScript

```javascript
const message = document.getElementById('success-message');

// Show
message.hidden = false;  // or message.removeAttribute('hidden');

// Hide
message.hidden = true;   // or message.setAttribute('hidden', '');
```

---

### Use Cases

**1. Conditional content:**
```html
<form id="login-form">
  <input type="email" name="email">
  <input type="password" name="password">
  <button type="submit">Login</button>
</form>

<div id="success-message" hidden>
  Login successful! Redirecting...
</div>

<script>
document.getElementById('login-form').addEventListener('submit', async (e) => {
  e.preventDefault();
  const success = await login();
  if (success) {
    document.getElementById('login-form').hidden = true;
    document.getElementById('success-message').hidden = false;
  }
});
</script>
```

**2. Progressive disclosure:**
```html
<button onclick="toggleDetails()">Show Details</button>

<div id="details" hidden>
  <p>Additional information...</p>
</div>

<script>
function toggleDetails() {
  const details = document.getElementById('details');
  details.hidden = !details.hidden;
}
</script>
```

---

## The `contenteditable` Attribute

### Purpose

Makes an element's content **editable** by the user.

```html
<div contenteditable="true">
  Edit this text directly!
</div>
```

### Values

- `contenteditable="true"` - Editable
- `contenteditable="false"` - Not editable
- `contenteditable=""` (empty) - Inherits from parent

---

### Use Cases

**1. Simple rich text editor:**
```html
<div 
  id="editor"
  contenteditable="true" 
  style="border: 1px solid #ccc; padding: 10px; min-height: 200px;"
>
  Start typing...
</div>

<button onclick="getContent()">Get HTML</button>

<script>
function getContent() {
  const html = document.getElementById('editor').innerHTML;
  console.log(html);
}
</script>
```

**2. Inline editing:**
```html
<h1 contenteditable="true" id="title">
  Click to edit title
</h1>

<script>
const title = document.getElementById('title');

title.addEventListener('blur', () => {
  // Save to server when user finishes editing
  saveTitle(title.textContent);
});
</script>
```

**3. Mixed editable/non-editable:**
```html
<div contenteditable="true">
  You can edit this text, but not 
  <span contenteditable="false" style="background: yellow;">
    this highlighted part
  </span>.
</div>
```

---

### Gotchas

**1. Messy HTML output:**
```html
<!-- User types: "Hello" -->
<!-- Actual HTML might be: -->
<div><br><span style="font-weight: 400;">Hello</span></div>
```

**Solution:** Sanitize content before saving.

**2. Security risk (XSS):**
```javascript
// ❌ Dangerous
element.innerHTML = userInput;  // User could inject <script>

// ✅ Safe
element.textContent = userInput;  // Escapes HTML
```

---

## Other Important Global Attributes

### `title`

Displays **tooltip** on hover.

```html
<button title="Click to save your changes">Save</button>
<abbr title="HyperText Markup Language">HTML</abbr>
```

**Accessibility note:** Don't rely on `title` for critical information - not all devices show tooltips.

---

### `lang`

Specifies **language** of element's content.

```html
<html lang="en">...</html>  <!-- Whole page in English -->

<p>English text</p>
<p lang="fr">Texte français</p>
<p lang="es">Texto español</p>
```

**Why it matters:**
- Screen readers use correct pronunciation
- Search engines understand content language
- Browser spell-check uses correct dictionary

---

### `dir`

Text **direction**: `ltr` (left-to-right) or `rtl` (right-to-left).

```html
<html dir="ltr">...</html>  <!-- English, most languages -->

<p dir="rtl">مرحبا بك</p>  <!-- Arabic, Hebrew -->
```

---

### `draggable`

Enable **drag-and-drop**.

```html
<div draggable="true" ondragstart="drag(event)">
  Drag me!
</div>

<div ondrop="drop(event)" ondragover="allowDrop(event)">
  Drop zone
</div>

<script>
function drag(ev) {
  ev.dataTransfer.setData("text", ev.target.id);
}

function allowDrop(ev) {
  ev.preventDefault();
}

function drop(ev) {
  ev.preventDefault();
  const data = ev.dataTransfer.getData("text");
  ev.target.appendChild(document.getElementById(data));
}
</script>
```

---

## Best Practices Summary

✅ **Use `id` for unique elements**, `class` for reusable styles

✅ **Use `data-*` for custom data**, not invented attributes

✅ **Use `tabindex="0"` or `-1`**, never positive integers

✅ **Use `hidden` for semantics**, CSS `display: none` for presentation

✅ **Sanitize `contenteditable` output** before saving

✅ **Set `lang` on page and mixed-language content**

✅ **Don't rely on `title` for critical info** (accessibility)

✅ **Validate HTML** to catch invalid attribute usage

---

## Resources

- [MDN: Global attributes](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes)
- [W3C: HTML spec](https://html.spec.whatwg.org/multipage/dom.html#global-attributes)
- [HTML attribute reference](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes)
- [Data attributes guide](https://developer.mozilla.org/en-US/docs/Learn/HTML/Howto/Use_data_attributes)
