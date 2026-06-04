# Web App Manifest

## Introduction

The Web App Manifest is a JSON file that provides metadata about your Progressive Web App, enabling it to be installed on a user's device and launched like a native application. It controls how your app appears when installed, including the app icon, name, theme colors, and display mode.

A properly configured manifest is essential for creating installable PWAs that provide native-like experiences across different platforms and devices.

## Basic Manifest Structure

### Minimal manifest.json

```json
{
  "name": "My Progressive Web App",
  "short_name": "My PWA",
  "description": "A Progressive Web App that does amazing things",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#2196F3",
  "icons": [
    {
      "src": "/icons/icon-192.png",
      "sizes": "192x192",
      "type": "image/png"
    },
    {
      "src": "/icons/icon-512.png",
      "sizes": "512x512",
      "type": "image/png"
    }
  ]
}
```

### Linking the Manifest

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>My PWA</title>
    
    <!-- Link to manifest -->
    <link rel="manifest" href="/manifest.json">
    
    <!-- Theme color for browser UI -->
    <meta name="theme-color" content="#2196F3">
    
    <!-- iOS specific -->
    <meta name="apple-mobile-web-app-capable" content="yes">
    <meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
    <meta name="apple-mobile-web-app-title" content="My PWA">
    <link rel="apple-touch-icon" href="/icons/icon-192.png">
</head>
<body>
    <h1>My Progressive Web App</h1>
</body>
</html>
```

## Complete Manifest Example

```json
{
  "name": "Todo App - Manage Your Tasks",
  "short_name": "Todo",
  "description": "A powerful todo application that works offline and syncs across devices",
  "lang": "en-US",
  "start_url": "/?source=pwa",
  "scope": "/",
  "display": "standalone",
  "orientation": "any",
  "background_color": "#ffffff",
  "theme_color": "#667eea",
  "icons": [
    {
      "src": "/icons/icon-72.png",
      "sizes": "72x72",
      "type": "image/png",
      "purpose": "any"
    },
    {
      "src": "/icons/icon-96.png",
      "sizes": "96x96",
      "type": "image/png",
      "purpose": "any"
    },
    {
      "src": "/icons/icon-128.png",
      "sizes": "128x128",
      "type": "image/png",
      "purpose": "any"
    },
    {
      "src": "/icons/icon-144.png",
      "sizes": "144x144",
      "type": "image/png",
      "purpose": "any"
    },
    {
      "src": "/icons/icon-152.png",
      "sizes": "152x152",
      "type": "image/png",
      "purpose": "any"
    },
    {
      "src": "/icons/icon-192.png",
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "any maskable"
    },
    {
      "src": "/icons/icon-384.png",
      "sizes": "384x384",
      "type": "image/png",
      "purpose": "any"
    },
    {
      "src": "/icons/icon-512.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "any maskable"
    }
  ],
  "screenshots": [
    {
      "src": "/screenshots/desktop-1.png",
      "sizes": "1280x720",
      "type": "image/png",
      "form_factor": "wide"
    },
    {
      "src": "/screenshots/mobile-1.png",
      "sizes": "750x1334",
      "type": "image/png",
      "form_factor": "narrow"
    }
  ],
  "categories": ["productivity", "utilities"],
  "shortcuts": [
    {
      "name": "New Todo",
      "short_name": "New",
      "description": "Create a new todo item",
      "url": "/new",
      "icons": [
        {
          "src": "/icons/new-96.png",
          "sizes": "96x96"
        }
      ]
    },
    {
      "name": "Today's Tasks",
      "short_name": "Today",
      "description": "View today's tasks",
      "url": "/today",
      "icons": [
        {
          "src": "/icons/today-96.png",
          "sizes": "96x96"
        }
      ]
    }
  ],
  "share_target": {
    "action": "/share",
    "method": "POST",
    "enctype": "multipart/form-data",
    "params": {
      "title": "title",
      "text": "text",
      "url": "url",
      "files": [
        {
          "name": "file",
          "accept": ["image/*", "application/pdf"]
        }
      ]
    }
  },
  "related_applications": [
    {
      "platform": "play",
      "url": "https://play.google.com/store/apps/details?id=com.example.todo",
      "id": "com.example.todo"
    }
  ],
  "prefer_related_applications": false
}
```

## Manifest Properties

### Display Modes

```json
{
  "display": "standalone"
}
```

Available modes:
- **fullscreen**: No browser UI (games, media players)
- **standalone**: Looks like native app, no browser UI
- **minimal-ui**: Minimal browser UI (back button, reload)
- **browser**: Normal browser experience

```javascript
// Detect display mode
function getDisplayMode() {
    const isStandalone = window.matchMedia('(display-mode: standalone)').matches;
    
    if (isStandalone) {
        console.log('Running as installed PWA');
    } else {
        console.log('Running in browser');
    }
    
    return isStandalone ? 'standalone' : 'browser';
}

