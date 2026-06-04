# Service Worker Lifecycle

## Table of Contents
- [Introduction](#introduction)
- [Service Worker Registration](#service-worker-registration)
- [Install Event](#install-event)
- [Activate Event](#activate-event)
- [Update Mechanism](#update-mechanism)
- [skipWaiting](#skipwaiting)
- [clients.claim()](#clientsclaim)
- [Common Misconceptions](#common-misconceptions)
- [Performance Implications](#performance-implications)
- [Browser Differences](#browser-differences)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Service Workers are a powerful feature that enable Progressive Web Apps by providing offline functionality, background sync, and push notifications. Understanding the Service Worker lifecycle is crucial for implementing reliable caching strategies and ensuring smooth updates. The lifecycle consists of registration, installation, activation, and update phases, each with specific purposes and timing.

## Service Worker Registration

### Basic Registration

```javascript
// Client-side: Registering a Service Worker
class ServiceWorkerRegistration {
  static async register() {
    // Check if Service Workers are supported
    if (!('serviceWorker' in navigator)) {
      console.log('Service Workers not supported');
      return null;
    }

    try {
      // Register the service worker
      const registration = await navigator.serviceWorker.register('/sw.js', {
        scope: '/' // Default scope is the location of the SW file
      });

      console.log('Service Worker registered:', registration);
      console.log('Scope:', registration.scope);

      return registration;
    } catch (error) {
      console.error('Service Worker registration failed:', error);
      return null;
    }
  }

  // Registration with custom scope
  static async registerWithScope() {
    const registration = await navigator.serviceWorker.register('/sw.js', {
      scope: '/app/' // Only controls URLs under /app/
    });

    return registration;
  }

  // Registration with options
  static async registerWithOptions() {
    const registration = await navigator.serviceWorker.register('/sw.js', {
      scope: '/',
      type: 'classic', // or 'module' for ES modules
      updateViaCache: 'none' // 'imports', 'all', or 'none'
    });

    return registration;
  }
}

// Practical registration with error handling
async function registerServiceWorker() {
  if (!('serviceWorker' in navigator)) {
    console.warn('Service Workers not supported');
    return;
  }

  // Wait for page load to avoid competing with other resources
  if (document.readyState === 'loading') {
    window.addEventListener('load', () => registerServiceWorker());
    return;
  }

  try {
    const registration = await navigator.serviceWorker.register('/sw.js');
    
    console.log('SW registered:', registration);
    
    // Check registration state
    if (registration.installing) {
      console.log('Service Worker installing');
    } else if (registration.waiting) {
      console.log('Service Worker waiting');
    } else if (registration.active) {
      console.log('Service Worker active');
    }
    
    // Listen for updates
    registration.addEventListener('updatefound', () => {
      console.log('New Service Worker found');
      const newWorker = registration.installing;
      
      newWorker.addEventListener('statechange', () => {
        console.log('SW state changed:', newWorker.state);
      });
    });
    
  } catch (error) {
    console.error('SW registration failed:', error);
  }
}

// Call registration
registerServiceWorker();
```

### Scope and Control

```javascript
// Understanding Service Worker scope
class ServiceWorkerScope {
  static examples() {
    return {
      defaultScope: {
        swLocation: '/sw.js',
        scope: '/', // Entire origin
        controls: ['/', '/about', '/app/page1', '/api/data']
      },
      
      restrictedScope: {
        swLocation: '/sw.js',
        scope: '/app/',
        controls: ['/app/page1', '/app/page2'],
        doesNotControl: ['/', '/about', '/other/page']
      },
      
      subfolderScope: {
        swLocation: '/app/sw.js',
        defaultScope: '/app/', // Cannot control above /app/
        maxScope: '/app/'
      },
      
      headerOverride: {
        header: 'Service-Worker-Allowed: /',
        allows: 'SW at /app/sw.js to control entire origin',
        usage: 'Override scope restriction via HTTP header'
      }
    };
  }

  // Checking if current page is controlled
  static isControlled() {
    if (navigator.serviceWorker.controller) {
      console.log('Page is controlled by SW:', navigator.serviceWorker.controller.scriptURL);
      return true;
    } else {
      console.log('Page is not controlled by SW');
      return false;
    }
  }

  // Getting the controlling Service Worker
  static getController() {
    return navigator.serviceWorker.controller;
  }

  // Waiting for control
  static async waitForControl() {
    if (navigator.serviceWorker.controller) {
      return navigator.serviceWorker.controller;
    }

    return new Promise((resolve) => {
      navigator.serviceWorker.addEventListener('controllerchange', () => {
        resolve(navigator.serviceWorker.controller);
      });
    });
  }
}

// Practical scope management
class ScopeManagement {
  // Multiple Service Workers for different sections
  static async registerMultiple() {
    // Main app Service Worker
    const mainSW = await navigator.serviceWorker.register('/sw-main.js', {
      scope: '/'
    });

    // Admin section Service Worker
    const adminSW = await navigator.serviceWorker.register('/admin/sw-admin.js', {
      scope: '/admin/'
    });

    return { mainSW, adminSW };
  }

  // Checking which SW controls current page
  static getCurrentSW() {
    const controller = navigator.serviceWorker.controller;
    
    if (controller) {
      const scriptURL = controller.scriptURL;
      const url = new URL(scriptURL);
      
      return {
        scriptURL,
        pathname: url.pathname,
        isMainSW: url.pathname === '/sw.js',
        isAdminSW: url.pathname === '/admin/sw-admin.js'
      };
    }
    
    return null;
  }
}
```

### Registration States

```javascript
// Service Worker registration states
class RegistrationStates {
  static async monitorStates() {
    const registration = await navigator.serviceWorker.register('/sw.js');

    // Three possible states in registration
    const states = {
      installing: registration.installing,  // SW is installing
      waiting: registration.waiting,        // SW installed, waiting to activate
      active: registration.active           // SW is active and controlling pages
    };

    console.log('Registration states:', states);

    // Monitor state changes
    if (registration.installing) {
      this.trackStateChanges(registration.installing, 'installing');
    }

    if (registration.waiting) {
      this.trackStateChanges(registration.waiting, 'waiting');
    }

    if (registration.active) {
      this.trackStateChanges(registration.active, 'active');
    }

    return registration;
  }

  static trackStateChanges(worker, initialState) {
    console.log(`Worker started in ${initialState} state`);

    worker.addEventListener('statechange', () => {
      console.log(`State changed from ${initialState} to ${worker.state}`);
      
      // Possible states: installing, installed, activating, activated, redundant
      switch (worker.state) {
        case 'installing':
          console.log('Service Worker installing...');
          break;
        case 'installed':
          console.log('Service Worker installed');
          break;
        case 'activating':
          console.log('Service Worker activating...');
          break;
        case 'activated':
          console.log('Service Worker activated');
          break;
        case 'redundant':
          console.log('Service Worker redundant (replaced or failed)');
          break;
      }
    });
  }

  // Complete lifecycle monitoring
  static async monitorFullLifecycle() {
    const registration = await navigator.serviceWorker.register('/sw.js');

    return new Promise((resolve) => {
      const states = [];

      const checkWorker = (worker, name) => {
        if (!worker) return;

        states.push({ name, state: worker.state });

        worker.addEventListener('statechange', () => {
          states.push({ name, state: worker.state });
          
          if (worker.state === 'activated') {
            resolve({ registration, states });
          }
        });
      };

      checkWorker(registration.installing, 'installing');
      checkWorker(registration.waiting, 'waiting');
      checkWorker(registration.active, 'active');
    });
  }
}
```

## Install Event

### Basic Installation

```javascript
// Service Worker: Install event
// File: sw.js

const CACHE_NAME = 'my-app-v1';
const urlsToCache = [
  '/',
  '/styles/main.css',
  '/scripts/app.js',
  '/images/logo.png'
];

// Install event - cache resources
self.addEventListener('install', (event) => {
  console.log('Service Worker installing...');

  // Wait until caching is complete
  event.waitUntil(
    caches.open(CACHE_NAME)
      .then((cache) => {
        console.log('Opened cache');
        return cache.addAll(urlsToCache);
      })
      .then(() => {
        console.log('All resources cached');
      })
      .catch((error) => {
        console.error('Cache failed:', error);
        throw error; // Installation fails if caching fails
      })
  );
});

// Install with error handling
self.addEventListener('install', (event) => {
  event.waitUntil(
    (async () => {
      try {
        const cache = await caches.open(CACHE_NAME);
        
        // Cache critical resources
        const criticalResources = [
          '/',
          '/styles/critical.css',
          '/scripts/app.js'
        ];
        
        await cache.addAll(criticalResources);
        console.log('Critical resources cached');
        
        // Cache optional resources (don't fail install if these fail)
        const optionalResources = [
          '/images/banner.jpg',
          '/images/background.jpg'
        ];
        
        await Promise.allSettled(
          optionalResources.map(url => cache.add(url))
        );
        
        console.log('Installation complete');
      } catch (error) {
        console.error('Installation failed:', error);
        throw error;
      }
    })()
  );
});
```

### Advanced Installation Strategies

```javascript
// Advanced install patterns
class InstallStrategies {
  // Strategy 1: Versioned caching
  static versionedInstall() {
    const VERSION = '1.2.3';
    const CACHE_NAME = `my-app-${VERSION}`;

    self.addEventListener('install', (event) => {
      event.waitUntil(
        caches.open(CACHE_NAME)
          .then(cache => cache.addAll([
            '/',
            '/styles/main.css',
            '/scripts/app.js'
          ]))
      );
    });
  }

  // Strategy 2: Progressive caching
  static progressiveInstall() {
    self.addEventListener('install', (event) => {
      event.waitUntil(
        (async () => {
          const cache = await caches.open(CACHE_NAME);
          
          // Phase 1: Essential resources (must succeed)
          const essential = ['/', '/app.js', '/styles.css'];
          await cache.addAll(essential);
          
          // Phase 2: Important resources (log failures but don't fail install)
          const important = ['/logo.png', '/fonts/main.woff2'];
          const importantResults = await Promise.allSettled(
            important.map(url => cache.add(url))
          );
          
          importantResults.forEach((result, index) => {
            if (result.status === 'rejected') {
              console.warn(`Failed to cache ${important[index]}`);
            }
          });
          
          // Phase 3: Nice-to-have resources (completely optional)
          const optional = ['/images/banner1.jpg', '/images/banner2.jpg'];
          Promise.allSettled(optional.map(url => cache.add(url)));
          
          console.log('Progressive installation complete');
        })()
      );
    });
  }

  // Strategy 3: Conditional caching
  static conditionalInstall() {
    self.addEventListener('install', (event) => {
      event.waitUntil(
        (async () => {
          const cache = await caches.open(CACHE_NAME);
          
          // Always cache core resources
          await cache.addAll([
            '/',
            '/app.js',
            '/styles.css'
          ]);
          
          // Conditionally cache based on client info
          const clients = await self.clients.matchAll();
          
          if (clients.length > 0) {
            // Fetch client preferences
            const preferences = await this.getClientPreferences(clients[0]);
            
            if (preferences.highQuality) {
              await cache.addAll([
                '/images/hq/banner.jpg',
                '/images/hq/background.jpg'
              ]);
            } else {
              await cache.addAll([
                '/images/lq/banner.jpg',
                '/images/lq/background.jpg'
              ]);
            }
          }
        })()
      );
    });
  }

  // Strategy 4: Streaming installation
  static streamingInstall() {
    self.addEventListener('install', (event) => {
      event.waitUntil(
        (async () => {
          const cache = await caches.open(CACHE_NAME);
          const resources = [
            '/', '/app.js', '/styles.css',
            '/page1', '/page2', '/page3',
            '/img1.jpg', '/img2.jpg', '/img3.jpg'
          ];
          
          // Cache resources one by one
          for (const url of resources) {
            try {
              await cache.add(url);
              console.log(`Cached: ${url}`);
              
              // Send progress to clients
              const clients = await self.clients.matchAll();
              clients.forEach(client => {
                client.postMessage({
                  type: 'INSTALL_PROGRESS',
                  cached: url,
                  total: resources.length
                });
              });
            } catch (error) {
              console.error(`Failed to cache ${url}:`, error);
            }
          }
        })()
      );
    });
  }
}

// Client-side: Monitoring installation progress
class InstallMonitoring {
  static monitorInstallation() {
    navigator.serviceWorker.addEventListener('message', (event) => {
      if (event.data.type === 'INSTALL_PROGRESS') {
        const progress = (event.data.cached.length / event.data.total) * 100;
        console.log(`Installation progress: ${progress.toFixed(0)}%`);
        
        // Update UI
        this.updateProgressBar(progress);
      }
    });
  }

  static updateProgressBar(progress) {
    const progressBar = document.getElementById('install-progress');
    if (progressBar) {
      progressBar.style.width = `${progress}%`;
      progressBar.setAttribute('aria-valuenow', progress);
    }
  }
}
```

### Install Event Best Practices

```javascript
// Best practices for install event
class InstallBestPractices {
  // 1. Keep install fast
  static fastInstall() {
    self.addEventListener('install', (event) => {
      event.waitUntil(
        caches.open(CACHE_NAME).then(cache => {
          // Only cache critical resources (< 1MB total)
          return cache.addAll([
            '/',
            '/app.js',      // 50KB
            '/styles.css',  // 20KB
            '/logo.svg'     // 5KB
          ]);
          // Total: ~75KB - installs in < 1 second on 3G
        })
      );
    });
  }

  // 2. Handle failures gracefully
  static gracefulFailure() {
    self.addEventListener('install', (event) => {
      event.waitUntil(
        caches.open(CACHE_NAME)
          .then(cache => cache.addAll(urlsToCache))
          .catch((error) => {
            console.error('Install failed:', error);
            
            // Log to analytics
            this.logInstallFailure(error);
            
            // Re-throw to fail installation
            throw error;
          })
      );
    });
  }

  // 3. Use cache busting for updated resources
  static cacheBusting() {
    const VERSION = '1.2.3';
    const urlsToCache = [
      '/',
      `/app.js?v=${VERSION}`,
      `/styles.css?v=${VERSION}`,
      '/logo.svg' // Static assets don't need cache busting
    ];

    self.addEventListener('install', (event) => {
      event.waitUntil(
        caches.open(`my-app-${VERSION}`)
          .then(cache => cache.addAll(urlsToCache))
      );
    });
  }

  // 4. Avoid blocking on non-critical resources
  static nonBlockingCache() {
    self.addEventListener('install', (event) => {
      event.waitUntil(
        (async () => {
          const cache = await caches.open(CACHE_NAME);
          
          // Block on critical resources
          await cache.addAll([
            '/',
            '/app.js',
            '/styles.css'
          ]);
          
          // Don't block on optional resources
          cache.addAll([
            '/images/banner.jpg',
            '/images/background.jpg'
          ]).catch(error => {
            console.warn('Optional caching failed:', error);
          });
        })()
      );
    });
  }
}
```

## Activate Event

### Basic Activation

```javascript
// Service Worker: Activate event
// File: sw.js

self.addEventListener('activate', (event) => {
  console.log('Service Worker activating...');

  event.waitUntil(
    (async () => {
      // Clean up old caches
      const cacheNames = await caches.keys();
      
      await Promise.all(
        cacheNames
          .filter(cacheName => cacheName !== CACHE_NAME)
          .map(cacheName => {
            console.log('Deleting old cache:', cacheName);
            return caches.delete(cacheName);
          })
      );
      
      console.log('Old caches cleaned up');
    })()
  );
});

// Activate with client claim
self.addEventListener('activate', (event) => {
  event.waitUntil(
    (async () => {
      // Clean up old caches
      const cacheNames = await caches.keys();
      await Promise.all(
        cacheNames
          .filter(name => name.startsWith('my-app-') && name !== CACHE_NAME)
          .map(name => caches.delete(name))
      );

      // Take control of all clients immediately
      await self.clients.claim();
      
      console.log('Activation complete, claimed clients');
    })()
  );
});
```

### Advanced Activation Strategies

```javascript
// Advanced activation patterns
class ActivateStrategies {
  // Strategy 1: Whitelist-based cache cleanup
  static whitelistCleanup() {
    const CURRENT_CACHES = {
      static: 'my-app-static-v1',
      dynamic: 'my-app-dynamic-v1',
      images: 'my-app-images-v1'
    };

    self.addEventListener('activate', (event) => {
      event.waitUntil(
        caches.keys().then((cacheNames) => {
          const validCacheNames = Object.values(CURRENT_CACHES);
          
          return Promise.all(
            cacheNames.map((cacheName) => {
              if (!validCacheNames.includes(cacheName)) {
                console.log('Deleting cache:', cacheName);
                return caches.delete(cacheName);
              }
            })
          );
        })
      );
    });
  }

  // Strategy 2: Selective cache cleanup
  static selectiveCleanup() {
    self.addEventListener('activate', (event) => {
      event.waitUntil(
        (async () => {
          const cacheNames = await caches.keys();
          
          // Delete old version caches
          const oldCaches = cacheNames.filter(name => {
            const match = name.match(/my-app-v(\d+)/);
            if (match) {
              const version = parseInt(match[1]);
              return version < 3; // Keep only last 3 versions
            }
            return false;
          });
          
          await Promise.all(oldCaches.map(name => caches.delete(name)));
          
          // Clean up dynamic cache (keep only recent entries)
          const dynamicCache = await caches.open('my-app-dynamic-v1');
          const requests = await dynamicCache.keys();
          
          if (requests.length > 50) {
            // Remove oldest entries
            const toDelete = requests.slice(0, requests.length - 50);
            await Promise.all(toDelete.map(req => dynamicCache.delete(req)));
          }
        })()
      );
    });
  }

  // Strategy 3: Migration between versions
  static migrationActivation() {
    self.addEventListener('activate', (event) => {
      event.waitUntil(
        (async () => {
          // Check if this is an update
          const oldCache = await caches.open('my-app-v1');
          const oldRequests = await oldCache.keys();
          
          if (oldRequests.length > 0) {
            console.log('Migrating from v1 to v2');
            
            // Copy still-valid resources from old cache
            const newCache = await caches.open('my-app-v2');
            const resourcesToKeep = ['/logo.svg', '/fonts/main.woff2'];
            
            for (const url of resourcesToKeep) {
              const response = await oldCache.match(url);
              if (response) {
                await newCache.put(url, response);
                console.log('Migrated:', url);
              }
            }
          }
          
          // Delete old caches
          const cacheNames = await caches.keys();
          await Promise.all(
            cacheNames
              .filter(name => name !== 'my-app-v2')
              .map(name => caches.delete(name))
          );
          
          // Claim clients
          await self.clients.claim();
        })()
      );
    });
  }

  // Strategy 4: Notification on activation
  static notifyClients() {
    self.addEventListener('activate', (event) => {
      event.waitUntil(
        (async () => {
          // Clean up caches
          const cacheNames = await caches.keys();
          await Promise.all(
            cacheNames
              .filter(name => name !== CACHE_NAME)
              .map(name => caches.delete(name))
          );
          
          // Notify all clients about update
          const clients = await self.clients.matchAll();
          clients.forEach(client => {
            client.postMessage({
              type: 'SW_ACTIVATED',
              version: CACHE_NAME,
              message: 'New version activated!'
            });
          });
          
          // Claim clients
          await self.clients.claim();
        })()
      );
    });
  }
}

// Client-side: Handling activation notifications
class ActivationNotifications {
  static setupListener() {
    navigator.serviceWorker.addEventListener('message', (event) => {
      if (event.data.type === 'SW_ACTIVATED') {
        console.log('SW activated:', event.data.version);
        
        // Show notification to user
        this.showUpdateNotification(event.data.message);
      }
    });
  }

  static showUpdateNotification(message) {
    // Show toast/banner
    const banner = document.createElement('div');
    banner.className = 'update-banner';
    banner.innerHTML = `
      ${message}
      <button onclick="location.reload()">Refresh</button>
    `;
    document.body.appendChild(banner);
  }
}
```

### Activation Timing

```javascript
// Understanding activation timing
class ActivationTiming {
  static scenarios() {
    return {
      scenario1: {
        name: 'First installation',
        sequence: [
          'User visits site',
          'SW registered',
          'Install event fires',
          'Activate event fires immediately',
          'SW takes control on next navigation'
        ],
        note: 'First page load not controlled unless using clients.claim()'
      },
      
      scenario2: {
        name: 'Update with open tabs',
        sequence: [
          'New SW version detected',
          'Install event fires',
          'SW waits in "waiting" state',
          'User closes all tabs',
          'Activate event fires when reopened',
          'New SW takes control'
        ],
        note: 'Waiting state prevents breaking open tabs'
      },
      
      scenario3: {
        name: 'Update with skipWaiting',
        sequence: [
          'New SW version detected',
          'Install event fires',
          'skipWaiting() called',
          'Activate event fires immediately',
          'Old SW becomes redundant',
          'New SW takes control'
        ],
        note: 'Immediate activation, may break open tabs'
      },
      
      scenario4: {
        name: 'Update with user prompt',
        sequence: [
          'New SW version detected',
          'Install event fires',
          'SW waits in "waiting" state',
          'Show update banner to user',
          'User clicks "Update"',
          'Call skipWaiting()',
          'Activate event fires',
          'Reload page'
        ],
        note: 'User-controlled update flow'
      }
    };
  }

  // Detecting activation state
  static async checkActivationState() {
    const registration = await navigator.serviceWorker.ready;
    
    const state = {
      installing: !!registration.installing,
      waiting: !!registration.waiting,
      active: !!registration.active,
      controlled: !!navigator.serviceWorker.controller
    };
    
    console.log('Activation state:', state);
    return state;
  }
}
```

## Update Mechanism

### Detecting Updates

```javascript
// Detecting Service Worker updates
class ServiceWorkerUpdates {
  static async setupUpdateDetection() {
    const registration = await navigator.serviceWorker.register('/sw.js');

    // Check for updates periodically
    setInterval(() => {
      registration.update();
    }, 60 * 60 * 1000); // Check every hour

    // Listen for update found
    registration.addEventListener('updatefound', () => {
      const newWorker = registration.installing;
      console.log('New Service Worker found');

      newWorker.addEventListener('statechange', () => {
        console.log('New SW state:', newWorker.state);
        
        if (newWorker.state === 'installed' && navigator.serviceWorker.controller) {
          // New SW installed, old SW still active
          console.log('New version available!');
          this.showUpdatePrompt(registration);
        }
      });
    });

    return registration;
  }

  static showUpdatePrompt(registration) {
    // Show UI to user
    const banner = document.createElement('div');
    banner.innerHTML = `
      <div class="update-banner">
        <p>New version available!</p>
        <button id="update-button">Update</button>
        <button id="dismiss-button">Later</button>
      </div>
    `;
    document.body.appendChild(banner);

    document.getElementById('update-button').addEventListener('click', () => {
      if (registration.waiting) {
        // Tell waiting SW to skip waiting
        registration.waiting.postMessage({ type: 'SKIP_WAITING' });
      }
    });

    document.getElementById('dismiss-button').addEventListener('click', () => {
      banner.remove();
    });
  }

  // Manual update check
  static async checkForUpdates() {
    const registration = await navigator.serviceWorker.getRegistration();
    
    if (registration) {
      await registration.update();
      console.log('Update check complete');
    }
  }

  // Complete update flow
  static async setupCompleteUpdateFlow() {
    const registration = await navigator.serviceWorker.register('/sw.js');

    // Track update state
    let refreshing = false;

    // Reload page when new SW takes control
    navigator.serviceWorker.addEventListener('controllerchange', () => {
      if (refreshing) return;
      refreshing = true;
      window.location.reload();
    });

    // Check for updates
    registration.addEventListener('updatefound', () => {
      const newWorker = registration.installing;

      newWorker.addEventListener('statechange', () => {
        if (newWorker.state === 'installed' && navigator.serviceWorker.controller) {
          // Show update UI
          const updateBanner = document.createElement('div');
          updateBanner.className = 'update-available';
          updateBanner.innerHTML = `
            <p>New version available!</p>
            <button id="reload-button">Reload</button>
          `;
          document.body.appendChild(updateBanner);

          document.getElementById('reload-button').addEventListener('click', () => {
            newWorker.postMessage({ type: 'SKIP_WAITING' });
          });
        }
      });
    });

    // Periodic update checks
    setInterval(() => registration.update(), 60 * 60 * 1000);
    
    // Check on page focus
    document.addEventListener('visibilitychange', () => {
      if (!document.hidden) {
        registration.update();
      }
    });

    return registration;
  }
}

// Service Worker: Handling update messages
// File: sw.js
self.addEventListener('message', (event) => {
  if (event.data.type === 'SKIP_WAITING') {
    self.skipWaiting();
  }
});
```

### Update Triggers

```javascript
// What triggers Service Worker updates
class UpdateTriggers {
  static triggers() {
    return {
      navigation: {
        description: 'Browser checks for updates on navigation',
        timing: 'Every 24 hours',
        example: 'User navigates to different page on site'
      },
      
      manualCheck: {
        description: 'Calling registration.update()',
        timing: 'Immediate',
        example: 'registration.update()'
      },
      
      pushSync: {
        description: 'Push or sync events trigger update check',
        timing: 'On event',
        example: 'Push notification received'
      },
      
      fetchHandler: {
        description: 'Fetch events for SW script',
        timing: 'On SW script request',
        note: 'Browser bypasses HTTP cache for SW script'
      }
    };
  }

  // Update detection algorithm
  static updateDetection() {
    return {
      step1: 'Fetch /sw.js (bypass HTTP cache)',
      step2: 'Byte-by-byte comparison with current SW',
      step3: 'If different, trigger update',
      step4: 'Install new SW',
      step5: 'Wait for old SW to be unused',
      step6: 'Activate new SW',
      
      note: [
        'Even 1 byte difference triggers update',
        'Comments and whitespace count',
        'Use build hashes for versioning'
      ]
    };
  }

  // Cache busting for imports
  static importCacheBusting() {
    // Service Worker
    const VERSION = '1.2.3';
    
    // Import scripts with version
    self.importScripts(`/sw-utils.js?v=${VERSION}`);
    self.importScripts(`/sw-cache.js?v=${VERSION}`);
    
    // This ensures imported scripts are updated
  }
}
```

### Update Strategies

```javascript
// Different update strategies
class UpdateStrategies {
  // Strategy 1: Conservative (default)
  static conservativeUpdate() {
    // Service Worker
    self.addEventListener('install', (event) => {
      event.waitUntil(
        caches.open('my-app-v2')
          .then(cache => cache.addAll(urls))
      );
      // Don't call skipWaiting()
    });

    self.addEventListener('activate', (event) => {
      event.waitUntil(
        caches.keys()
          .then(names => Promise.all(
            names.filter(n => n !== 'my-app-v2')
              .map(n => caches.delete(n))
          ))
      );
    });

    // Client: Show update prompt
    // User must close all tabs or refresh
  }

  // Strategy 2: Aggressive (immediate)
  static aggressiveUpdate() {
    // Service Worker
    self.addEventListener('install', (event) => {
      event.waitUntil(
        caches.open('my-app-v2')
          .then(cache => cache.addAll(urls))
      );
      // Skip waiting immediately
      self.skipWaiting();
    });

    self.addEventListener('activate', (event) => {
      event.waitUntil(
        (async () => {
          // Clean up
          const names = await caches.keys();
          await Promise.all(
            names.filter(n => n !== 'my-app-v2')
              .map(n => caches.delete(n))
          );
          // Claim clients immediately
          await self.clients.claim();
        })()
      );
    });

    // Client: Auto-reload on controller change
    navigator.serviceWorker.addEventListener('controllerchange', () => {
      window.location.reload();
    });
  }

  // Strategy 3: User-prompted
  static userPromptedUpdate() {
    // Client
    async function setupUpdate() {
      const registration = await navigator.serviceWorker.register('/sw.js');

      registration.addEventListener('updatefound', () => {
        const newWorker = registration.installing;

        newWorker.addEventListener('statechange', () => {
          if (newWorker.state === 'installed' && navigator.serviceWorker.controller) {
            // Show prompt
            showUpdatePrompt(() => {
              newWorker.postMessage({ type: 'SKIP_WAITING' });
            });
          }
        });
      });

      // Reload when new SW takes over
      let refreshing = false;
      navigator.serviceWorker.addEventListener('controllerchange', () => {
        if (refreshing) return;
        refreshing = true;
        window.location.reload();
      });
    }

    // Service Worker
    self.addEventListener('message', (event) => {
      if (event.data.type === 'SKIP_WAITING') {
        self.skipWaiting();
      }
    });
  }

  // Strategy 4: Background update
  static backgroundUpdate() {
    // Client: Check for updates in background
    async function backgroundCheck() {
      const registration = await navigator.serviceWorker.getRegistration();
      
      if (registration) {
        // Check every 10 minutes
        setInterval(() => {
          registration.update();
        }, 10 * 60 * 1000);
        
        // Check when tab becomes visible
        document.addEventListener('visibilitychange', () => {
          if (!document.hidden) {
            registration.update();
          }
        });
      }
    }

    // Service Worker: Install silently
    self.addEventListener('install', (event) => {
      event.waitUntil(
        caches.open('my-app-v2')
          .then(cache => cache.addAll(urls))
      );
      // Don't skip waiting - wait for user to close tabs
    });
  }
}
```

## skipWaiting

### Understanding skipWaiting

```javascript
// skipWaiting() explained
class SkipWaitingExplanation {
  static defaultBehavior() {
    return {
      without: [
        'New SW installs',
        'New SW waits in "waiting" state',
        'Old SW continues controlling pages',
        'New SW activates only when all tabs closed',
        'User must close all tabs or wait'
      ],
      
      with: [
        'New SW installs',
        'skipWaiting() called',
        'New SW immediately activates',
        'Old SW becomes redundant',
        'New SW takes control'
      ]
    };
  }

  // When to use skipWaiting
  static useCases() {
    return {
      appropriate: [
        'Critical bug fixes',
        'Security patches',
        'User-initiated updates',
        'Major version changes with user confirmation'
      ],
      
      inappropriate: [
        'Regular updates',
        'Minor changes',
        'Without user notification',
        'When breaking changes possible'
      ]
    };
  }
}

// Using skipWaiting
class SkipWaitingUsage {
  // Method 1: Call in install event
  static installSkipWaiting() {
    // Service Worker
    self.addEventListener('install', (event) => {
      event.waitUntil(
        (async () => {
          const cache = await caches.open('my-app-v2');
          await cache.addAll(urls);
          
          // Skip waiting after successful install
          await self.skipWaiting();
        })()
      );
    });
  }

  // Method 2: Call on message
  static messageSkipWaiting() {
    // Service Worker
    self.addEventListener('message', (event) => {
      if (event.data.type === 'SKIP_WAITING') {
        self.skipWaiting();
      }
    });

    // Client
    async function skipWaiting() {
      const registration = await navigator.serviceWorker.getRegistration();
      
      if (registration?.waiting) {
        registration.waiting.postMessage({ type: 'SKIP_WAITING' });
      }
    }
  }

  // Method 3: Conditional skipWaiting
  static conditionalSkipWaiting() {
    // Service Worker
    const IS_CRITICAL_UPDATE = true; // Set based on version

    self.addEventListener('install', (event) => {
      event.waitUntil(
        (async () => {
          const cache = await caches.open('my-app-v2');
          await cache.addAll(urls);
          
          if (IS_CRITICAL_UPDATE) {
            // Critical updates skip waiting
            await self.skipWaiting();
          }
          // Non-critical updates wait normally
        })()
      );
    });
  }

  // Complete user-prompted flow
  static completeUserFlow() {
    // Client
    class UpdateManager {
      static async setup() {
        const registration = await navigator.serviceWorker.register('/sw.js');

        registration.addEventListener('updatefound', () => {
          const newWorker = registration.installing;
          
          newWorker.addEventListener('statechange', () => {
            if (newWorker.state === 'installed' && navigator.serviceWorker.controller) {
              this.promptUserForUpdate(newWorker);
            }
          });
        });

        // Handle controller change
        let refreshing = false;
        navigator.serviceWorker.addEventListener('controllerchange', () => {
          if (refreshing) return;
          refreshing = true;
          
          // Save scroll position, form data, etc.
          this.savePageState();
          
          window.location.reload();
        });
      }

      static promptUserForUpdate(newWorker) {
        const modal = document.createElement('div');
        modal.className = 'update-modal';
        modal.innerHTML = `
          <div class="modal-content">
            <h2>Update Available</h2>
            <p>A new version is available. Update now?</p>
            <button id="update-yes">Update</button>
            <button id="update-no">Later</button>
          </div>
        `;
        document.body.appendChild(modal);

        document.getElementById('update-yes').addEventListener('click', () => {
          newWorker.postMessage({ type: 'SKIP_WAITING' });
          modal.remove();
        });

        document.getElementById('update-no').addEventListener('click', () => {
          modal.remove();
          // Remind later
          setTimeout(() => this.promptUserForUpdate(newWorker), 60 * 60 * 1000);
        });
      }

      static savePageState() {
        const state = {
          scrollY: window.scrollY,
          formData: this.getFormData(),
          timestamp: Date.now()
        };
        sessionStorage.setItem('pageState', JSON.stringify(state));
      }

      static getFormData() {
        const forms = document.querySelectorAll('form');
        const data = {};
        
        forms.forEach(form => {
          const formData = new FormData(form);
          data[form.id] = Object.fromEntries(formData);
        });
        
        return data;
      }
    }

    UpdateManager.setup();
  }
}
```

### skipWaiting Pitfalls

```javascript
// Common skipWaiting problems
class SkipWaitingPitfalls {
  static pitfall1_BrokenState() {
    return {
      problem: 'Old page running with new SW',
      scenario: [
        'User has page open',
        'New SW installs and calls skipWaiting()',
        'New SW activates with new cache',
        'Old page tries to fetch from cache',
        'Cache structure changed = broken page'
      ],
      solution: 'Always reload page after skipWaiting()',
      code: `
        // Client
        navigator.serviceWorker.addEventListener('controllerchange', () => {
          window.location.reload();
        });
      `
    };
  }

  static pitfall2_DataLoss() {
    return {
      problem: 'User loses unsaved data on reload',
      scenario: [
        'User filling out form',
        'Update available',
        'User clicks update',
        'Page reloads immediately',
        'Form data lost'
      ],
      solution: 'Save state before reload',
      code: `
        // Save form data
        navigator.serviceWorker.addEventListener('controllerchange', () => {
          saveFormData();
          window.location.reload();
        });
      `
    };
  }

  static pitfall3_RaceCondition() {
    return {
      problem: 'Multiple tabs cause race conditions',
      scenario: [
        'User has 3 tabs open',
        'Update arrives',
        'All tabs call skipWaiting()',
        'Race condition on activation',
        'Inconsistent state across tabs'
      ],
      solution: 'Coordinate updates across tabs',
      code: `
        // Use BroadcastChannel
        const channel = new BroadcastChannel('sw-update');
        
        channel.addEventListener('message', (event) => {
          if (event.data === 'UPDATE_AVAILABLE') {
            // Only first tab handles update
            if (document.hidden) return;
            promptUpdate();
          }
        });
      `
    };
  }

  // Safe skipWaiting pattern
  static safePattern() {
    // Service Worker
    self.addEventListener('message', (event) => {
      if (event.data.type === 'SKIP_WAITING') {
        self.skipWaiting();
      }
    });

    self.addEventListener('activate', (event) => {
      event.waitUntil(
        (async () => {
          // Clean up old caches
          await cleanupCaches();
          
          // Claim all clients
          await self.clients.claim();
          
          // Notify all clients to reload
          const clients = await self.clients.matchAll();
          clients.forEach(client => {
            client.postMessage({
              type: 'SW_UPDATED',
              message: 'Please reload the page'
            });
          });
        })()
      );
    });

    // Client
    navigator.serviceWorker.addEventListener('message', (event) => {
      if (event.data.type === 'SW_UPDATED') {
        // Save state
        savePageState();
        
        // Show brief message
        showToast('Updating...');
        
        // Reload after short delay
        setTimeout(() => {
          window.location.reload();
        }, 1000);
      }
    });
  }
}
```

## clients.claim()

### Understanding clients.claim()

```javascript
// clients.claim() explained
class ClientsClaimExplanation {
  static defaultBehavior() {
    return {
      without: [
        'First page load: Not controlled by SW',
        'User must navigate to another page',
        'Second page load: Controlled by SW',
        'First page never controlled'
      ],
      
      with: [
        'First page load: Not controlled initially',
        'SW installs and activates',
        'clients.claim() called',
        'First page now controlled by SW',
        'All pages controlled immediately'
      ]
    };
  }

  static visualization() {
    return `
      Without clients.claim():
      Page Load -> SW Register -> SW Install -> SW Activate
                                                    |
                                              (Page not controlled)
                                                    |
                                          Next Navigation
                                                    |
                                          (Page now controlled)

      With clients.claim():
      Page Load -> SW Register -> SW Install -> SW Activate -> clients.claim()
                                                                       |
                                                              (Page controlled!)
    `;
  }
}

// Using clients.claim()
class ClientsClaimUsage {
  // Basic usage
  static basicClaim() {
    // Service Worker
    self.addEventListener('activate', (event) => {
      event.waitUntil(
        self.clients.claim()
      );
    });
  }

  // Complete activation with claim
  static completeActivation() {
    // Service Worker
    self.addEventListener('activate', (event) => {
      event.waitUntil(
        (async () => {
          // Clean up old caches
          const cacheNames = await caches.keys();
          await Promise.all(
            cacheNames
              .filter(name => name !== CACHE_NAME)
              .map(name => caches.delete(name))
          );
          
          // Take control of all clients
          await self.clients.claim();
          
          console.log('Activated and claimed clients');
        })()
      );
    });
  }

  // Claim with notification
  static claimWithNotification() {
    // Service Worker
    self.addEventListener('activate', (event) => {
      event.waitUntil(
        (async () => {
          await self.clients.claim();
          
          // Notify all clients that SW is now controlling them
          const clients = await self.clients.matchAll();
          clients.forEach(client => {
            client.postMessage({
              type: 'SW_CONTROLLING',
              message: 'Service Worker is now active'
            });
          });
        })()
      );
    });

    // Client
    navigator.serviceWorker.addEventListener('message', (event) => {
      if (event.data.type === 'SW_CONTROLLING') {
        console.log('SW took control:', event.data.message);
        
        // Refresh cached data
        refreshData();
      }
    });
  }

  // Conditional claim
  static conditionalClaim() {
    // Service Worker
    self.addEventListener('activate', (event) => {
      event.waitUntil(
        (async () => {
          // Check if this is first install or update
          const isUpdate = await self.clients.matchAll().then(
            clients => clients.length > 0
          );
          
          if (!isUpdate) {
            // First install: claim immediately
            await self.clients.claim();
            console.log('First install: claimed clients');
          } else {
            // Update: let user trigger reload
            console.log('Update: waiting for user reload');
          }
        })()
      );
    });
  }
}
```

### clients.claim() Use Cases

```javascript
// When to use clients.claim()
class ClientsClaimUseCases {
  static useCase1_FirstVisit() {
    return {
      scenario: 'User visits site for first time',
      problem: 'First page load not controlled by SW',
      solution: 'Use clients.claim() to control immediately',
      benefits: [
        'Offline works on first visit',
        'Consistent experience',
        'No need to navigate to get SW features'
      ],
      code: `
        self.addEventListener('activate', (event) => {
          event.waitUntil(self.clients.claim());
        });
      `
    };
  }

  static useCase2_OfflineFirst() {
    return {
      scenario: 'Offline-first application',
      problem: 'Need immediate offline capability',
      solution: 'Combine skipWaiting() and clients.claim()',
      code: `
        self.addEventListener('install', () => {
          self.skipWaiting();
        });

        self.addEventListener('activate', (event) => {
          event.waitUntil(self.clients.claim());
        });
      `
    };
  }

  static useCase3_CriticalUpdate() {
    return {
      scenario: 'Critical bug fix deployment',
      problem: 'Need immediate update across all tabs',
      solution: 'Use both skipWaiting() and clients.claim()',
      code: `
        // Service Worker
        const IS_CRITICAL = true;

        self.addEventListener('install', (event) => {
          if (IS_CRITICAL) {
            event.waitUntil(
              caches.open(CACHE_NAME)
                .then(cache => cache.addAll(urls))
                .then(() => self.skipWaiting())
            );
          }
        });

        self.addEventListener('activate', (event) => {
          if (IS_CRITICAL) {
            event.waitUntil(
              (async () => {
                await cleanupCaches();
                await self.clients.claim();
                
                // Force reload all clients
                const clients = await self.clients.matchAll();
                clients.forEach(client => {
                  client.postMessage({ type: 'FORCE_RELOAD' });
                });
              })()
            );
          }
        });

        // Client
        navigator.serviceWorker.addEventListener('message', (event) => {
          if (event.data.type === 'FORCE_RELOAD') {
            window.location.reload();
          }
        });
      `
    };
  }
}
```

### Combining skipWaiting and clients.claim

```javascript
// Best practices for combining both
class CombinedBestPractices {
  // Pattern 1: Aggressive update (use with caution)
  static aggressiveUpdate() {
    // Service Worker
    self.addEventListener('install', (event) => {
      console.log('Installing new version');
      event.waitUntil(
        caches.open('my-app-v2')
          .then(cache => cache.addAll(urls))
          .then(() => self.skipWaiting())
      );
    });

    self.addEventListener('activate', (event) => {
      console.log('Activating new version');
      event.waitUntil(
        (async () => {
          await cleanupOldCaches();
          await self.clients.claim();
          
          // Notify clients
          const clients = await self.clients.matchAll();
          clients.forEach(client => {
            client.postMessage({
              type: 'UPDATE_COMPLETE',
              version: 'v2'
            });
          });
        })()
      );
    });

    // Client: Handle update
    navigator.serviceWorker.addEventListener('message', (event) => {
      if (event.data.type === 'UPDATE_COMPLETE') {
        console.log('Updated to:', event.data.version);
        // Refresh UI, refetch data, etc.
      }
    });
  }

  // Pattern 2: Safe controlled update
  static safeControlledUpdate() {
    // Service Worker
    let UPDATE_APPROVED = false;

    self.addEventListener('message', (event) => {
      if (event.data.type === 'APPROVE_UPDATE') {
        UPDATE_APPROVED = true;
        self.skipWaiting();
      }
    });

    self.addEventListener('install', (event) => {
      event.waitUntil(
        caches.open('my-app-v2')
          .then(cache => cache.addAll(urls))
      );
      // Don't skip waiting automatically
    });

    self.addEventListener('activate', (event) => {
      event.waitUntil(
        (async () => {
          if (UPDATE_APPROVED) {
            await cleanupOldCaches();
            await self.clients.claim();
            console.log('Update approved and activated');
          }
        })()
      );
    });

    // Client: User approval flow
    async function promptForUpdate() {
      const registration = await navigator.serviceWorker.getRegistration();
      
      if (registration?.waiting) {
        const userApproved = confirm('Update available. Update now?');
        
        if (userApproved) {
          registration.waiting.postMessage({ type: 'APPROVE_UPDATE' });
          
          // Wait for controller change
          navigator.serviceWorker.addEventListener('controllerchange', () => {
            window.location.reload();
          });
        }
      }
    }
  }

  // Pattern 3: First-install only
  static firstInstallOnly() {
    // Service Worker
    self.addEventListener('install', (event) => {
      event.waitUntil(
        (async () => {
          const cache = await caches.open('my-app-v1');
          await cache.addAll(urls);
          
          // Check if there are existing clients
          const clients = await self.clients.matchAll();
          
          if (clients.length === 0) {
            // First install: skip waiting
            await self.skipWaiting();
          }
          // If clients exist, it's an update - wait normally
        })()
      );
    });

    self.addEventListener('activate', (event) => {
      event.waitUntil(
        (async () => {
          await cleanupOldCaches();
          
          // Always claim clients
          await self.clients.claim();
        })()
      );
    });
  }
}
```

## Common Misconceptions

### Misconception 1: "Service Worker caches everything automatically"

**Reality:** You must explicitly define what to cache in install event. Nothing is cached automatically.

```javascript
const reality = {
  automatic: 'Nothing',
  manual: 'Everything must be explicitly cached in install event or fetch handler',
  example: 'cache.addAll(urls) or cache.put(request, response)'
};
```

### Misconception 2: "skipWaiting() makes updates instant"

**Reality:** skipWaiting() only skips the waiting phase. The page must still reload to use the new SW.

```javascript
const timeline = {
  withoutSkipWaiting: 'Install -> Wait -> (close tabs) -> Activate',
  withSkipWaiting: 'Install -> Activate immediately',
  pageRefresh: 'Still required to use new SW version'
};
```

### Misconception 3: "clients.claim() updates existing requests"

**Reality:** clients.claim() only affects future requests. Current in-flight requests use the old SW or no SW.

```javascript
const behavior = {
  before: 'Requests go to network or old SW',
  claim: 'clients.claim() called',
  after: 'New requests go to new SW',
  current: 'In-flight requests unchanged'
};
```

### Misconception 4: "Service Worker updates check on every page load"

**Reality:** Updates check max once per 24 hours by default on navigation.

```javascript
const updateChecks = {
  navigation: 'Once per 24 hours',
  manual: 'registration.update() bypasses 24-hour limit',
  pushSync: 'Updates check on push/sync events',
  softReload: 'Ctrl+R uses cached SW script',
  hardReload: 'Ctrl+Shift+R always checks for updates'
};
```

## Performance Implications

### Lifecycle Performance Impact

```javascript
// Performance considerations
class LifecyclePerformance {
  static installPerformance() {
    return {
      cacheSize: {
        recommendation: '< 1MB for critical resources',
        reason: 'Large caches delay installation',
        impact: 'User waits before SW active'
      },
      
      numberOfRequests: {
        recommendation: '< 30 resources in initial cache',
        reason: 'Each request has overhead',
        strategy: 'Cache critical resources only, lazy-load others'
      },
      
      timing: {
        fast3G: '~2-5 seconds for 500KB',
        slow3G: '~10-30 seconds for 500KB',
        optimization: 'Show installation progress to user'
      }
    };
  }

  static activatePerformance() {
    return {
      cacheCleanup: {
        impact: 'Minimal if few old caches',
        recommendation: 'Delete old caches in activate',
        timing: '< 100ms typically'
      },
      
      claimClients: {
        impact: 'Very fast (< 10ms)',
        note: 'No network requests involved'
      }
    };
  }

  static updatePerformance() {
    return {
      detection: {
        impact: 'Single network request for SW script',
        timing: '~100-500ms',
        note: 'Happens in background'
      },
      
      installation: {
        impact: 'Same as initial install',
        timing: 'Depends on resources to cache',
        benefit: 'Happens in background while old SW serves'
      },
      
      activation: {
        impact: 'Minimal, only affects new navigations',
        userImpact: 'None if user doesn't refresh'
      }
    };
  }

  // Optimization strategies
  static optimizations() {
    return {
      opt1: {
        name: 'Minimize initial cache',
        code: `
          // Only cache critical resources
          const criticalResources = [
            '/',
            '/app.js',
            '/critical.css'
          ];
          
          self.addEventListener('install', (event) => {
            event.waitUntil(
              caches.open(CACHE_NAME)
                .then(cache => cache.addAll(criticalResources))
            );
          });
        `
      },
      
      opt2: {
        name: 'Progressive caching',
        code: `
          // Cache critical immediately, rest over time
          self.addEventListener('fetch', (event) => {
            event.respondWith(
              caches.match(event.request)
                .then(response => {
                  if (response) return response;
                  
                  return fetch(event.request).then(response => {
                    // Cache non-critical resources on demand
                    caches.open('dynamic-cache')
                      .then(cache => cache.put(event.request, response.clone()));
                    
                    return response;
                  });
                })
            );
          });
        `
      },
      
      opt3: {
        name: 'Background sync for updates',
        code: `
          // Client: Check updates in background
          setInterval(async () => {
            const registration = await navigator.serviceWorker.getRegistration();
            registration?.update();
          }, 60 * 60 * 1000); // Check every hour
        `
      }
    };
  }
}
```

## Browser Differences

### Service Worker Support

```javascript
const browserSupport = {
  chrome: {
    since: 'v40 (2015)',
    features: {
      basic: 'Full support',
      backgroundSync: 'v49+',
      pushAPI: 'v42+',
      navigationPreload: 'v59+',
      cacheAPI: 'Full support'
    },
    quirks: [
      'Max scope restriction can be overridden with header',
      'Update checks throttled to once per 24 hours'
    ]
  },
  
  firefox: {
    since: 'v44 (2016)',
    features: {
      basic: 'Full support',
      backgroundSync: 'Not supported',
      pushAPI: 'v44+',
      navigationPreload: 'v59+',
      cacheAPI: 'Full support'
    },
    quirks: [
      'Private browsing prevents SW registration',
      'Can disable in about:config'
    ]
  },
  
  safari: {
    since: 'v11.1 (2018)',
    features: {
      basic: 'Full support',
      backgroundSync: 'Not supported',
      pushAPI: 'v16+ (2022)',
      navigationPreload: 'v15.4+',
      cacheAPI: 'Full support'
    },
    quirks: [
      'Later adoption than other browsers',
      'Some limitations on iOS',
      'Push API only on iOS 16.4+'
    ]
  },
  
  edge: {
    legacy: 'v17+ (2017), limited support',
    chromium: 'v79+ (2020), same as Chrome',
    recommendation: 'Use Chromium Edge'
  }
};

// Feature detection
class ServiceWorkerFeatureDetection {
  static detectSupport() {
    return {
      serviceWorker: 'serviceWorker' in navigator,
      pushManager: 'PushManager' in window,
      syncManager: 'SyncManager' in window,
      backgroundFetch: 'BackgroundFetchManager' in window,
      navigationPreload: 'navigationPreload' in ServiceWorkerRegistration.prototype
    };
  }

  static async checkFeatures() {
    const features = this.detectSupport();
    
    console.log('Service Worker Support:');
    Object.entries(features).forEach(([feature, supported]) => {
      console.log(`  ${feature}: ${supported ? '✓' : '✗'}`);
    });
    
    return features;
  }
}
```

## Interview Questions

### Question 1: Explain the complete Service Worker lifecycle from registration to activation

**Answer:** The lifecycle has four main phases: 1) Registration - client calls navigator.serviceWorker.register('/sw.js'), browser downloads and parses SW script. 2) Installation - install event fires, SW caches critical resources. If event.waitUntil() promise rejects, installation fails and SW is discarded. 3) Waiting - installed SW waits for old SW to be unused (all tabs closed). 4) Activation - activate event fires, SW cleans up old caches, optionally calls clients.claim(). After activation, SW controls pages on next navigation. Key point: first page load isn't controlled unless clients.claim() is used.

### Question 2: What's the difference between skipWaiting() and clients.claim()?

**Answer:** skipWaiting() skips the waiting phase, allowing new SW to activate immediately even with open tabs using old SW. Without it, new SW waits until all tabs using old SW are closed. clients.claim() makes the SW take control of all uncontrolled pages immediately, including the first page load. Without it, SW only controls pages on subsequent navigations. They solve different problems: skipWaiting() is about when activation happens, clients.claim() is about which pages are controlled. First-install use case: call both in activate event for immediate offline capability. Update use case: be cautious with skipWaiting() as it can break open tabs - better to prompt user.

### Question 3: Why might a new Service Worker stay in the "waiting" state, and how do you handle it?

**Answer:** A new SW stays in waiting state when there are still pages controlled by the old SW. This prevents breaking open tabs with incompatible cache structures. Handle it by: 1) Default approach - let it wait, user closes tabs naturally, 2) Show update prompt - detect waiting state via updatefound event, show UI asking user to reload, call skipWaiting() when approved, 3) Background updates - periodically check for updates, prompt when user returns after closing tabs, 4) Critical updates - use skipWaiting() automatically but notify all tabs to reload, saving their state first. Always reload page after skipWaiting() via controllerchange event listener.

### Question 4: What happens if the install event fails?

**Answer:** If event.waitUntil() promise in install event rejects, the entire installation fails. The SW is discarded and marked as redundant. The old SW (if any) continues running. The failed SW never reaches waiting or active state. Common failure causes: network error fetching resources to cache, cache quota exceeded, syntax error in SW script. Best practices: only cache critical resources in install, handle cache.addAll() errors gracefully, cache optional resources without failing install (Promise.allSettled), keep total cache size reasonable (< 1MB critical resources). Use try-catch and proper error logging to diagnose failures.

### Question 5: How do Service Worker updates work, and what triggers them?

**Answer:** Update triggers: 1) Navigation to in-scope page (max once per 24 hours), 2) Push/sync events, 3) Manual registration.update() call, 4) First fetch event in 24 hours. Update algorithm: browser fetches /sw.js bypassing HTTP cache, performs byte-by-byte comparison with current SW. Even 1 byte difference (including comments) triggers update. New SW installs in background while old SW serves pages. After install, new SW waits (unless skipWaiting()). Activates when old SW unused or skipWaiting() called. Best practice: add version numbers/hashes to SW script, use importScripts with cache-busting for dependencies, test updates with DevTools "Update on reload", implement user-prompted update flow for better UX.

