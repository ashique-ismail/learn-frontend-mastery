# Label Association, Fieldset/Legend, and Accessibility

## Learning Objectives
- Master proper label association techniques for form accessibility
- Understand fieldset and legend for grouping form controls
- Learn ARIA attributes for enhanced form accessibility
- Implement keyboard navigation and screen reader support
- Build forms that meet WCAG 2.2 standards

---

## Label Association

### Why Labels Matter

Labels are **critical** for accessibility:
- **Screen readers** announce label text when focus moves to input
- **Click target** - clicking label focuses the input (larger hit area)
- **Context** - users understand what to enter
- **Validation** - clear connection between error messages and fields

**Without label:**
```html
<!-- ❌ Inaccessible -->
<input type="text" placeholder="Enter your name">
```

**With label:**
```html
<!-- ✅ Accessible -->
<label for="name">Name:</label>
<input type="text" id="name" name="name">
```

---

### Label Association Methods

#### 1. Explicit Association (Recommended)

Uses `for` attribute pointing to input's `id`.

```html
<label for="email">Email address:</label>
<input type="email" id="email" name="email" required>
```

**Benefits:**
- Works with any HTML structure (label and input don't need to be adjacent)
- Most flexible approach
- Clear semantic relationship

**Multiple labels for one input:**
```html
<label for="username" class="main-label">Username:</label>
<input 
  type="text" 
  id="username" 
  name="username"
  aria-describedby="username-help"
>
<label for="username" class="hint">
  Letters and numbers only, 3-20 characters
</label>
```

---

#### 2. Implicit Association (Wrapping)

Input nested inside label.

```html
<label>
  Email address:
  <input type="email" name="email" required>
</label>
```

**Benefits:**
- Simpler HTML (no need for `id` and `for`)
- Automatic association

**Drawbacks:**
- Less flexible for styling
- Can't have label and input far apart in DOM
- Less explicit in code

**Common pattern with custom styling:**
```html
<label class="form-field">
  <span class="label-text">Email:</span>
  <input type="email" name="email">
</label>

<style>
.form-field {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.label-text {
  font-weight: 600;
  font-size: 14px;
}
</style>
```

---

### Label Best Practices

#### Visible Labels vs Placeholder

```html
<!-- ❌ Bad: Placeholder as label -->
<input type="email" placeholder="Email address">

<!-- ✅ Good: Both label and placeholder -->
<label for="email">Email address:</label>
<input 
  type="email" 
  id="email" 
  placeholder="you@example.com"
>
```

**Why placeholders aren't enough:**
- Disappear when user starts typing
- Low contrast (often gray text)
- Not announced by all screen readers
- Can confuse users about what's already filled

---

#### Required Field Indicators

```html
<!-- Method 1: Visual indicator in label -->
<label for="name">
  Name <span aria-label="required">*</span>
</label>
<input type="text" id="name" required>

<!-- Method 2: Text indicator -->
<label for="email">
  Email <span class="required-text">(required)</span>
</label>
<input type="email" id="email" required>

<!-- Method 3: Icon with proper ARIA -->
<label for="phone">
  Phone 
  <svg aria-label="required" role="img">
    <use href="#asterisk-icon"></use>
  </svg>
</label>
<input type="tel" id="phone" required>

<style>
.required-text {
  font-size: 0.875em;
  color: #666;
}

[aria-label="required"] {
  color: #d00;
  font-weight: bold;
}
</style>
```

**Legend for required fields:**
```html
<form>
  <p class="required-legend">
    Fields marked with <span aria-hidden="true">*</span> are required
  </p>
  
  <!-- form fields -->
</form>
```

---

#### Label Position

**Top-aligned (recommended for most forms):**
```html
<div class="form-field">
  <label for="name">Name:</label>
  <input type="text" id="name">
</div>

<style>
.form-field {
  display: flex;
  flex-direction: column;
  gap: 4px;
  margin-bottom: 16px;
}
</style>
```

**Benefits:**
- Faster scanning (vertical eye movement)
- Works well for long labels
- Mobile-friendly
- Better for translation (labels can expand)

---

**Left-aligned:**
```html
<div class="form-field">
  <label for="email">Email address:</label>
  <input type="email" id="email">
</div>

<style>
.form-field {
  display: grid;
  grid-template-columns: 150px 1fr;
  gap: 12px;
  align-items: center;
  margin-bottom: 16px;
}
</style>
```

**Use case:** Short forms with brief labels (e.g., login forms)

---

#### Multi-line Labels

```html
<label for="terms">
  I accept the 
  <a href="/terms" target="_blank">Terms of Service</a>
  and 
  <a href="/privacy" target="_blank">Privacy Policy</a>
</label>
<input type="checkbox" id="terms" required>

<style>
label {
  display: inline-block;
  max-width: 400px;
  line-height: 1.5;
}
</style>
```

---

### Special Cases

#### Checkbox and Radio Labels

```html
<!-- ✅ Good: Label after input -->
<div class="checkbox-field">
  <input type="checkbox" id="newsletter" name="newsletter">
  <label for="newsletter">Subscribe to newsletter</label>
</div>

<!-- Also good: Wrapped -->
<label class="checkbox-label">
  <input type="checkbox" name="newsletter">
  Subscribe to newsletter
</label>

<style>
.checkbox-field {
  display: flex;
  align-items: center;
  gap: 8px;
}

.checkbox-label {
  display: flex;
  align-items: center;
  gap: 8px;
  cursor: pointer;
}
</style>
```

---

#### File Input Labels

```html
<!-- Native file input (often hidden) -->
<input type="file" id="file-upload" name="file" hidden>

<!-- Custom styled label acts as button -->
<label for="file-upload" class="file-upload-btn">
  <svg aria-hidden="true"><!-- upload icon --></svg>
  Choose File
</label>

<span id="file-name">No file chosen</span>

<style>
.file-upload-btn {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  padding: 10px 20px;
  background: #007bff;
  color: white;
  border-radius: 4px;
  cursor: pointer;
}

.file-upload-btn:hover {
  background: #0056b3;
}
</style>

<script>
const fileInput = document.getElementById('file-upload');
const fileName = document.getElementById('file-name');

fileInput.addEventListener('change', (e) => {
  const file = e.target.files[0];
  fileName.textContent = file ? file.name : 'No file chosen';
});
</script>
```

---

#### Search Inputs

```html
<!-- With visible label -->
<label for="search">Search:</label>
<input type="search" id="search" name="q">

<!-- With visually hidden label (for layout) -->
<label for="search" class="sr-only">Search</label>
<input 
  type="search" 
  id="search" 
  name="q"
  placeholder="Search..."
  aria-label="Search"
>

<style>
/* Screen reader only (visually hidden but accessible) */
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
</style>
```

---

## Fieldset and Legend

### Purpose

**`<fieldset>`** groups related form controls.

**`<legend>`** provides a caption/title for the group.

**Benefits:**
- Semantic grouping
- Screen readers announce legend when entering group
- Visual grouping (browser default border)
- Improved form structure

---

### Basic Usage

```html
<fieldset>
  <legend>Personal Information</legend>
  
  <label for="first-name">First name:</label>
  <input type="text" id="first-name" name="first-name" required>
  
  <label for="last-name">Last name:</label>
  <input type="text" id="last-name" name="last-name" required>
  
  <label for="dob">Date of birth:</label>
  <input type="date" id="dob" name="dob" required>
</fieldset>
```

---

### Radio Button Groups (Essential)

```html
<fieldset>
  <legend>Shipping method:</legend>
  
  <div>
    <input type="radio" id="standard" name="shipping" value="standard" checked>
    <label for="standard">Standard (5-7 days) - Free</label>
  </div>
  
  <div>
    <input type="radio" id="express" name="shipping" value="express">
    <label for="express">Express (2-3 days) - $10</label>
  </div>
  
  <div>
    <input type="radio" id="overnight" name="shipping" value="overnight">
    <label for="overnight">Overnight - $25</label>
  </div>
</fieldset>
```

**Screen reader experience:**
- Announces: "Shipping method, group"
- Then for each option: "Standard (5-7 days) - Free, radio button, checked"

---

### Checkbox Groups

```html
<fieldset>
  <legend>Select your interests:</legend>
  
  <div>
    <input type="checkbox" id="tech" name="interests" value="tech">
    <label for="tech">Technology</label>
  </div>
  
  <div>
    <input type="checkbox" id="sports" name="interests" value="sports">
    <label for="sports">Sports</label>
  </div>
  
  <div>
    <input type="checkbox" id="music" name="interests" value="music">
    <label for="music">Music</label>
  </div>
</fieldset>
```

---

### Nested Fieldsets

```html
<form>
  <fieldset>
    <legend>Account Information</legend>
    
    <label for="username">Username:</label>
    <input type="text" id="username" required>
    
    <label for="email">Email:</label>
    <input type="email" id="email" required>
    
    <fieldset>
      <legend>Account Type:</legend>
      
      <div>
        <input type="radio" id="personal" name="account-type" value="personal">
        <label for="personal">Personal</label>
      </div>
      
      <div>
        <input type="radio" id="business" name="account-type" value="business">
        <label for="business">Business</label>
      </div>
    </fieldset>
  </fieldset>
  
  <button type="submit">Create Account</button>
</form>
```

---

### Styling Fieldsets

**Remove default styling:**
```css
fieldset {
  border: none;
  margin: 0;
  padding: 0;
  min-width: 0; /* Fix IE flexbox bug */
}

legend {
  padding: 0;
  font-weight: 600;
  font-size: 1.125rem;
  margin-bottom: 12px;
}
```

**Custom styling:**
```css
fieldset {
  border: 2px solid #e0e0e0;
  border-radius: 8px;
  padding: 20px;
  margin-bottom: 24px;
}

legend {
  padding: 0 8px;
  font-weight: 600;
  font-size: 1.125rem;
  color: #333;
}
```

**Card-style fieldset:**
```css
fieldset {
  border: 1px solid #ddd;
  border-radius: 8px;
  padding: 24px;
  margin-bottom: 24px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.05);
  background: white;
}

legend {
  font-size: 1.25rem;
  font-weight: 700;
  color: #1a1a1a;
  padding: 0 8px;
}
```

---

### Disabled Fieldsets

**Disable all inputs in group:**
```html
<fieldset disabled>
  <legend>Shipping Address (unavailable for digital products)</legend>
  
  <label for="street">Street:</label>
  <input type="text" id="street" name="street">
  
  <label for="city">City:</label>
  <input type="text" id="city" name="city">
</fieldset>

<style>
fieldset:disabled {
  opacity: 0.6;
  pointer-events: none;
}
</style>
```

---

## ARIA Attributes for Forms

### `aria-label`

Provides accessible name when visible label isn't present.

```html
<!-- Icon button without visible text -->
<button type="submit" aria-label="Submit form">
  <svg><!-- submit icon --></svg>
</button>

<!-- Search with icon only -->
<input 
  type="search" 
  name="q"
  aria-label="Search products"
  placeholder="Search..."
>
```

**When to use:**
- Icon buttons
- Decorative labels
- When visible label doesn't provide enough context

---

### `aria-labelledby`

References one or more elements as the label.

```html
<h2 id="login-heading">Sign In</h2>

<form aria-labelledby="login-heading">
  <label for="username">Username:</label>
  <input type="text" id="username">
  
  <label for="password">Password:</label>
  <input type="password" id="password">
  
  <button type="submit">Sign In</button>
</form>
```

**Multiple references:**
```html
<span id="price-label">Price:</span>
<span id="price-currency">USD</span>

<input 
  type="number" 
  aria-labelledby="price-label price-currency"
  step="0.01"
>
<!-- Screen reader: "Price USD, edit text" -->
```

---

### `aria-describedby`

Links input to description/help text.

```html
<label for="password">Password:</label>
<input 
  type="password" 
  id="password"
  aria-describedby="password-requirements"
  required
>
<div id="password-requirements">
  Must be at least 8 characters with uppercase, lowercase, and numbers
</div>
```

**Multiple descriptions:**
```html
<label for="username">Username:</label>
<input 
  type="text" 
  id="username"
  aria-describedby="username-help username-error"
>
<div id="username-help">3-20 characters, letters and numbers only</div>
<div id="username-error" role="alert" hidden>
  Username is already taken
</div>
```

---

### `aria-required`

Indicates required field (redundant with `required` attribute).

```html
<!-- Modern browsers: use required attribute -->
<input type="text" id="name" required>

<!-- Legacy support: add aria-required -->
<input type="text" id="name" required aria-required="true">
```

**Note:** `required` attribute is sufficient in modern browsers. Use `aria-required` only for custom components.

---

### `aria-invalid`

Indicates validation state.

```html
<label for="email">Email:</label>
<input 
  type="email" 
  id="email"
  aria-invalid="false"
  aria-describedby="email-error"
>
<div id="email-error" role="alert" hidden></div>

<script>
const emailInput = document.getElementById('email');
const emailError = document.getElementById('email-error');

emailInput.addEventListener('blur', () => {
  if (!emailInput.validity.valid) {
    emailInput.setAttribute('aria-invalid', 'true');
    emailError.textContent = 'Please enter a valid email address';
    emailError.hidden = false;
  } else {
    emailInput.setAttribute('aria-invalid', 'false');
    emailError.hidden = true;
  }
});
</script>
```

---

### `role="alert"` and `aria-live`

Announces dynamic messages to screen readers.

```html
<!-- Error message -->
<div id="form-error" role="alert" aria-live="assertive" hidden>
  Please fix the errors below before submitting
</div>

<!-- Success message -->
<div id="form-success" role="status" aria-live="polite" hidden>
  Form submitted successfully
</div>

<script>
form.addEventListener('submit', (e) => {
  e.preventDefault();
  
  if (!form.checkValidity()) {
    const errorDiv = document.getElementById('form-error');
    errorDiv.hidden = false;
    // Screen reader announces immediately (assertive)
  } else {
    const successDiv = document.getElementById('form-success');
    successDiv.hidden = false;
    // Screen reader announces when convenient (polite)
  }
});
</script>
```

**`aria-live` values:**
- `off` - No announcement (default)
- `polite` - Announce at next opportunity (success messages)
- `assertive` - Announce immediately (errors, warnings)

---

## Keyboard Navigation

### Tab Order

**Default tab order:** Order in DOM (source order).

```html
<!-- Tab order: 1. First name, 2. Last name, 3. Email, 4. Submit -->
<form>
  <input type="text" id="first-name">
  <input type="text" id="last-name">
  <input type="email" id="email">
  <button type="submit">Submit</button>
</form>
```

**Visual order should match tab order** for intuitive navigation.

---

### Custom Tab Order (Anti-pattern)

```html
<!-- ❌ Don't use positive tabindex values -->
<input type="text" tabindex="3">
<input type="text" tabindex="1">
<input type="text" tabindex="2">
<!-- Confusing and breaks natural flow -->
```

**Better:** Reorder in DOM or use CSS Grid/Flexbox for visual reordering.

---

### Skip Navigation Link

```html
<a href="#main-content" class="skip-link">Skip to main content</a>

<nav><!-- navigation --></nav>

<main id="main-content" tabindex="-1">
  <h1>Page Title</h1>
  <!-- page content -->
</main>

<style>
.skip-link {
  position: absolute;
  top: -40px;
  left: 0;
  background: #000;
  color: #fff;
  padding: 8px;
  text-decoration: none;
  z-index: 100;
}

.skip-link:focus {
  top: 0;
}
</style>
```

---

### Focus Management

**Focus first invalid field:**
```javascript
form.addEventListener('submit', (e) => {
  if (!form.checkValidity()) {
    e.preventDefault();
    const firstInvalid = form.querySelector(':invalid');
    firstInvalid?.focus();
  }
});
```

**Focus confirmation message:**
```javascript
form.addEventListener('submit', async (e) => {
  e.preventDefault();
  
  await submitForm();
  
  const successMessage = document.getElementById('success');
  successMessage.hidden = false;
  successMessage.tabIndex = -1;
  successMessage.focus();
});
```

---

## Common Mistakes

❌ **Using placeholder as label**
```html
<!-- Wrong -->
<input type="email" placeholder="Email address">

<!-- Correct -->
<label for="email">Email address:</label>
<input type="email" id="email" placeholder="you@example.com">
```

❌ **Missing `for`/`id` association**
```html
<!-- Wrong: no connection -->
<label>Email:</label>
<input type="email" name="email">

<!-- Correct -->
<label for="email">Email:</label>
<input type="email" id="email" name="email">
```

❌ **Fieldset without legend**
```html
<!-- Wrong: missing context -->
<fieldset>
  <input type="radio" name="size" value="small">
  <input type="radio" name="size" value="large">
</fieldset>

<!-- Correct -->
<fieldset>
  <legend>Choose size:</legend>
  <input type="radio" id="small" name="size" value="small">
  <label for="small">Small</label>
  <input type="radio" id="large" name="size" value="large">
  <label for="large">Large</label>
</fieldset>
```

❌ **Incorrect ARIA usage**
```html
<!-- Wrong: aria-label on non-input element -->
<div aria-label="Email">
  <input type="email">
</div>

<!-- Correct -->
<label for="email">Email:</label>
<input type="email" id="email">
```

---

## Best Practices

✅ **Always provide visible labels** - Don't rely on placeholders

✅ **Use explicit label association** (`for`/`id`) - Most flexible and clear

✅ **Group related inputs with fieldset** - Especially radio buttons and checkboxes

✅ **Provide clear error messages** - Link with `aria-describedby`

✅ **Ensure logical tab order** - Visual order should match DOM order

✅ **Test with keyboard only** - All interactions should be possible without mouse

✅ **Test with screen reader** - NVDA (Windows), JAWS (Windows), VoiceOver (Mac/iOS)

✅ **Use native HTML elements** - Better accessibility than custom components

✅ **Indicate required fields clearly** - Visual and programmatic indication

✅ **Provide instructions before form** - Not just inline validation

---

## Resources

- [MDN: label element](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/label)
- [MDN: fieldset element](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/fieldset)
- [W3C: Forms Tutorial](https://www.w3.org/WAI/tutorials/forms/)
- [WebAIM: Creating Accessible Forms](https://webaim.org/techniques/forms/)
- [WCAG 2.2: Labels or Instructions](https://www.w3.org/WAI/WCAG22/Understanding/labels-or-instructions.html)
- [A11y Project: Form best practices](https://www.a11yproject.com/posts/how-to-write-accessible-forms/)
