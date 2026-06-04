# Native Validation

## Learning Objectives
- Master HTML5 native form validation attributes
- Understand validation timing and user experience patterns
- Learn CSS pseudo-classes for validation states
- Implement custom validation messages and patterns
- Balance client-side and server-side validation

---

## Overview of Native Validation

HTML5 provides **built-in validation** without JavaScript, offering:
- **Immediate feedback** - Validation on form submission
- **Native UI** - Browser-provided error messages
- **Accessibility** - Screen reader support
- **Zero JavaScript** - Works without any scripting

**Basic example:**
```html
<form>
  <input type="email" required>
  <button type="submit">Submit</button>
</form>
<!-- Browser blocks submission and shows error if email is invalid -->
```

---

## Validation Attributes

### `required`

**Purpose:** Field must be filled before submission.

```html
<label for="name">Name (required):</label>
<input type="text" id="name" name="name" required>
```

**Works on:**
- Text inputs, textareas, selects
- Checkboxes, radio buttons
- File inputs

**Checkbox example:**
```html
<input type="checkbox" id="terms" required>
<label for="terms">I accept the terms and conditions</label>
<!-- User must check before submitting -->
```

**Select example:**
```html
<label for="country">Country:</label>
<select id="country" name="country" required>
  <option value="">-- Select a country --</option>
  <option value="us">United States</option>
  <option value="ca">Canada</option>
  <option value="uk">United Kingdom</option>
</select>
<!-- First option with empty value acts as placeholder -->
```

---

### `minlength` and `maxlength`

**Purpose:** Constrain text input length.

```html
<label for="username">Username (3-20 characters):</label>
<input 
  type="text" 
  id="username" 
  name="username"
  minlength="3"
  maxlength="20"
  required
>
```

**Behavior:**
- `maxlength` - **Prevents typing** beyond limit (hard limit)
- `minlength` - **Validates on submit** (soft limit)

**With character counter:**
```html
<label for="bio">Bio (max 200 characters):</label>
<textarea 
  id="bio" 
  name="bio"
  maxlength="200"
></textarea>
<div id="char-count">0 / 200</div>

<script>
const textarea = document.getElementById('bio');
const counter = document.getElementById('char-count');

textarea.addEventListener('input', () => {
  counter.textContent = `${textarea.value.length} / 200`;
});
</script>
```

---

### `min` and `max`

**Purpose:** Constrain numeric and date values.

**Numeric example:**
```html
<label for="age">Age (18-100):</label>
<input 
  type="number" 
  id="age" 
  name="age"
  min="18"
  max="100"
  required
>
```

**Date range:**
```html
<label for="appointment">Appointment (next 30 days):</label>
<input 
  type="date" 
  id="appointment" 
  name="appointment"
  min="2025-05-26"
  max="2025-06-25"
  required
>
```

**Dynamic date constraints:**
```html
<label for="start-date">Start date:</label>
<input type="date" id="start-date" name="start-date" required>

<label for="end-date">End date:</label>
<input type="date" id="end-date" name="end-date" required>

<script>
const startDate = document.getElementById('start-date');
const endDate = document.getElementById('end-date');

// Set today as minimum for start date
const today = new Date().toISOString().split('T')[0];
startDate.min = today;

// Update end date minimum when start date changes
startDate.addEventListener('change', () => {
  endDate.min = startDate.value;
  
  // Reset end date if it's before new start date
  if (endDate.value && endDate.value < startDate.value) {
    endDate.value = '';
  }
});
</script>
```

---

### `step`

**Purpose:** Define valid value increments.

**Number with decimals:**
```html
<label for="price">Price (in dollars):</label>
<input 
  type="number" 
  id="price" 
  name="price"
  min="0"
  step="0.01"
  placeholder="0.00"
  required
>
<!-- Allows: 0.00, 0.01, 0.02, 1.50, etc. -->
```