### Question 6: Why use clients.claim() and what are the risks?

**Answer:** Use clients.claim() to make SW control pages immediately, especially first page load. Benefits: offline works on first visit, consistent experience across all tabs, no navigation needed to activate SW features. Risks: SW and page may expect different resource versions, can break pages if cache structure changed, race conditions if page loads resources before claim completes. Safe patterns: only use on first install (check clients.matchAll().length === 0), always use with consistent cache strategy, notify clients after claiming so they can refresh data, combine with proper cache versioning. Critical for offline-first apps where immediate offline capability is required. Skip for update scenarios - let user control when to switch versions.

### Question 7: How do you implement a safe user-prompted update flow?

**Answer:** Proper flow: 1) Detect update - listen for updatefound event on registration, 2) Check state - when new SW reaches 'installed' state and controller exists, update available, 3) Show UI - display non-intrusive banner/toast with "Update" button, 4) User triggers - on click, send message to new SW to skipWaiting(), 5) Reload - listen for controllerchange event, save page state (scroll position, form data), reload page, 6) Restore state - after reload, restore saved state from sessionStorage. Key: never auto-reload without user consent, always save state before reload, handle case where user dismisses prompt (show again later), use BroadcastChannel to coordinate across tabs. This prevents data loss and gives user control.

### Question 8: What's the purpose of the activate event and what should you do in it?

