# Autocomplete, Autofill, and Autofocus Semantics

## Learning Objectives
- Master the `autocomplete` attribute for optimized form filling
- Understand browser autofill behavior and how to control it
- Learn when and how to use `autofocus` appropriately
- Implement user-friendly autocomplete experiences
- Balance convenience with security and privacy

---

## The `autocomplete` Attribute

### Purpose and Benefits

The `autocomplete` attribute tells browsers **what kind of data** is expected in an input field, enabling:
- **Faster form filling** - One-click autofill from saved data
- **Fewer typos** - Data comes from browser's database
- **Better UX** - Especially on mobile devices
- **Privacy** - Users control what's saved
- **Accessibility** - Consistent data entry for users with cognitive disabilities

---

### Basic Syntax

```html
<!-- Enable autocomplete (default) -->
<input type="text" name="name" autocomplete="name">

<!-- Disable autocomplete -->
<input type="text" name="secret" autocomplete="off">

<!-- Autocomplete on form level -->
<form autocomplete="on">
  <input type="text" name="email">
  <input type="text" name="address">
</form>

<form autocomplete="off">
  <input type="text" name="otp">
</form>
```

---

### Common Autocomplete Values

#### Personal Information

```html
<!-- Name -->
<input type="text" name="name" autocomplete="name">
<input type="text" name="first-name" autocomplete="given-name">
<input type="text" name="last-name" autocomplete="family-name">
<input type="text" name="middle-name" autocomplete="additional-name">

<!-- Name affixes -->
<input type="text" name="prefix" autocomplete="honorific-prefix">
<!-- Example: Mr., Mrs., Dr. -->

<input type="text" name="suffix" autocomplete="honorific-suffix">
<!-- Example: Jr., Sr., III -->

<!-- Nickname -->
<input type="text" name="nickname" autocomplete="nickname">

<!-- Email -->
<input type="email" name="email" autocomplete="email">

<!-- Phone -->
<input type="tel" name="phone" autocomplete="tel">
<input type="tel" name="mobile" autocomplete="tel">
<input type="tel" name="country-code" autocomplete="tel-country-code">
<input type="tel" name="national" autocomplete="tel-national">
<input type="tel" name="area-code" autocomplete="tel-area-code">
<input type="tel" name="local" autocomplete="tel-local">
<input type="tel" name="extension" autocomplete="tel-extension">

<!-- Username -->
<input type="text" name="username" autocomplete="username">
```

---

#### Address Information

```html
<!-- Full address -->
<textarea name="address" autocomplete="street-address"></textarea>

<!-- Address components -->
<input type="text" name="address-line1" autocomplete="address-line1">
<input type="text" name="address-line2" autocomplete="address-line2">
<input type="text" name="address-line3" autocomplete="address-line3">

<!-- City/locality -->
<input type="text" name="city" autocomplete="address-level2">

<!-- State/province/region -->
<input type="text" name="state" autocomplete="address-level1">

<!-- ZIP/postal code -->
<input type="text" name="zip" autocomplete="postal-code">

<!-- Country -->
<select name="country" autocomplete="country">
  <option value="US">United States</option>
  <option value="CA">Canada</option>
</select>

<select name="country-code" autocomplete="country-name">
  <option value="United States">United States</option>
  <option value="Canada">Canada</option>
</select>
```

**Complete address form:**
```html
<fieldset>
  <legend>Shipping Address</legend>
  
  <label for="street">Street Address:</label>
  <input type="text" id="street" name="street" autocomplete="shipping street-address" required>
  
  <label for="city">City:</label>
  <input type="text" id="city" name="city" autocomplete="shipping address-level2" required>
  
  <label for="state">State:</label>
  <input type="text" id="state" name="state" autocomplete="shipping address-level1" required>
  
  <label for="zip">ZIP Code:</label>
  <input type="text" id="zip" name="zip" autocomplete="shipping postal-code" required>
  
  <label for="country">Country:</label>
  <select id="country" name="country" autocomplete="shipping country" required>
    <option value="US">United States</option>
    <option value="CA">Canada</option>
  </select>
</fieldset>
```

---

#### Payment Information

