# Browser Caching Strategies

## Table of Contents
- [Introduction](#introduction)
- [Cache-Control Headers](#cache-control-headers)
- [ETag and Last-Modified](#etag-and-last-modified)
- [Memory Cache vs Disk Cache](#memory-cache-vs-disk-cache)
- [Service Worker Cache API](#service-worker-cache-api)
- [Cache Invalidation Strategies](#cache-invalidation-strategies)
- [Common Misconceptions](#common-misconceptions)
- [Performance Implications](#performance-implications)
- [Browser Differences](#browser-differences)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Browser caching is one of the most effective performance optimizations available. Understanding how browsers cache resources, how to control caching behavior, and how to invalidate stale caches is crucial for building fast, reliable web applications. This guide covers HTTP caching, browser cache types, Service Worker caching, and practical invalidation strategies.

## Cache-Control Headers

### Basic Cache-Control Directives

```javascript
// Cache-Control directive examples
class CacheControlBasics {
  static directives() {
    return {
      // No caching
      noStore: {
        header: 'Cache-Control: no-store',
        meaning: 'Never cache this response',
        usage: 'Sensitive data, personal information',
        behavior: 'Browser always requests from server',
        example: 'User profile, banking data'
      },

      // No cache without validation
      noCache: {
        header: 'Cache-Control: no-cache',
        meaning: 'Cache but revalidate before use',
        usage: 'Dynamic content that changes',
        behavior: 'Browser caches but validates with server (ETag/Last-Modified)',
        example: 'News articles, social media feeds'
      },

      // Public caching
      public: {
        header: 'Cache-Control: public, max-age=31536000',
        meaning: 'Can be cached by browser and CDN',
        usage: 'Static assets, public resources',
        behavior: 'Shared caches (CDN) can store',
        example: 'CSS, JS, images with hashed filenames'
      },

      // Private caching
      private: {
        header: 'Cache-Control: private, max-age=3600',
        meaning: 'Only browser can cache, not CDN',
        usage: 'User-specific content',
        behavior: 'Browser caches but CDN cannot',
        example: 'Personalized HTML pages'
      },

      // Fresh for duration
      maxAge: {
        header: 'Cache-Control: max-age=3600',
        meaning: 'Fresh for 3600 seconds (1 hour)',
        usage: 'Content with known update frequency',
        behavior: 'Browser serves from cache without request',
        example: 'API responses, frequently updated assets'
      },

      // Revalidation
      mustRevalidate: {
        header: 'Cache-Control: max-age=3600, must-revalidate',
        meaning: 'Must revalidate after max-age expires',
        usage: 'Important to have fresh data',
        behavior: 'Cannot serve stale response',
        example: 'Financial data, inventory'
      },

      // Stale while revalidate
      staleWhileRevalidate: {
        header: 'Cache-Control: max-age=3600, stale-while-revalidate=86400',
        meaning: 'Serve stale up to 86400s while fetching fresh',
        usage: 'Balance freshness and performance',
        behavior: 'Returns stale immediately, updates in background',
        example: 'Social media content, news feeds'
      },

      // Immutable
      immutable: {
        header: 'Cache-Control: public, max-age=31536000, immutable',
        meaning: 'Resource never changes',
        usage: 'Content-hashed assets',
        behavior: 'Skip revalidation even on reload',
        example: 'app.abc123.js, style.def456.css'
      }
    };
  }

  // Server-side implementation examples
  static serverExamples() {
    return {
      // Express.js
      express: `
        const express = require('express');
        const app = express();

        // Static assets (immutable)
        app.use('/static', express.static('public', {
          maxAge: '1y',
          immutable: true
        }));

        // HTML pages (revalidate)
        app.get('/', (req, res) => {
          res.set('Cache-Control', 'no-cache');
          res.send(html);
        });

        // API responses (short-lived)
        app.get('/api/data', (req, res) => {
          res.set('Cache-Control', 'public, max-age=60');
          res.json(data);
        });

        // User-specific data
        app.get('/api/profile', (req, res) => {
          res.set('Cache-Control', 'private, max-age=300');
          res.json(profile);
        });
      `,

      // Nginx
      nginx: `
        # Static assets
        location /static/ {
          expires 1y;
          add_header Cache-Control "public, immutable";
        }

        # HTML pages
        location / {
          add_header Cache-Control "no-cache";
        }

        # API responses
        location /api/ {
          add_header Cache-Control "public, max-age=60";
        }
      `,

      // Apache
      apache: `
        # .htaccess
        <IfModule mod_headers.c>
          # Static assets
          <FilesMatch "\\.(css|js|jpg|png|woff2)$">
            Header set Cache-Control "public, max-age=31536000, immutable"
          </FilesMatch>

          # HTML
          <FilesMatch "\\.html$">
            Header set Cache-Control "no-cache"
          </FilesMatch>
        </IfModule>
      `
    };
  }
}

// Practical caching strategies
class CachingStrategies {
  // Strategy 1: Content-based (fingerprinting)
  static contentBased() {
    return {
      concept: 'Embed content hash in filename',
      pattern: 'app.[contenthash].js',
      cacheHeader: 'Cache-Control: public, max-age=31536000, immutable',
      
      benefits: [
        'Cache forever (1 year)',
        'Automatic invalidation on change',
        'No revalidation needed',
        'Perfect cache hit rate'
      ],
      
      implementation: `
        // Webpack configuration
        module.exports = {
          output: {
            filename: '[name].[contenthash].js',
            path: path.resolve(__dirname, 'dist')
          }
        };

        // Generated files:
        // app.abc123def456.js
        // vendor.789ghi012jkl.js
        
        // HTML automatically updated:
        // <script src="/app.abc123def456.js"></script>
      `
    };
  }

  // Strategy 2: Time-based (TTL)
  static timeBased() {
    return {
      concept: 'Cache for specific duration',
      
      short: {
        duration: '60-300 seconds',
        usage: 'API responses, frequently changing data',
        header: 'Cache-Control: public, max-age=60'
      },
      
      medium: {
        duration: '1-24 hours',
        usage: 'Semi-static content, daily updates',
        header: 'Cache-Control: public, max-age=3600'
      },
      
      long: {
        duration: '1 week - 1 year',
        usage: 'Static assets, rarely changing',
        header: 'Cache-Control: public, max-age=604800'
      }
    };
  }

  // Strategy 3: Stale-while-revalidate
  static staleWhileRevalidate() {
    return {
      concept: 'Serve stale content while fetching fresh',
      header: 'Cache-Control: max-age=60, stale-while-revalidate=3600',
      
      behavior: [
        'First 60s: Serve from cache (fresh)',
        '60s - 1h: Serve from cache (stale), fetch new in background',
        '> 1h: Fetch from network'
      ],
      
      benefits: [
        'Instant response from cache',
        'Background update for next request',
        'Resilient to slow networks'
      ],
      
      useCase: 'News articles, social feeds, comments',
      
      implementation: `
        // Server
        app.get('/api/feed', (req, res) => {
          res.set('Cache-Control', 'max-age=60, stale-while-revalidate=3600');
          res.json(getFeed());
        });

        // Browser behavior:
        // t=0s: Request -> Server (200, cached)
        // t=30s: Request -> Cache (instant)
        // t=90s: Request -> Cache (instant) + Background fetch
        // t=91s: Request -> Cache (now fresh from background fetch)
      `
    };
  }

  // Strategy 4: Network-first with fallback
  static networkFirst() {
    return {
      concept: 'Try network, fallback to cache',
      header: 'Cache-Control: no-cache',
      
      behavior: [
        'Always request from network',
        'If network fails, use cache',
        'Always revalidate'
      ],
      
      useCase: 'Critical data that must be fresh',
      
      implementation: `
        // Service Worker
        self.addEventListener('fetch', (event) => {
          event.respondWith(
            fetch(event.request)
              .then(response => {
                // Cache the response
                const cache = caches.open('dynamic');
                cache.put(event.request, response.clone());
                return response;
              })
              .catch(() => {
                // Network failed, try cache
                return caches.match(event.request);
              })
          );
        });
      `
    };
  }
}
```

### Advanced Cache-Control Patterns

```javascript
// Complex caching scenarios
class AdvancedCaching {
  // Pattern 1: Vary by request headers
  static varyHeader() {
    return {
      header: 'Vary: Accept-Encoding, Accept-Language',
      meaning: 'Cache varies by these request headers',
      
      example: {
        request1: {
          headers: { 'Accept-Encoding': 'gzip' },
          cached: 'response-gzip'
        },
        request2: {
          headers: { 'Accept-Encoding': 'br' },
          cached: 'response-brotli'
        }
      },
      
      implementation: `
        // Server
        app.get('/api/data', (req, res) => {
          res.set('Cache-Control', 'public, max-age=3600');
          res.set('Vary', 'Accept-Language');
          
          const lang = req.headers['accept-language'];
          res.json(getData(lang));
        });
      `,
      
      warning: 'Vary: * prevents caching entirely'
    };
  }

  // Pattern 2: Conditional requests
  static conditionalRequests() {
    return {
      concept: 'Cache but always revalidate',
      headers: {
        request: 'Cache-Control: no-cache',
        validation: 'ETag or Last-Modified'
      },
      
      flow: [
        'Browser: GET /data (If-None-Match: "abc123")',
        'Server: 304 Not Modified (if unchanged)',
        'Browser: Use cached response',
        'OR',
        'Server: 200 OK with new data (if changed)'
      ],
      
      benefits: [
        'Always fresh data',
        'Save bandwidth on 304',
        'Faster than full download'
      ]
    };
  }

  // Pattern 3: Surrogate-Control (CDN)
  static surrogateControl() {
    return {
      concept: 'Separate cache rules for CDN and browser',
      
      headers: {
        cdnOnly: 'Surrogate-Control: max-age=3600',
        browserOnly: 'Cache-Control: private, max-age=0',
        both: 'Surrogate-Control: max-age=3600\nCache-Control: public, max-age=60'
      },
      
      example: `
        // Long CDN cache, short browser cache
        app.get('/api/data', (req, res) => {
          res.set('Surrogate-Control', 'max-age=86400'); // CDN: 24h
          res.set('Cache-Control', 'public, max-age=300'); // Browser: 5m
          res.json(data);
        });
      `,
      
      useCase: 'CDN-hosted APIs with frequent browser updates'
    };
  }

  // Pattern 4: Cache key customization
  static cacheKeys() {
    return {
      concept: 'Control what makes responses unique',
      
      standard: {
        key: 'URL + HTTP method',
        example: 'GET https://example.com/api/users'
      },
      
      withVary: {
        key: 'URL + Method + Vary headers',
        example: 'GET https://example.com/api/users + Accept-Language: en'
      },
      
      custom: {
        description: 'Service Worker can use custom keys',
        example: `
          // Cache by URL without query params
          const cacheKey = new URL(request.url).pathname;
          caches.match(cacheKey);
          
          // Cache by custom identifier
          const cacheKey = \`user-\${userId}-data\`;
          caches.match(cacheKey);
        `
      }
    };
  }
}
```

## ETag and Last-Modified

### ETag (Entity Tag)

```javascript
// ETag implementation and usage
class ETagImplementation {
  static basics() {
    return {
      concept: 'Unique identifier for resource version',
      format: 'ETag: "abc123def456" or W/"abc123" (weak)',
      
      strongETag: {
        format: 'ETag: "abc123"',
        meaning: 'Byte-for-byte identical',
        usage: 'Content-based hash'
      },
      
      weakETag: {
        format: 'ETag: W/"abc123"',
        meaning: 'Semantically equivalent',
        usage: 'Gzipped content, minor changes acceptable'
      }
    };
  }

  // Server-side ETag generation
  static generateETag() {
    const crypto = require('crypto');

    // Content-based ETag
    function contentETag(content) {
      const hash = crypto
        .createHash('sha256')
        .update(content)
        .digest('hex')
        .substring(0, 16);
      
      return `"${hash}"`;
    }

    // Timestamp-based ETag
    function timestampETag(lastModified) {
      const timestamp = lastModified.getTime();
      return `"${timestamp}"`;
    }

    // Weak ETag for dynamic content
    function weakETag(content) {
      const hash = crypto
        .createHash('md5')
        .update(content)
        .digest('hex')
        .substring(0, 8);
      
      return `W/"${hash}"`;
    }

    return { contentETag, timestampETag, weakETag };
  }

  // Express implementation
  static expressETag() {
    return `
      const express = require('express');
      const crypto = require('crypto');
      const app = express();

      // Enable ETag for all responses
      app.enable('etag');

      // Custom ETag generator
      app.set('etag', (body, encoding) => {
        const hash = crypto
          .createHash('sha256')
          .update(body, encoding)
          .digest('hex')
          .substring(0, 16);
        return \`"\${hash}"\`;
      });

      // Manual ETag handling
      app.get('/api/data', (req, res) => {
        const data = getData();
        const etag = generateETag(JSON.stringify(data));
        
        // Check If-None-Match
        if (req.headers['if-none-match'] === etag) {
          res.status(304).end(); // Not Modified
          return;
        }
        
        // Send with ETag
        res.set('ETag', etag);
        res.set('Cache-Control', 'no-cache');
        res.json(data);
      });

      function generateETag(content) {
        return crypto
          .createHash('sha256')
          .update(content)
          .digest('hex')
          .substring(0, 16);
      }
    `;
  }

  // Conditional request flow
  static conditionalFlow() {
    return {
      firstRequest: {
        client: 'GET /api/data',
        server: '200 OK\nETag: "abc123"\n{data}',
        cached: 'Response with ETag stored'
      },
      
      subsequentRequest: {
        client: 'GET /api/data\nIf-None-Match: "abc123"',
        server: '304 Not Modified (unchanged)\nOR\n200 OK with new ETag (changed)',
        benefit: '304 = no body sent, saves bandwidth'
      },
      
      savings: {
        fullResponse: '10 KB',
        notModified: '~200 bytes (headers only)',
        savings: '98% bandwidth reduction'
      }
    };
  }
}

// Last-Modified implementation
class LastModifiedImplementation {
  static basics() {
    return {
      concept: 'Timestamp of last modification',
      format: 'Last-Modified: Wed, 21 Oct 2025 07:28:00 GMT',
      validation: 'If-Modified-Since header',
      
      vsETag: {
        lastModified: 'Second precision, may miss rapid changes',
        etag: 'Version-based, catches all changes',
        recommendation: 'Use both when possible'
      }
    };
  }

  // Express implementation
  static expressLastModified() {
    return `
      app.get('/api/data', (req, res) => {
        const data = getData();
        const lastModified = new Date(data.updatedAt);
        
        // Check If-Modified-Since
        const ifModifiedSince = req.headers['if-modified-since'];
        if (ifModifiedSince) {
          const modifiedSince = new Date(ifModifiedSince);
          if (lastModified <= modifiedSince) {
            res.status(304).end(); // Not Modified
            return;
          }
        }
        
        // Send with Last-Modified
        res.set('Last-Modified', lastModified.toUTCString());
        res.set('Cache-Control', 'no-cache');
        res.json(data);
      });
    `;
  }

  // Combined ETag + Last-Modified
  static combined() {
    return `
      app.get('/api/data', (req, res) => {
        const data = getData();
        const etag = generateETag(JSON.stringify(data));
        const lastModified = new Date(data.updatedAt);
        
        // Check both validators
        const ifNoneMatch = req.headers['if-none-match'];
        const ifModifiedSince = req.headers['if-modified-since'];
        
        let notModified = false;
        
        // ETag takes precedence
        if (ifNoneMatch) {
          notModified = (ifNoneMatch === etag);
        } else if (ifModifiedSince) {
          notModified = (lastModified <= new Date(ifModifiedSince));
        }
        
        if (notModified) {
          res.status(304).end();
          return;
        }
        
        // Send with both validators
        res.set('ETag', etag);
        res.set('Last-Modified', lastModified.toUTCString());
        res.set('Cache-Control', 'no-cache');
        res.json(data);
      });
    `;
  }
}
```

## Memory Cache vs Disk Cache

### Understanding Browser Cache Types

```javascript
// Browser cache hierarchy
class BrowserCacheTypes {
  static hierarchy() {
    return {
      memoryCache: {
        location: 'RAM',
        speed: 'Fastest (~1ms)',
        size: 'Small (~50-100MB)',
        persistence: 'Session only (cleared on tab close)',
        priority: 'Checked first',
        
        cached: [
          'Recently accessed resources',
          'Images currently visible',
          'Scripts for active page',
          'Preloaded resources'
        ],
        
        notCached: [
          'Very large resources (>10MB)',
          'Resources with no-store',
          'Cross-origin resources (sometimes)'
        ]
      },
      
      diskCache: {
        location: 'Hard drive/SSD',
        speed: 'Fast (~5-50ms)',
        size: 'Large (~100MB - 1GB+)',
        persistence: 'Survives browser restart',
        priority: 'Checked after memory cache',
        
        cached: [
          'All cacheable resources',
          'Evicted from memory cache',
          'Large files',
          'Persistent cache'
        ]
      },
      
      serviceWorkerCache: {
        location: 'IndexedDB (disk)',
        speed: 'Fast (~5-50ms)',
        size: 'Large (50% available disk space)',
        persistence: 'Permanent until deleted',
        priority: 'Checked before HTTP cache',
        
        cached: [
          'Explicitly cached resources',
          'Offline support files',
          'App shell',
          'API responses (if cached)'
        ]
      },
      
      httpCache: {
        description: 'General term for memory + disk cache',
        controlled: 'Cache-Control headers',
        automatic: 'Browser decides memory vs disk'
      }
    };
  }

  // Cache lookup order
  static lookupOrder() {
    return {
      step1: 'Service Worker (if registered and controlling)',
      step2: 'Memory Cache (if present)',
      step3: 'Disk Cache (if present and fresh)',
      step4: 'Network (fetch from server)',
      
      visualization: `
        Request -> SW? -> Memory? -> Disk? -> Network
                   ↓       ↓        ↓         ↓
                  Found   Found    Found    Fetch
      `
    };
  }

  // Identifying cache type in DevTools
  static devToolsIdentification() {
    return {
      chrome: {
        memoryCache: 'Size column shows "(memory cache)"',
        diskCache: 'Size column shows "(disk cache)"',
        serviceWorker: 'Size column shows "(ServiceWorker)"',
        
        example: `
          Name                  Status  Size
          app.js                200     (memory cache)
          styles.css            200     (disk cache)
          api/data              200     (ServiceWorker)
          logo.png              200     1.2 KB
        `
      },
      
      disableCache: {
        softReload: 'Ctrl+R: Uses memory cache',
        hardReload: 'Ctrl+Shift+R: Bypasses all caches',
        devToolsDisable: 'Network tab: "Disable cache" checkbox'
      }
    };
  }
}

// Memory cache specifics
class MemoryCacheSpecifics {
  static behavior() {
    return {
      population: [
        'First load: Resource fetched, stored in memory',
        'Same page: Served from memory (instant)',
        'New tab: May share memory cache (same origin)',
        'Tab close: Memory cache cleared'
      ],
      
      optimization: [
        'Preload hints populate memory cache',
        'Recently viewed images stay in memory',
        'LRU eviction when full',
        'Cleared on memory pressure'
      ],
      
      forcedStorage: `
        <!-- Preload to force memory cache -->
        <link rel="preload" href="/styles.css" as="style">
        <link rel="preload" href="/logo.png" as="image">
        
        <!-- Resources loaded into memory immediately -->
      `
    };
  }

  // Memory cache limits
  static limits() {
    return {
      chrome: {
        total: '~100MB per origin',
        perResource: '~10MB max',
        behavior: 'LRU eviction when full'
      },
      
      firefox: {
        total: '~50MB',
        behavior: 'Clears on memory pressure'
      },
      
      safari: {
        total: 'Variable based on system memory',
        behavior: 'Aggressive clearing'
      }
    };
  }
}

// Disk cache specifics
class DiskCacheSpecifics {
  static behavior() {
    return {
      storage: [
        'All cacheable HTTP responses',
        'Survives browser restart',
        'Shared across tabs/windows',
        'Quota-managed by browser'
      ],
      
      eviction: [
        'LRU (Least Recently Used)',
        'Quota exceeded',
        'User clears cache',
        'Origin deleted'
      ],
      
      location: {
        chrome: {
          windows: '%LocalAppData%\\Google\\Chrome\\User Data\\Default\\Cache',
          mac: '~/Library/Caches/Google/Chrome/Default/Cache',
          linux: '~/.cache/google-chrome/Default/Cache'
        },
        firefox: {
          windows: '%AppData%\\Local\\Mozilla\\Firefox\\Profiles\\xxx.default\\cache2',
          mac: '~/Library/Caches/Firefox/Profiles/xxx.default/cache2',
          linux: '~/.cache/mozilla/firefox/xxx.default/cache2'
        }
      }
    };
  }

  static quotas() {
    return {
      chrome: {
        default: '~80% of available disk space',
        perOrigin: '~20% of total quota',
        minimum: '1GB'
      },
      
      firefox: {
        default: '~50% of available disk space',
        perOrigin: '~10% of total quota'
      },
      
      checking: `
        // Check quota usage
        if (navigator.storage && navigator.storage.estimate) {
          navigator.storage.estimate().then(estimate => {
            console.log('Usage:', estimate.usage);
            console.log('Quota:', estimate.quota);
            console.log('Percentage:', 
              (estimate.usage / estimate.quota * 100).toFixed(2) + '%');
          });
        }
      `
    };
  }
}
```

## Service Worker Cache API

### Basic Cache API Usage

```javascript
// Service Worker Cache API
class CacheAPIBasics {
  // Opening caches
  static async openCache() {
    // Open or create cache
    const cache = await caches.open('my-cache-v1');
    
    // Cache name is just a string - you control versioning
    return cache;
  }

  // Adding resources to cache
  static async addToCache() {
    const cache = await caches.open('my-cache-v1');
    
    // Add single URL
    await cache.add('/styles.css');
    
    // Add multiple URLs
    await cache.addAll([
      '/',
      '/app.js',
      '/styles.css',
      '/logo.png'
    ]);
    
    // Add with custom request/response
    const response = await fetch('/api/data');
    await cache.put('/api/data', response);
  }

  // Retrieving from cache
  static async getFromCache() {
    const cache = await caches.open('my-cache-v1');
    
    // Match specific request
    const response = await cache.match('/styles.css');
    
    if (response) {
      console.log('Found in cache');
      return response;
    } else {
      console.log('Not in cache');
      return null;
    }
  }

  // Deleting from cache
  static async deleteFromCache() {
    const cache = await caches.open('my-cache-v1');
    
    // Delete specific request
    const deleted = await cache.delete('/old-file.css');
    console.log('Deleted:', deleted);
    
    // Delete entire cache
    await caches.delete('my-cache-v1');
  }

  // Listing cache contents
  static async listCacheContents() {
    const cache = await caches.open('my-cache-v1');
    
    // Get all cached requests
    const requests = await cache.keys();
    
    console.log('Cached resources:');
    requests.forEach(request => {
      console.log('  ', request.url);
    });
    
    return requests;
  }

  // Listing all caches
  static async listAllCaches() {
    const cacheNames = await caches.keys();
    
    console.log('All caches:');
    cacheNames.forEach(name => {
      console.log('  ', name);
    });
    
    return cacheNames;
  }
}

// Advanced Cache API patterns
class AdvancedCacheAPI {
  // Cache-first strategy
  static cacheFirst() {
    return `
      self.addEventListener('fetch', (event) => {
        event.respondWith(
          caches.match(event.request)
            .then(response => {
              // Return cached response or fetch from network
              return response || fetch(event.request);
            })
        );
      });
    `;
  }

  // Network-first strategy
  static networkFirst() {
    return `
      self.addEventListener('fetch', (event) => {
        event.respondWith(
          fetch(event.request)
            .then(response => {
              // Cache the fresh response
              const cache = caches.open('dynamic-v1');
              cache.then(c => c.put(event.request, response.clone()));
              return response;
            })
            .catch(() => {
              // Network failed, try cache
              return caches.match(event.request);
            })
        );
      });
    `;
  }

  // Stale-while-revalidate
  static staleWhileRevalidate() {
    return `
      self.addEventListener('fetch', (event) => {
        event.respondWith(
          caches.match(event.request)
            .then(cachedResponse => {
              const fetchPromise = fetch(event.request)
                .then(networkResponse => {
                  // Update cache in background
                  caches.open('dynamic-v1')
                    .then(cache => cache.put(event.request, networkResponse.clone()));
                  return networkResponse;
                });
              
              // Return cached response immediately, or wait for network
              return cachedResponse || fetchPromise;
            })
        );
      });
    `;
  }

  // Cache with timeout
  static cacheWithTimeout() {
    return `
      self.addEventListener('fetch', (event) => {
        event.respondWith(
          Promise.race([
            fetch(event.request),
            new Promise((resolve, reject) => {
              setTimeout(() => reject(new Error('Timeout')), 3000);
            })
          ])
          .then(response => {
            // Network succeeded, cache it
            const cache = caches.open('dynamic-v1');
            cache.then(c => c.put(event.request, response.clone()));
            return response;
          })
          .catch(() => {
            // Network failed or timeout, use cache
            return caches.match(event.request);
          })
        );
      });
    `;
  }

  // Selective caching
  static selectiveCaching() {
    return `
      self.addEventListener('fetch', (event) => {
        const url = new URL(event.request.url);
        
        // Only cache same-origin requests
        if (url.origin !== location.origin) {
          event.respondWith(fetch(event.request));
          return;
        }
        
        // Don't cache API requests
        if (url.pathname.startsWith('/api/')) {
          event.respondWith(fetch(event.request));
          return;
        }
        
        // Cache static assets
        event.respondWith(
          caches.match(event.request)
            .then(response => response || fetch(event.request))
        );
      });
    `;
  }

  // Cache expiration
  static cacheExpiration() {
    return `
      const CACHE_NAME = 'my-cache-v1';
      const MAX_AGE = 7 * 24 * 60 * 60 * 1000; // 7 days
      
      self.addEventListener('fetch', (event) => {
        event.respondWith(
          caches.open(CACHE_NAME).then(async (cache) => {
            const cachedResponse = await cache.match(event.request);
            
            if (cachedResponse) {
              const cachedDate = new Date(cachedResponse.headers.get('date'));
              const age = Date.now() - cachedDate.getTime();
              
              if (age < MAX_AGE) {
                return cachedResponse; // Still fresh
              }
            }
            
            // Expired or not cached, fetch fresh
            const response = await fetch(event.request);
            cache.put(event.request, response.clone());
            return response;
          })
        );
      });
    `;
  }

  // Cache size management
  static cacheSizeManagement() {
    return `
      const CACHE_NAME = 'images-v1';
      const MAX_ITEMS = 50;
      
      async function trimCache(cacheName, maxItems) {
        const cache = await caches.open(cacheName);
        const keys = await cache.keys();
        
        if (keys.length > maxItems) {
          // Delete oldest items
          const toDelete = keys.slice(0, keys.length - maxItems);
          await Promise.all(toDelete.map(key => cache.delete(key)));
        }
      }
      
      self.addEventListener('fetch', (event) => {
        if (event.request.url.match(/\\.(jpg|png|gif|svg)$/)) {
          event.respondWith(
            caches.open(CACHE_NAME).then(async (cache) => {
              const response = await fetch(event.request);
              cache.put(event.request, response.clone());
              
              // Trim cache to max items
              trimCache(CACHE_NAME, MAX_ITEMS);
              
              return response;
            })
          );
        }
      });
    `;
  }
}
```

### Complete Caching Strategies

```javascript
// Real-world caching strategies
class CachingPatterns {
  // Pattern 1: App Shell pattern
  static appShell() {
    return `
      const CACHE_NAME = 'app-shell-v1';
      const APP_SHELL = [
        '/',
        '/app.js',
        '/styles.css',
        '/icons/logo.svg'
      ];
      
      // Install: Cache app shell
      self.addEventListener('install', (event) => {
        event.waitUntil(
          caches.open(CACHE_NAME)
            .then(cache => cache.addAll(APP_SHELL))
        );
      });
      
      // Fetch: Serve app shell from cache, content from network
      self.addEventListener('fetch', (event) => {
        const url = new URL(event.request.url);
        
        // App shell: Cache-first
        if (APP_SHELL.includes(url.pathname)) {
          event.respondWith(
            caches.match(event.request)
              .then(response => response || fetch(event.request))
          );
          return;
        }
        
        // Content: Network-first
        event.respondWith(
          fetch(event.request)
            .catch(() => caches.match(event.request))
        );
      });
    `;
  }

  // Pattern 2: Multi-cache strategy
  static multiCache() {
    return `
      const CACHES = {
        static: 'static-v1',
        dynamic: 'dynamic-v1',
        images: 'images-v1'
      };
      
      self.addEventListener('install', (event) => {
        event.waitUntil(
          caches.open(CACHES.static).then(cache => {
            return cache.addAll([
              '/',
              '/app.js',
              '/styles.css'
            ]);
          })
        );
      });
      
      self.addEventListener('fetch', (event) => {
        const url = new URL(event.request.url);
        
        // Images: Cache-first with size limit
        if (url.pathname.match(/\\.(jpg|png|gif|svg)$/)) {
          event.respondWith(handleImages(event.request));
          return;
        }
        
        // API: Network-first with cache fallback
        if (url.pathname.startsWith('/api/')) {
          event.respondWith(handleAPI(event.request));
          return;
        }
        
        // Static assets: Cache-first
        event.respondWith(handleStatic(event.request));
      });
      
      async function handleImages(request) {
        const cache = await caches.open(CACHES.images);
        const cached = await cache.match(request);
        
        if (cached) return cached;
        
        const response = await fetch(request);
        cache.put(request, response.clone());
        
        // Limit image cache size
        const keys = await cache.keys();
        if (keys.length > 100) {
          await cache.delete(keys[0]);
        }
        
        return response;
      }
      
      async function handleAPI(request) {
        try {
          const response = await fetch(request);
          const cache = await caches.open(CACHES.dynamic);
          cache.put(request, response.clone());
          return response;
        } catch {
          return caches.match(request);
        }
      }
      
      async function handleStatic(request) {
        const cached = await caches.match(request);
        return cached || fetch(request);
      }
    `;
  }

  // Pattern 3: Background sync with cache
  static backgroundSync() {
    return `
      // Service Worker
      const QUEUE_NAME = 'api-queue';
      
      self.addEventListener('fetch', (event) => {
        if (event.request.method === 'POST') {
          event.respondWith(handlePost(event.request));
        }
      });
      
      async function handlePost(request) {
        try {
          return await fetch(request);
        } catch {
          // Offline: Queue request
          const cache = await caches.open(QUEUE_NAME);
          const cloned = request.clone();
          await cache.put(\`queue-\${Date.now()}\`, cloned);
          
          // Register background sync
          await self.registration.sync.register('sync-queue');
          
          return new Response(JSON.stringify({ queued: true }), {
            headers: { 'Content-Type': 'application/json' }
          });
        }
      }
      
      self.addEventListener('sync', (event) => {
        if (event.tag === 'sync-queue') {
          event.waitUntil(syncQueue());
        }
      });
      
      async function syncQueue() {
        const cache = await caches.open(QUEUE_NAME);
        const requests = await cache.keys();
        
        for (const request of requests) {
          try {
            const cached = await cache.match(request);
            await fetch(cached);
            await cache.delete(request);
          } catch {
            // Keep in queue, will retry next sync
          }
        }
      }
    `;
  }
}
```

## Cache Invalidation Strategies

### Invalidation Techniques

```javascript
// Cache invalidation approaches
class CacheInvalidation {
  // Strategy 1: Versioned cache names
  static versionedCacheNames() {
    return `
      const VERSION = '1.2.3';
      const CACHE_NAME = \`my-app-\${VERSION}\`;
      
      // Install: Create new cache
      self.addEventListener('install', (event) => {
        event.waitUntil(
          caches.open(CACHE_NAME)
            .then(cache => cache.addAll(urls))
        );
      });
      
      // Activate: Delete old caches
      self.addEventListener('activate', (event) => {
        event.waitUntil(
          caches.keys().then(names => {
            return Promise.all(
              names.filter(name => name !== CACHE_NAME)
                .map(name => caches.delete(name))
            );
          })
        );
      });
      
      // Result: Automatic invalidation on version change
    `;
  }

  // Strategy 2: Content-based (hash in filename)
  static contentBasedInvalidation() {
    return {
      concept: 'Filename includes content hash',
      example: 'app.abc123.js, styles.def456.css',
      
      build: `
        // Webpack generates:
        // app.abc123.js
        // styles.def456.css
        
        // HTML references:
        <script src="/app.abc123.js"></script>
        <link rel="stylesheet" href="/styles.def456.css">
        
        // Cache-Control:
        Cache-Control: public, max-age=31536000, immutable
      `,
      
      benefits: [
        'Automatic invalidation (new hash = new file)',
        'Perfect caching (immutable)',
        'No cache busting query params',
        'CDN-friendly'
      ],
      
      implementation: `
        // Webpack config
        output: {
          filename: '[name].[contenthash].js',
          chunkFilename: '[name].[contenthash].chunk.js'
        }
      `
    };
  }

  // Strategy 3: Cache busting query params
  static queryParamBusting() {
    return {
      concept: 'Add version/timestamp to URL',
      examples: [
        '/app.js?v=1.2.3',
        '/styles.css?t=1634567890',
        '/api/data?_=1634567890'
      ],
      
      implementation: `
        // Client-side
        const version = '1.2.3';
        const script = document.createElement('script');
        script.src = \`/app.js?v=\${version}\`;
        document.head.appendChild(script);
        
        // Or timestamp
        const timestamp = Date.now();
        fetch(\`/api/data?_=\${timestamp}\`);
      `,
      
      downsides: [
        'Different URLs for same content',
        'CDN treats as different resource',
        'Less cache efficiency'
      ]
    };
  }

  // Strategy 4: Programmatic cache clearing
  static programmaticClearing() {
    return `
      // Client-side: Clear specific cache
      async function clearCache(cacheName) {
        const deleted = await caches.delete(cacheName);
        console.log(\`Cache \${cacheName} deleted:\`, deleted);
      }
      
      // Clear all caches
      async function clearAllCaches() {
        const names = await caches.keys();
        await Promise.all(names.map(name => caches.delete(name)));
        console.log('All caches cleared');
      }
      
      // Clear old versions
      async function clearOldVersions(currentVersion) {
        const names = await caches.keys();
        const oldCaches = names.filter(name => {
          return name.startsWith('my-app-') && name !== \`my-app-\${currentVersion}\`;
        });
        
        await Promise.all(oldCaches.map(name => caches.delete(name)));
      }
      
      // Service Worker: Message-based clearing
      self.addEventListener('message', (event) => {
        if (event.data.type === 'CLEAR_CACHE') {
          event.waitUntil(
            caches.delete(event.data.cacheName)
          );
        }
      });
      
      // Client sends message
      navigator.serviceWorker.controller.postMessage({
        type: 'CLEAR_CACHE',
        cacheName: 'old-cache-v1'
      });
    `;
  }

  // Strategy 5: HTTP header invalidation
  static headerInvalidation() {
    return {
      clearSiteData: {
        header: 'Clear-Site-Data: "cache"',
        clears: 'All cached data for origin',
        usage: 'On logout, major update',
        
        example: `
          // Server
          app.post('/logout', (req, res) => {
            res.set('Clear-Site-Data', '"cache", "storage"');
            res.json({ success: true });
          });
        `
      },
      
      noStore: {
        header: 'Cache-Control: no-store',
        prevents: 'Any caching',
        usage: 'Sensitive data'
      },
      
      noCache: {
        header: 'Cache-Control: no-cache',
        forces: 'Revalidation before use',
        usage: 'Must be fresh data'
      }
    };
  }

  // Strategy 6: Time-based expiration
  static timeBasedExpiration() {
    return `
      // Service Worker: Expire old cached responses
      async function expireOldCache(cacheName, maxAge) {
        const cache = await caches.open(cacheName);
        const requests = await cache.keys();
        const now = Date.now();
        
        for (const request of requests) {
          const response = await cache.match(request);
          const date = new Date(response.headers.get('date'));
          const age = now - date.getTime();
          
          if (age > maxAge) {
            await cache.delete(request);
            console.log('Expired:', request.url);
          }
        }
      }
      
      // Run periodically
      setInterval(() => {
        expireOldCache('dynamic-v1', 7 * 24 * 60 * 60 * 1000); // 7 days
      }, 60 * 60 * 1000); // Check every hour
    `;
  }

  // Strategy 7: User-initiated clearing
  static userInitiated() {
    return `
      // Client-side UI
      class CacheManager {
        static async clearAllCaches() {
          const names = await caches.keys();
          await Promise.all(names.map(name => caches.delete(name)));
          
          // Also unregister Service Worker
          const registrations = await navigator.serviceWorker.getRegistrations();
          await Promise.all(registrations.map(reg => reg.unregister()));
          
          // Reload page
          window.location.reload(true);
        }
        
        static async getCacheInfo() {
          const names = await caches.keys();
          const info = await Promise.all(
            names.map(async (name) => {
              const cache = await caches.open(name);
              const keys = await cache.keys();
              return { name, count: keys.length };
            })
          );
          
          return info;
        }
        
        static renderUI() {
          return \`
            <div class="cache-manager">
              <h3>Cache Management</h3>
              <button onclick="CacheManager.clearAllCaches()">
                Clear All Caches
              </button>
              <button onclick="CacheManager.getCacheInfo().then(console.log)">
                Show Cache Info
              </button>
            </div>
          \`;
        }
      }
    `;
  }
}

// Combined invalidation strategy
class CombinedInvalidation {
  static completeStrategy() {
    return {
      staticAssets: {
        strategy: 'Content-hashed filenames',
        caching: 'Cache-Control: public, max-age=31536000, immutable',
        invalidation: 'Automatic (new hash = new file)'
      },
      
      html: {
        strategy: 'Always revalidate',
        caching: 'Cache-Control: no-cache',
        invalidation: 'ETag validation'
      },
      
      api: {
        strategy: 'Short TTL + stale-while-revalidate',
        caching: 'Cache-Control: max-age=60, stale-while-revalidate=3600',
        invalidation: 'Time-based + manual trigger'
      },
      
      serviceWorker: {
        strategy: 'Versioned cache names',
        caching: 'Explicit cache.addAll()',
        invalidation: 'Automatic on activate'
      },
      
      userData: {
        strategy: 'Private, short-lived',
        caching: 'Cache-Control: private, max-age=300',
        invalidation: 'Logout triggers Clear-Site-Data'
      }
    };
  }
}
```

## Common Misconceptions

### Misconception 1: "no-cache means don't cache"

**Reality:** no-cache means cache but revalidate before use. no-store means don't cache.

```javascript
const clarification = {
  'no-cache': {
    meaning: 'Cache but revalidate with server before use',
    behavior: 'Browser sends If-None-Match or If-Modified-Since',
    use: 'Dynamic content that must be fresh'
  },
  'no-store': {
    meaning: 'Do not cache at all',
    behavior: 'Always fetch from server',
    use: 'Sensitive data, private information'
  },
  'max-age=0': {
    meaning: 'Cache but immediately stale',
    behavior: 'Similar to no-cache, must revalidate',
    use: 'Force revalidation'
  }
};
```

### Misconception 2: "Private cache means secure"

**Reality:** Private only prevents CDN caching. Browser still caches and data is not encrypted.

```javascript
const reality = {
  private: 'Only browser caches, not CDN/proxy',
  notPrivate: 'Data still cached in browser (disk)',
  notEncrypted: 'Cache is not encrypted',
  stillAccessible: 'Anyone with disk access can read',
  trueSecurity: 'Use HTTPS + proper authentication'
};
```

### Misconception 3: "Clearing cache fixes all issues"

**Reality:** Many "cache" issues are actually JavaScript, localStorage, or Service Worker problems.

```javascript
const actualCulprits = {
  serviceWorker: 'Serving old cached responses',
  localStorage: 'Old data in localStorage',
  inMemoryState: 'Application state not cleared',
  cookies: 'Old cookies still present',
  dns: 'DNS cache (not browser cache)',
  
  clearing: {
    httpCache: 'Clear browser cache',
    swCache: 'Unregister Service Worker + clear caches',
    storage: 'Clear localStorage/sessionStorage/IndexedDB',
    cookies: 'Clear cookies',
    all: 'Hard reload (Ctrl+Shift+R) or incognito'
  }
};
```

### Misconception 4: "ETag and Last-Modified do the same thing"

**Reality:** Different mechanisms with different precision and use cases.

```javascript
const differences = {
  etag: {
    basis: 'Content-based identifier',
    precision: 'Exact version',
    changes: 'On any content change',
    best: 'Frequently changing content'
  },
  lastModified: {
    basis: 'Timestamp',
    precision: 'Second precision',
    changes: 'On file modification',
    best: 'File-based resources'
  },
  recommendation: 'Use both for best compatibility'
};
```

## Performance Implications

### Cache Hit vs Miss Performance

```javascript
// Performance impact of caching
class CachePerformance {
  static metrics() {
    return {
      cacheHit: {
        memoryCache: '~1-5ms',
        diskCache: '~5-50ms',
        serviceWorker: '~5-50ms',
        description: 'Near-instant response'
      },
      
      cacheMiss: {
        3G: '~500-2000ms',
        4G: '~100-500ms',
        WiFi: '~50-200ms',
        description: 'Full network roundtrip'
      },
      
      improvement: {
        typical: '10-100x faster',
        mobile: '100-1000x faster on slow 3G',
        offline: 'Infinite (works offline vs not working)'
      }
    };
  }

  // Measuring cache performance
  static measureCachePerformance() {
    return `
      // Performance Observer
      const observer = new PerformanceObserver((list) => {
        list.getEntries().forEach(entry => {
          const cache = entry.transferSize === 0 ? 'cache' : 'network';
          console.log(\`\${entry.name}: \${cache} (\${entry.duration.toFixed(2)}ms)\`);
        });
      });
      
      observer.observe({ entryTypes: ['resource'] });
      
      // Or manual measurement
      performance.getEntriesByType('resource').forEach(entry => {
        const fromCache = entry.transferSize === 0 && entry.decodedBodySize > 0;
        console.log(entry.name, {
          duration: entry.duration,
          transferSize: entry.transferSize,
          fromCache
        });
      });
    `;
  }

  // Cache optimization strategies
  static optimizations() {
    return {
      opt1: {
        name: 'Preload critical resources',
        code: '<link rel="preload" href="/critical.css" as="style">',
        benefit: 'Load into memory cache early'
      },
      
      opt2: {
        name: 'Minimize cache misses',
        strategies: [
          'Increase max-age',
          'Use stale-while-revalidate',
          'Implement Service Worker offline fallback'
        ]
      },
      
      opt3: {
        name: 'Optimize cache size',
        strategies: [
          'Compress responses (gzip/brotli)',
          'Remove unused code',
          'Lazy load non-critical resources'
        ]
      },
      
      opt4: {
        name: 'Cache strategically',
        strategies: [
          'Critical assets: Cache-first',
          'API data: Network-first with cache fallback',
          'Images: Cache-first with size limit',
          'HTML: Network-first or revalidate'
        ]
      }
    };
  }
}

// Real-world performance impact
class RealWorldImpact {
  static caseStudy() {
    return {
      scenario: 'E-commerce product page',
      
      withoutCaching: {
        html: '200ms',
        css: '150ms',
        js: '300ms',
        images: '800ms (4x 200ms)',
        fonts: '200ms',
        total: '1650ms',
        lcp: '1000ms' // Largest image
      },
      
      withCaching: {
        html: '100ms (revalidate)',
        css: '5ms (disk cache)',
        js: '5ms (disk cache)',
        images: '20ms (disk cache, 4x 5ms)',
        fonts: '5ms (disk cache)',
        total: '135ms',
        lcp: '105ms' // Cached image
      },
      
      improvement: {
        total: '92% faster (1650ms -> 135ms)',
        lcp: '90% faster (1000ms -> 105ms)',
        userExperience: 'Nearly instant vs slow load'
      }
    };
  }
}
```

## Browser Differences

### Cache Behavior Across Browsers

```javascript
const browserDifferences = {
  chrome: {
    memoryCache: '~100MB per origin',
    diskCache: '~80% of available disk',
    serviceWorker: 'Full support',
    heuristics: 'Aggressive caching of cacheable responses',
    quirks: [
      'Memory cache cleared on tab close',
      'Disk cache shared across profiles',
      'Private browsing uses memory only'
    ]
  },
  
  firefox: {
    memoryCache: '~50MB',
    diskCache: '~50% of available disk',
    serviceWorker: 'Full support',
    heuristics: 'Conservative caching',
    quirks: [
      'Separate cache per profile',
      'Private browsing blocks Service Workers',
      'Can disable cache in about:config'
    ]
  },
  
  safari: {
    memoryCache: 'Variable based on device memory',
    diskCache: 'Limited on iOS',
    serviceWorker: 'Full support (iOS 11.3+)',
    heuristics: 'Very aggressive eviction on iOS',
    quirks: [
      'Clears cache aggressively on iOS',
      'Limited cache on older iOS devices',
      'Private browsing blocks some features',
      'ITP affects cross-site caching'
    ]
  },
  
  edge: {
    legacy: 'Limited Service Worker support',
    chromium: 'Same as Chrome (v79+)',
    recommendation: 'Use Chromium Edge'
  }
};

// Cache control support
const cacheControlSupport = {
  immutable: {
    chrome: 'v65+',
    firefox: 'v49+',
    safari: 'v11+',
    benefit: 'Skip revalidation on reload'
  },
  
  staleWhileRevalidate: {
    chrome: 'v75+',
    firefox: 'v68+',
    safari: 'Not supported',
    fallback: 'Use Service Worker implementation'
  },
  
  staleIfError: {
    chrome: 'v75+',
    firefox: 'v68+',
    safari: 'Not supported',
    benefit: 'Serve stale on server error'
  }
};
```

## Interview Questions

### Question 1: Explain the difference between Cache-Control: no-cache and no-store

**Answer:** no-cache doesn't mean "don't cache" - it means cache the response but revalidate with the server before using it. Browser stores response and sends If-None-Match (ETag) or If-Modified-Since header to check if it's still fresh. Server responds with 304 Not Modified if unchanged, saving bandwidth. no-store means don't cache at all - never store the response anywhere, always fetch from server. Use no-cache for dynamic content that needs to be fresh (HTML pages, APIs), use no-store for sensitive data (banking, personal info). no-cache + ETag provides good balance of freshness and performance.

### Question 2: How does memory cache differ from disk cache, and when is each used?

**Answer:** Memory cache stores responses in RAM - extremely fast (~1ms) but cleared on tab/browser close, limited size (~50-100MB). Used for recently accessed resources, images currently on page, preloaded resources. Disk cache stores on hard drive/SSD - slower (~5-50ms) but persists across sessions, much larger (~100MB-1GB+). Used for all cacheable responses, survives browser restart. Lookup order: Service Worker (if registered) -> Memory cache -> Disk cache -> Network. Browser automatically chooses which cache based on: recency (recent = memory), size (large = disk), available memory, resource type. Users cannot directly control which cache is used.

### Question 3: What is stale-while-revalidate and when would you use it?

**Answer:** stale-while-revalidate allows serving stale cached content while fetching fresh content in background. Header: Cache-Control: max-age=60, stale-while-revalidate=3600. Behavior: 0-60s: serve from cache (fresh), 60s-1h: serve from cache instantly (stale) while fetching update in background for next request, >1h: fetch from network. Benefits: instant response from cache, automatic background updates, resilient to slow networks. Perfect for: social feeds, news articles, comments, non-critical data where slight staleness acceptable. Don't use for: financial data, inventory, user profile changes, any data requiring immediate accuracy. Improves perceived performance significantly - users get instant response.

### Question 4: How does ETag validation work and what are its benefits?

**Answer:** ETag is unique identifier for resource version, generated by server (usually content hash or timestamp). Flow: 1) First request: Server responds with ETag header, 2) Browser caches response with ETag, 3) Next request with Cache-Control: no-cache: Browser sends If-None-Match: {etag}, 4) Server compares: if match -> 304 Not Modified (no body), if different -> 200 with new content + new ETag. Benefits: ~98% bandwidth savings on 304 (headers only vs full response), guaranteed freshness (always validates), works with dynamic content. Strong ETag "abc123" means byte-identical, Weak ETag W/"abc123" means semantically equivalent (minor differences OK). Use with Cache-Control: no-cache for best freshness/performance balance.

### Question 5: What's the purpose of the immutable directive and when should you use it?

**Answer:** immutable tells browser resource will never change, skip revalidation even on page reload. Header: Cache-Control: public, max-age=31536000, immutable. Without immutable: even with long max-age, browsers revalidate on reload (user pressed Ctrl+R). With immutable: skip revalidation until max-age expires. Use ONLY with content-hashed filenames: app.abc123.js, style.def456.css. Benefits: eliminates unnecessary revalidation requests, faster reloads, reduces server load. Don't use on resources that might change with same URL - will serve stale forever. Best practice: build tools generate hashed filenames automatically (Webpack [contenthash]), set immutable + 1 year max-age. Perfect for CDN-served static assets.

### Question 6: How do you implement an effective cache invalidation strategy?

**Answer:** Cache invalidation needs multi-layered approach: 1) Static assets: content-hashed filenames (app.abc123.js) + immutable caching, automatic invalidation when hash changes, 2) HTML: Cache-Control: no-cache + ETag, always fresh but efficient with 304s, 3) APIs: short TTL (60-300s) + stale-while-revalidate, balance freshness and speed, 4) Service Worker: versioned cache names (my-cache-v1), delete old caches in activate event, 5) User actions: Clear-Site-Data header on logout, programmatic cache.delete() for manual clearing. Key principle: make new content have different identifier than old content. Hardest problem in CS: "There are only two hard things - cache invalidation and naming things."

### Question 7: What's the difference between public and private cache directives?

**Answer:** public means response can be cached by any cache - browser, CDN, proxy. private means only browser can cache, CDN/proxy must not. Use public for: truly public resources (CSS, JS, images), same for all users, CDN-served assets. Use private for: user-specific content (personalized HTML, user profile), authenticated responses, anything that varies by user. Example: public API endpoint -> public, max-age=3600. User dashboard HTML -> private, max-age=300. Importance: prevents CDN from serving user A's data to user B. Note: private doesn't encrypt or secure data, just controls which caches can store it. Still need HTTPS and proper auth.

### Question 8: How does Service Worker caching differ from HTTP caching?

**Answer:** HTTP cache (memory + disk) is automatic, controlled by headers, shared across all pages. Service Worker cache is programmatic, controlled by JavaScript, origin-specific. Key differences: Control - HTTP: server decides via headers, SW: developer decides via code. Strategy - HTTP: cache-first or revalidate, SW: any strategy (network-first, cache-first, stale-while-revalidate, custom). Persistence - HTTP: evicted by browser heuristics, SW: persists until explicitly deleted. Offline - HTTP: no offline support, SW: full offline capability. Priority - SW cache checked before HTTP cache. Update - HTTP: automatic based on headers, SW: manual control via versioning. Best practice: use both - SW for app shell and offline, HTTP for CDN assets. SW provides ultimate control but requires more code.

## Key Takeaways

1. **Cache-Control is King**: Cache-Control header is primary mechanism for controlling caching. max-age sets freshness lifetime, no-cache forces revalidation, no-store prevents caching entirely, immutable optimizes static assets.

2. **no-cache != no-store**: no-cache means cache but revalidate before use (with ETag/Last-Modified). no-store means never cache. This confusion causes many issues.

3. **Memory vs Disk Cache**: Memory cache (RAM) is fastest (~1ms) but session-only. Disk cache (storage) is fast (~5-50ms) and persistent. Browser automatically manages which is used.

4. **ETag Enables Efficient Revalidation**: ETag with no-cache provides best freshness/performance balance. 304 responses save ~98% bandwidth while guaranteeing freshness. Always generate strong ETags when possible.

5. **Content Hashing Solves Invalidation**: Embed content hash in filename (app.abc123.js) + immutable + 1 year max-age. Automatic invalidation when content changes. Perfect for CDN static assets.

6. **stale-while-revalidate Best of Both**: Serves stale content instantly while fetching fresh in background. Perfect for non-critical content where slight staleness acceptable. Dramatically improves perceived performance.

7. **Service Worker Ultimate Control**: Programmatic caching with any strategy. Enables offline, background sync, push notifications. Checked before HTTP cache. Requires careful version management.

8. **Cache Lookup Order**: Service Worker (if controlling) -> Memory cache -> Disk cache -> Network. Understanding this order is crucial for debugging cache issues.

9. **private for User Data**: Use private for user-specific content to prevent CDN serving one user's data to another. public for truly public resources. Critical for security.

10. **Invalidation is Hard**: Cache invalidation requires coordinated strategy across all layers. Use content hashing for static assets, versioning for SW caches, short TTLs for dynamic data, Clear-Site-Data for user actions.

## Resources

### Official Specifications
- **HTTP Caching RFC 7234**: https://tools.ietf.org/html/rfc7234
- **Cache-Control Header**: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control
- **Service Worker Spec**: https://w3c.github.io/ServiceWorker/

### Implementation Guides
- **MDN HTTP Caching**: https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching
- **Google Caching Best Practices**: https://web.dev/http-cache/
- **Service Worker Caching Strategies**: https://developers.google.com/web/tools/workbox/modules/workbox-strategies

### Tools
- **Chrome DevTools**: Network tab Size column shows cache source
- **Lighthouse**: Audit for caching issues
- **Cache Control Header Tester**: https://redbot.org/

### Articles
- **Caching Best Practices**: https://jakearchibald.com/2016/caching-best-practices/
- **Offline Cookbook**: https://web.dev/offline-cookbook/
- **Cache API**: https://developer.mozilla.org/en-US/docs/Web/API/Cache

### Libraries
- **Workbox**: https://developers.google.com/web/tools/workbox - Google's Service Worker library
- **sw-toolbox**: https://github.com/GoogleChrome/sw-toolbox - Service Worker helper (deprecated but educational)