**Time intervals:**
```html
<label for="appointment">Appointment time (15-minute intervals):</label>
<input 
  type="time" 
  id="appointment" 
  name="appointment"
  min="09:00"
  max="17:00"
  step="900"
  required
>
<!-- 900 seconds = 15 minutes -->
<!-- Allows: 09:00, 09:15, 09:30, etc. -->
```

**Any value (disable step validation):**
```html
<input type="number" step="any">
<!-- Allows any decimal precision -->
```

---

### `pattern`

**Purpose:** Validate against a regular expression.

**Basic examples:**
```html
<!-- ZIP code -->
<input 
  type="text" 
  pattern="[0-9]{5}"
  title="5-digit ZIP code"
  placeholder="12345"
>

<!-- Phone number -->
<input 
  type="tel" 
  pattern="[0-9]{3}-[0-9]{3}-[0-9]{4}"
  title="Format: 123-456-7890"
  placeholder="555-123-4567"
>

<!-- Alphanumeric username -->
<input 
  type="text" 
  pattern="[a-zA-Z0-9]+"
  title="Letters and numbers only"
>
```

**Advanced patterns:**
```html
<!-- Strong password -->
<input 
  type="password" 
  pattern="(?=.*\d)(?=.*[a-z])(?=.*[A-Z]).{8,}"
  title="Must contain at least one number, one uppercase and lowercase letter, and at least 8 characters"
>

<!-- Hex color -->
<input 
  type="text" 
  pattern="#[0-9A-Fa-f]{6}"
  title="Hex color format: #RRGGBB"
  placeholder="#FF0000"
>

<!-- Credit card (basic) -->
<input 
  type="text" 
  pattern="[0-9]{13,19}"
  title="13-19 digit card number"
  inputmode="numeric"
>

<!-- URL with specific domain -->
<input 
  type="url" 
  pattern="https://github\.com/.*"
  title="Must be a GitHub URL starting with https://"
>
```

**Important:** Pattern must match the **entire value** (implicit `^...$` anchors).

```html
<!-- Wrong: allows extra characters -->
<input pattern="[0-9]{5}">  <!-- Matches "12345abc" -->

<!-- Correct: enforces exact match -->
<input pattern="[0-9]{5}">  <!-- Only matches "12345" (due to implicit anchors) -->
```

---

### Input Type-Specific Validation

**`type="email"`**
```html
<!-- Single email -->
<input type="email" required>
<!-- Validates: user@domain.com -->

<!-- Multiple emails -->
<input type="email" multiple required>
<!-- Validates: user1@domain.com, user2@domain.com -->

<!-- Stricter validation -->
<input 
  type="email" 
  pattern="[a-z0-9._%+-]+@[a-z0-9.-]+\.[a-z]{2,}$"
  required
>
```

**`type="url"`**
```html
<!-- Must include protocol -->
<input type="url" required>
<!-- Valid: https://example.com -->
<!-- Invalid: example.com -->

<!-- Force https -->
<input type="url" pattern="https://.*" required>
```

---

## Validation Timing

### When Validation Occurs

**1. On form submission (default):**
```html
<form>
  <input type="email" required>
  <button type="submit">Submit</button>
</form>
<!-- Validation triggers when submit is clicked -->
```

**2. On user interaction (with JavaScript):**
```html
<input 
  type="email" 
  id="email" 
  required
  oninput="this.setCustomValidity('')"
  oninvalid="this.setCustomValidity('Please enter a valid email address')"
>
```

**3. Real-time validation (as user types):**
```html
<input 
  type="email" 
  id="email" 
  required
>
<span id="email-error" class="error"></span>

<script>
const email = document.getElementById('email');
const error = document.getElementById('email-error');

email.addEventListener('input', () => {
  if (email.validity.valid) {
    error.textContent = '';
  } else {
    error.textContent = email.validationMessage;
  }
});
</script>
```

---

## CSS Validation Pseudo-Classes

### `:valid` and `:invalid`

**Basic styling:**
```css
input:valid {
  border-color: green;
}

input:invalid {
  border-color: red;
}
```

