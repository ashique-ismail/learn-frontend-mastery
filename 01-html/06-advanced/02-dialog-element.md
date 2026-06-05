# Dialog element

## The Idea

**In plain English:** A dialog element is a special pop-up box that appears on top of the rest of a webpage, asking for your attention or a decision before you can go back to what you were doing. Think of it as a webpage's way of tapping you on the shoulder and saying "hold on, look at this first."

**Real-world analogy:** Imagine you are in a library and you try to walk out with a book. A librarian steps out from behind a counter, stands in front of you, and asks "Did you check this out?" You cannot get past them until you answer — either you say "Yes" and walk out, or "No" and hand it back. Everything else in the library is still there but you cannot interact with it while the librarian is blocking your path.

- The librarian stepping out = the `<dialog>` element appearing on screen
- The librarian blocking the exit = the modal backdrop that prevents clicking anything else
- Saying "Yes" or "No" to the librarian = clicking a button that calls `dialog.close()` with a return value

---

## Overview

The `<dialog>` element represents a modal or non-modal dialog box. It provides built-in functionality for showing, hiding, and managing focus with minimal JavaScript.

## Basic dialog

```html
<dialog id="my-dialog">
  <h2>Dialog Title</h2>
  <p>Dialog content goes here</p>
  <button id="close-btn">Close</button>
</dialog>

<button id="open-btn">Open Dialog</button>

<script>
const dialog = document.getElementById('my-dialog');
const openBtn = document.getElementById('open-btn');
const closeBtn = document.getElementById('close-btn');

openBtn.addEventListener('click', () => {
  dialog.showModal(); // Opens as modal
});

closeBtn.addEventListener('click', () => {
  dialog.close();
});
</script>
```

## Opening methods

### showModal()

Opens as modal dialog (blocking backdrop, focus trap).

```javascript
dialog.showModal();
```

**Characteristics:**

- Backdrop overlay (::backdrop)
- Focus trapped inside dialog
- Closes with Escape key
- Dismisses clicks on backdrop
- Top layer (above all other content)

### show()

Opens as non-modal dialog (no backdrop, no focus trap).

```javascript
dialog.show();
```

**Characteristics:**

- No backdrop
- No focus trap
- Doesn't close with Escape
- Positioned in normal flow

## Closing methods

### close()

```javascript
dialog.close();
```

### close(returnValue)

```javascript
dialog.close('confirmed');
```

Access return value:

```javascript
dialog.addEventListener('close', () => {
  console.log('Return value:', dialog.returnValue);
});
```

## Styling the dialog

### Basic styling

```css
dialog {
  border: none;
  border-radius: 8px;
  padding: 24px;
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.2);
  max-width: 500px;
}

dialog h2 {
  margin-top: 0;
}
```

### Backdrop styling

```css
dialog::backdrop {
  background: rgba(0, 0, 0, 0.5);
  backdrop-filter: blur(4px);
}
```

### Open state

```css
dialog[open] {
  animation: slideIn 0.3s ease;
}

@keyframes slideIn {
  from {
    opacity: 0;
    transform: translateY(-50px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}
```

## Form in dialog

### Using form method="dialog"

```html
<dialog id="form-dialog">
  <form method="dialog">
    <h2>Edit Profile</h2>
    
    <label for="name">Name</label>
    <input type="text" id="name" required>
    
    <label for="email">Email</label>
    <input type="email" id="email" required>
    
    <div class="buttons">
      <button type="submit" value="cancel">Cancel</button>
      <button type="submit" value="confirm">Save</button>
    </div>
  </form>
</dialog>

<script>
const dialog = document.getElementById('form-dialog');

dialog.addEventListener('close', () => {
  if (dialog.returnValue === 'confirm') {
    const name = document.getElementById('name').value;
    const email = document.getElementById('email').value;
    console.log('Saved:', { name, email });
  }
});
</script>
```

## Confirmation dialog

```html
<dialog id="confirm-dialog">
  <h2>Confirm Action</h2>
  <p>Are you sure you want to delete this item? This action cannot be undone.</p>
  
  <div class="buttons">
    <button id="cancel-btn">Cancel</button>
    <button id="confirm-btn" class="danger">Delete</button>
  </div>
</dialog>

<script>
class ConfirmDialog {
  constructor(dialogId) {
    this.dialog = document.getElementById(dialogId);
    this.cancelBtn = this.dialog.querySelector('#cancel-btn');
    this.confirmBtn = this.dialog.querySelector('#confirm-btn');
    
    this.cancelBtn.addEventListener('click', () => {
      this.dialog.close('cancel');
    });
    
    this.confirmBtn.addEventListener('click', () => {
      this.dialog.close('confirm');
    });
  }
  
  show() {
    return new Promise((resolve) => {
      this.dialog.showModal();
      
      this.dialog.addEventListener('close', () => {
        resolve(this.dialog.returnValue === 'confirm');
      }, { once: true });
    });
  }
}

// Usage
const confirmDialog = new ConfirmDialog('confirm-dialog');

deleteBtn.addEventListener('click', async () => {
  const confirmed = await confirmDialog.show();
  
  if (confirmed) {
    deleteItem();
  }
});
</script>
```

