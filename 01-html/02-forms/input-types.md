# Input Types

## Learning Objectives
- Master all HTML5 input types and their specific behaviors
- Understand browser validation and user experience implications
- Learn when to use each input type for optimal UX and accessibility
- Implement proper fallback strategies for unsupported input types

---

## Overview of Input Types

HTML5 introduced many specialized input types beyond the basic `text` field. Each type provides:
- **Native validation** - Built-in client-side validation
- **Optimized UI** - Mobile keyboards and pickers tailored to the input
- **Semantic meaning** - Clear intent for assistive technologies
- **Graceful degradation** - Fallback to `type="text"` in unsupported browsers

```html
<!-- Basic syntax -->
<input type="text" name="username" id="username">
<input type="email" name="email" id="email">
<input type="number" name="age" id="age">
```

---

## Text-Based Input Types

### `type="text"`

**Purpose:** Single-line plain text input (default type).

```html
<label for="username">Username:</label>
<input 
  type="text" 
  id="username" 
  name="username"
  placeholder="Enter username"
  maxlength="20"
  required
>
```

**Common attributes:**
- `placeholder` - Hint text (disappears on focus)
- `maxlength` - Maximum character count
- `minlength` - Minimum character count
- `size` - Visual width (in characters)
- `autocomplete` - Enable/disable autocomplete

**Use cases:**
- Names, usernames, titles
- Short text fields
- Single-line addresses

---

### `type="email"`

**Purpose:** Email address input with built-in validation.

```html
<label for="email">Email:</label>
<input 
  type="email" 
  id="email" 
  name="email"
  placeholder="user@example.com"
  required
  autocomplete="email"
>
```

**Validation:**
- Must contain `@` symbol
- Must have characters before and after `@`
- More lenient than RFC 5322 spec

**Browser behavior:**
- Mobile: Shows email-optimized keyboard (@ and . keys prominent)
- Desktop: Basic validation on submit

**Multiple emails:**
```html
<input 
  type="email" 
  name="recipients"
  multiple
  placeholder="email1@example.com, email2@example.com"
>
```

**Pattern for stricter validation:**
```html
<input 
  type="email" 
  pattern="[a-z0-9._%+-]+@[a-z0-9.-]+\.[a-z]{2,}$"
  title="Please enter a valid email address"
>
```

---

### `type="tel"`

**Purpose:** Telephone number input.

```html
<label for="phone">Phone:</label>
<input 
  type="tel" 
  id="phone" 
  name="phone"
  placeholder="+1 (555) 123-4567"
  pattern="[0-9]{3}-[0-9]{3}-[0-9]{4}"
  title="Format: 123-456-7890"
  autocomplete="tel"
>
```

**Key points:**
- **No built-in validation** (phone formats vary globally)
- Mobile: Shows numeric keypad
- Use `pattern` attribute for format validation

**International format:**
```html
<input 
  type="tel" 
  pattern="^\+?[1-9]\d{1,14}$"
  title="Enter phone number in E.164 format (+1234567890)"
>
```

---

### `type="url"`

**Purpose:** Web URL input with validation.

```html
<label for="website">Website:</label>
<input 
  type="url" 
  id="website" 
  name="website"
  placeholder="https://example.com"
  required
>
```

**Validation:**
- Must start with a protocol (`http://`, `https://`, `ftp://`, etc.)
- Browser validates basic URL structure

**Mobile behavior:**
- Shows keyboard with `.com` and `/` keys

**Pattern for specific protocols:**
```html
<input 
  type="url" 
  pattern="https://.*"
  title="URL must start with https://"
>
```

---

### `type="search"`

**Purpose:** Search query input.

```html
<label for="search">Search:</label>
<input 
  type="search" 
  id="search" 
  name="q"
  placeholder="Search..."
  autocomplete="off"
>
```

**Differences from `text`:**
- Some browsers show a clear button (×)
- Rounded corners in some browsers (Safari)
- Enter key may show "Search" instead of "Go"

**With datalist (autocomplete suggestions):**
```html
<input 
  type="search" 
  list="search-suggestions"
  placeholder="Search products..."
>

<datalist id="search-suggestions">
  <option value="Laptop">
  <option value="Mouse">
  <option value="Keyboard">
</datalist>
```

---

### `type="password"`

**Purpose:** Password input (obscured text).

