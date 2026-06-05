# Clickjacking and X-Frame-Options

## The Idea

**In plain English:** Clickjacking is a trick where a bad actor secretly places your website invisibly on top of their own page, so when you think you're clicking a button on their site, you're actually clicking a button on yours — without ever knowing it. "Framing" means embedding one website inside another like a picture in a picture frame.

**Real-world analogy:** Imagine a scammer places a sheet of glass over a voting button at a kiosk. The glass is invisible, and printed on the kiosk's front panel is a big sign saying "Click here for a free coffee." You tap the sign — but the glass is what you actually press, and behind the glass is the real voting button you never meant to touch.

- The voting kiosk = your real website (bank, social media, etc.)
- The sheet of invisible glass = the transparent iframe the attacker lays over their page
- The "free coffee" sign = the fake decoy content the attacker shows you
- Your tap = your real mouse click, unknowingly landing on the hidden button

---

## Overview

Clickjacking (UI redressing) is an attack where a malicious page embeds your application in a transparent iframe and tricks users into clicking on UI elements they can't see. The user thinks they're interacting with the attacker's page but is actually clicking buttons in your app — confirming transactions, enabling permissions, or deleting accounts. This guide covers the attack mechanics, X-Frame-Options, CSP `frame-ancestors`, and detection techniques.

## How Clickjacking Works

```
Clickjacking Attack Flow:

Attacker's page                    Your banking app
┌──────────────────────────────┐   ┌───────────────────────────┐
│                              │   │  [Send $1000 to attacker] │
│  "Click here to WIN $1000!"  │   │  [Confirm transfer]       │
│         ┌──────────┐         │   └───────────────────────────┘
│         │  CLICK   │         │
│         │   HERE   │         │
│         └──────────┘         │
└──────────────────────────────┘

What actually happens:
┌──────────────────────────────┐
│  Attacker's visible page     │  z-index: 1 (visible)
│                              │
│  ┌────────────────────────┐  │
│  │ iframe: bank.com       │  │  z-index: 2 (invisible, opacity: 0)
│  │  [Confirm transfer] ←──┼──┼── positioned directly over "CLICK HERE"
│  └────────────────────────┘  │
│  "Click here to WIN $1000!"  │  z-index: 1
└──────────────────────────────┘

User clicks "CLICK HERE" → actually clicks "Confirm transfer" in bank.com iframe
```

### Minimal Attack HTML

```html
<!-- attacker.html -->
<!DOCTYPE html>
<html>
<head>
  <style>
    #decoy {
      position: absolute;
      top: 200px;
      left: 300px;
      z-index: 1;
      font-size: 24px;
      cursor: pointer;
    }

    #target-frame {
      position: absolute;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
      opacity: 0;           /* invisible — victim can't see it */
      z-index: 2;           /* on top of decoy button */
    }
  </style>
</head>
<body>
  <div id="decoy">🎉 Click here to claim your prize!</div>

  <!-- victim.example.com is the target application -->
  <iframe
    id="target-frame"
    src="https://victim.example.com/account/settings"
  ></iframe>
</body>
</html>
```

## Defense 1: X-Frame-Options Header

`X-Frame-Options` is a response header that controls whether a page can be embedded in a frame:

```
X-Frame-Options: DENY
  → Page cannot be framed by ANY origin (including same-origin)

X-Frame-Options: SAMEORIGIN
  → Page can only be framed by the same origin (e.g., same.example.com)

X-Frame-Options: ALLOW-FROM https://trusted.example.com
  → Page can be framed only by the specified origin
  ⚠️  ALLOW-FROM is deprecated and not supported in Chrome/Firefox
```

### Setting X-Frame-Options in Node.js

```typescript
// Express — manual header
app.use((req, res, next) => {
  res.setHeader('X-Frame-Options', 'DENY');
  next();
});

// Express with Helmet (recommended)
import helmet from 'helmet';

app.use(
  helmet.frameguard({
    action: 'deny',          // or 'sameorigin'
  })
);

// Next.js — next.config.js
module.exports = {
  async headers() {
    return [
      {
        source: '/:path*',
        headers: [
          {
            key: 'X-Frame-Options',
            value: 'DENY',
          },
        ],
      },
    ];
  },
};
```

## Defense 2: CSP frame-ancestors (Modern Standard)

`frame-ancestors` is the CSP directive that supersedes X-Frame-Options. It's more flexible — it supports multiple trusted origins, wildcards, and has full browser support:

```
Content-Security-Policy: frame-ancestors 'none'
  → No framing allowed by anyone

Content-Security-Policy: frame-ancestors 'self'
  → Same origin only (equivalent to SAMEORIGIN)

Content-Security-Policy: frame-ancestors 'self' https://trusted-partner.com
  → Same origin OR trusted-partner.com

Content-Security-Policy: frame-ancestors https://*.example.com
  → Any subdomain of example.com
```

