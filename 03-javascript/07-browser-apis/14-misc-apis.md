# Clipboard, Notifications, Geolocation, and File System Access

## Overview

Modern browsers expose a rich set of platform APIs that go well beyond DOM manipulation. This file covers four commonly-interviewed groups: the **Clipboard API** for reading/writing system clipboard data, the **Notifications API** for system-level push notifications, the **Geolocation API** for device location, and the **File System Access API** for reading and writing files on the local filesystem. Each requires explicit user permission and has important security and UX considerations.

## Clipboard API

The modern async Clipboard API replaces the deprecated `document.execCommand('copy')` approach.

### Writing to the Clipboard

```javascript
// Write plain text
async function copyText(text) {
  try {
    await navigator.clipboard.writeText(text);
    console.log('Copied!');
  } catch (err) {
    console.error('Copy failed:', err.name); // NotAllowedError if not permitted
  }
}

// Write rich content (multiple MIME types)
async function copyRich(html, plainText) {
  const htmlBlob = new Blob([html], { type: 'text/html' });
  const textBlob = new Blob([plainText], { type: 'text/plain' });

  await navigator.clipboard.write([
    new ClipboardItem({
      'text/html': htmlBlob,
      'text/plain': textBlob,
    }),
  ]);
}

// Write an image
async function copyImage(imageBlob) {
  await navigator.clipboard.write([
    new ClipboardItem({ 'image/png': imageBlob }),
  ]);
}
```

### Reading from the Clipboard

```javascript
// Read text (requires 'clipboard-read' permission)
async function pasteText() {
  try {
    const text = await navigator.clipboard.readText();
    console.log('Pasted:', text);
    return text;
  } catch (err) {
    console.error('Paste failed:', err.name); // NotAllowedError
  }
}

// Read rich content
async function pasteRich() {
  const items = await navigator.clipboard.read();

  for (const item of items) {
    console.log('Types:', item.types); // ['text/plain', 'text/html']

    if (item.types.includes('text/html')) {
      const blob = await item.getType('text/html');
      const html = await blob.text();
      console.log('HTML:', html);
    }
  }
}
```

### Permissions

```javascript
// clipboard-write: auto-granted for user-initiated events
// clipboard-read: requires explicit permission prompt
const result = await navigator.permissions.query({ name: 'clipboard-read' });
console.log(result.state); // 'granted' | 'denied' | 'prompt'

result.onchange = () => console.log('Permission changed:', result.state);
```

### Clipboard Events (Intercept Cut/Copy/Paste)

```javascript
// Intercept paste
document.addEventListener('paste', (event) => {
  const text = event.clipboardData.getData('text/plain');
  const html = event.clipboardData.getData('text/html');
  event.preventDefault();
  insertContent(text); // custom handling
});

// Intercept copy — customize what gets copied
document.addEventListener('copy', (event) => {
  const selection = window.getSelection().toString();
  event.clipboardData.setData('text/plain', selection + '\n— My App');
  event.clipboardData.setData('text/html', `<p>${selection}</p><cite>— My App</cite>`);
  event.preventDefault();
});

// Files
document.addEventListener('paste', (event) => {
  const files = [...event.clipboardData.files];
  files.forEach(file => uploadFile(file));
});
```

## Notifications API

### Requesting Permission

```javascript
// Permission values: 'granted' | 'denied' | 'default'
async function requestNotificationPermission() {
  if (!('Notification' in window)) {
    console.log('Notifications not supported');
    return;
  }

  if (Notification.permission === 'granted') return true;
  if (Notification.permission === 'denied') return false;

  const permission = await Notification.requestPermission();
  return permission === 'granted';
}
```

### Creating Notifications

```javascript
async function notify(title, options = {}) {
  if (!await requestNotificationPermission()) return;

  const notification = new Notification(title, {
    body: 'You have a new message from Alice.',
    icon: '/icons/notification.png',
    badge: '/icons/badge.png',    // small icon for mobile
    image: '/images/preview.jpg', // large image
    tag: 'chat-alice',            // replaces existing notification with same tag
    requireInteraction: false,    // don't auto-dismiss
    silent: false,                // play sound
    data: { url: '/chat/alice', userId: 42 },
    actions: [                    // buttons (Service Worker only)
      { action: 'reply', title: 'Reply' },
      { action: 'dismiss', title: 'Dismiss' },
    ],
  });

  notification.onclick = (event) => {
    event.preventDefault(); // prevent focus to browser window
    window.open(notification.data.url, '_blank');
    notification.close();
  };

  notification.onclose = () => console.log('Notification dismissed');
  notification.onerror = (e) => console.error(e);

  // Auto-close after 5 seconds
  setTimeout(() => notification.close(), 5000);
}
```