**Problem:** Shows invalid state immediately on page load.

**Solution:** Only show validation after user interaction.

```css
/* Don't show validation on pristine inputs */
input:invalid {
  border-color: #ccc;
}

/* Show validation after user touches field */
input:invalid:not(:focus):not(:placeholder-shown) {
  border-color: red;
}

input:valid:not(:focus):not(:placeholder-shown) {
  border-color: green;
}
```

**With icons:**
```css
input {
  padding-right: 30px;
  background-repeat: no-repeat;
  background-position: right 10px center;
  background-size: 20px;
}

input:valid:not(:focus):not(:placeholder-shown) {
  background-image: url('check-icon.svg');
  border-color: green;
}

input:invalid:not(:focus):not(:placeholder-shown) {
  background-image: url('error-icon.svg');
  border-color: red;
}
```

---

### `:required` and `:optional`

```css
/* Style required fields */
label:has(+ input:required)::after {
  content: " *";
  color: red;
}

/* Or mark optional fields */
input:optional + label::after {
  content: " (optional)";
  color: #666;
  font-size: 0.875em;
}
```

---

### `:in-range` and `:out-of-range`

For `<input type="number">`, `type="range"`, `type="date"`, etc.

```css
input:in-range {
  border-color: green;
}

input:out-of-range {
  border-color: red;
}
```

**Example:**
```html
<label for="quantity">Quantity (1-10):</label>
<input 
  type="number" 
  id="quantity" 
  min="1" 
  max="10"
  value="5"
>

<style>
#quantity:out-of-range {
  border: 2px solid red;
}

#quantity:out-of-range + .hint::before {
  content: "⚠️ ";
}
</style>

<span class="hint">Please enter a value between 1 and 10</span>
```

---

### `:user-invalid` (New)

Shows invalid state **only after user interaction**.

```css
/* Old way (shows invalid immediately) */
input:invalid {
  border-color: red;
}

/* New way (shows invalid only after user edits and leaves field) */
input:user-invalid {
  border-color: red;
}
```

**Browser support:** Chrome 88+, Firefox 88+, Safari 16.5+

**Fallback:**
```css
/* Progressive enhancement */
input:invalid:not(:focus):not(:placeholder-shown) {
  border-color: red;
}

/* Override with :user-invalid if supported */
@supports selector(:user-invalid) {
  input:invalid:not(:focus):not(:placeholder-shown) {
    border-color: initial;
  }
  
  input:user-invalid {
    border-color: red;
  }
}
```

---

## Custom Validation Messages

### Using `title` Attribute

```html
<input 
  type="text" 
  pattern="[0-9]{5}"
  title="Please enter a 5-digit ZIP code"
  required
>
<!-- Browser shows this message when validation fails -->
```

---

### Using `setCustomValidity()`

**Basic example:**
```html
<input 
  type="text" 
  id="username" 
  required
>

<script>
const username = document.getElementById('username');

username.addEventListener('input', () => {
  if (username.value.length < 3) {
    username.setCustomValidity('Username must be at least 3 characters');
  } else if (username.value.length > 20) {
    username.setCustomValidity('Username must be at most 20 characters');
  } else if (!/^[a-zA-Z0-9_]+$/.test(username.value)) {
    username.setCustomValidity('Username can only contain letters, numbers, and underscores');
  } else {
    username.setCustomValidity(''); // Clear custom error
  }
});
</script>
```

**Important:** Always clear custom validity when input becomes valid!

---

### Password Confirmation

```html
<label for="password">Password:</label>
<input 
  type="password" 
  id="password" 
  name="password"
  minlength="8"
  required
>

<label for="confirm-password">Confirm password:</label>
<input 
  type="password" 
  id="confirm-password" 
  name="confirm-password"
  required
>

<script>
const password = document.getElementById('password');
const confirmPassword = document.getElementById('confirm-password');

function validatePassword() {
  if (password.value !== confirmPassword.value) {
    confirmPassword.setCustomValidity('Passwords do not match');
  } else {
    confirmPassword.setCustomValidity('');
  }
}

password.addEventListener('change', validatePassword);
confirmPassword.addEventListener('input', validatePassword);
</script>
```