### CSP frame-ancestors vs X-Frame-Options

```
Comparison:

Feature                      X-Frame-Options    CSP frame-ancestors
─────────────────────────────────────────────────────────────────────
Browser support              All browsers       All modern browsers
Multiple trusted origins     ❌ No              ✅ Yes
Wildcard subdomains          ❌ No              ✅ Yes
Syntax                       Response header    CSP header directive
Overrides X-Frame-Options    N/A                ✅ Yes (CSP wins)
Recommended going forward    No (legacy)        Yes
```

```typescript
// Express — comprehensive clickjacking protection
app.use((req, res, next) => {
  // Set both for maximum compatibility
  res.setHeader('X-Frame-Options', 'DENY');
  res.setHeader(
    'Content-Security-Policy',
    "frame-ancestors 'none'"
  );
  next();
});

// For apps that legitimately need embedding (e.g., embeddable widgets)
app.get('/widget', (req, res) => {
  const allowedOrigins = [
    'https://partner.example.com',
    'https://dashboard.example.com',
  ];
  const origin = req.headers.origin || req.headers.referer;
  const isAllowed = allowedOrigins.some((o) => origin?.startsWith(o));

  if (isAllowed) {
    res.setHeader(
      'Content-Security-Policy',
      `frame-ancestors 'self' ${allowedOrigins.join(' ')}`
    );
  } else {
    res.setHeader('Content-Security-Policy', "frame-ancestors 'none'");
  }
  // ⚠️  DO NOT set based on user-controlled input — validate strictly
});
```

## Defense 3: JavaScript Frame-Busting (Deprecated)

Before `X-Frame-Options`, JavaScript "frame busters" were the only protection:

```javascript
// ❌ JavaScript frame busting — DO NOT rely on this
// It can be bypassed with sandbox="allow-scripts" on the iframe
if (top !== self) {
  top.location = self.location; // try to break out of iframe
}

// Bypass: attacker uses:
<iframe src="https://victim.com" sandbox="allow-scripts allow-forms">
// sandbox="allow-scripts" without "allow-top-navigation" prevents the redirect

// ✅ HTTP headers are the only reliable defense
```

## Detecting Framing Attempts

```typescript
// Server-side: log and alert on suspicious referer headers
app.use((req, res, next) => {
  const referer = req.headers.referer;
  if (referer) {
    const refererUrl = new URL(referer);
    const requestUrl = new URL(`https://${req.headers.host}${req.path}`);

    if (refererUrl.hostname !== requestUrl.hostname) {
      // Cross-origin request — could be legitimate or framing attempt
      // Log for analysis, don't block (breaks legitimate cross-origin flows)
      console.warn(`Cross-origin request: referer=${referer} path=${req.path}`);
    }
  }
  next();
});

// Client-side: detect and handle being framed (e.g., for analytics)
if (window.self !== window.top) {
  // Page is in an iframe
  window.top?.postMessage({ type: 'FRAMING_DETECTED', src: window.location.href }, '*');
  // You could also redirect or show a warning banner
}
```

## Protecting Sensitive Actions

For highly sensitive actions (confirm payment, delete account), add additional verification beyond framing protection:

```typescript
// 1. Require re-authentication for sensitive actions
app.post('/api/transfer', requireRecentAuth, async (req, res) => {
  // Even if clickjacked, user still needs to authenticate
});

// 2. Custom confirmation dialogs with random tokens
// A framebusted confirmation dialog the attacker can't predict
function ConfirmDangerousAction({ onConfirm }: { onConfirm: () => void }) {
  const [confirmText, setConfirmText] = useState('');
  // Generate a random word the user must type — attacker can't pre-fill it
  const requiredText = useMemo(() => generateRandomWord(), []);

  return (
    <Dialog>
      <p>Type "<strong>{requiredText}</strong>" to confirm deletion of your account</p>
      <input value={confirmText} onChange={(e) => setConfirmText(e.target.value)} />
      <button
        disabled={confirmText !== requiredText}
        onClick={onConfirm}
      >
        Confirm
      </button>
    </Dialog>
  );
}

// 3. Time-based CSRF tokens also mitigate clickjacking for form submissions
```

## Legitimate iframe Embedding

If your app genuinely needs to be embeddable (payment widgets, embedded dashboards, auth flows):

```typescript
// Allow specific origins via frame-ancestors
// The iframe src app sets its own headers