### Service Worker Notifications (Recommended)

Notifications created from a Service Worker persist even after the page closes and support action buttons:

```javascript
// In sw.js
self.addEventListener('push', (event) => {
  const data = event.data.json();

  event.waitUntil(
    self.registration.showNotification(data.title, {
      body: data.body,
      icon: '/icon.png',
      data: { url: data.url },
      actions: [
        { action: 'open', title: 'Open' },
        { action: 'close', title: 'Dismiss' },
      ],
    })
  );
});

self.addEventListener('notificationclick', (event) => {
  event.notification.close();

  if (event.action === 'close') return;

  event.waitUntil(
    clients.matchAll({ type: 'window' }).then(windows => {
      const target = event.notification.data.url;
      const existing = windows.find(w => w.url === target);
      if (existing) return existing.focus();
      return clients.openWindow(target);
    })
  );
});
```

## Geolocation API

### One-Time Position

```javascript
function getCurrentPosition(options = {}) {
  return new Promise((resolve, reject) => {
    if (!('geolocation' in navigator)) {
      reject(new Error('Geolocation not supported'));
      return;
    }

    navigator.geolocation.getCurrentPosition(resolve, reject, {
      enableHighAccuracy: false, // GPS vs. network/IP (battery tradeoff)
      timeout: 10000,            // ms before error (TIMEOUT error)
      maximumAge: 60000,         // accept cached position up to 1 min old
      ...options,
    });
  });
}

async function getLocation() {
  try {
    const position = await getCurrentPosition();
    const { latitude, longitude, accuracy, altitude, heading, speed } = position.coords;
    const timestamp = position.timestamp;

    console.log(`${latitude}, ${longitude} ± ${accuracy}m`);
    return { latitude, longitude, accuracy };
  } catch (err) {
    switch (err.code) {
      case GeolocationPositionError.PERMISSION_DENIED:     // 1
        console.error('User denied geolocation');
        break;
      case GeolocationPositionError.POSITION_UNAVAILABLE: // 2
        console.error('Position unavailable');
        break;
      case GeolocationPositionError.TIMEOUT:              // 3
        console.error('Geolocation timed out');
        break;
    }
  }
}
```

### Continuous Tracking

```javascript
// watchPosition returns a watcher ID
const watcherId = navigator.geolocation.watchPosition(
  (position) => {
    updateMap(position.coords.latitude, position.coords.longitude);
  },
  (error) => {
    console.error('Tracking error:', error.message);
  },
  { enableHighAccuracy: true, timeout: 5000 }
);

// Stop tracking
navigator.geolocation.clearWatch(watcherId);

// As a class
class LocationTracker {
  #watcherId = null;
  #subscribers = new Set();

  start(options = {}) {
    this.#watcherId = navigator.geolocation.watchPosition(
      (pos) => this.#subscribers.forEach(fn => fn(pos)),
      (err) => this.#subscribers.forEach(fn => fn(null, err)),
      options
    );
  }

  stop() {
    if (this.#watcherId !== null) {
      navigator.geolocation.clearWatch(this.#watcherId);
      this.#watcherId = null;
    }
  }

  subscribe(fn) {
    this.#subscribers.add(fn);
    return () => this.#subscribers.delete(fn);
  }
}
```

## File System Access API

The File System Access API (origin trial, widely available in Chromium) allows reading and writing files on the local filesystem, not just files dragged into the browser or selected via `<input type="file">`.

### Opening Files for Reading

```javascript
async function openTextFile() {
  const [fileHandle] = await window.showOpenFilePicker({
    types: [
      {
        description: 'Text files',
        accept: { 'text/plain': ['.txt', '.md'] },
      },
    ],
    multiple: false,
    excludeAcceptAllOption: false,
  });

  const file = await fileHandle.getFile();
  const text = await file.text();
  return { name: file.name, text, fileHandle };
}

// Fallback for browsers without File System Access API
async function openFile() {
  if ('showOpenFilePicker' in window) {
    return openTextFile();
  }
  // Fallback: input element
  return new Promise((resolve) => {
    const input = document.createElement('input');
    input.type = 'file';
    input.accept = '.txt,.md';
    input.onchange = async () => {
      const file = input.files[0];
      const text = await file.text();
      resolve({ name: file.name, text, fileHandle: null });
    };
    input.click();
  });
}
```