---

### Async Validation (e.g., username availability)

```html
<label for="username">Username:</label>
<input 
  type="text" 
  id="username" 
  name="username"
  required
>
<span id="username-status"></span>

<script>
const username = document.getElementById('username');
const status = document.getElementById('username-status');
let debounceTimer;

username.addEventListener('input', () => {
  clearTimeout(debounceTimer);
  
  debounceTimer = setTimeout(async () => {
    if (username.value.length >= 3) {
      status.textContent = 'Checking...';
      
      // Simulate API call
      const available = await checkUsernameAvailability(username.value);
      
      if (!available) {
        username.setCustomValidity('Username is already taken');
        status.textContent = '❌ Not available';
      } else {
        username.setCustomValidity('');
        status.textContent = '✅ Available';
      }
    }
  }, 500);
});

async function checkUsernameAvailability(username) {
  const response = await fetch(`/api/check-username?username=${username}`);
  const data = await response.json();
  return data.available;
}
</script>
```

---

## Validation API

### Validity State Properties

```javascript
const input = document.getElementById('email');

// Check specific validity states
console.log(input.validity.valid);          // true/false - overall validity
console.log(input.validity.valueMissing);   // required but empty
console.log(input.validity.typeMismatch);   // wrong type (e.g., invalid email)
console.log(input.validity.patternMismatch);// doesn't match pattern
console.log(input.validity.tooLong);        // exceeds maxlength
console.log(input.validity.tooShort);       // below minlength
console.log(input.validity.rangeUnderflow); // below min
console.log(input.validity.rangeOverflow);  // above max
console.log(input.validity.stepMismatch);   // doesn't match step
console.log(input.validity.badInput);       // browser can't convert value
console.log(input.validity.customError);    // setCustomValidity() was called

// Get validation message
console.log(input.validationMessage);       // "Please enter a valid email address"
```

---

### Manual Validation

**`checkValidity()`** - Validates element, returns boolean, **doesn't trigger UI**:
```javascript
if (!input.checkValidity()) {
  console.log('Invalid:', input.validationMessage);
}
```

**`reportValidity()`** - Validates element, returns boolean, **shows UI**:
```javascript
if (!input.reportValidity()) {
  // Browser shows error message
  console.log('Invalid:', input.validationMessage);
}
```

**Form-level validation:**
```javascript
const form = document.getElementById('my-form');

// Validate all fields
if (form.checkValidity()) {
  console.log('Form is valid');
} else {
  form.reportValidity(); // Show all errors
}
```

---

### Custom Validation on Submit

```html
<form id="registration-form" novalidate>
  <input type="text" id="username" required>
  <input type="email" id="email" required>
  <input type="password" id="password" required>
  <button type="submit">Register</button>
</form>

<script>
const form = document.getElementById('registration-form');

form.addEventListener('submit', async (e) => {
  e.preventDefault();
  
  // Clear previous custom errors
  form.querySelectorAll('input').forEach(input => {
    input.setCustomValidity('');
  });
  
  // Run custom validations
  const username = document.getElementById('username');
  const available = await checkUsernameAvailability(username.value);
  
  if (!available) {
    username.setCustomValidity('Username is already taken');
  }
  
  // Check all validity
  if (!form.checkValidity()) {
    form.reportValidity();
    return;
  }
  
  // Form is valid - submit
  submitForm(new FormData(form));
});
</script>
```

---

## Disabling Native Validation

### `novalidate` Attribute

**On form:**
```html
<form novalidate>
  <input type="email" required>
  <button type="submit">Submit</button>
</form>
<!-- Native validation is disabled - use JavaScript validation instead -->
```