**Answer:** Activate event fires after SW passes waiting state, before it starts controlling pages. Purpose: perform cleanup and initialization before SW starts handling fetches. Common tasks: 1) Delete old caches - caches.keys() then delete non-current caches, prevents storage bloat, 2) Call clients.claim() if needed - take immediate control, 3) Migrate data between cache versions if needed, 4) Initialize SW state, 5) Notify clients of activation. Don't do: long-running tasks (delays SW startup), fetch resources (do in install), complex operations that block activation. Pattern: use event.waitUntil() to keep SW alive during async cleanup, handle errors gracefully, log activation completion. Activation is one-time setup - keep it fast and focused on cleanup.

## Key Takeaways

1. **Registration Scope Critical**: SW scope defaults to its location. Can only control paths at or below its directory unless Service-Worker-Allowed header overrides. Scope determines which pages SW controls.

2. **Install for Critical Resources**: Install event is for caching critical resources only. Keep fast (< 1MB). If event.waitUntil() promise rejects, installation fails completely. Cache optional resources separately.

3. **Waiting State Purpose**: New SW waits for old SW to be unused (all tabs closed) to prevent breaking open tabs with incompatible versions. This is a feature, not a bug.

4. **skipWaiting() Use Sparingly**: Immediately activates new SW, can break open tabs. Only use for: critical fixes, first installs, user-prompted updates. Always reload page after skipWaiting() to prevent version mismatch.