### Writing Files

```javascript
// Save to an existing file handle (no picker shown again)
async function saveFile(fileHandle, content) {
  const writable = await fileHandle.createWritable();
  await writable.write(content);
  await writable.close();
}

// Save As — pick a new file location
async function saveAs(content, suggestedName = 'file.txt') {
  const fileHandle = await window.showSaveFilePicker({
    suggestedName,
    types: [
      {
        description: 'Text file',
        accept: { 'text/plain': ['.txt'] },
      },
    ],
  });

  const writable = await fileHandle.createWritable();
  await writable.write(content);
  await writable.close();

  return fileHandle; // persist for future saves without picker
}

// Writing binary data (e.g., generated image)
async function saveBlob(blob, name) {
  const handle = await window.showSaveFilePicker({ suggestedName: name });
  const writable = await handle.createWritable();
  await writable.write(blob);
  await writable.close();
}
```

### Directory Access

```javascript
// Open an entire directory
const dirHandle = await window.showDirectoryPicker({ mode: 'readwrite' });

// Iterate entries
for await (const [name, handle] of dirHandle.entries()) {
  console.log(name, handle.kind); // 'file' or 'directory'
}

// Get a specific file
const fileHandle = await dirHandle.getFileHandle('config.json', { create: false });
const file = await fileHandle.getFile();
const config = JSON.parse(await file.text());

// Create a new file
const newHandle = await dirHandle.getFileHandle('output.json', { create: true });
const writable = await newHandle.createWritable();
await writable.write(JSON.stringify(data));
await writable.close();

// Verify permission still exists (user may have revoked it)
const perm = await dirHandle.queryPermission({ mode: 'readwrite' });
if (perm !== 'granted') {
  await dirHandle.requestPermission({ mode: 'readwrite' });
}
```

### Drag and Drop Integration

```javascript
// Accepting dropped files (works in all browsers)
dropZone.addEventListener('dragover', (e) => e.preventDefault());
dropZone.addEventListener('drop', async (e) => {
  e.preventDefault();
  const items = [...e.dataTransfer.items];

  for (const item of items) {
    if (item.kind !== 'file') continue;

    // Modern: get FileSystemHandle for read/write
    if (item.getAsFileSystemHandle) {
      const handle = await item.getAsFileSystemHandle();
      if (handle.kind === 'file') {
        const file = await handle.getFile();
        processFile(file, handle);
      }
    } else {
      // Legacy fallback
      const file = item.getAsFile();
      processFile(file, null);
    }
  }
});
```

## Comparison Table

| API | Permission Required | Scope | Browser Support |
|-----|--------------------|----|------|
| Clipboard write | Auto (user gesture) | Same-origin page | Wide |
| Clipboard read | Prompt required | Same-origin page | Wide |
| Notifications (page) | Prompt required | Page lifetime | Wide |
| Notifications (SW) | Prompt required | Background, persistent | Wide |
| Geolocation | Prompt required | Page | Wide |
| File System Access | Prompt per file/dir | Local filesystem | Chromium |

## Best Practices

### 1. Never Request Permissions Without User Intent

```javascript
// Bad: requesting permission on page load
window.addEventListener('load', () => Notification.requestPermission());

// Good: tie permission requests to meaningful user actions
notifyBtn.addEventListener('click', () => requestNotificationPermission());
locationBtn.addEventListener('click', () => getLocation());
```

### 2. Handle Permission Denial Gracefully

```javascript
try {
  await navigator.clipboard.writeText(text);
  showToast('Copied!');
} catch {
  // Silently provide a manual alternative
  showModal('Press Ctrl+C to copy the selected text');
}
```

### 3. Feature-Detect Before Using

```javascript
if ('showOpenFilePicker' in window) {
  // Modern File System Access API
} else {
  // Fallback: <input type="file">
}

if ('geolocation' in navigator) {
  // Geolocation supported
}
```

### 4. Persist File Handles in IndexedDB for Re-use