```html
<label for="password">Password:</label>
<input 
  type="password" 
  id="password" 
  name="password"
  minlength="8"
  required
  autocomplete="current-password"
>
```

**Behavior:**
- Text is masked with dots or asterisks
- No spell-check
- No autocorrect
- Copy/paste may be restricted in some browsers

**Password visibility toggle:**
```html
<div class="password-field">
  <input 
    type="password" 
    id="password" 
    name="password"
  >
  <button 
    type="button" 
    onclick="togglePassword()"
    aria-label="Show password"
  >
    Show
  </button>
</div>

<script>
function togglePassword() {
  const input = document.getElementById('password');
  const button = event.target;
  
  if (input.type === 'password') {
    input.type = 'text';
    button.textContent = 'Hide';
    button.setAttribute('aria-label', 'Hide password');
  } else {
    input.type = 'password';
    button.textContent = 'Show';
    button.setAttribute('aria-label', 'Show password');
  }
}
</script>
```

**Autocomplete values:**
- `new-password` - For registration/password change forms
- `current-password` - For login forms

---

## Numeric Input Types

### `type="number"`

**Purpose:** Numeric input with spinner controls.

```html
<label for="quantity">Quantity:</label>
<input 
  type="number" 
  id="quantity" 
  name="quantity"
  min="1"
  max="100"
  step="1"
  value="1"
  required
>
```

**Attributes:**
- `min` - Minimum value
- `max` - Maximum value
- `step` - Increment/decrement step
- `value` - Default value

**Validation:**
- Accepts integers and decimals
- Scientific notation (1e10)
- Rejects non-numeric characters

**Mobile behavior:**
- Shows numeric keypad

**Decimal numbers:**
```html
<input 
  type="number" 
  step="0.01"
  min="0"
  placeholder="0.00"
>
```

**When NOT to use `type="number"`:**

```html
<!-- ❌ Bad: Credit card numbers (use type="text" with inputmode="numeric") -->
<input type="number" placeholder="Credit card">

<!-- ✅ Good: -->
<input 
  type="text" 
  inputmode="numeric" 
  pattern="[0-9]{13,19}"
  placeholder="1234 5678 9012 3456"
>

<!-- ❌ Bad: ZIP codes (leading zeros get stripped) -->
<input type="number" placeholder="ZIP">

<!-- ✅ Good: -->
<input 
  type="text" 
  inputmode="numeric" 
  pattern="[0-9]{5}"
  placeholder="12345"
>
```

**Why?** `type="number"` strips leading zeros, allows scientific notation, and shows spinner controls which don't make sense for IDs/codes.

---

### `type="range"`

**Purpose:** Slider control for numeric selection.

```html
<label for="volume">Volume:</label>
<input 
  type="range" 
  id="volume" 
  name="volume"
  min="0"
  max="100"
  step="1"
  value="50"
>
<output for="volume">50</output>

<script>
const range = document.getElementById('volume');
const output = document.querySelector('output[for="volume"]');

range.addEventListener('input', (e) => {
  output.textContent = e.target.value;
});
</script>
```

**Attributes:**
- `min` - Minimum value (default: 0)
- `max` - Maximum value (default: 100)
- `step` - Increment step (default: 1)
- `value` - Initial value

**Visual feedback with datalist:**
```html
<input 
  type="range" 
  min="0" 
  max="100" 
  step="25"
  list="markers"
>

<datalist id="markers">
  <option value="0" label="0%">
  <option value="25" label="25%">
  <option value="50" label="50%">
  <option value="75" label="75%">
  <option value="100" label="100%">
</datalist>
```

**Use cases:**
- Volume controls
- Brightness/contrast adjustments
- Price filters
- Rating scales

---

## Date and Time Input Types

### `type="date"`

**Purpose:** Date picker (year, month, day).

```html
<label for="birthday">Birthday:</label>
<input 
  type="date" 
  id="birthday" 
  name="birthday"
  min="1900-01-01"
  max="2025-12-31"
  value="2000-01-01"
>
```

**Value format:** `YYYY-MM-DD` (ISO 8601)

**Browser support:**
- Good support in modern browsers
- Safari on desktop: native picker since Safari 14.1
- Fallback: Shows text input in unsupported browsers

