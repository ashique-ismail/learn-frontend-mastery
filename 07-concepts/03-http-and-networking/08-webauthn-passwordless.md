# WebAuthn and Passwordless Authentication

## The Idea

**In plain English:** WebAuthn is a way for websites to let you log in using your fingerprint, face, or a physical security key instead of a password. Your device holds a secret key that never leaves it, and the website only ever sees the matching public key — so there is nothing useful to steal from the server.

**Real-world analogy:** Think of a hotel that gives you a physical key card when you check in. The card has a unique chip inside (your private key) that only opens your room's lock. The hotel programs the lock (stores the public key) and never needs to keep a copy of the chip itself. When you tap the card on the lock, the lock checks whether the card matches — without the card ever "sending" its secret anywhere.

- The key card = the private key stored on your device (fingerprint sensor, security key, or phone)
- The room lock = the server, which holds the matching public key
- Tapping the card on the lock = the browser asking your device to sign a one-time challenge so the server can verify it

---

## Overview

WebAuthn (Web Authentication API) is a W3C standard that enables phishing-resistant, passwordless authentication using public-key cryptography. It is the browser API component of the FIDO2 specification. Instead of a password, the user authenticates via a hardware security key (YubiKey), biometric device (Touch ID, Face ID, Windows Hello), or a passkey synced across devices via iCloud/Google Password Manager.

Senior engineers are expected to understand the protocol flow, credential lifecycle, and how to architect a WebAuthn-based system.

---

## Why WebAuthn Matters

| Problem with passwords | WebAuthn solution |
|---|---|
| Phishable (users enter on fake sites) | Credential is origin-bound — can't be used on a different domain |
| Reused across sites | Each registration generates a new key pair per origin |
| Stolen from server DB | Server only stores the public key — useless if leaked |
| Brute-forceable | Asymmetric cryptography — no shared secret to brute-force |
| Requires server-side hashing (bcrypt, Argon2) | No password hashing needed |

---

## Core Concepts

### Parties

- **Relying Party (RP):** Your server/application. Identified by `rpId` (usually the domain).
- **Authenticator:** The device holding the private key — a hardware key, platform authenticator (Touch ID), or passkey provider.
- **User Agent:** The browser, which mediates between RP and authenticator.

### Credential Types

- **Platform authenticator:** Built into the device (Touch ID, Face ID, Windows Hello, Android biometric). Tied to that device.
- **Roaming authenticator:** External hardware key (YubiKey, Titan key). Portable but physically present required.
- **Passkey:** A synced platform credential. Apple syncs via iCloud Keychain, Google via Google Password Manager. Works across the user's devices.

### Key Concepts

- **Challenge:** A random server-generated nonce sent with every registration/authentication request. Prevents replay attacks.
- **Attestation:** During registration, the authenticator can prove its model/manufacturer to the RP (used in high-security contexts).
- **Assertion:** During authentication, the authenticator proves it holds the private key by signing the challenge.
- **User Verification (UV):** Whether the authenticator required biometric/PIN before signing (`required`, `preferred`, `discouraged`).

---

## Registration Flow

```
Browser                     Server
  |                            |
  |  1. GET /register/begin    |
  |  ─────────────────────────>|
  |                            | generate challenge + options
  |  2. PublicKeyCredentialCreationOptions
  |  <─────────────────────────|
  |                            |
  |  3. navigator.credentials.create(options)
  |     ↓ OS biometric prompt  |
  |     ↓ authenticator creates key pair
  |     ↓ signs clientDataJSON + attestationObject
  |                            |
  |  4. POST /register/complete|
  |     { credential }         |
  |  ─────────────────────────>|
  |                            | verify signature, store public key
  |  5. { success: true }      |
  |  <─────────────────────────|
```

### Browser — Registration

