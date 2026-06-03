# HTTPS and TLS

## Table of Contents
- [Introduction](#introduction)
- [TLS Handshake Explained](#tls-handshake-explained)
- [Certificate Management](#certificate-management)
- [HSTS (HTTP Strict Transport Security)](#hsts-http-strict-transport-security)
- [Mixed Content Warnings](#mixed-content-warnings)
- [SSL/TLS Versions and Cipher Suites](#ssltls-versions-and-cipher-suites)
- [Certificate Pinning](#certificate-pinning)
- [Implementation Examples](#implementation-examples)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [When to Use/Not to Use](#when-to-usenot-to-use)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

**HTTPS (HTTP Secure)** is the secure version of HTTP, using **TLS (Transport Layer Security)** to encrypt communication between client and server. It protects against eavesdropping, tampering, and man-in-the-middle attacks.

### Why HTTPS Matters

```
┌─────────────────────────────────────────────────────────────┐
│              HTTP vs HTTPS                                   │
└─────────────────────────────────────────────────────────────┘

HTTP (Insecure):
   Client ──── Plain Text ───> Server
           Password: secret123
           
   Attacker can:
   ✗ Read all data (eavesdropping)
   ✗ Modify data (tampering)
   ✗ Impersonate server (MITM)

HTTPS (Secure):
   Client ──── Encrypted ───> Server
           �����������������
           
   Attacker sees:
   ✓ Only encrypted gibberish
   ✓ Cannot read data
   ✓ Cannot modify without detection
   ✓ Cannot impersonate (certificate validation)

Benefits:
  ├─ Confidentiality (encryption)
  ├─ Integrity (tampering detection)
  ├─ Authentication (certificate verification)
  └─ Trust (browser indicators)
```

## TLS Handshake Explained

### TLS 1.3 Handshake (Simplified)

```
┌─────────────────────────────────────────────────────────────┐
│              TLS 1.3 Handshake Flow                          │
└─────────────────────────────────────────────────────────────┘

1. ClientHello
   ┌────────┐                           ┌────────┐
   │ Client │ ────────────────────────> │ Server │
   └────────┘                           └────────┘
   
   Sends:
   - TLS version (1.3)
   - Random number
   - Cipher suites supported
   - Key share (Diffie-Hellman)
   - Extensions (SNI, ALPN)

2. ServerHello + Certificate + Finished
   ┌────────┐                           ┌────────┐
   │ Client │ <──────────────────────── │ Server │
   └────────┘                           └────────┘
   
   Sends:
   - Selected cipher suite
   - Server certificate
   - Key share
   - Encrypted handshake (Finished)

3. Client Finished
   ┌────────┐                           ┌────────┐
   │ Client │ ────────────────────────> │ Server │
   └────────┘                           └────────┘
   
   Sends:
   - Encrypted handshake (Finished)

4. Application Data
   ┌────────┐ <───── Encrypted ──────> ┌────────┐
   │ Client │                           │ Server │
   └────────┘                           └────────┘
   
   Both sides can now send encrypted data

Key Improvements in TLS 1.3:
  ├─ Fewer round trips (faster)
  ├─ Forward secrecy mandatory
  ├─ Removed weak ciphers
  └─ Always encrypted handshake
```

### Detailed Handshake Steps

```typescript
// TLS handshake simulation (conceptual)
class TLSHandshake {
  // Step 1: Client Hello
  createClientHello(): ClientHello {
    return {
      protocolVersion: 'TLS 1.3',
      random: crypto.randomBytes(32),
      cipherSuites: [
        'TLS_AES_256_GCM_SHA384',
        'TLS_AES_128_GCM_SHA256',
        'TLS_CHACHA20_POLY1305_SHA256'
      ],
      extensions: {
        serverName: 'example.com',  // SNI
        supportedGroups: ['x25519', 'secp256r1'],
        signatureAlgorithms: ['rsa_pss_rsae_sha256', 'ecdsa_secp256r1_sha256'],
        alpn: ['h2', 'http/1.1']  // Protocol negotiation
      },
      keyShare: this.generateDHKeyShare()
    };
  }

  // Step 2: Server Response
  createServerHello(clientHello: ClientHello): ServerHello {
    return {
      protocolVersion: 'TLS 1.3',
      random: crypto.randomBytes(32),
      selectedCipherSuite: 'TLS_AES_256_GCM_SHA384',
      keyShare: this.generateDHKeyShare(),
      certificate: this.loadCertificate(),
      certificateVerify: this.signHandshake(),
      finished: this.computeFinished()
    };
  }

  // Step 3: Derive Session Keys
  deriveSessionKeys(
    clientKeyShare: Buffer,
    serverKeyShare: Buffer,
    clientRandom: Buffer,
    serverRandom: Buffer
  ): SessionKeys {
    // Diffie-Hellman key exchange
    const sharedSecret = this.computeDH(clientKeyShare, serverKeyShare);
    
    // Key derivation
    const masterSecret = this.hkdfExtract(sharedSecret);
    
    return {
      clientWriteKey: this.hkdfExpand(masterSecret, 'client write key'),
      serverWriteKey: this.hkdfExpand(masterSecret, 'server write key'),
      clientWriteIV: this.hkdfExpand(masterSecret, 'client write iv'),
      serverWriteIV: this.hkdfExpand(masterSecret, 'server write iv')
    };
  }

  // Verify certificate chain
  verifyCertificate(certificate: Certificate): boolean {
    // 1. Check certificate validity period
    const now = new Date();
    if (now < certificate.notBefore || now > certificate.notAfter) {
      return false;
    }

    // 2. Verify certificate chain to trusted root
    let current = certificate;
    while (current.issuer !== current.subject) {
      const issuer = this.findIssuerCertificate(current.issuer);
      if (!issuer) return false;
      
      if (!this.verifySignature(current, issuer.publicKey)) {
        return false;
      }
      
      current = issuer;
    }

    // 3. Check if root is trusted
    if (!this.isTrustedRoot(current)) {
      return false;
    }

    // 4. Verify hostname matches certificate
    if (!this.verifyHostname(certificate, this.hostname)) {
      return false;
    }

    // 5. Check certificate revocation (OCSP/CRL)
    if (this.isRevoked(certificate)) {
      return false;
    }

    return true;
  }
}
```

## Certificate Management

### Certificate Structure

```typescript
// X.509 Certificate structure
interface X509Certificate {
  // Certificate metadata
  version: number;  // v3 = 3
  serialNumber: string;
  
  // Signature algorithm
  signatureAlgorithm: string;  // e.g., 'sha256WithRSAEncryption'
  
  // Issuer (CA)
  issuer: {
    country: string;
    organization: string;
    commonName: string;
  };
  
  // Validity period
  notBefore: Date;
  notAfter: Date;
  
  // Subject (certificate holder)
  subject: {
    country: string;
    state: string;
    locality: string;
    organization: string;
    commonName: string;  // Domain name
  };
  
  // Public key
  publicKey: {
    algorithm: string;  // RSA, ECDSA
    size: number;  // 2048, 4096
    key: Buffer;
  };
  
  // Extensions
  extensions: {
    // Subject Alternative Names (multiple domains)
    subjectAltName: string[];  // ['example.com', 'www.example.com', '*.example.com']
    
    // Key usage
    keyUsage: string[];  // ['digitalSignature', 'keyEncipherment']
    
    // Extended key usage
    extKeyUsage: string[];  // ['serverAuth', 'clientAuth']
    
    // Basic constraints
    basicConstraints: {
      ca: boolean;
      pathLen?: number;
    };
    
    // Authority Key Identifier
    authorityKeyIdentifier: Buffer;
    
    // Subject Key Identifier
    subjectKeyIdentifier: Buffer;
    
    // CRL Distribution Points
    crlDistributionPoints: string[];
    
    // Authority Information Access (OCSP)
    authorityInfoAccess: {
      ocsp: string[];
      caIssuers: string[];
    };
  };
  
  // Signature
  signature: Buffer;
}
```

### Obtaining Certificates

```typescript
// Let's Encrypt with ACME protocol
import * as acme from 'acme-client';

class CertificateManager {
  private client: acme.Client;

  async obtainCertificate(domains: string[]): Promise<{
    certificate: string;
    privateKey: string;
  }> {
    // 1. Create ACME client
    const accountPrivateKey = await acme.forge.createPrivateKey();
    
    this.client = new acme.Client({
      directoryUrl: acme.directory.letsencrypt.production,
      accountKey: accountPrivateKey
    });

    // 2. Create account
    await this.client.createAccount({
      termsOfServiceAgreed: true,
      contact: ['mailto:admin@example.com']
    });

    // 3. Create CSR (Certificate Signing Request)
    const [key, csr] = await acme.forge.createCsr({
      commonName: domains[0],
      altNames: domains
    });

    // 4. Request certificate
    const cert = await this.client.auto({
      csr,
      email: 'admin@example.com',
      termsOfServiceAgreed: true,
      challengeCreateFn: this.createChallenge.bind(this),
      challengeRemoveFn: this.removeChallenge.bind(this)
    });

    return {
      certificate: cert.toString(),
      privateKey: key.toString()
    };
  }

  // HTTP-01 challenge
  private async createChallenge(
    authz: any,
    challenge: any,
    keyAuthorization: string
  ): Promise<void> {
    // Serve challenge at: http://example.com/.well-known/acme-challenge/{token}
    const token = challenge.token;
    
    // Store for HTTP server to serve
    await this.storeChallengeResponse(token, keyAuthorization);
    
    console.log(`HTTP challenge created: ${token}`);
  }

  private async removeChallenge(
    authz: any,
    challenge: any,
    keyAuthorization: string
  ): Promise<void> {
    const token = challenge.token;
    await this.removeChallengeResponse(token);
    console.log(`HTTP challenge removed: ${token}`);
  }

  // Auto-renewal
  async setupAutoRenewal(domains: string[]): Promise<void> {
    setInterval(async () => {
      const cert = await this.loadCertificate();
      const expiryDate = this.getCertificateExpiry(cert);
      const daysUntilExpiry = this.getDaysUntil(expiryDate);

      // Renew 30 days before expiry
      if (daysUntilExpiry <= 30) {
        console.log('Renewing certificate...');
        const newCert = await this.obtainCertificate(domains);
        await this.installCertificate(newCert);
        console.log('Certificate renewed successfully');
      }
    }, 24 * 60 * 60 * 1000); // Check daily
  }
}
```

### Node.js HTTPS Server Configuration

```typescript
// Express HTTPS server
import express from 'express';
import https from 'https';
import fs from 'fs';

const app = express();

// Load SSL/TLS certificates
const options: https.ServerOptions = {
  key: fs.readFileSync('/path/to/private.key'),
  cert: fs.readFileSync('/path/to/certificate.crt'),
  ca: fs.readFileSync('/path/to/ca-bundle.crt'), // Certificate chain
  
  // TLS configuration
  minVersion: 'TLSv1.2',  // Minimum TLS version
  ciphers: [
    'ECDHE-RSA-AES256-GCM-SHA384',
    'ECDHE-RSA-AES128-GCM-SHA256',
    'ECDHE-RSA-CHACHA20-POLY1305'
  ].join(':'),
  honorCipherOrder: true,  // Prefer server cipher order
  
  // Enable OCSP stapling
  requestCert: false,
  rejectUnauthorized: true
};

// Create HTTPS server
const httpsServer = https.createServer(options, app);

// HTTP to HTTPS redirect
const httpApp = express();
httpApp.use((req, res) => {
  res.redirect(301, `https://${req.headers.host}${req.url}`);
});

httpApp.listen(80);
httpsServer.listen(443);

console.log('HTTPS server running on port 443');
console.log('HTTP redirect server running on port 80');
```

## HSTS (HTTP Strict Transport Security)

### HSTS Header Configuration

```typescript
// HSTS implementation
class HSTSMiddleware {
  // Basic HSTS
  static basic() {
    return (req: express.Request, res: express.Response, next: express.NextFunction) => {
      res.setHeader(
        'Strict-Transport-Security',
        'max-age=31536000'  // 1 year
      );
      next();
    };
  }

  // HSTS with subdomains
  static withSubdomains() {
    return (req: express.Request, res: express.Response, next: express.NextFunction) => {
      res.setHeader(
        'Strict-Transport-Security',
        'max-age=31536000; includeSubDomains'
      );
      next();
    };
  }

  // HSTS with preload
  static withPreload() {
    return (req: express.Request, res: express.Response, next: express.NextFunction) => {
      res.setHeader(
        'Strict-Transport-Security',
        'max-age=63072000; includeSubDomains; preload'  // 2 years
      );
      next();
    };
  }

  // Conditional HSTS (development vs production)
  static conditional() {
    return (req: express.Request, res: express.Response, next: express.NextFunction) => {
      if (process.env.NODE_ENV === 'production') {
        res.setHeader(
          'Strict-Transport-Security',
          'max-age=31536000; includeSubDomains; preload'
        );
      }
      next();
    };
  }
}

// Usage
app.use(HSTSMiddleware.withPreload());

// Using Helmet middleware
import helmet from 'helmet';

app.use(helmet.hsts({
  maxAge: 31536000,  // 1 year in seconds
  includeSubDomains: true,
  preload: true
}));
```

### HSTS Preload List

```typescript
// Steps to join HSTS preload list
class HSTSPreload {
  /*
   * Requirements:
   * 1. Serve valid HTTPS certificate
   * 2. Redirect HTTP to HTTPS (all subdomains)
   * 3. Serve HSTS header on all subdomains:
   *    - max-age >= 31536000 (1 year)
   *    - includeSubDomains directive
   *    - preload directive
   * 4. Submit domain at hstspreload.org
   */

  static checkEligibility(domain: string): PreloadEligibility {
    return {
      hasValidCert: this.hasValidCertificate(domain),
      redirectsHTTP: this.checksHTTPRedirect(domain),
      hasHSTSHeader: this.hasProperHSTSHeader(domain),
      includesSubdomains: this.includesSubdomainDirective(domain),
      hasPreloadDirective: this.hasPreloadDirective(domain),
      meetsMaxAge: this.meetsMaxAgeRequirement(domain)
    };
  }

  static async submitToPreload(domain: string): Promise<void> {
    const eligibility = this.checkEligibility(domain);
    
    if (!Object.values(eligibility).every(v => v)) {
      throw new Error('Domain not eligible for HSTS preload');
    }

    // Submit at https://hstspreload.org
    console.log(`Domain ${domain} is eligible for HSTS preload`);
    console.log('Submit at: https://hstspreload.org');
  }
}
```

### HSTS Benefits and Risks

```
┌─────────────────────────────────────────────────────────────┐
│              HSTS Impact                                     │
└─────────────────────────────────────────────────────────────┘

Benefits:
  ✓ Prevents SSL stripping attacks
  ✓ Automatic HTTPS upgrade
  ✓ No HTTP requests (even first visit with preload)
  ✓ Protection against downgrade attacks
  ✓ Better SEO (search engines prefer HTTPS)

Risks:
  ✗ Cannot easily revert to HTTP
  ✗ max-age timer must expire
  ✗ Affects all subdomains (with includeSubDomains)
  ✗ Preload is nearly permanent (6+ months to remove)
  ✗ Certificate issues lock users out completely

Recommendation:
  1. Start with short max-age (e.g., 1 day)
  2. Gradually increase (1 week → 1 month → 1 year)
  3. Monitor for issues
  4. Only add includeSubDomains when ready
  5. Only submit for preload when confident
```

## Mixed Content Warnings

### Types of Mixed Content

```typescript
// Mixed content occurs when HTTPS page loads HTTP resources

interface MixedContentTypes {
  // Active mixed content (BLOCKED by browsers)
  active: {
    scripts: '<script src="http://example.com/script.js">',
    stylesheets: '<link rel="stylesheet" href="http://example.com/style.css">',
    iframes: '<iframe src="http://example.com">',
    xhr: 'fetch("http://example.com/api")',
    websockets: 'new WebSocket("ws://example.com")',
    objects: '<object data="http://example.com/file.pdf">'
  };
  
  // Passive mixed content (WARNING, not blocked)
  passive: {
    images: '<img src="http://example.com/image.jpg">',
    audio: '<audio src="http://example.com/sound.mp3">',
    video: '<video src="http://example.com/video.mp4">'
  };
}
```

### Fixing Mixed Content

```typescript
// Solutions for mixed content issues

// 1. Use protocol-relative URLs (legacy, not recommended)
// ❌ Deprecated approach
const legacyURL = '//example.com/resource.js';
// <script src="//example.com/resource.js"></script>

// 2. Use HTTPS explicitly (recommended)
// ✓ Best practice
const secureURL = 'https://example.com/resource.js';
// <script src="https://example.com/resource.js"></script>

// 3. Upgrade Insecure Requests (CSP directive)
app.use((req, res, next) => {
  res.setHeader(
    'Content-Security-Policy',
    'upgrade-insecure-requests'
  );
  next();
});
// Browser automatically upgrades HTTP to HTTPS

// 4. Fix in React/Angular applications
class SecureResourceLoader {
  // React: Ensure HTTPS URLs
  static sanitizeURL(url: string): string {
    if (url.startsWith('http://')) {
      return url.replace('http://', 'https://');
    }
    return url;
  }

  // Component
  static ImageComponent: React.FC<{ src: string }> = ({ src }) => {
    const secureSrc = SecureResourceLoader.sanitizeURL(src);
    return <img src={secureSrc} alt="Secure image" />;
  };
}

// 5. Detect mixed content in development
class MixedContentDetector {
  static detectMixedContent(): void {
    if (window.location.protocol === 'https:') {
      // Listen for mixed content warnings
      const observer = new PerformanceObserver((list) => {
        for (const entry of list.getEntries()) {
          if (entry.name.startsWith('http://')) {
            console.warn('Mixed content detected:', entry.name);
          }
        }
      });

      observer.observe({ entryTypes: ['resource'] });
    }
  }

  // Report mixed content to server
  static reportMixedContent(url: string): void {
    fetch('/api/report-mixed-content', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ url, page: window.location.href })
    });
  }
}
```

## SSL/TLS Versions and Cipher Suites

### Protocol Versions

```
┌─────────────────────────────────────────────────────────────┐
│              TLS Version Comparison                          │
└─────────────────────────────────────────────────────────────┘

┌──────────┬─────────┬──────────┬─────────────────────────┐
│ Protocol │  Year   │  Status  │       Notes             │
├──────────┼─────────┼──────────┼─────────────────────────┤
│ SSL 2.0  │  1995   │  INSECURE│  Never use              │
│ SSL 3.0  │  1996   │  INSECURE│  POODLE attack          │
│ TLS 1.0  │  1999   │  DEPRECATED  Weak by modern standards │
│ TLS 1.1  │  2006   │  DEPRECATED  Remove support         │
│ TLS 1.2  │  2008   │  ✓ SECURE│  Minimum recommended    │
│ TLS 1.3  │  2018   │  ✓ BEST  │  Preferred              │
└──────────┴─────────┴──────────┴─────────────────────────┘
```

### Cipher Suite Configuration

```typescript
// Recommended cipher suites
const recommendedCiphers = {
  // TLS 1.3 (automatic, secure by default)
  tls13: [
    'TLS_AES_256_GCM_SHA384',
    'TLS_AES_128_GCM_SHA256',
    'TLS_CHACHA20_POLY1305_SHA256'
  ],
  
  // TLS 1.2 (for compatibility)
  tls12: [
    // ECDHE provides forward secrecy
    'ECDHE-RSA-AES256-GCM-SHA384',
    'ECDHE-RSA-AES128-GCM-SHA256',
    'ECDHE-RSA-CHACHA20-POLY1305',
    
    // ECDSA (if using ECDSA certificates)
    'ECDHE-ECDSA-AES256-GCM-SHA384',
    'ECDHE-ECDSA-AES128-GCM-SHA256',
    'ECDHE-ECDSA-CHACHA20-POLY1305'
  ],
  
  // Weak ciphers to avoid
  weak: [
    'RC4',      // Broken
    'DES',      // Too weak
    '3DES',     // Deprecated
    'MD5',      // Broken
    'NULL',     // No encryption!
    'EXPORT',   // Intentionally weakened
    'anon'      // No authentication
  ]
};

// Node.js configuration
const tlsOptions = {
  minVersion: 'TLSv1.2',
  maxVersion: 'TLSv1.3',
  ciphers: [
    ...recommendedCiphers.tls13,
    ...recommendedCiphers.tls12
  ].join(':'),
  honorCipherOrder: true,  // Server chooses cipher
  ecdhCurve: 'auto'  // Prefer X25519
};

// Nginx configuration
/*
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers 'TLS_AES_256_GCM_SHA384:TLS_AES_128_GCM_SHA256:ECDHE-RSA-AES256-GCM-SHA384';
ssl_prefer_server_ciphers on;
*/
```

### Testing TLS Configuration

```typescript
// Test TLS configuration
class TLSConfigurationTester {
  async testConfiguration(domain: string): Promise<TLSTestResults> {
    return {
      supportedVersions: await this.testSupportedVersions(domain),
      cipherSuites: await this.testCipherSuites(domain),
      certificateValidity: await this.testCertificate(domain),
      vulnerabilities: await this.testVulnerabilities(domain),
      grade: await this.calculateGrade(domain)
    };
  }

  private async testSupportedVersions(domain: string): Promise<string[]> {
    const versions = ['TLSv1', 'TLSv1.1', 'TLSv1.2', 'TLSv1.3'];
    const supported = [];

    for (const version of versions) {
      if (await this.supportsVersion(domain, version)) {
        supported.push(version);
      }
    }

    return supported;
  }

  private async testVulnerabilities(domain: string): Promise<Vulnerability[]> {
    const vulnerabilities = [];

    // Test for common vulnerabilities
    if (await this.testHeartbleed(domain)) {
      vulnerabilities.push({
        name: 'Heartbleed',
        severity: 'CRITICAL',
        cvss: 10.0
      });
    }

    if (await this.testPOODLE(domain)) {
      vulnerabilities.push({
        name: 'POODLE',
        severity: 'HIGH',
        cvss: 7.5
      });
    }

    if (await this.testBEAST(domain)) {
      vulnerabilities.push({
        name: 'BEAST',
        severity: 'MEDIUM',
        cvss: 5.0
      });
    }

    return vulnerabilities;
  }

  // Use SSL Labs API for comprehensive testing
  async sslLabsTest(domain: string): Promise<any> {
    const response = await fetch(
      `https://api.ssllabs.com/api/v3/analyze?host=${domain}`
    );
    return response.json();
  }
}

// Testing tools:
// - https://www.ssllabs.com/ssltest/
// - testssl.sh (command-line)
// - nmap with ssl-enum-ciphers script
```

## Certificate Pinning

### HTTP Public Key Pinning (HPKP) - Deprecated

```typescript
// HPKP is deprecated due to risks, but concept is important

// ❌ HPKP (deprecated, don't use)
res.setHeader(
  'Public-Key-Pins',
  'pin-sha256="base64=="; pin-sha256="backup-base64=="; max-age=5184000; includeSubDomains'
);
// Risk: Misconfiguration can lock users out permanently

// ✓ Better alternative: Certificate Transparency + Expect-CT
res.setHeader(
  'Expect-CT',
  'max-age=86400, enforce, report-uri="https://example.com/ct-report"'
);
```

### Application-Level Certificate Pinning

```typescript
// Mobile app certificate pinning (conceptual)
class CertificatePinning {
  private pinnedFingerprints = [
    'sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=',
    'sha256/BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB='  // Backup
  ];

  async validateCertificate(cert: X509Certificate): Promise<boolean> {
    // Compute certificate fingerprint
    const fingerprint = this.computeFingerprint(cert);
    
    // Check against pinned fingerprints
    if (!this.pinnedFingerprints.includes(fingerprint)) {
      throw new Error('Certificate pinning validation failed');
    }

    return true;
  }

  private computeFingerprint(cert: X509Certificate): string {
    // Hash the Subject Public Key Info (SPKI)
    const spki = cert.publicKey.exportSPKI();
    const hash = crypto.createHash('sha256').update(spki).digest('base64');
    return `sha256/${hash}`;
  }

  // React Native example (pseudo-code)
  static configureReactNative() {
    /*
    import { SSLPinning } from 'react-native-ssl-pinning';

    SSLPinning.fetch('https://api.example.com/data', {
      method: 'GET',
      pkPinning: true,
      sslPinning: {
        certs: ['certificate1', 'certificate2']  // From assets
      }
    });
    */
  }
}
```

## Common Mistakes

### 1. Using Self-Signed Certificates in Production

```typescript
// ❌ BAD: Self-signed certificate
// Triggers browser warnings
// No chain of trust

// ✓ GOOD: Use Let's Encrypt or commercial CA
const certManager = new CertificateManager();
const cert = await certManager.obtainCertificate(['example.com']);
```

### 2. Weak TLS Configuration

```typescript
// ❌ BAD: Allowing old protocols and weak ciphers
const weakConfig = {
  minVersion: 'TLSv1',  // Too old!
  ciphers: 'ALL',  // Includes weak ciphers
  honorCipherOrder: false
};

// ✓ GOOD: Strong configuration
const strongConfig = {
  minVersion: 'TLSv1.2',
  maxVersion: 'TLSv1.3',
  ciphers: recommendedCiphers.join(':'),
  honorCipherOrder: true
};
```

### 3. Not Setting HSTS Header

```typescript
// ❌ BAD: No HSTS header
// Vulnerable to SSL stripping

// ✓ GOOD: Set HSTS header
app.use((req, res, next) => {
  res.setHeader(
    'Strict-Transport-Security',
    'max-age=31536000; includeSubDomains; preload'
  );
  next();
});
```

### 4. Mixed Content Issues

```typescript
// ❌ BAD: HTTP resources on HTTPS page
<script src="http://cdn.example.com/library.js"></script>

// ✓ GOOD: HTTPS resources
<script src="https://cdn.example.com/library.js"></script>

// ✓ GOOD: CSP upgrade-insecure-requests
res.setHeader('Content-Security-Policy', 'upgrade-insecure-requests');
```

### 5. Expired Certificates

```typescript
// ❌ BAD: No monitoring or auto-renewal
// Certificate expires, site goes down

// ✓ GOOD: Automated monitoring and renewal
class CertificateMonitor {
  async setupMonitoring(): Promise<void> {
    setInterval(async () => {
      const cert = await this.loadCertificate();
      const daysUntilExpiry = this.getDaysUntilExpiry(cert);
      
      if (daysUntilExpiry <= 30) {
        await this.sendExpiryAlert(daysUntilExpiry);
      }
      
      if (daysUntilExpiry <= 7) {
        await this.autoRenewCertificate();
      }
    }, 24 * 60 * 60 * 1000); // Daily check
  }
}
```

## Best Practices

### 1. Use Automated Certificate Management

```typescript
// Automated certificate management with Let's Encrypt
import * as greenlock from 'greenlock-express';

const app = greenlock.init({
  packageRoot: __dirname,
  configDir: './greenlock.d',
  maintainerEmail: 'admin@example.com',
  cluster: false
}).serve(expressApp);

// Certificates automatically obtained and renewed
```

### 2. Implement Complete HTTPS Strategy

```typescript
// Comprehensive HTTPS configuration
class HTTPSStrategy {
  static implement(app: express.Application): void {
    // 1. Force HTTPS redirect
    app.use((req, res, next) => {
      if (!req.secure && process.env.NODE_ENV === 'production') {
        return res.redirect(301, `https://${req.headers.host}${req.url}`);
      }
      next();
    });

    // 2. Set HSTS header
    app.use((req, res, next) => {
      if (req.secure) {
        res.setHeader(
          'Strict-Transport-Security',
          'max-age=31536000; includeSubDomains; preload'
        );
      }
      next();
    });

    // 3. Upgrade insecure requests
    app.use((req, res, next) => {
      res.setHeader('Content-Security-Policy', 'upgrade-insecure-requests');
      next();
    });

    // 4. Additional security headers
    app.use(helmet());
  }
}

HTTPSStrategy.implement(app);
```

### 3. Regular Security Audits

```typescript
// Automated security testing
class SecurityAuditor {
  async runAudits(): Promise<AuditResults> {
    return {
      tls: await this.auditTLS(),
      certificate: await this.auditCertificate(),
      headers: await this.auditSecurityHeaders(),
      mixedContent: await this.checkMixedContent()
    };
  }

  async auditTLS(): Promise<TLSAuditResult> {
    // Use SSL Labs API
    const results = await fetch(
      `https://api.ssllabs.com/api/v3/analyze?host=${this.domain}`
    );
    
    return results.json();
  }

  async auditCertificate(): Promise<CertificateAudit> {
    const cert = await this.getCertificate();
    
    return {
      valid: this.isCertificateValid(cert),
      daysUntilExpiry: this.getDaysUntilExpiry(cert),
      chain: this.validateCertificateChain(cert),
      revocation: await this.checkRevocation(cert)
    };
  }

  // Schedule regular audits
  scheduleAudits(): void {
    setInterval(async () => {
      const results = await this.runAudits();
      
      if (results.hasIssues()) {
        await this.alertSecurityTeam(results);
      }
      
      await this.storeAuditResults(results);
    }, 7 * 24 * 60 * 60 * 1000); // Weekly
  }
}
```

## When to Use/Not to Use

### When HTTPS is Required

**Always use HTTPS for:**
1. **Authentication pages** - Login, registration, password reset
2. **Payment processing** - Any financial transactions
3. **Personal data** - Forms collecting sensitive information
4. **API endpoints** - Especially with authentication
5. **Admin panels** - Backend administration interfaces
6. **All pages** - Modern best practice: HTTPS everywhere

**HTTPS is NOT optional:**
- PCI DSS requires HTTPS for payment processing
- Modern browsers flag HTTP as "Not Secure"
- SEO benefits (Google ranks HTTPS higher)
- Required for modern APIs (Service Workers, Geolocation, etc.)

### HTTP Acceptable (Rare Cases)

**HTTP might be okay for:**
- Internal development servers (with proper network isolation)
- Localhost development
- Public read-only content with no user interaction
- Legacy systems in controlled environments

**But consider:** Even these benefit from HTTPS

## Interview Questions

### Q1: Explain the TLS handshake process.

**Answer:** TLS handshake establishes secure connection:

**TLS 1.3 handshake (simplified):**

1. **ClientHello:** Client sends supported TLS versions, cipher suites, random number, and initial key share
2. **ServerHello:** Server selects cipher suite, sends certificate, key share, and encrypted handshake
3. **Client Finished:** Client verifies certificate, derives session keys, sends encrypted confirmation
4. **Application Data:** Both sides can now send encrypted data

**Key points:**
- **Certificate verification:** Client validates server's certificate chain to trusted root CA
- **Key exchange:** Diffie-Hellman used to establish shared secret
- **Forward secrecy:** Session keys derived, protecting past sessions if private key compromised
- **TLS 1.3 benefits:** Fewer round trips (faster), mandatory forward secrecy, encrypted handshake

**TLS 1.3 vs 1.2:** TLS 1.3 requires 1-RTT vs 2-RTT for TLS 1.2, improving performance.

### Q2: What is HSTS and why is it important?

**Answer:** HTTP Strict Transport Security (HSTS) forces browsers to use HTTPS.

**How it works:**
```
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

**Benefits:**
1. **Prevents SSL stripping:** Attacker can't downgrade to HTTP
2. **No HTTP requests:** Browser automatically uses HTTPS
3. **Protection persists:** Works even on first visit (with preload)

**Directives:**
- **max-age:** How long browser remembers (1 year = 31536000 seconds)
- **includeSubDomains:** Apply to all subdomains
- **preload:** Eligible for browser preload list

**Risks:**
- Cannot easily revert to HTTP during max-age period
- includeSubDomains affects ALL subdomains
- Preload is nearly permanent (6+ months to remove)

**Best practice:**
1. Start with short max-age (1 day)
2. Test thoroughly
3. Gradually increase max-age
4. Add includeSubDomains when ready
5. Submit for preload only when confident

### Q3: What is mixed content and how do you fix it?

**Answer:** Mixed content occurs when HTTPS page loads HTTP resources.

**Types:**

**Active (blocked):**
- Scripts, stylesheets, iframes, XHR/Fetch, WebSockets
- Allows attacker to modify page or steal data
- Browsers block completely

**Passive (warning):**
- Images, audio, video
- Less dangerous but still insecure
- Browsers show warning

**Solutions:**

**1. Use HTTPS URLs:**
```html
<!-- ❌ Mixed content -->
<script src="http://example.com/script.js"></script>

<!-- ✓ Fixed -->
<script src="https://example.com/script.js"></script>
```

**2. CSP upgrade-insecure-requests:**
```
Content-Security-Policy: upgrade-insecure-requests
```
Browser automatically upgrades HTTP to HTTPS.

**3. Remove protocol (legacy):**
```html
<script src="//example.com/script.js"></script>
```
Uses same protocol as page (not recommended).

**Detection:** Browser DevTools Console shows mixed content warnings.

### Q4: Explain forward secrecy in TLS.

**Answer:** Forward secrecy (Perfect Forward Secrecy/PFS) ensures past sessions remain secure even if private key is compromised.

**How it works:**

**Without forward secrecy (RSA key exchange):**
1. Client encrypts session key with server's public key
2. Server decrypts with private key
3. If attacker records traffic and later steals private key, can decrypt all past sessions

**With forward secrecy (Diffie-Hellman):**
1. Each session uses unique, ephemeral keys
2. Keys never transmitted, derived from public parameters
3. Keys discarded after session
4. Even if private key stolen, cannot decrypt past sessions

**Cipher suites with forward secrecy:**
- ECDHE-RSA-AES256-GCM-SHA384 (✓ has forward secrecy)
- RSA-AES256-GCM-SHA384 (✗ no forward secrecy)

Look for "ECDHE" or "DHE" in cipher suite name.

**TLS 1.3:** Forward secrecy mandatory (all cipher suites use it).

**Why important:** Protects historical data if long-term keys compromised.

### Q5: How do you ensure certificate validity programmatically?

**Answer:** Multi-step certificate validation:

**1. Check validity period:**
```typescript
const now = new Date();
if (now < cert.notBefore || now > cert.notAfter) {
  throw new Error('Certificate expired or not yet valid');
}
```

**2. Verify certificate chain:**
```typescript
// Check each certificate signed by next in chain
let current = cert;
while (!current.isSelfSigned()) {
  const issuer = findIssuer(current);
  if (!verifySignature(current, issuer.publicKey)) {
    throw new Error('Invalid signature');
  }
  current = issuer;
}

// Verify root is trusted
if (!isTrustedRoot(current)) {
  throw new Error('Untrusted root certificate');
}
```

**3. Verify hostname:**
```typescript
// Check Common Name or Subject Alternative Names
if (!cert.subjectAltNames.includes(hostname)) {
  throw new Error('Hostname mismatch');
}
```

**4. Check revocation:**
```typescript
// OCSP (Online Certificate Status Protocol)
const ocspResponse = await fetchOCSP(cert.ocspURL, cert);
if (ocspResponse.status === 'revoked') {
  throw new Error('Certificate revoked');
}

// Or CRL (Certificate Revocation List)
const crl = await fetchCRL(cert.crlURL);
if (crl.contains(cert.serialNumber)) {
  throw new Error('Certificate revoked');
}
```

**5. Verify key usage:**
```typescript
if (!cert.extKeyUsage.includes('serverAuth')) {
  throw new Error('Certificate not valid for server authentication');
}
```

**Node.js:** Most validation automatic with `https` module, but can access certificate with:
```typescript
const req = https.request(options, (res) => {
  const cert = res.socket.getPeerCertificate();
  // Validate as needed
});
```

### Q6: What are the differences between TLS 1.2 and TLS 1.3?

**Answer:** TLS 1.3 major improvements:

**1. Performance:**
- TLS 1.3: 1-RTT handshake (faster)
- TLS 1.2: 2-RTT handshake
- TLS 1.3: 0-RTT possible for resumption

**2. Security:**
- Removed weak cipher suites (RSA key exchange, CBC mode)
- Forward secrecy mandatory (ECDHE required)
- Handshake always encrypted (better privacy)

**3. Simplification:**
- TLS 1.3: 5 cipher suites
- TLS 1.2: 37+ cipher suites
- Easier to configure securely

**4. Cipher suites:**
```
TLS 1.3:
- TLS_AES_256_GCM_SHA384
- TLS_AES_128_GCM_SHA256
- TLS_CHACHA20_POLY1305_SHA256

TLS 1.2 (many, including):
- ECDHE-RSA-AES256-GCM-SHA384
- ECDHE-RSA-AES128-GCM-SHA256
```

**5. Downgrade protection:**
- TLS 1.3 includes downgrade attack detection

**Recommendation:** Use TLS 1.3 where possible, maintain TLS 1.2 for compatibility.

### Q7: How do you handle certificate renewal without downtime?

**Answer:** Zero-downtime certificate renewal:

**1. Automated renewal (Let's Encrypt + certbot):**
```bash
# Certbot automatically renews 30 days before expiry
certbot renew --deploy-hook "systemctl reload nginx"
```

**2. Graceful reload:**
```typescript
// Node.js: Reload certificates without restarting
import { readFileSync } from 'fs';

let tlsOptions = loadTLSOptions();

// Watch for certificate changes
watchCertificates((newCert, newKey) => {
  tlsOptions.cert = newCert;
  tlsOptions.key = newKey;
  
  // Server automatically uses new certificates for new connections
  // Existing connections continue with old certificates
});

// No server restart needed
```

**3. Load balancer rotation:**
```
1. Update certificate on Server A
2. Reload Server A (now uses new cert)
3. Update certificate on Server B
4. Reload Server B
5. Both servers now have new certificate
6. Zero downtime
```

**4. DNS-based validation:**
```typescript
// Renew certificate without touching web server
// Certbot uses DNS challenge instead of HTTP challenge
certbot certonly --dns-route53 -d example.com
```

**5. Monitoring:**
```typescript
// Alert before expiry
const daysUntilExpiry = getCertificateExpiry();
if (daysUntilExpiry <= 30) {
  sendAlert('Certificate expiring soon');
}

// Auto-renew at 30 days
if (daysUntilExpiry <= 30) {
  await renewCertificate();
}
```

**Best practice:** Automated renewal with monitoring and alerting.

### Q8: What security headers should you set for HTTPS sites?

**Answer:** Essential security headers:

**1. HSTS:**
```
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```
Forces HTTPS usage.

**2. CSP:**
```
Content-Security-Policy: default-src 'self'; script-src 'self' 'strict-dynamic'
```
Prevents XSS attacks.

**3. X-Content-Type-Options:**
```
X-Content-Type-Options: nosniff
```
Prevents MIME sniffing.

**4. X-Frame-Options:**
```
X-Frame-Options: DENY
```
Prevents clickjacking.

**5. Referrer-Policy:**
```
Referrer-Policy: strict-origin-when-cross-origin
```
Controls referrer information.

**6. Permissions-Policy:**
```
Permissions-Policy: geolocation=(), microphone=(), camera=()
```
Controls feature access.

**Implementation:**
```typescript
import helmet from 'helmet';

app.use(helmet({
  hsts: {
    maxAge: 31536000,
    includeSubDomains: true,
    preload: true
  },
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'", "'strict-dynamic'"]
    }
  },
  xContentTypeOptions: true,
  xFrameOptions: { action: 'deny' }
}));
```

**Testing:** Use securityheaders.com to audit headers.

## Key Takeaways

1. **Always use HTTPS in production** - Essential for security, privacy, and modern web features; HTTP is deprecated
2. **TLS 1.2 minimum, prefer TLS 1.3** - Disable old protocols (SSL, TLS 1.0/1.1); use strong cipher suites with forward secrecy
3. **Implement HSTS with preload** - Prevent SSL stripping attacks; gradually increase max-age; submit for preload when confident
4. **Automate certificate management** - Use Let's Encrypt or similar; implement auto-renewal; monitor expiration dates
5. **Fix all mixed content** - Use HTTPS for all resources; implement upgrade-insecure-requests CSP directive
6. **Validate certificates properly** - Check validity period, chain, hostname, and revocation status
7. **Use strong cipher suites** - Prefer ECDHE for forward secrecy; avoid weak ciphers (RC4, DES, 3DES, MD5)
8. **Set comprehensive security headers** - HSTS, CSP, X-Frame-Options, X-Content-Type-Options for defense in depth
9. **Monitor and audit regularly** - Use SSL Labs for testing; set up expiration alerts; conduct security audits
10. **Plan for certificate rotation** - Implement zero-downtime renewal; use graceful reloads; maintain backup certificates

## Resources

### Official Documentation
- [RFC 8446: TLS 1.3](https://tools.ietf.org/html/rfc8446)
- [RFC 6797: HSTS](https://tools.ietf.org/html/rfc6797)
- [MDN: Mixed Content](https://developer.mozilla.org/en-US/docs/Web/Security/Mixed_content)

### Tools and Services
- [SSL Labs SSL Test](https://www.ssllabs.com/ssltest/) - Comprehensive TLS testing
- [Let's Encrypt](https://letsencrypt.org/) - Free SSL/TLS certificates
- [Certbot](https://certbot.eff.org/) - Automatic certificate management
- [testssl.sh](https://testssl.sh/) - Command-line SSL/TLS testing

### Libraries
- [acme-client](https://www.npmjs.com/package/acme-client) - ACME protocol client for Node.js
- [greenlock-express](https://www.npmjs.com/package/greenlock-express) - Automated Let's Encrypt for Express
- [helmet](https://helmetjs.github.io/) - Security headers middleware

### Articles and Guides
- [Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/)
- [OWASP Transport Layer Protection Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Transport_Layer_Protection_Cheat_Sheet.html)
- [High Performance Browser Networking](https://hpbn.co/) - Chapter on TLS

### Testing and Monitoring
- [Certificate Transparency Logs](https://crt.sh/)
- [HSTS Preload List](https://hstspreload.org/)
- [Security Headers](https://securityheaders.com/)