**Date range:**
```html
<label for="checkin">Check-in:</label>
<input 
  type="date" 
  id="checkin" 
  name="checkin"
  min="2025-05-26"
>

<label for="checkout">Check-out:</label>
<input 
  type="date" 
  id="checkout" 
  name="checkout"
  min="2025-05-27"
>

<script>
document.getElementById('checkin').addEventListener('change', (e) => {
  const checkin = new Date(e.target.value);
  const nextDay = new Date(checkin);
  nextDay.setDate(nextDay.getDate() + 1);
  
  const checkout = document.getElementById('checkout');
  checkout.min = nextDay.toISOString().split('T')[0];
});
</script>
```

---

### `type="time"`

**Purpose:** Time picker (hours and minutes).

```html
<label for="appointment">Appointment time:</label>
<input 
  type="time" 
  id="appointment" 
  name="appointment"
  min="09:00"
  max="18:00"
  step="900"
  value="12:00"
>
```

**Value format:** `HH:MM` or `HH:MM:SS`

**Attributes:**
- `step` - Step in seconds (900 = 15 minutes)
- `min` / `max` - Time range

**Use cases:**
- Appointment scheduling
- Opening hours
- Alarm settings

---

### `type="datetime-local"`

**Purpose:** Date and time picker (no timezone).

```html
<label for="meeting">Meeting:</label>
<input 
  type="datetime-local" 
  id="meeting" 
  name="meeting"
  min="2025-05-26T00:00"
  max="2025-12-31T23:59"
>
```

**Value format:** `YYYY-MM-DDTHH:MM`

**Note:** No timezone info (use server-side timezone handling).

---

### `type="month"`

**Purpose:** Month and year picker.

```html
<label for="expiry">Card expiry:</label>
<input 
  type="month" 
  id="expiry" 
  name="expiry"
  min="2025-05"
>
```

**Value format:** `YYYY-MM`

**Use cases:**
- Credit card expiry
- Monthly reports
- Subscription periods

---

### `type="week"`

**Purpose:** Week and year picker.

```html
<label for="week">Select week:</label>
<input 
  type="week" 
  id="week" 
  name="week"
  value="2025-W21"
>
```

**Value format:** `YYYY-WNN` (W = week)

**Use cases:**
- Weekly schedules
- Project planning
- Timesheet submission

---

## Color and File Input Types

### `type="color"`

**Purpose:** Color picker.

```html
<label for="bg-color">Background color:</label>
<input 
  type="color" 
  id="bg-color" 
  name="bg-color"
  value="#ff0000"
>

<script>
document.getElementById('bg-color').addEventListener('input', (e) => {
  document.body.style.backgroundColor = e.target.value;
});
</script>
```

**Value format:** Hex color code (`#RRGGBB`)

**Default:** `#000000` (black)

**Browser behavior:**
- Opens native color picker
- Returns lowercase hex value

**With opacity:**
```html
<!-- Color picker doesn't support alpha channel -->
<!-- Use separate inputs: -->
<input type="color" id="color" value="#ff0000">
<input type="range" id="opacity" min="0" max="1" step="0.1" value="1">

<script>
function updateColor() {
  const color = document.getElementById('color').value;
  const opacity = document.getElementById('opacity').value;
  
  // Convert hex to rgba
  const r = parseInt(color.substr(1,2), 16);
  const g = parseInt(color.substr(3,2), 16);
  const b = parseInt(color.substr(5,2), 16);
  
  document.body.style.backgroundColor = `rgba(${r}, ${g}, ${b}, ${opacity})`;
}

document.getElementById('color').addEventListener('input', updateColor);
document.getElementById('opacity').addEventListener('input', updateColor);
</script>
```

---

### `type="file"`

**Purpose:** File upload input.

**Single file:**
```html
<label for="avatar">Profile picture:</label>
<input 
  type="file" 
  id="avatar" 
  name="avatar"
  accept="image/png, image/jpeg"
  required
>
```

**Multiple files:**
```html
<label for="photos">Upload photos:</label>
<input 
  type="file" 
  id="photos" 
  name="photos"
  accept="image/*"
  multiple
>
```

**Attributes:**
- `accept` - Allowed file types (MIME types or extensions)
- `multiple` - Allow multiple files
- `capture` - Mobile: use camera/microphone