```javascript
// Step 2 response from server (simplified)
const creationOptions = {
  challenge: base64urlDecode(serverChallenge), // ArrayBuffer
  rp: {
    name: 'Acme Corp',
    id:   'acme.com',       // must match or be suffix of current domain
  },
  user: {
    id:          crypto.getRandomValues(new Uint8Array(16)), // opaque user handle
    name:        'alice@acme.com',
    displayName: 'Alice',
  },
  pubKeyCredParams: [
    { type: 'public-key', alg: -7  },  // ES256 (ECDSA P-256) — preferred
    { type: 'public-key', alg: -257 }, // RS256 (RSA PKCS1v15) — fallback
  ],
  authenticatorSelection: {
    authenticatorAttachment: 'platform',    // 'platform' | 'cross-platform' | omit for any
    residentKey:             'required',    // enables usernameless login
    userVerification:        'required',    // require biometric/PIN
  },
  timeout:     60_000,
  attestation: 'none', // 'none' | 'indirect' | 'direct' — 'none' for most apps
};

// Step 3: create credential
const credential = await navigator.credentials.create({
  publicKey: creationOptions,
});

// Step 4: send to server
const registration = {
  id:       credential.id,
  rawId:    base64urlEncode(credential.rawId),
  type:     credential.type,
  response: {
    clientDataJSON:    base64urlEncode(credential.response.clientDataJSON),
    attestationObject: base64urlEncode(credential.response.attestationObject),
  },
};

await fetch('/register/complete', {
  method: 'POST',
  body: JSON.stringify(registration),
});
```

### Server — Registration Verification (Node.js with SimpleWebAuthn)

```typescript
import { verifyRegistrationResponse } from '@simplewebauthn/server';

async function completeRegistration(userId: string, body: RegistrationResponseJSON) {
  const expectedChallenge = await getStoredChallenge(userId); // from session/Redis

  const verification = await verifyRegistrationResponse({
    response:             body,
    expectedChallenge,
    expectedOrigin:       'https://acme.com',
    expectedRPID:         'acme.com',
    requireUserVerification: true,
  });

  if (!verification.verified) throw new Error('Registration failed');

  // Store the public key — this is all you need; no password!
  const { credentialID, credentialPublicKey, counter } = verification.registrationInfo!;

  await db.credentials.create({
    userId,
    credentialId:  credentialID,      // Buffer
    publicKey:     credentialPublicKey, // Buffer (COSE-encoded)
    counter,                            // for clone detection
    deviceType:    verification.registrationInfo!.credentialDeviceType,
  });
}
```

---

## Authentication Flow

```
Browser                     Server
  |                            |
  |  1. POST /auth/begin       |
  |     { username? }          |
  |  ─────────────────────────>|
  |                            | lookup stored credentials for user
  |                            | generate new challenge
  |  2. PublicKeyCredentialRequestOptions
  |  <─────────────────────────|
  |                            |
  |  3. navigator.credentials.get(options)
  |     ↓ user selects account + biometric
  |     ↓ authenticator signs challenge
  |                            |
  |  4. POST /auth/complete    |
  |     { assertion }          |
  |  ─────────────────────────>|
  |                            | verify signature with stored public key
  |                            | increment counter
  |  5. Set session cookie     |
  |  <─────────────────────────|
```

### Browser — Authentication

```javascript
// Step 2 response from server
const requestOptions = {
  challenge: base64urlDecode(serverChallenge),
  rpId: 'acme.com',
  allowCredentials: [   // omit for usernameless (discoverable credentials)
    {
      type: 'public-key',
      id:   base64urlDecode(storedCredentialId),
    },
  ],
  userVerification: 'required',
  timeout: 60_000,
};

// Step 3: get assertion
const assertion = await navigator.credentials.get({
  publicKey: requestOptions,
});

// Step 4: send to server
await fetch('/auth/complete', {
  method: 'POST',
  body: JSON.stringify({
    id:       assertion.id,
    rawId:    base64urlEncode(assertion.rawId),
    type:     assertion.type,
    response: {
      clientDataJSON:      base64urlEncode(assertion.response.clientDataJSON),
      authenticatorData:   base64urlEncode(assertion.response.authenticatorData),
      signature:           base64urlEncode(assertion.response.signature),
      userHandle:          assertion.response.userHandle
        ? base64urlEncode(assertion.response.userHandle)
        : null,
    },
  }),
});
```

### Server — Authentication Verification