// For an embeddable payment widget:
app.get('/widget/payment', (req, res) => {
  const allowedParents = process.env.ALLOWED_EMBED_ORIGINS?.split(',') ?? [];

  res.setHeader(
    'Content-Security-Policy',
    `frame-ancestors 'self' ${allowedParents.join(' ')}`
  );
  // DO NOT set X-Frame-Options — it doesn't support multiple origins
  // CSP frame-ancestors takes precedence when both are set

  res.sendFile('payment-widget.html');
});

// For the parent page embedding the widget:
// No special headers needed — frame-ancestors is a directive on the FRAMED page
```

## Common Mistakes

### 1. Relying on JavaScript Frame Busting

```javascript
// ❌ Defeated by sandbox attribute
if (top !== self) top.location = self.location;

// ✅ HTTP headers only
// X-Frame-Options: DENY
// Content-Security-Policy: frame-ancestors 'none'
```

### 2. Using X-Frame-Options ALLOW-FROM

```
// ❌ Deprecated, not supported in Chrome/Firefox
X-Frame-Options: ALLOW-FROM https://partner.com

// ✅ Use CSP frame-ancestors
Content-Security-Policy: frame-ancestors https://partner.com
```

### 3. Only Protecting GET Pages, Not POST Endpoints

```typescript
// ❌ Protecting the page but not the action endpoint
// Attacker doesn't need to see the button — they can iframe the action URL
app.get('/settings', (req, res) => {
  res.setHeader('X-Frame-Options', 'DENY');
  // ...
});

// POST endpoint has no framing protection
app.post('/api/settings', async (req, res) => {
  // ← No X-Frame-Options/CSP on the form action!
  // Form submission via hidden iframe still works
});

// ✅ Add CSRF protection to state-changing endpoints
// CSRF tokens prevent cross-origin form submissions regardless of framing
```

### 4. Setting Headers Only on Authenticated Pages

```typescript
// ❌ Login page is unprotected — attacker frames the login form
// Steals credentials via overlay
app.get('/dashboard', isAuthenticated, (req, res) => {
  res.setHeader('X-Frame-Options', 'DENY'); // only on auth'd pages
});

// ✅ Apply globally, including login and signup pages
app.use(helmet.frameguard({ action: 'deny' }));
```

## Interview Questions

### 1. What is clickjacking and why is JavaScript frame-busting an insufficient defense?

**Answer:** Clickjacking (UI redressing) embeds a target site in a transparent iframe and positions it over decoy UI, tricking users into clicking invisible elements. JavaScript frame-busting tries to escape the iframe by setting `top.location`, but it's defeated by the `sandbox` attribute on the iframe — specifically `sandbox="allow-scripts allow-forms"` which permits scripts but not top-level navigation. Since the defense can be bypassed by the attacker who controls the iframe element, only HTTP headers that the browser enforces unconditionally (`X-Frame-Options`, CSP `frame-ancestors`) are reliable defenses.

### 2. What is the difference between X-Frame-Options and CSP frame-ancestors?

**Answer:** `X-Frame-Options` is a legacy header with three values: `DENY` (no framing), `SAMEORIGIN` (same origin only), and `ALLOW-FROM origin` (single trusted origin — deprecated and unsupported in Chrome/Firefox). CSP `frame-ancestors` is the modern replacement: it supports multiple trusted origins, wildcards (`https://*.example.com`), is part of the broader CSP framework, and when both are present, CSP takes precedence. For new implementations, use CSP `frame-ancestors` exclusively. Keep `X-Frame-Options` only for legacy browser compatibility.

### 3. How would you implement clickjacking protection for an application that has some legitimate embeddable pages?

**Answer:** Apply CSP `frame-ancestors 'none'` globally as a default. For pages that need to be embeddable, override with the specific allowed parent origins: `frame-ancestors 'self' https://partner.example.com`. This is enforced per-page by the framed page's own headers — the parent doesn't control it. Maintain a server-side allowlist of trusted origins; never derive the `frame-ancestors` value from user-controlled input. Also consider what actions are available in the embedded context and add CSRF tokens or re-authentication requirements for sensitive operations even within an allowed embedding.

### 4. How does clickjacking relate to CSRF, and how are the defenses different?

**Answer:** Both attacks involve tricking users into performing unintended actions on a target site. CSRF forges a request from another origin (the user's browser makes a request the user didn't intend, using their credentials). Clickjacking physically tricks the user into clicking an element they can't see. CSRF defenses (same-site cookies, CSRF tokens) don't prevent clickjacking — the clickjacked page is actually loaded and rendered, so cookies are present legitimately. Framing headers (`X-Frame-Options`, `frame-ancestors`) don't prevent CSRF — CSRF attacks don't need to frame the page at all. A complete defense needs both.