**Common accept values:**
```html
<!-- Images -->
<input type="file" accept="image/*">
<input type="file" accept="image/png, image/jpeg">
<input type="file" accept=".png, .jpg, .jpeg">

<!-- Documents -->
<input type="file" accept=".pdf, .doc, .docx">
<input type="file" accept="application/pdf">

<!-- Video -->
<input type="file" accept="video/*">

<!-- Audio -->
<input type="file" accept="audio/*">
```

**Accessing files with JavaScript:**
```html
<input type="file" id="file-input" accept="image/*">
<img id="preview" style="max-width: 200px;">

<script>
document.getElementById('file-input').addEventListener('change', (e) => {
  const file = e.target.files[0];
  
  if (file) {
    console.log('Name:', file.name);
    console.log('Size:', file.size, 'bytes');
    console.log('Type:', file.type);
    
    // Show preview
    const reader = new FileReader();
    reader.onload = (e) => {
      document.getElementById('preview').src = e.target.result;
    };
    reader.readAsDataURL(file);
  }
});
</script>
```

**Drag and drop upload:**
```html
<div 
  id="drop-zone"
  style="border: 2px dashed #ccc; padding: 40px; text-align: center;"
>
  Drag files here or <label for="file-input" style="color: blue; cursor: pointer;">browse</label>
  <input type="file" id="file-input" multiple hidden>
</div>

<script>
const dropZone = document.getElementById('drop-zone');
const fileInput = document.getElementById('file-input');

// Prevent default drag behaviors
['dragenter', 'dragover', 'dragleave', 'drop'].forEach(eventName => {
  dropZone.addEventListener(eventName, preventDefaults, false);
  document.body.addEventListener(eventName, preventDefaults, false);
});

function preventDefaults(e) {
  e.preventDefault();
  e.stopPropagation();
}

// Highlight drop zone
['dragenter', 'dragover'].forEach(eventName => {
  dropZone.addEventListener(eventName, () => {
    dropZone.style.borderColor = '#000';
  });
});

['dragleave', 'drop'].forEach(eventName => {
  dropZone.addEventListener(eventName, () => {
    dropZone.style.borderColor = '#ccc';
  });
});

// Handle drop
dropZone.addEventListener('drop', (e) => {
  const files = e.dataTransfer.files;
  handleFiles(files);
});

// Handle file input
fileInput.addEventListener('change', (e) => {
  handleFiles(e.target.files);
});

function handleFiles(files) {
  [...files].forEach(file => {
    console.log('Uploaded:', file.name);
  });
}
</script>
```

**Mobile camera capture:**
```html
<!-- Opens camera directly on mobile -->
<input type="file" accept="image/*" capture="environment">

<!-- Front camera -->
<input type="file" accept="image/*" capture="user">

<!-- Video recording -->
<input type="file" accept="video/*" capture="environment">

<!-- Audio recording -->
<input type="file" accept="audio/*" capture="user">
```

---

## Other Input Types

### `type="checkbox"`

**Purpose:** Toggle option (on/off).

```html
<input 
  type="checkbox" 
  id="terms" 
  name="terms"
  value="accepted"
  required
>
<label for="terms">I accept the terms and conditions</label>
```

**Checked by default:**
```html
<input type="checkbox" id="newsletter" checked>
<label for="newsletter">Subscribe to newsletter</label>
```

**Multiple checkboxes (same name):**
```html
<fieldset>
  <legend>Select toppings:</legend>
  
  <input type="checkbox" id="cheese" name="toppings" value="cheese">
  <label for="cheese">Cheese</label>
  
  <input type="checkbox" id="pepperoni" name="toppings" value="pepperoni">
  <label for="pepperoni">Pepperoni</label>
  
  <input type="checkbox" id="mushrooms" name="toppings" value="mushrooms">
  <label for="mushrooms">Mushrooms</label>
</fieldset>
```

---

### `type="radio"`

**Purpose:** Single selection from multiple options.

```html
<fieldset>
  <legend>Select size:</legend>
  
  <input type="radio" id="small" name="size" value="small">
  <label for="small">Small</label>
  
  <input type="radio" id="medium" name="size" value="medium" checked>
  <label for="medium">Medium</label>
  
  <input type="radio" id="large" name="size" value="large">
  <label for="large">Large</label>
</fieldset>
```