```html
<!-- Credit card number -->
<input 
  type="text" 
  name="cc-number" 
  autocomplete="cc-number"
  inputmode="numeric"
  pattern="[0-9]{13,19}"
>

<!-- Cardholder name -->
<input type="text" name="cc-name" autocomplete="cc-name">

<!-- Expiry date -->
<input type="month" name="cc-exp" autocomplete="cc-exp">

<!-- Or separate month and year -->
<input type="text" name="cc-exp-month" autocomplete="cc-exp-month" placeholder="MM">
<input type="text" name="cc-exp-year" autocomplete="cc-exp-year" placeholder="YYYY">

<!-- CVV/CVC -->
<input 
  type="text" 
  name="cc-csc" 
  autocomplete="cc-csc"
  inputmode="numeric"
  pattern="[0-9]{3,4}"
>

<!-- Card type -->
<select name="cc-type" autocomplete="cc-type">
  <option value="visa">Visa</option>
  <option value="mastercard">Mastercard</option>
  <option value="amex">American Express</option>
</select>
```

**Complete payment form:**
```html
<fieldset>
  <legend>Payment Information</legend>
  
  <label for="cc-name">Name on Card:</label>
  <input type="text" id="cc-name" name="cc-name" autocomplete="cc-name" required>
  
  <label for="cc-number">Card Number:</label>
  <input 
    type="text" 
    id="cc-number" 
    name="cc-number" 
    autocomplete="cc-number"
    inputmode="numeric"
    pattern="[0-9]{13,19}"
    required
  >
  
  <div class="form-row">
    <div>
      <label for="cc-exp">Expiry:</label>
      <input type="month" id="cc-exp" name="cc-exp" autocomplete="cc-exp" required>
    </div>
    
    <div>
      <label for="cc-csc">CVV:</label>
      <input 
        type="text" 
        id="cc-csc" 
        name="cc-csc" 
        autocomplete="cc-csc"
        inputmode="numeric"
        pattern="[0-9]{3,4}"
        required
      >
    </div>
  </div>
</fieldset>
```

---

#### Authentication

```html
<!-- Current password (login) -->
<input 
  type="password" 
  name="password" 
  autocomplete="current-password"
  required
>

<!-- New password (registration/change) -->
<input 
  type="password" 
  name="new-password" 
  autocomplete="new-password"
  required
>

<!-- One-time password -->
<input 
  type="text" 
  name="otp" 
  autocomplete="one-time-code"
  inputmode="numeric"
  pattern="[0-9]{6}"
>
```

**Login form:**
```html
<form method="POST" action="/login">
  <label for="username">Username or Email:</label>
  <input 
    type="text" 
    id="username" 
    name="username" 
    autocomplete="username"
    required
  >
  
  <label for="password">Password:</label>
  <input 
    type="password" 
    id="password" 
    name="password" 
    autocomplete="current-password"
    required
  >
  
  <button type="submit">Sign In</button>
</form>
```

**Registration form:**
```html
<form method="POST" action="/register">
  <label for="email">Email:</label>
  <input 
    type="email" 
    id="email" 
    name="email" 
    autocomplete="email"
    required
  >
  
  <label for="username">Username:</label>
  <input 
    type="text" 
    id="username" 
    name="username" 
    autocomplete="username"
    required
  >
  
  <label for="new-password">Password:</label>
  <input 
    type="password" 
    id="new-password" 
    name="new-password" 
    autocomplete="new-password"
    required
  >
  
  <button type="submit">Create Account</button>
</form>
```

---

#### Work and Organization

```html
<!-- Company/organization -->
<input type="text" name="company" autocomplete="organization">

<!-- Job title -->
<input type="text" name="title" autocomplete="organization-title">
```

---

#### Other Values