**On submit button:**
```html
<form>
  <input type="email" required>
  <button type="submit">Submit</button>
  <button type="submit" formnovalidate>Save Draft</button>
</form>
<!-- "Save Draft" bypasses validation -->
```

**Use case:** Implementing custom validation UI.

```javascript
const form = document.querySelector('form[novalidate]');

form.addEventListener('submit', (e) => {
  e.preventDefault();
  
  // Custom validation logic
  const errors = validateForm(form);
  
  if (errors.length > 0) {
    displayErrors(errors);
  } else {
    form.submit();
  }
});
```

---

## Common Mistakes

❌ **Forgetting to clear custom validity**
```javascript
// Wrong: custom error persists even when fixed
input.addEventListener('input', () => {
  if (input.value.length < 3) {
    input.setCustomValidity('Too short');
  }
  // Missing: else { input.setCustomValidity(''); }
});

// Correct:
input.addEventListener('input', () => {
  if (input.value.length < 3) {
    input.setCustomValidity('Too short');
  } else {
    input.setCustomValidity('');
  }
});
```

❌ **Showing validation errors too early**
```css
/* Wrong: shows errors immediately */
input:invalid {
  border-color: red;
}

/* Correct: show errors after interaction */
input:invalid:not(:focus):not(:placeholder-shown) {
  border-color: red;
}
```

❌ **Not providing clear error messages**
```html
<!-- Wrong: generic message -->
<input type="text" pattern="[0-9]{5}" title="Invalid format">

<!-- Correct: specific message -->
<input type="text" pattern="[0-9]{5}" title="Please enter a 5-digit ZIP code">
```

❌ **Relying only on HTML5 validation**
```javascript
// Wrong: HTML5 validation can be bypassed
// Users can disable JavaScript, edit HTML, or submit via curl

// Correct: Always validate server-side too
app.post('/submit', (req, res) => {
  const errors = validateData(req.body);
  if (errors.length > 0) {
    return res.status(400).json({ errors });
  }
  // Process valid data
});
```

❌ **Using `maxlength` for passwords**
```html
<!-- Wrong: limits password strength -->
<input type="password" maxlength="20">

<!-- Correct: use minlength only -->
<input type="password" minlength="8">
```

---

## Best Practices

✅ **Combine HTML5 validation with JavaScript** - Use HTML5 for basic validation, JavaScript for complex rules

✅ **Validate on both client and server** - Never trust client-side validation alone

✅ **Show validation errors at the right time** - Not too early (annoying), not too late (confusing)

✅ **Provide clear, actionable error messages** - Tell users exactly what's wrong and how to fix it

✅ **Use `aria-describedby` for error messages**
```html
<label for="email">Email:</label>
<input 
  type="email" 
  id="email" 
  aria-describedby="email-error"
  required
>
<span id="email-error" role="alert"></span>
```

✅ **Focus first invalid field on submit**
```javascript
form.addEventListener('submit', (e) => {
  if (!form.checkValidity()) {
    e.preventDefault();
    const firstInvalid = form.querySelector(':invalid');
    firstInvalid.focus();
  }
});
```

✅ **Indicate required fields clearly**
```html
<label for="name">
  Name <span aria-label="required">*</span>
</label>
<input type="text" id="name" required>
```

✅ **Test validation across browsers** - Different browsers show different messages

✅ **Debounce async validation** - Don't hammer your API on every keystroke

---

## Resources

- [MDN: Client-side form validation](https://developer.mozilla.org/en-US/docs/Learn/Forms/Form_validation)
- [MDN: Constraint validation](https://developer.mozilla.org/en-US/docs/Web/HTML/Constraint_validation)
- [HTML5 validation pseudo-classes](https://developer.mozilla.org/en-US/docs/Web/CSS/:invalid)
- [W3C: Form validation](https://www.w3.org/WAI/tutorials/forms/validation/)
- [Smashing Magazine: Form validation UX](https://www.smashingmagazine.com/2022/09/inline-validation-web-forms-ux/)