## Alert dialog

```html
<dialog id="alert-dialog">
  <h2 id="alert-title">Alert</h2>
  <p id="alert-message"></p>
  <button id="alert-ok">OK</button>
</dialog>

<script>
class AlertDialog {
  constructor() {
    this.dialog = document.getElementById('alert-dialog');
    this.title = document.getElementById('alert-title');
    this.message = document.getElementById('alert-message');
    this.okBtn = document.getElementById('alert-ok');
    
    this.okBtn.addEventListener('click', () => {
      this.dialog.close();
    });
  }
  
  show(title, message) {
    this.title.textContent = title;
    this.message.textContent = message;
    this.dialog.showModal();
    this.okBtn.focus();
  }
}

// Usage
const alertDialog = new AlertDialog();
alertDialog.show('Success', 'Your changes have been saved.');
</script>
```

## Prompt dialog

```html
<dialog id="prompt-dialog">
  <form method="dialog">
    <h2 id="prompt-title">Input Required</h2>
    <p id="prompt-message"></p>
    
    <input type="text" id="prompt-input" required>
    
    <div class="buttons">
      <button type="submit" value="cancel">Cancel</button>
      <button type="submit" value="ok">OK</button>
    </div>
  </form>
</dialog>

<script>
class PromptDialog {
  constructor() {
    this.dialog = document.getElementById('prompt-dialog');
    this.form = this.dialog.querySelector('form');
    this.title = document.getElementById('prompt-title');
    this.message = document.getElementById('prompt-message');
    this.input = document.getElementById('prompt-input');
  }
  
  show(title, message, defaultValue = '') {
    return new Promise((resolve) => {
      this.title.textContent = title;
      this.message.textContent = message;
      this.input.value = defaultValue;
      
      this.dialog.showModal();
      this.input.focus();
      this.input.select();
      
      this.dialog.addEventListener('close', () => {
        if (this.dialog.returnValue === 'ok') {
          resolve(this.input.value);
        } else {
          resolve(null);
        }
      }, { once: true });
    });
  }
}

// Usage
const promptDialog = new PromptDialog();

renameBtn.addEventListener('click', async () => {
  const newName = await promptDialog.show(
    'Rename File',
    'Enter new name:',
    'document.txt'
  );
  
  if (newName) {
    renameFile(newName);
  }
});
</script>
```

## Accessibility

### ARIA attributes

```html
<dialog 
  id="accessible-dialog"
  role="dialog"
  aria-labelledby="dialog-title"
  aria-describedby="dialog-desc">
  
  <h2 id="dialog-title">Dialog Title</h2>
  <p id="dialog-desc">Dialog description</p>
  
  <button>Close</button>
</dialog>
```

### Focus management

```javascript
const dialog = document.getElementById('my-dialog');

dialog.addEventListener('close', () => {
  // Return focus to trigger element
  triggerElement.focus();
});

// Focus first interactive element when opened
dialog.showModal();
const firstButton = dialog.querySelector('button');
firstButton.focus();
```

### Escape key handling

Built-in with `showModal()`, but can be customized:

```javascript
dialog.addEventListener('cancel', (e) => {
  // Prevent default escape behavior
  e.preventDefault();
  
  // Custom behavior
  if (hasUnsavedChanges) {
    confirmClose();
  } else {
    dialog.close();
  }
});
```

## Backdrop click handling

```javascript
dialog.addEventListener('click', (e) => {
  // Close when clicking backdrop
  if (e.target === dialog) {
    dialog.close();
  }
});
```

Or prevent backdrop clicks:

```javascript
dialog.addEventListener('click', (e) => {
  if (e.target === dialog) {
    e.preventDefault();
    // Optionally shake dialog or show message
  }
});
```

## Complete modal example

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Dialog Example</title>
  <style>
    dialog {
      border: none;
      border-radius: 8px;
      padding: 0;
      max-width: 500px;
      box-shadow: 0 4px 16px rgba(0, 0, 0, 0.2);
    }
    
    dialog::backdrop {
      background: rgba(0, 0, 0, 0.5);
      backdrop-filter: blur(2px);
    }
    
    .dialog-header {
      padding: 20px 24px;
      border-bottom: 1px solid #eee;
      display: flex;
      justify-content: space-between;
      align-items: center;
    }
    
    .dialog-header h2 {
      margin: 0;
      font-size: 20px;
    }
    
    .dialog-body {
      padding: 24px;
    }
    
    .dialog-footer {
      padding: 16px 24px;
      border-top: 1px solid #eee;
      display: flex;
      gap: 8px;
      justify-content: flex-end;
    }
    
    .close-button {
      background: none;
      border: none;
      font-size: 24px;
      cursor: pointer;
      color: #666;
      padding: 0;
      width: 32px;
      height: 32px;
      border-radius: 4px;
    }
    
    .close-button:hover {
      background: #f5f5f5;
    }
    
    button {
      padding: 8px 16px;
      border: none;
      border-radius: 4px;
      cursor: pointer;
      font-size: 14px;
    }
    
    .btn-primary {
      background: #0066cc;
      color: white;
    }
    
    .btn-primary:hover {
      background: #0052a3;
    }
    
    .btn-secondary {
      background: #f5f5f5;
      color: #333;
    }
    
    .btn-secondary:hover {
      background: #e0e0e0;
    }
  </style>