// Listen for display mode changes
if ('matchMedia' in window) {
    window.matchMedia('(display-mode: standalone)').addEventListener('change', (e) => {
        console.log('Display mode changed:', e.matches ? 'standalone' : 'browser');
    });
}
```

### Orientation

```json
{
  "orientation": "portrait"
}
```

Options: `any`, `natural`, `landscape`, `portrait`, `portrait-primary`, `portrait-secondary`, `landscape-primary`, `landscape-secondary`

### App Shortcuts

```json
{
  "shortcuts": [
    {
      "name": "Search",
      "short_name": "Search",
      "description": "Search for items",
      "url": "/search?utm_source=homescreen",
      "icons": [
        {
          "src": "/icons/search-96.png",
          "sizes": "96x96",
          "type": "image/png"
        }
      ]
    }
  ]
}
```

### Share Target

```json
{
  "share_target": {
    "action": "/share",
    "method": "POST",
    "enctype": "multipart/form-data",
    "params": {
      "title": "title",
      "text": "text",
      "url": "url",
      "files": [
        {
          "name": "images",
          "accept": ["image/*"]
        }
      ]
    }
  }
}
```

```javascript
// Handle shared content
if (window.location.pathname === '/share') {
    const formData = await request.formData();
    const title = formData.get('title');
    const text = formData.get('text');
    const url = formData.get('url');
    const files = formData.getAll('images');
    
    console.log('Shared:', { title, text, url, files });
}
```

## Installation Prompt

### Handling Install Prompt

```javascript
// app.js - Install prompt handler
let deferredPrompt;
const installButton = document.getElementById('install-button');

window.addEventListener('beforeinstallprompt', (e) => {
    // Prevent default browser prompt
    e.preventDefault();
    
    // Store event for later use
    deferredPrompt = e;
    
    // Show custom install button
    installButton.style.display = 'block';
    
    console.log('Install prompt available');
});

installButton.addEventListener('click', async () => {
    if (!deferredPrompt) {
        return;
    }
    
    // Show install prompt
    deferredPrompt.prompt();
    
    // Wait for user choice
    const { outcome } = await deferredPrompt.userChoice;
    console.log(`User ${outcome === 'accepted' ? 'accepted' : 'dismissed'} the install prompt`);
    
    // Clear prompt
    deferredPrompt = null;
    installButton.style.display = 'none';
});

window.addEventListener('appinstalled', (e) => {
    console.log('PWA installed successfully');
    
    // Track installation
    analytics.track('pwa_installed');
    
    // Hide install button
    installButton.style.display = 'none';
});
```

### Custom Install UI

```html
<!-- install-banner.html -->
<div id="install-banner" class="install-banner" style="display: none;">
    <div class="banner-content">
        <img src="/icons/icon-72.png" alt="App icon">
        <div class="banner-text">
            <h3>Install My PWA</h3>
            <p>Add to your home screen for easy access</p>
        </div>
        <button id="install-accept" class="btn-primary">Install</button>
        <button id="install-dismiss" class="btn-secondary">Not now</button>
    </div>
</div>

<style>
.install-banner {
    position: fixed;
    bottom: 0;
    left: 0;
    right: 0;
    background: white;
    box-shadow: 0 -2px 10px rgba(0,0,0,0.1);
    padding: 16px;
    z-index: 1000;
    animation: slideUp 0.3s ease-out;
}

@keyframes slideUp {
    from { transform: translateY(100%); }
    to { transform: translateY(0); }
}

.banner-content {
    display: flex;
    align-items: center;
    gap: 16px;
}

.banner-content img {
    width: 48px;
    height: 48px;
    border-radius: 8px;
}

.banner-text {
    flex: 1;
}

.banner-text h3 {
    margin: 0;
    font-size: 16px;
}

.banner-text p {
    margin: 4px 0 0 0;
    font-size: 14px;
    color: #666;
}
</style>

<script>
const banner = document.getElementById('install-banner');
const acceptBtn = document.getElementById('install-accept');
const dismissBtn = document.getElementById('install-dismiss');
let deferredPrompt;

window.addEventListener('beforeinstallprompt', (e) => {
    e.preventDefault();
    deferredPrompt = e;
    
    // Check if user previously dismissed
    if (!localStorage.getItem('install-dismissed')) {
        banner.style.display = 'block';
    }
});

acceptBtn.addEventListener('click', async () => {
    if (deferredPrompt) {
        deferredPrompt.prompt();
        const { outcome } = await deferredPrompt.userChoice;
        
        if (outcome === 'accepted') {
            localStorage.setItem('install-accepted', 'true');
        }
        
        banner.style.display = 'none';
        deferredPrompt = null;
    }
});