**Key difference from checkbox:**
- Radio buttons with same `name` are mutually exclusive
- One must always be selected (can't uncheck all)

---

### `type="hidden"`

**Purpose:** Hidden data submission.

```html
<form>
  <input type="hidden" name="user_id" value="12345">
  <input type="hidden" name="csrf_token" value="abc123xyz">
  
  <input type="text" name="comment" placeholder="Add comment">
  <button type="submit">Submit</button>
</form>
```

**Use cases:**
- CSRF tokens
- User IDs
- Form step tracking
- A/B test variants

---

## Browser Support and Fallbacks

### Feature Detection

```javascript
function supportsInputType(type) {
  const input = document.createElement('input');
  input.setAttribute('type', type);
  return input.type === type;
}

// Test support
console.log('Date input:', supportsInputType('date'));
console.log('Color input:', supportsInputType('color'));

// Conditional polyfill loading
if (!supportsInputType('date')) {
  // Load date picker polyfill
  loadPolyfill('datepicker');
}
```

### Fallback Strategies

**1. Feature detection + polyfill:**
```html
<input type="date" id="date-input">

<script>
if (!supportsInputType('date')) {
  // Load library like Flatpickr, Pikaday, or DatePicker
  import('flatpickr').then(flatpickr => {
    flatpickr('#date-input', {
      dateFormat: 'Y-m-d'
    });
  });
}
</script>
```

**2. Progressive enhancement:**
```html
<!-- Works everywhere (text input fallback) -->
<input 
  type="date" 
  placeholder="YYYY-MM-DD"
  pattern="\d{4}-\d{2}-\d{2}"
>
```

**3. Manual fallback:**
```html
<label for="date">Date:</label>
<input type="date" id="date" name="date">

<!-- Separate selects as fallback -->
<select name="date-year">...</select>
<select name="date-month">...</select>
<select name="date-day">...</select>

<script>
if (supportsInputType('date')) {
  // Hide selects
  document.querySelector('[name="date-year"]').style.display = 'none';
  document.querySelector('[name="date-month"]').style.display = 'none';
  document.querySelector('[name="date-day"]').style.display = 'none';
} else {
  // Hide date input
  document.getElementById('date').style.display = 'none';
}
</script>
```

---

## Common Mistakes

❌ **Using `type="number"` for non-numeric IDs**
```html
<!-- Wrong: strips leading zeros -->
<input type="number" placeholder="ZIP code">

<!-- Correct: -->
<input type="text" inputmode="numeric" pattern="[0-9]{5}">
```

❌ **Not providing labels**
```html
<!-- Wrong: accessibility issue -->
<input type="email" placeholder="Email">

<!-- Correct: -->
<label for="email">Email:</label>
<input type="email" id="email" placeholder="user@example.com">
```

❌ **Wrong date format**
```html
<!-- Wrong: -->
<input type="date" value="05/26/2025">

<!-- Correct: ISO format -->
<input type="date" value="2025-05-26">
```

❌ **Relying only on client-side validation**
```html
<!-- Not enough: users can bypass HTML5 validation -->
<input type="email" required>

<!-- Always validate on server too -->
```

❌ **Using `type="tel"` expecting validation**
```html
<!-- Wrong: no automatic validation -->
<input type="tel" required>

<!-- Correct: add pattern -->
<input type="tel" pattern="[0-9]{3}-[0-9]{3}-[0-9]{4}" required>
```

---

## Best Practices

✅ **Use specific input types** - Better UX, especially on mobile

✅ **Always include labels** - Accessibility and usability

✅ **Use appropriate autocomplete values** - Faster form filling

✅ **Provide fallbacks** - Test in older browsers

✅ **Combine with validation attributes** - `required`, `pattern`, `min`, `max`

✅ **Use `inputmode` for text inputs** - Optimized mobile keyboards

```html
<!-- Text input but numeric keyboard -->
<input type="text" inputmode="numeric" pattern="[0-9]*">

<!-- Text input but email keyboard -->
<input type="text" inputmode="email">
```

✅ **Test on real devices** - Input types behave differently across platforms

✅ **Validate server-side** - Never trust client-side validation alone

✅ **Use `step="any"` for precise decimals** - Avoids rounding issues

```html
<input type="number" step="any">
```

---

## Resources

- [MDN: input types](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/input)
- [Can I use: Input types](https://caniuse.com/?search=input%20type)
- [HTML5 input types test](https://inputtypes.com/)
- [Mobile input types](https://better-mobile-inputs.netlify.app/)
- [A11y: Form inputs](https://www.a11yproject.com/posts/how-to-write-accessible-forms/)