</head>
<body>
  <button id="open-dialog">Open Dialog</button>
  
  <dialog id="my-dialog">
    <div class="dialog-header">
      <h2>Dialog Title</h2>
      <button class="close-button" aria-label="Close">&times;</button>
    </div>
    
    <div class="dialog-body">
      <p>This is the dialog content. You can put any content here.</p>
      <p>The dialog will trap focus and can be closed with the Escape key.</p>
    </div>
    
    <div class="dialog-footer">
      <button class="btn-secondary" data-action="cancel">Cancel</button>
      <button class="btn-primary" data-action="confirm">Confirm</button>
    </div>
  </dialog>
  
  <script>
    const dialog = document.getElementById('my-dialog');
    const openBtn = document.getElementById('open-dialog');
    const closeBtn = dialog.querySelector('.close-button');
    const cancelBtn = dialog.querySelector('[data-action="cancel"]');
    const confirmBtn = dialog.querySelector('[data-action="confirm"]');
    
    openBtn.addEventListener('click', () => {
      dialog.showModal();
    });
    
    closeBtn.addEventListener('click', () => {
      dialog.close('cancel');
    });
    
    cancelBtn.addEventListener('click', () => {
      dialog.close('cancel');
    });
    
    confirmBtn.addEventListener('click', () => {
      dialog.close('confirm');
    });
    
    // Close on backdrop click
    dialog.addEventListener('click', (e) => {
      if (e.target === dialog) {
        dialog.close('cancel');
      }
    });
    
    // Handle close event
    dialog.addEventListener('close', () => {
      console.log('Dialog closed with return value:', dialog.returnValue);
    });
  </script>
</body>
</html>
```

## Fullscreen dialog

```css
dialog.fullscreen {
  max-width: none;
  max-height: none;
  width: 100vw;
  height: 100vh;
  margin: 0;
  padding: 0;
  border-radius: 0;
}
```

```javascript
const dialog = document.getElementById('my-dialog');
dialog.classList.add('fullscreen');
dialog.showModal();
```

## Animation

### Opening animation

```css
dialog {
  animation: slideUp 0.3s ease;
}

@keyframes slideUp {
  from {
    opacity: 0;
    transform: translateY(100px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

dialog::backdrop {
  animation: fadeIn 0.3s ease;
}

@keyframes fadeIn {
  from {
    opacity: 0;
  }
  to {
    opacity: 1;
  }
}
```

### Closing animation

```javascript
dialog.addEventListener('close', () => {
  dialog.classList.add('closing');
});

dialog.addEventListener('animationend', () => {
  dialog.classList.remove('closing');
});
```

```css
dialog.closing {
  animation: slideDown 0.3s ease;
}

@keyframes slideDown {
  from {
    opacity: 1;
    transform: translateY(0);
  }
  to {
    opacity: 0;
    transform: translateY(100px);
  }
}
```

## Browser support

Modern browsers support `<dialog>`:

- Chrome 37+
- Firefox 98+
- Safari 15.4+
- Edge 79+

### Polyfill

For older browsers:

```html
<script src="https://cdn.jsdelivr.net/npm/dialog-polyfill@latest/dist/dialog-polyfill.js"></script>
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/dialog-polyfill@latest/dist/dialog-polyfill.css">

<script>
const dialog = document.querySelector('dialog');
dialogPolyfill.registerDialog(dialog);
</script>
```

## Best practices

1. Always use `showModal()` for modal dialogs
2. Provide clear way to close (X button, Cancel button, Escape key)
3. Label dialogs with `aria-labelledby` and `aria-describedby`
4. Return focus to trigger element on close
5. Handle backdrop clicks appropriately
6. Use form method="dialog" for forms
7. Provide keyboard navigation
8. Test with screen readers
9. Consider animation performance
10. Handle edge cases (rapid open/close, nested dialogs)

## Key takeaways

- `<dialog>` element provides native modal/non-modal dialogs
- Use `showModal()` for modal, `show()` for non-modal
- Built-in focus trap and Escape key handling with `showModal()`
- Style backdrop with `::backdrop` pseudo-element
- Use `form method="dialog"` for forms in dialogs
- Access return value via `dialog.returnValue`
- Handle `close` event for cleanup
- Provide accessible labels and focus management
- Click backdrop or press Escape to close
- Consider polyfill for older browsers
