# Live regions

## The Idea

**In plain English:** A live region is a special area on a webpage that automatically tells screen reader software (a tool that reads the screen aloud for people who can't see it) whenever the content inside it changes, so users do not have to move their cursor there to find out what updated.

**Real-world analogy:** Think of a sports stadium scoreboard operator who calls out score changes over a loudspeaker. Fans in the stands do not have to walk up and look at the scoreboard themselves — the announcement comes to them.

- The loudspeaker = the `aria-live` attribute (it broadcasts changes to whoever is listening)
- The scoreboard box = the HTML element marked as a live region (the area being watched for changes)
- The score changing = dynamic content being updated by JavaScript

---

## Overview

Live regions announce dynamic content changes to screen reader users without requiring focus changes. They're essential for real-time updates like status messages, notifications, and changing content.

## aria-live attribute

Controls how and when updates are announced.

### aria-live="polite"

Announces changes during natural pauses in screen reader speech.

```html
<div aria-live="polite">
  Status message will be announced when user pauses
</div>
```

**Use for:**
- Status messages
- Non-urgent notifications
- Progress updates
- Form validation feedback

### aria-live="assertive"

Immediately interrupts screen reader to announce changes.

```html
<div aria-live="assertive">
  Critical alert announced immediately
</div>
```

**Use for:**
- Error messages
- Critical alerts
- Time-sensitive information
- Important warnings

**Use sparingly** - interrupting is disruptive.

### aria-live="off"

No announcements (default for most elements).

```html
<div aria-live="off">
  Changes not announced
</div>
```

## role="status"

Equivalent to `aria-live="polite"` with `aria-atomic="true"`.

```html
<div role="status">
  Item added to cart
</div>
```

**Use for:**
- Status updates
- Success messages
- Progress indicators
- Non-critical notifications

## role="alert"

Equivalent to `aria-live="assertive"` with `aria-atomic="true"`.

```html
<div role="alert">
  Error: Please correct the form
</div>
```

**Use for:**
- Error messages
- Critical warnings
- Time-sensitive alerts
- Important notifications

## aria-atomic

Controls whether entire region or just changes are announced.

### aria-atomic="true"

Announces entire region content.

```html
<div aria-live="polite" aria-atomic="true">
  <p>Items in cart: <span id="count">3</span></p>
</div>
<!-- Announces: "Items in cart: 3" when count changes -->
```

### aria-atomic="false"

Announces only changed content (default).

```html
<div aria-live="polite" aria-atomic="false">
  <p>Items in cart: <span id="count">3</span></p>
</div>
<!-- Announces: "3" when count changes -->
```

## aria-relevant

Specifies which changes trigger announcements.

```html
<!-- Announce additions only (default for most) -->
<div aria-live="polite" aria-relevant="additions">
  New items announced
</div>

<!-- Announce removals only -->
<div aria-live="polite" aria-relevant="removals">
  Removed items announced
</div>

<!-- Announce text changes -->
<div aria-live="polite" aria-relevant="text">
  Text changes announced
</div>

<!-- Announce multiple types -->
<div aria-live="polite" aria-relevant="additions text">
  Additions and text changes announced
</div>

<!-- Announce all changes -->
<div aria-live="polite" aria-relevant="all">
  All changes announced
</div>
```

## Common patterns

### Status message

```html
<div role="status" aria-live="polite" aria-atomic="true">
  <span id="status-message"></span>
</div>

<script>
function showStatus(message) {
  const status = document.getElementById('status-message');
  status.textContent = message;
  
  // Clear after 5 seconds
  setTimeout(() => {
    status.textContent = '';
  }, 5000);
}

// Usage
showStatus('Settings saved successfully');
</script>
```

### Error alert

```html
<div role="alert" aria-live="assertive" aria-atomic="true" id="error-alert" hidden>
  <span id="error-message"></span>
</div>

<script>
function showError(message) {
  const alert = document.getElementById('error-alert');
  const messageEl = document.getElementById('error-message');
  
  messageEl.textContent = message;
  alert.hidden = false;
  
  // Optional: auto-hide after delay
  setTimeout(() => {
    alert.hidden = true;
  }, 5000);
}

// Usage
showError('Please enter a valid email address');
</script>
```

### Loading indicator

```html
<div role="status" aria-live="polite" aria-atomic="true">
  <span id="loading-status"></span>
</div>

<script>
async function loadData() {
  const status = document.getElementById('loading-status');
  
  status.textContent = 'Loading data...';
  
  try {
    const data = await fetch('/api/data');
    status.textContent = 'Data loaded successfully';
  } catch (error) {
    status.textContent = 'Error loading data';
  }
  
  // Clear after announcement
  setTimeout(() => {
    status.textContent = '';
  }, 3000);
}
</script>
```

### Cart counter

```html
<div aria-live="polite" aria-atomic="true">
  <span class="visually-hidden">Items in cart:</span>
  <span id="cart-count">0</span>
</div>

<script>
let cartCount = 0;

function addToCart() {
  cartCount++;
  document.getElementById('cart-count').textContent = cartCount;
  // Announces: "Items in cart: 3"
}
</script>
```

### Form validation

```html
<form>
  <label for="email">Email</label>
  <input 
    type="email" 
    id="email"
    aria-describedby="email-error"
    aria-invalid="false">
  
  <div 
    id="email-error" 
    role="alert" 
    aria-live="assertive"
    aria-atomic="true">
    <!-- Error message inserted here -->
  </div>
  
  <button type="submit">Submit</button>
</form>

<script>
const emailInput = document.getElementById('email');
const emailError = document.getElementById('email-error');

emailInput.addEventListener('blur', () => {
  if (!emailInput.validity.valid) {
    emailInput.setAttribute('aria-invalid', 'true');
    emailError.textContent = 'Please enter a valid email address';
  } else {
    emailInput.setAttribute('aria-invalid', 'false');
    emailError.textContent = '';
  }
});
</script>
```

### Search results

```html
<div>
  <label for="search">Search</label>
  <input type="search" id="search">
  
  <div role="status" aria-live="polite" aria-atomic="true">
    <span id="search-results-count"></span>
  </div>
  
  <div id="search-results">
    <!-- Results inserted here -->
  </div>
</div>

<script>
const searchInput = document.getElementById('search');
const resultsCount = document.getElementById('search-results-count');
const resultsContainer = document.getElementById('search-results');

searchInput.addEventListener('input', async (e) => {
  const query = e.target.value;
  
  if (query.length < 3) {
    resultsCount.textContent = '';
    resultsContainer.innerHTML = '';
    return;
  }
  
  const results = await searchAPI(query);
  
  resultsCount.textContent = `${results.length} results found`;
  resultsContainer.innerHTML = renderResults(results);
});
</script>
```

### Timer/countdown

```html
<div role="timer" aria-live="off" aria-atomic="true">
  Time remaining: <span id="countdown">5:00</span>
</div>

<script>
let seconds = 300;
const countdown = document.getElementById('countdown');
const timerRegion = countdown.closest('[role="timer"]');

setInterval(() => {
  seconds--;
  
  const mins = Math.floor(seconds / 60);
  const secs = seconds % 60;
  countdown.textContent = `${mins}:${secs.toString().padStart(2, '0')}`;
  
  // Only announce at certain intervals
  if (seconds === 60) {
    timerRegion.setAttribute('aria-live', 'polite');
    setTimeout(() => {
      timerRegion.setAttribute('aria-live', 'off');
    }, 100);
  }
  
  if (seconds === 0) {
    timerRegion.setAttribute('aria-live', 'assertive');
  }
}, 1000);
</script>
```

### Chat/messaging

```html
<div class="chat">
  <div 
    id="messages" 
    role="log" 
    aria-live="polite"
    aria-atomic="false"
    aria-relevant="additions">
    <!-- Messages inserted here -->
  </div>
  
  <form>
    <label for="message-input">Message</label>
    <input type="text" id="message-input">
    <button>Send</button>
  </form>
</div>

<script>
function addMessage(author, text) {
  const messages = document.getElementById('messages');
  const message = document.createElement('div');
  message.innerHTML = `<strong>${author}:</strong> ${text}`;
  messages.appendChild(message);
  // New messages announced as they arrive
}
</script>
```

### Progress bar

```html
<div 
  role="progressbar" 
  aria-valuenow="0"
  aria-valuemin="0"
  aria-valuemax="100"
  aria-live="polite"
  aria-atomic="true"
  aria-label="Upload progress">
  <div class="progress-bar" style="width: 0%"></div>
  <span class="visually-hidden">0% complete</span>
</div>

<script>
function updateProgress(percent) {
  const progressbar = document.querySelector('[role="progressbar"]');
  const bar = progressbar.querySelector('.progress-bar');
  const text = progressbar.querySelector('.visually-hidden');
  
  progressbar.setAttribute('aria-valuenow', percent);
  bar.style.width = `${percent}%`;
  text.textContent = `${percent}% complete`;
  
  // Only announce at 25% intervals to avoid spam
  if (percent % 25 === 0 || percent === 100) {
    // Announcement happens automatically due to aria-live
  }
}
</script>
```

## Best practices

### 1. Include live region in initial page load

```html
<!-- Good: present on page load -->
<div role="status" aria-live="polite" id="status">
  <span id="status-message"></span>
</div>

<!-- Bad: created dynamically -->
<script>
// Live region may not work when created dynamically
const status = document.createElement('div');
status.setAttribute('role', 'status');
document.body.appendChild(status);
</script>
```

### 2. Use appropriate politeness level

```html
<!-- Polite for most status updates -->
<div role="status" aria-live="polite">
  Item saved
</div>

<!-- Assertive only for critical alerts -->
<div role="alert" aria-live="assertive">
  Error: Connection lost
</div>
```

### 3. Keep messages concise

```html
<!-- Good: concise -->
<div role="status">
  Email sent
</div>

<!-- Bad: verbose -->
<div role="status">
  Your email message has been successfully sent to the recipient and they should receive it shortly in their inbox
</div>
```

### 4. Clear messages after announcement

```javascript
function showStatus(message) {
  const status = document.getElementById('status');
  status.textContent = message;
  
  // Clear after delay to allow re-announcement
  setTimeout(() => {
    status.textContent = '';
  }, 5000);
}
```

### 5. Avoid announcement spam

```javascript
// Bad: announces every keystroke
searchInput.addEventListener('input', (e) => {
  resultsCount.textContent = `Searching for ${e.target.value}...`;
});

// Good: debounce announcements
let searchTimeout;
searchInput.addEventListener('input', (e) => {
  clearTimeout(searchTimeout);
  searchTimeout = setTimeout(() => {
    resultsCount.textContent = `Searching for ${e.target.value}...`;
  }, 500);
});
```

## Testing live regions

### Screen reader testing

- NVDA (Windows): Settings > Speech > Automatic reading of changes
- JAWS (Windows): Should work by default
- VoiceOver (macOS): Should work by default
- TalkBack (Android): Should work by default

### Browser DevTools

```javascript
// Monitor live region changes
const observer = new MutationObserver((mutations) => {
  mutations.forEach((mutation) => {
    console.log('Live region changed:', mutation.target);
  });
});

const liveRegion = document.querySelector('[aria-live]');
observer.observe(liveRegion, {
  childList: true,
  subtree: true,
  characterData: true
});
```

## Common mistakes

```html
<!-- Bad: dynamically created live region -->
<script>
const alert = document.createElement('div');
alert.setAttribute('role', 'alert');
alert.textContent = 'Error!';
document.body.appendChild(alert);
// May not be announced
</script>

<!-- Good: update existing live region -->
<div role="alert" id="alert"></div>
<script>
document.getElementById('alert').textContent = 'Error!';
// Will be announced
</script>

<!-- Bad: changing aria-live after content change -->
<div id="status">Content changed</div>
<script>
document.getElementById('status').setAttribute('aria-live', 'polite');
// Won't be announced
</script>

<!-- Good: aria-live set before content change -->
<div id="status" aria-live="polite"></div>
<script>
document.getElementById('status').textContent = 'Content changed';
// Will be announced
</script>

<!-- Bad: too many assertive regions -->
<div aria-live="assertive">Update 1</div>
<div aria-live="assertive">Update 2</div>
<div aria-live="assertive">Update 3</div>

<!-- Good: one polite region -->
<div role="status" aria-live="polite" id="status"></div>
```

## Special roles with implicit live region

### role="timer"

```html
<div role="timer">
  <span id="timer">5:00</span>
</div>
```

### role="log"

For chat or activity logs (aria-live="polite" implicit):

```html
<div role="log" aria-label="Chat messages">
  <!-- Messages appended here -->
</div>
```

### role="marquee"

For non-essential, frequently updating content:

```html
<div role="marquee" aria-label="Stock ticker">
  <!-- Stock prices -->
</div>
```

## Key takeaways

- Live regions announce dynamic content changes
- Use `aria-live="polite"` for most status updates
- Use `aria-live="assertive"` sparingly for critical alerts
- Use `role="status"` for polite status messages
- Use `role="alert"` for assertive error messages
- Include live regions in initial page load
- Keep announcements concise and meaningful
- Use `aria-atomic` to control what's announced
- Avoid announcement spam with debouncing
- Test with actual screen readers
- Clear messages after announcement
- Don't create live regions dynamically