```html
<!-- URL -->
<input type="url" name="website" autocomplete="url">

<!-- Photo/avatar -->
<input type="url" name="photo" autocomplete="photo">

<!-- Birthday -->
<input type="date" name="birthday" autocomplete="bday">

<!-- Or separate components -->
<input type="number" name="birth-day" autocomplete="bday-day" min="1" max="31">
<input type="number" name="birth-month" autocomplete="bday-month" min="1" max="12">
<input type="number" name="birth-year" autocomplete="bday-year" min="1900" max="2025">

<!-- Gender -->
<select name="gender" autocomplete="sex">
  <option value="">Prefer not to say</option>
  <option value="M">Male</option>
  <option value="F">Female</option>
</select>

<!-- Language preference -->
<select name="language" autocomplete="language">
  <option value="en">English</option>
  <option value="es">Spanish</option>
  <option value="fr">French</option>
</select>

<!-- Transaction amount -->
<input type="number" name="amount" autocomplete="transaction-amount" step="0.01">

<!-- Transaction currency -->
<select name="currency" autocomplete="transaction-currency">
  <option value="USD">USD</option>
  <option value="EUR">EUR</option>
  <option value="GBP">GBP</option>
</select>
```

---

### Section Tokens

Distinguish between multiple instances (e.g., billing vs shipping).

**Syntax:** `section-{name} {autocomplete-value}`

```html
<!-- Billing address -->
<fieldset>
  <legend>Billing Address</legend>
  
  <input type="text" name="billing-name" autocomplete="section-billing name">
  <input type="text" name="billing-street" autocomplete="section-billing street-address">
  <input type="text" name="billing-city" autocomplete="section-billing address-level2">
  <input type="text" name="billing-state" autocomplete="section-billing address-level1">
  <input type="text" name="billing-zip" autocomplete="section-billing postal-code">
</fieldset>

<!-- Shipping address -->
<fieldset>
  <legend>Shipping Address</legend>
  
  <input type="text" name="shipping-name" autocomplete="section-shipping name">
  <input type="text" name="shipping-street" autocomplete="section-shipping street-address">
  <input type="text" name="shipping-city" autocomplete="section-shipping address-level2">
  <input type="text" name="shipping-state" autocomplete="section-shipping address-level1">
  <input type="text" name="shipping-zip" autocomplete="section-shipping postal-code">
</fieldset>
```

**Alternative syntax (simpler, also works):**
```html
<!-- Billing -->
<input autocomplete="billing name">
<input autocomplete="billing street-address">

<!-- Shipping -->
<input autocomplete="shipping name">
<input autocomplete="shipping street-address">
```

---

### Disabling Autocomplete

**When to disable:**
- One-time codes (OTP, CAPTCHA)
- Sensitive admin forms
- Search boxes (sometimes)
- Forms with dynamic field meanings

```html
<!-- Disable on specific field -->
<input type="text" name="otp" autocomplete="off">

<!-- Disable on entire form -->
<form autocomplete="off">
  <input type="text" name="code">
  <input type="text" name="verification">
</form>
```

**Note:** Some browsers may ignore `autocomplete="off"` for username/password fields for security reasons.

**Force disable password autocomplete:**
```html
<input 
  type="password" 
  name="password" 
  autocomplete="new-password"
>
<!-- Using "new-password" instead of "off" works better -->
```

---

## Autofill Behavior

### How Browser Autofill Works

1. **User saves data** - Through form submission or browser prompt
2. **Browser analyzes form** - Uses `autocomplete`, `name`, `id`, labels
3. **Browser matches data** - Finds relevant saved information
4. **User triggers autofill** - Clicks field or uses keyboard shortcut
5. **Browser fills fields** - Populates all matching fields

---

### Autofill Heuristics

Browsers use multiple signals to determine what data to autofill:

**1. `autocomplete` attribute (primary):**
```html
<input type="email" name="user_email" autocomplete="email">
```

**2. `name` attribute:**
```html
<input type="text" name="first_name">
<!-- Browser infers this is a first name -->
```

**3. `id` attribute:**
```html
<input type="text" id="email-address">
```

**4. Associated label:**
```html
<label for="phone">Phone Number:</label>
<input type="tel" id="phone">
```

**5. Input type:**
```html
<input type="email">
<!-- Browser knows it's an email field -->
```

**Best practice:** Use explicit `autocomplete` attributes instead of relying on heuristics.

---

### Testing Autofill