```javascript
// FileSystemFileHandle can be stored in IndexedDB for future sessions
const db = await openDB('file-handles', 1, {
  upgrade(db) { db.createObjectStore('handles'); },
});
await db.put('handles', fileHandle, 'lastFile');
// Next session:
const handle = await db.get('handles', 'lastFile');
if (handle) {
  const perm = await handle.queryPermission({ mode: 'readwrite' });
  // Will need to re-prompt on new page load
}
```

## Interview Questions

### Q1: Why does the async Clipboard API require a user gesture for writing?

**Answer:** The Clipboard API is restricted to user-activated contexts (a result of a `click`, `keydown`, etc.) to prevent malicious pages from silently overwriting the user's clipboard. `writeText` and `write` are auto-granted during a user gesture in most browsers. `readText` and `read` require an explicit permission prompt because reading the clipboard could expose sensitive data the user copied from unrelated applications.

### Q2: What is the difference between the Notifications API on a page vs. from a Service Worker?

**Answer:** `new Notification()` created in a page context is visible only while the page is open and does not support action buttons in most browsers. Notifications created via `ServiceWorkerRegistration.showNotification()` persist independently of the page, can display action buttons, and can be interacted with even after the page is closed. The `notificationclick` event in the Service Worker handles button actions and can open windows or focus existing tabs.

### Q3: What are the error codes from Geolocation, and what does each mean?

**Answer:** `PERMISSION_DENIED` (1) means the user denied the permission prompt or it was blocked by policy. `POSITION_UNAVAILABLE` (2) means the device could not determine position (no GPS fix, no network data). `TIMEOUT` (3) means the `timeout` option elapsed before a position was obtained. Only `PERMISSION_DENIED` is permanent; the other two are transient conditions that may succeed on retry.

### Q4: What is the File System Access API, and how is it different from reading a file via `<input type="file">`?

**Answer:** `<input type="file">` provides a one-time snapshot read of a file selected by the user; you get a `File` object but no write access and no reference back to the original file path. The File System Access API returns a `FileSystemFileHandle` that can be used to read or write the file repeatedly across sessions (with permission), reflect updates saved by other applications, and access entire directory trees—enabling web apps to behave like native desktop editors.

### Q5: How would you implement a "copy to clipboard" button with a fallback for older browsers?

**Answer:** Try the async `navigator.clipboard.writeText()` inside a user-event handler. If it throws a `NotAllowedError` or is unavailable, create a temporary `<textarea>`, set its value, call `select()`, and use `document.execCommand('copy')` as the legacy fallback. Always wrap in try/catch and provide a visible error state (e.g., select the text for manual copy) when both fail.

## Common Pitfalls

### 1. Clipboard API Outside User Gesture

```javascript
// Fails: not in a user activation context
setTimeout(async () => {
  await navigator.clipboard.writeText('data'); // NotAllowedError
}, 1000);

// Works: inside a click handler
btn.addEventListener('click', async () => {
  await navigator.clipboard.writeText('data');
});
```

### 2. Notification Permission Prompt Too Early

```javascript
// Spamming users with permission prompts on load reduces grant rate
// Show a custom pre-prompt explaining why notifications are useful first
```

### 3. Not Calling clearWatch for Geolocation

```javascript
// Continuous GPS tracking drains battery if the component unmounts without cleanup
componentWillUnmount() {
  navigator.geolocation.clearWatch(this.watcherId);
}
```

### 4. FileSystemWritableFileStream Not Closed

```javascript
const writable = await fileHandle.createWritable();
await writable.write(data);
// Forgetting writable.close() leaves the file lock open
await writable.close(); // always close!
```

### 5. Assuming File System Access API is Available

```javascript
// Only Chromium supports showOpenFilePicker (2024)
// Firefox and Safari require the input element fallback
```

## Resources

- [MDN: Clipboard API](https://developer.mozilla.org/en-US/docs/Web/API/Clipboard_API)
- [MDN: Notifications API](https://developer.mozilla.org/en-US/docs/Web/API/Notifications_API)
- [MDN: Geolocation API](https://developer.mozilla.org/en-US/docs/Web/API/Geolocation_API)
- [MDN: File System Access API](https://developer.mozilla.org/en-US/docs/Web/API/File_System_Access_API)
- [web.dev: The File System Access API](https://web.dev/file-system-access/)
- [web.dev: Notifications](https://web.dev/notifications/)
- [Permissions API](https://developer.mozilla.org/en-US/docs/Web/API/Permissions_API)