```typescript
import { verifyAuthenticationResponse } from '@simplewebauthn/server';

async function completeAuthentication(userId: string, body: AuthenticationResponseJSON) {
  const storedCredential = await db.credentials.findByCredentialId(body.id);
  const expectedChallenge = await getStoredChallenge(userId);

  const verification = await verifyAuthenticationResponse({
    response:             body,
    expectedChallenge,
    expectedOrigin:       'https://acme.com',
    expectedRPID:         'acme.com',
    authenticator: {
      credentialID:        storedCredential.credentialId,
      credentialPublicKey: storedCredential.publicKey,
      counter:             storedCredential.counter,
    },
    requireUserVerification: true,
  });

  if (!verification.verified) throw new Error('Authentication failed');

  // Update counter (detects cloned authenticators if counter doesn't increment)
  await db.credentials.updateCounter(
    storedCredential.id,
    verification.authenticationInfo.newCounter
  );

  await createSession(userId);
}
```

---

## Passkeys

Passkeys are synced discoverable credentials. The user experience is:

1. Register on one device — credential syncs to all user's devices via cloud (iCloud/Google)
2. Authenticate on any device — no hardware key needed, no username required
3. Cross-device authentication — scan a QR code with your phone to authenticate on a desktop

```javascript
// Passkey registration — same API, different options
const creationOptions = {
  // ...
  authenticatorSelection: {
    authenticatorAttachment: 'platform',  // prefer built-in
    residentKey:             'required',  // must be discoverable (enables usernameless)
    userVerification:        'required',
  },
};

// Usernameless authentication — no allowCredentials needed
const requestOptions = {
  challenge: base64urlDecode(serverChallenge),
  rpId: 'acme.com',
  // allowCredentials: omitted — browser shows all available passkeys for this RP
  userVerification: 'required',
};

const assertion = await navigator.credentials.get({ publicKey: requestOptions });
// The userHandle in the response tells the server which user authenticated
```

---

## Conditional UI (Autofill Integration)

Modern browsers can suggest passkeys in the autofill dropdown of a username/password field:

```html
<input type="text" name="username" autocomplete="username webauthn" />
```

```javascript
// Check support
const available = await PublicKeyCredential.isConditionalMediationAvailable?.();

if (available) {
  // Start a "silent" get() — doesn't show modal, waits for autofill interaction
  navigator.credentials.get({
    mediation: 'conditional',
    publicKey: {
      challenge: base64urlDecode(serverChallenge),
      rpId: 'acme.com',
      userVerification: 'preferred',
    },
  }).then(completeAuthentication);
}
```

---

## Browser Support & Feature Detection

```javascript
if (!window.PublicKeyCredential) {
  // WebAuthn not supported — fall back to password
  showPasswordForm();
  return;
}

// Check platform authenticator availability
const available = await PublicKeyCredential.isUserVerifyingPlatformAuthenticatorAvailable();
if (!available) {
  // Offer hardware key or fallback to TOTP/password
}
```

---

## Security Considerations

- **Origin binding** — The `clientDataJSON` contains the origin. Phishing domains get different origins → signature verification fails → attack defeated.
- **Challenge freshness** — Expire stored challenges after 60 seconds. One-time use prevents replay attacks.
- **Counter validation** — If the counter does not increase, the credential may have been cloned. Reject or flag.
- **`rpId` validation** — Must equal the effective domain or a registrable suffix. `rp.id = 'acme.com'` works on `app.acme.com` but not on `evil.com`.
- **User verification** — `required` means the device enforced biometric/PIN. `preferred` and `discouraged` allow bypassing it on some authenticators.

---

## Interview Questions

**Q: How does WebAuthn prevent phishing?**
A: Credentials are cryptographically bound to the `rpId` (the origin's domain). When a user is on `evil-acme.com`, the authenticator receives a different `rpId` in the clientDataJSON. The server's `rpId` check fails, so the signature is rejected — the credential cannot be used on any domain other than the one it was registered for.

**Q: What is a passkey and how does it differ from a hardware security key?**
A: A passkey is a synced discoverable credential stored in a platform authenticator (iCloud Keychain, Google Password Manager). It syncs across all of the user's devices — no physical key needed. A hardware security key (YubiKey) is a roaming authenticator that stays on a physical device and never syncs.

**Q: What does `residentKey: 'required'` enable?**
A: It stores the user handle on the authenticator itself (a "discoverable credential"). This enables usernameless login — the server can omit `allowCredentials` in the authentication request and recover the user identity from the `userHandle` in the assertion response.

**Q: What is the counter field used for?**
A: Each assertion increments a counter stored on the authenticator. The server stores the last-seen counter. If an assertion arrives with a counter ≤ the stored value, the credential may have been cloned — the server should reject it and alert the user.