**Chrome DevTools:**
1. Right-click input → Inspect
2. Elements panel → Event Listeners → Show "autofill" events
3. Fill form and check which fields are autofilled

**Manual testing:**
1. Save data in browser (submit form or use browser's save prompt)
2. Clear form
3. Start typing in field
4. Check dropdown suggestions
5. Select suggestion to autofill

---

### Styling Autofilled Inputs

**Default browser styles:**
```css
/* Chrome/Safari: yellow background */
input:-webkit-autofill {
  background-color: #FAFFBD;
}

/* Firefox: light blue background */
input:-moz-autofill {
  background-color: #F0F8FF;
}
```

**Custom styling:**
```css
/* Override autofill background */
input:-webkit-autofill,
input:-webkit-autofill:hover,
input:-webkit-autofill:focus {
  -webkit-box-shadow: 0 0 0 1000px white inset;
  -webkit-text-fill-color: #000;
}

/* Dark theme autofill */
input:-webkit-autofill {
  -webkit-box-shadow: 0 0 0 1000px #1a1a1a inset;
  -webkit-text-fill-color: #fff;
}

/* Subtle highlight */
input:-webkit-autofill {
  border: 2px solid #4CAF50;
  background-color: #f0fff0 !important;
}
```

**Detect autofill with JavaScript:**
```javascript
const input = document.getElementById('email');

// Method 1: :-webkit-autofill pseudo-class
const isAutofilled = input.matches(':-webkit-autofill');

// Method 2: Listen for autofill event
input.addEventListener('change', (e) => {
  if (e.inputType === null) {
    // Likely autofilled (no user input)
    console.log('Field was autofilled');
  }
});

// Method 3: MutationObserver for style changes
const observer = new MutationObserver((mutations) => {
  mutations.forEach((mutation) => {
    if (mutation.attributeName === 'style') {
      console.log('Autofill detected');
    }
  });
});

observer.observe(input, { attributes: true });
```

---

## The `autofocus` Attribute

### Purpose

Automatically focuses an input when the page loads.

```html
<form>
  <input type="text" name="search" autofocus>
  <button type="submit">Search</button>
</form>
```

**Behavior:**
- Only **one** element can have autofocus per page
- Focus moves to that element on page load
- Useful for:
  - Search pages
  - Login pages
  - Form-centric pages

---

### When to Use

**✅ Good use cases:**

**Search pages:**
```html
<h1>Search</h1>
<form>
  <input type="search" name="q" autofocus placeholder="Search...">
  <button type="submit">Search</button>
</form>
```

**Login pages:**
```html
<form method="POST" action="/login">
  <label for="username">Username:</label>
  <input type="text" id="username" name="username" autofocus required>
  
  <label for="password">Password:</label>
  <input type="password" id="password" name="password" required>
  
  <button type="submit">Sign In</button>
</form>
```

**Single-purpose forms:**
```html
<h1>Subscribe to Newsletter</h1>
<form>
  <input type="email" name="email" autofocus required>
  <button type="submit">Subscribe</button>
</form>
```

---

### When NOT to Use

**❌ Bad use cases:**

**Long pages:**
```html
<!-- Bad: Page has content above form -->
<header><!-- 2 screens of content --></header>
<main>
  <form>
    <input type="text" autofocus> <!-- Jumps past content -->
  </form>
</main>
```

**Multiple forms:**
```html
<!-- Bad: Confusing which form is active -->
<form id="form1">
  <input type="text" autofocus>
</form>

<form id="form2">
  <input type="text">
</form>
```

**Modal dialogs (use JavaScript instead):**
```html
<!-- Bad: Modal not open yet -->
<dialog id="modal">
  <input type="text" autofocus>
</dialog>

<!-- Good: Focus when opening -->
<script>
const modal = document.getElementById('modal');
const input = modal.querySelector('input');

function openModal() {
  modal.showModal();
  input.focus(); // Explicit focus
}
</script>
```

---

### Accessibility Concerns

**Problem:** `autofocus` can disorient users:
- Screen reader users may miss page context
- Keyboard users may have scroll position jump
- Can interfere with browser's "scroll to previous position" feature

**Solution:** Consider user preferences:

```javascript
// Check if user has "prefers-reduced-motion"
const prefersReducedMotion = window.matchMedia('(prefers-reduced-motion: reduce)').matches;

if (!prefersReducedMotion) {
  document.getElementById('search').focus();
}
```

**Alternative:** Focus on user interaction:
```html
<button onclick="showSearchForm()">Search</button>

<form id="search-form" hidden>
  <input type="search" id="search-input">
  <button type="submit">Search</button>
</form>

<script>
function showSearchForm() {
  const form = document.getElementById('search-form');
  const input = document.getElementById('search-input');
  
  form.hidden = false;
  input.focus(); // Focus after user action
}
</script>
```

---

### Programmatic Focus

**Better alternative to `autofocus` in many cases:**

```javascript
// Focus after page load
window.addEventListener('load', () => {
  const input = document.getElementById('search');
  
  // Only focus if user hasn't interacted yet
  if (document.activeElement === document.body) {
    input.focus();
  }
});

// Focus with smooth scroll
const input = document.getElementById('email');
input.scrollIntoView({ behavior: 'smooth', block: 'center' });
input.focus();

// Focus with selection
const textInput = document.getElementById('edit-title');
textInput.focus();
textInput.select(); // Select all text for easy editing
```

---

## Common Mistakes

❌ **Wrong autocomplete values**
```html
<!-- Wrong -->
<input type="text" name="first-name" autocomplete="firstname">

<!-- Correct -->
<input type="text" name="first-name" autocomplete="given-name">
```

❌ **Disabling autocomplete unnecessarily**
```html
<!-- Wrong: Hurts UX for no reason -->
<form autocomplete="off">
  <input type="text" name="name">
  <input type="email" name="email">
</form>

<!-- Correct: Enable autocomplete -->
<form>
  <input type="text" name="name" autocomplete="name">
  <input type="email" name="email" autocomplete="email">
</form>
```

❌ **Multiple autofocus attributes**
```html
<!-- Wrong: Only one should have autofocus -->
<input type="text" autofocus>
<input type="text" autofocus> <!-- This one wins (last in DOM) -->
```

❌ **Autofocus on non-primary content**
```html
<!-- Wrong: Newsletter signup shouldn't steal focus -->
<article>
  <h1>Article Title</h1>
  <p>Article content...</p>
</article>

<aside>
  <form>
    <input type="email" autofocus> <!-- Annoying! -->
  </form>
</aside>
```

❌ **Not handling autofill styling**
```html
<!-- Wrong: Autofilled inputs look broken in dark theme -->
<style>
body { background: #000; color: #fff; }
input { background: #333; color: #fff; }
/* Autofilled inputs get yellow background from browser */
</style>

<!-- Correct: Override autofill styles -->
<style>
input:-webkit-autofill {
  -webkit-box-shadow: 0 0 0 1000px #333 inset;
  -webkit-text-fill-color: #fff;
}
</style>
```

---

## Best Practices

✅ **Always use appropriate autocomplete values** - Improves UX dramatically

✅ **Use section tokens for multiple addresses** - Billing vs shipping

✅ **Don't disable autocomplete unnecessarily** - Users appreciate autofill

✅ **Use `autofocus` sparingly** - Only on single-purpose pages

✅ **Style autofilled inputs** - Match your design system

✅ **Test autofill in multiple browsers** - Chrome, Firefox, Safari

✅ **Use `autocomplete="one-time-code"` for OTP** - iOS/Android will suggest codes from SMS

✅ **Combine with proper input types** - `type="email"`, `type="tel"`, etc.

✅ **Consider privacy** - Don't force users to save sensitive data

✅ **Provide manual option** - Don't rely solely on autofill

---

## Resources

- [MDN: autocomplete attribute](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/autocomplete)
- [HTML Standard: Autofill](https://html.spec.whatwg.org/multipage/form-control-infrastructure.html#autofill)
- [MDN: autofocus attribute](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/input#autofocus)
- [Chrome Developers: Autofill best practices](https://web.dev/sign-in-form-best-practices/)
- [WebKit: Autofill](https://webkit.org/blog/7769/how-to-fix-password-autofill-problems/)