5. **clients.claim() for Immediate Control**: Makes SW control current pages without navigation. Essential for offline-first first-visit experience. Risky for updates - may cause version conflicts.

6. **Activate for Cleanup**: Use activate event to delete old caches and initialize new version. Keep fast - blocking activation delays SW startup. Don't fetch resources in activate.

7. **Update Detection**: Browser checks for updates on navigation (max once per 24 hours), push/sync events, manual update() call. Byte-by-byte comparison - any change triggers update.

8. **First Load Not Controlled**: First page load isn't controlled by SW unless clients.claim() called in activate. Subsequent navigations are controlled. This surprises many developers.

9. **State Machine Complexity**: SW can be in installing, installed, activating, activated, or redundant states. Multiple SWs can exist simultaneously (old active, new waiting). Track state changes carefully.

10. **Lifecycle is Async**: All lifecycle events are async. Use event.waitUntil() to extend event lifetime for async operations. Event completes when promise settles. Handle errors or SW fails/becomes redundant.

## Resources

### Official Specifications
- **Service Worker Spec**: https://w3c.github.io/ServiceWorker/
- **MDN Service Worker API**: https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API
- **Service Worker Lifecycle**: https://developers.google.com/web/fundamentals/primers/service-workers/lifecycle

### Implementation Guides
- **Google Service Worker Guide**: https://developers.google.com/web/ilt/pwa/introduction-to-service-worker
- **Service Worker Cookbook**: https://serviceworke.rs/
- **Workbox** (Google's SW library): https://developers.google.com/web/tools/workbox

### Tools
- **Chrome DevTools**: Application tab -> Service Workers
- **Lighthouse**: PWA audit includes SW checks
- **Service Worker Toolbox**: https://github.com/GoogleChrome/sw-toolbox (deprecated but educational)

### Articles
- **Service Worker Lifecycle Explained**: https://web.dev/service-worker-lifecycle/
- **Understanding Service Worker**: https://javascript.info/service-workers
- **Progressive Web Apps**: https://web.dev/progressive-web-apps/

### Debugging
- **Chrome DevTools Service Worker Debug**: chrome://serviceworker-internals/
- **Firefox Service Worker Debug**: about:debugging#/runtime/this-firefox
- **Workbox Debugging**: https://developers.google.com/web/tools/workbox/guides/troubleshoot-and-debug