dismissBtn.addEventListener('click', () => {
    localStorage.setItem('install-dismissed', 'true');
    banner.style.display = 'none';
});
</script>
```

## Platform-Specific Configurations

### iOS Configuration

```html
<!-- iOS meta tags -->
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
<meta name="apple-mobile-web-app-title" content="My PWA">

<!-- iOS icons -->
<link rel="apple-touch-icon" href="/icons/icon-152.png">
<link rel="apple-touch-icon" sizes="180x180" href="/icons/icon-180.png">

<!-- iOS splash screens -->
<link rel="apple-touch-startup-image" 
      href="/splash/iphone-x.png" 
      media="(device-width: 375px) and (device-height: 812px) and (-webkit-device-pixel-ratio: 3)">
<link rel="apple-touch-startup-image" 
      href="/splash/iphone-8.png" 
      media="(device-width: 375px) and (device-height: 667px) and (-webkit-device-pixel-ratio: 2)">
```

### Android Configuration

```html
<!-- Android theme color -->
<meta name="theme-color" content="#2196F3">

<!-- Android status bar -->
<meta name="mobile-web-app-capable" content="yes">
```

## Maskable Icons

```json
{
  "icons": [
    {
      "src": "/icons/maskable-192.png",
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "maskable"
    },
    {
      "src": "/icons/maskable-512.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "maskable"
    },
    {
      "src": "/icons/any-192.png",
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "any"
    },
    {
      "src": "/icons/monochrome-192.png",
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "monochrome"
    }
  ]
}
```

## Manifest Validation

```javascript
// validate-manifest.js
async function validateManifest() {
    try {
        const response = await fetch('/manifest.json');
        const manifest = await response.json();
        
        const errors = [];
        const warnings = [];
        
        // Required fields
        if (!manifest.name && !manifest.short_name) {
            errors.push('Either name or short_name is required');
        }
        
        if (!manifest.start_url) {
            errors.push('start_url is required');
        }
        
        if (!manifest.icons || manifest.icons.length === 0) {
            errors.push('At least one icon is required');
        }
        
        // Recommended fields
        if (!manifest.display) {
            warnings.push('display property is recommended');
        }
        
        if (!manifest.theme_color) {
            warnings.push('theme_color is recommended');
        }
        
        if (!manifest.background_color) {
            warnings.push('background_color is recommended');
        }
        
        // Icon validation
        const hasLargeIcon = manifest.icons?.some(icon => 
            icon.sizes.split('x')[0] >= 192
        );
        
        if (!hasLargeIcon) {
            errors.push('At least one icon should be 192x192 or larger');
        }
        
        console.log('Validation results:', { errors, warnings });
        
        return { valid: errors.length === 0, errors, warnings };
    } catch (error) {
        console.error('Failed to validate manifest:', error);
        return { valid: false, errors: [error.message], warnings: [] };
    }
}
```

## Browser Support

| Feature | Chrome | Firefox | Safari | Edge |
|---------|--------|---------|--------|------|
| Basic Manifest | 39+ | ❌ | 11.3+ | 79+ |
| Install Prompt | 68+ | ❌ | ❌ | 79+ |
| Shortcuts | 84+ | ❌ | ❌ | 84+ |
| Share Target | 89+ | ❌ | ❌ | 89+ |

## Common Mistakes

1. **Missing required fields**
2. **Wrong icon sizes**
3. **Invalid JSON**
4. **Wrong MIME type**
5. **Not linking manifest**

## Best Practices

1. **Include all required fields**
2. **Provide multiple icon sizes**
3. **Use meaningful names**
4. **Set appropriate display mode**
5. **Define theme colors**
6. **Add shortcuts for common actions**
7. **Implement share target**
8. **Test on multiple devices**

## Interview Questions

1. What is the Web App Manifest?
2. What are the required manifest fields?
3. What are display modes?
4. How do you handle install prompts?
5. What are maskable icons?
6. How do shortcuts work?
7. What is share target?
8. How do you test manifest configuration?

## Key Takeaways

- Manifest enables PWA installation
- Required fields: name/short_name, start_url, icons
- Display modes control app presentation
- Install prompt can be customized
- Platform-specific configurations needed
- Multiple icon sizes required
- Shortcuts improve app access
- Share target enables content sharing

## Resources

- [MDN: Web App Manifest](https://developer.mozilla.org/en-US/docs/Web/Manifest)
- [Web.dev: Add a Web App Manifest](https://web.dev/add-manifest/)
- [Maskable.app](https://maskable.app/)
- [Manifest Generator](https://app-manifest.firebaseapp.com/)
- [PWA Builder](https://www.pwabuilder.com/)
