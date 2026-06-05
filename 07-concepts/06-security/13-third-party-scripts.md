# Third-Party Scripts Security

## The Idea

**In plain English:** When you add a third-party script (like an analytics tool or a chat widget) to your site, you're giving that external code full access to your page. If that script is compromised, every visitor to your site is at risk — so you need to load it carefully and limit what it can do.

**Real-world analogy:** Think of hiring a contractor to work in your home. You trust a reputable firm (a well-known vendor), but you still don't give them the keys to your safe (sensitive user data). You check their ID when they arrive (Subresource Integrity hash) and only let them into the rooms they need access to (Content Security Policy).

- Hiring the contractor = loading a `<script src="...">` from a third party
- Checking their ID = using a Subresource Integrity (`integrity=`) attribute
- Restricting which rooms they enter = Content Security Policy limiting script sources
- Finding out the firm was compromised = a supply-chain attack on the vendor's CDN

---

## Table of Contents
- [Introduction](#introduction)
- [Third-Party Security Risks](#third-party-security-risks)
- [Subresource Integrity (SRI)](#subresource-integrity-sri)
- [Iframe Sandboxing](#iframe-sandboxing)
- [CSP for Third-Party Scripts](#csp-for-third-party-scripts)
- [Vendor Security Assessment](#vendor-security-assessment)
- [Implementation Examples](#implementation-examples)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [When to Use/Not to Use](#when-to-usenot-to-use)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

**Third-party scripts** are external JavaScript files loaded from domains you don't control. They're common in modern web applications (analytics, ads, social widgets) but introduce significant security risks.

### The Third-Party Problem

```
┌─────────────────────────────────────────────────────────────┐
│          Third-Party Script Landscape                        │
└─────────────────────────────────────────────────────────────┘

Your Web Application
      ├── Google Analytics  ───> Tracks user behavior
      ├── Facebook SDK      ───> Social sharing
      ├── Stripe.js         ───> Payment processing
      ├── Intercom          ───> Customer support chat
      ├── Google Ads        ───> Advertising
      └── Hotjar            ───> Session recording

Each script:
  ✓ Runs in your page's context
  ✓ Has full access to DOM
  ✓ Can read cookies
  ✓ Can make requests
  ✓ Can modify your page
  ✗ You don't control the code
  ✗ Could be compromised
  ✗ Could change without notice

Average website: 20-30 third-party scripts
Each is a potential security risk!
```

## Third-Party Security Risks

### Attack Vectors

```typescript
// Third-party script risks

interface ThirdPartyRisk {
  // 1. Supply Chain Compromise
  supplyChai: {
    description: 'Third-party service itself is hacked';
    example: 'British Airways Magecart attack (2018)';
    impact: 'Attacker steals credit card data from all sites using script';
  };

  // 2. Man-in-the-Middle (MITM)
  mitm: {
    description: 'Script modified during transmission';
    example: 'ISP injecting ads via HTTP';
    impact: 'Users get malicious version of script';
  };

  // 3. Malicious Updates
  maliciousUpdate: {
    description: 'Legitimate script updates with malicious code';
    example: 'Compromised npm package maintainer';
    impact: 'All users automatically get malicious code';
  };

  // 4. Data Exfiltration
  dataExfiltration: {
    description: 'Script sends sensitive data to third party';
    example: 'Analytics script reading form data';
    impact: 'Passwords, credit cards leaked';
  };

  // 5. Cross-Site Scripting (XSS)
  xss: {
    description: 'Script creates XSS vulnerabilities';
    example: 'Vulnerable widget code';
    impact: 'Attackers execute code on your site';
  };
}
```

### Real-World Incidents

```
┌─────────────────────────────────────────────────────────────┐
│          Notable Third-Party Script Attacks                  │
└─────────────────────────────────────────────────────────────┘

British Airways (2018):
  • Magecart attack via third-party payment script
  • 380,000 payment card details stolen
  • £183 million GDPR fine
  
Ticketmaster (2018):
  • Compromised Inbenta chatbot widget
  • 40,000 customers affected
  • £1.25 million fine

Newegg (2018):
  • Magecart via compromised script
  • Credit card skimming
  • One month before detection

Lesson: Third-party scripts are HIGH-RISK!
```

### Security Assessment

```typescript
// Evaluate third-party script risk

class ThirdPartyRiskAssessor {
  async assessScript(scriptUrl: string): Promise<RiskAssessment> {
    return {
      vendor: await this.assessVendor(scriptUrl),
      technical: await this.assessTechnical(scriptUrl),
      data: await this.assessDataHandling(scriptUrl),
      compliance: await this.assessCompliance(scriptUrl),
      overallRisk: this.calculateRisk()
    };
  }

  private async assessVendor(scriptUrl: string): Promise<VendorRisk> {
    const domain = new URL(scriptUrl).hostname;
    
    return {
      reputation: await this.checkReputation(domain),
      securityPractices: await this.checkSecurityPractices(domain),
      incidentHistory: await this.checkIncidentHistory(domain),
      financialStability: await this.checkFinancialStability(domain),
      support: await this.checkSupportQuality(domain)
    };
  }

  private async assessTechnical(scriptUrl: string): Promise<TechnicalRisk> {
    const script = await this.fetchScript(scriptUrl);
    
    return {
      hasHttps: scriptUrl.startsWith('https://'),
      hasSRI: await this.checkSRISupport(scriptUrl),
      cspCompatible: await this.checkCSPCompatibility(script),
      codeQuality: await this.analyzeCode(script),
      dependencies: await this.analyzeDependencies(script),
      obfuscation: this.detectObfuscation(script)
    };
  }

  private async assessDataHandling(scriptUrl: string): Promise<DataRisk> {
    return {
      accessesDOM: this.checkDOMAccess(await this.fetchScript(scriptUrl)),
      readsCookies: this.checkCookieAccess(await this.fetchScript(scriptUrl)),
      externalRequests: await this.trackExternalRequests(scriptUrl),
      localStorage: this.checkLocalStorageAccess(await this.fetchScript(scriptUrl)),
      formData: this.checkFormAccess(await this.fetchScript(scriptUrl))
    };
  }

  private async assessCompliance(scriptUrl: string): Promise<ComplianceRisk> {
    return {
      gdprCompliant: await this.checkGDPRCompliance(scriptUrl),
      privacyPolicy: await this.checkPrivacyPolicy(scriptUrl),
      dataProcessingAgreement: await this.checkDPA(scriptUrl),
      subProcessors: await this.checkSubProcessors(scriptUrl)
    };
  }

  private calculateRisk(): 'LOW' | 'MEDIUM' | 'HIGH' | 'CRITICAL' {
    // Risk scoring algorithm
    // Consider all factors and return overall risk
    return 'MEDIUM';
  }
}
```

## Subresource Integrity (SRI)

### SRI Implementation

```typescript
// Subresource Integrity: Verify script integrity

// ❌ BAD: No integrity check
<script src="https://cdn.example.com/library.js"></script>
// If CDN compromised, malicious code executes

// ✓ GOOD: SRI hash verifies integrity
<script
  src="https://cdn.example.com/library.js"
  integrity="sha384-oqVuAfXRKap7fdgcCY5uykM6+R9GqQ8K/uxy9rx7HNQlGYl1kPzQho1wx4JwY8wC"
  crossorigin="anonymous"
></script>
// Browser verifies hash before executing

// Generate SRI hash
import crypto from 'crypto';
import fs from 'fs';

function generateSRIHash(filePath: string, algorithm = 'sha384'): string {
  const fileContent = fs.readFileSync(filePath);
  const hash = crypto
    .createHash(algorithm)
    .update(fileContent)
    .digest('base64');
  
  return `${algorithm}-${hash}`;
}

// Usage
const hash = generateSRIHash('./node_modules/react/umd/react.production.min.js');
console.log(hash);
// Output: sha384-oqVuAfXRKap7fdgcCY5uykM6+R9GqQ8K/uxy9rx7HNQlGYl1kPzQho1wx4JwY8wC

// React component with SRI
const SecureExternalScript: React.FC<{
  src: string;
  integrity: string;
}> = ({ src, integrity }) => {
  useEffect(() => {
    const script = document.createElement('script');
    script.src = src;
    script.integrity = integrity;
    script.crossOrigin = 'anonymous';
    script.async = true;
    
    script.onerror = () => {
      console.error('Script failed to load or integrity check failed');
    };
    
    document.body.appendChild(script);
    
    return () => {
      document.body.removeChild(script);
    };
  }, [src, integrity]);
  
  return null;
};

// Usage
<SecureExternalScript
  src="https://cdn.example.com/library.js"
  integrity="sha384-oqVuAfXRKap7fdgcCY5uykM6+R9GqQ8K..."
/>
```

### Automated SRI Generation

```typescript
// Webpack plugin for automatic SRI

class SRIPlugin {
  apply(compiler: any) {
    compiler.hooks.compilation.tap('SRIPlugin', (compilation: any) => {
      compilation.hooks.processAssets.tap(
        {
          name: 'SRIPlugin',
          stage: compiler.webpack.Compilation.PROCESS_ASSETS_STAGE_OPTIMIZE
        },
        (assets: any) => {
          // Generate SRI hashes for all assets
          const sriHashes = new Map<string, string>();
          
          Object.keys(assets).forEach((filename) => {
            if (filename.endsWith('.js') || filename.endsWith('.css')) {
              const source = assets[filename].source();
              const hash = crypto
                .createHash('sha384')
                .update(source)
                .digest('base64');
              
              sriHashes.set(filename, `sha384-${hash}`);
            }
          });
          
          // Inject SRI into HTML files
          Object.keys(assets).forEach((filename) => {
            if (filename.endsWith('.html')) {
              let content = assets[filename].source();
              
              sriHashes.forEach((hash, file) => {
                // Add integrity attribute to script tags
                const scriptRegex = new RegExp(
                  `<script([^>]*)src=["']([^"']*${file})["']([^>]*)>`,
                  'g'
                );
                
                content = content.replace(
                  scriptRegex,
                  `<script$1src="$2"$3 integrity="${hash}" crossorigin="anonymous">`
                );
                
                // Add integrity attribute to link tags
                const linkRegex = new RegExp(
                  `<link([^>]*)href=["']([^"']*${file})["']([^>]*)>`,
                  'g'
                );
                
                content = content.replace(
                  linkRegex,
                  `<link$1href="$2"$3 integrity="${hash}" crossorigin="anonymous">`
                );
              });
              
              assets[filename] = {
                source: () => content,
                size: () => content.length
              };
            }
          });
        }
      );
    });
  }
}

// Webpack configuration
module.exports = {
  plugins: [
    new SRIPlugin()
  ]
};
```

### SRI with Fallback

```typescript
// Handle SRI failures with fallback

class SRIWithFallback {
  async loadScript(
    primary: { src: string; integrity: string },
    fallback: { src: string; integrity: string }
  ): Promise<void> {
    return new Promise((resolve, reject) => {
      const script = document.createElement('script');
      script.src = primary.src;
      script.integrity = primary.integrity;
      script.crossOrigin = 'anonymous';
      script.async = true;
      
      script.onerror = () => {
        console.warn('Primary script failed, trying fallback');
        
        // Try fallback
        const fallbackScript = document.createElement('script');
        fallbackScript.src = fallback.src;
        fallbackScript.integrity = fallback.integrity;
        fallbackScript.crossOrigin = 'anonymous';
        fallbackScript.async = true;
        
        fallbackScript.onerror = () => {
          reject(new Error('Both primary and fallback scripts failed'));
        };
        
        fallbackScript.onload = () => resolve();
        
        document.body.appendChild(fallbackScript);
      };
      
      script.onload = () => resolve();
      
      document.body.appendChild(script);
    });
  }
}

// Usage
const loader = new SRIWithFallback();

await loader.loadScript(
  {
    src: 'https://cdn1.example.com/library.js',
    integrity: 'sha384-primary-hash'
  },
  {
    src: 'https://cdn2.example.com/library.js',
    integrity: 'sha384-fallback-hash'
  }
);
```

## Iframe Sandboxing

### Sandboxed Iframes

```html
<!-- ❌ BAD: Unsandboxed iframe -->
<iframe src="https://third-party.com/widget"></iframe>
<!-- Third-party has full access -->

<!-- ✓ GOOD: Sandboxed iframe -->
<iframe
  src="https://third-party.com/widget"
  sandbox="allow-scripts allow-same-origin"
></iframe>
<!-- Limited permissions -->

<!-- Sandbox permissions -->
<iframe
  src="https://third-party.com/widget"
  sandbox="
    allow-scripts
    allow-same-origin
    allow-forms
    allow-popups
  "
></iframe>
```

### Sandbox Permissions

```typescript
// Iframe sandbox configuration

interface SandboxPermissions {
  // Allow JavaScript execution
  'allow-scripts': boolean;
  
  // Allow same-origin access
  'allow-same-origin': boolean;
  
  // Allow form submission
  'allow-forms': boolean;
  
  // Allow popups
  'allow-popups': boolean;
  
  // Allow popup to escape sandbox
  'allow-popups-to-escape-sandbox': boolean;
  
  // Allow pointer lock
  'allow-pointer-lock': boolean;
  
  // Allow top navigation
  'allow-top-navigation': boolean;
  
  // Allow modal dialogs
  'allow-modals': boolean;
  
  // Allow automatic features
  'allow-orientation-lock': boolean;
  
  // Allow presentation
  'allow-presentation': boolean;
}

// React component with configurable sandbox
const SandboxedIframe: React.FC<{
  src: string;
  permissions: Array<keyof SandboxPermissions>;
  csp?: string;
}> = ({ src, permissions, csp }) => {
  const sandboxValue = permissions.join(' ');
  
  return (
    <iframe
      src={src}
      sandbox={sandboxValue}
      csp={csp}
      style={{
        border: 'none',
        width: '100%',
        height: '100%'
      }}
    />
  );
};

// Usage: Minimal permissions
<SandboxedIframe
  src="https://third-party.com/widget"
  permissions={['allow-scripts']}
/>

// Usage: Standard widget
<SandboxedIframe
  src="https://third-party.com/widget"
  permissions={[
    'allow-scripts',
    'allow-same-origin',
    'allow-forms'
  ]}
  csp="default-src 'self' https://third-party.com"
/>
```

### PostMessage Communication

```typescript
// Secure communication with sandboxed iframe

// Parent window
class IframeMessenger {
  private iframe: HTMLIFrameElement;
  private trustedOrigin: string;
  private messageHandlers = new Map<string, (data: any) => void>();

  constructor(iframe: HTMLIFrameElement, trustedOrigin: string) {
    this.iframe = iframe;
    this.trustedOrigin = trustedOrigin;
    
    window.addEventListener('message', this.handleMessage.bind(this));
  }

  sendMessage(type: string, data: any): void {
    this.iframe.contentWindow?.postMessage(
      { type, data },
      this.trustedOrigin
    );
  }

  onMessage(type: string, handler: (data: any) => void): void {
    this.messageHandlers.set(type, handler);
  }

  private handleMessage(event: MessageEvent): void {
    // Verify origin
    if (event.origin !== this.trustedOrigin) {
      console.warn('Message from untrusted origin:', event.origin);
      return;
    }

    // Verify source
    if (event.source !== this.iframe.contentWindow) {
      console.warn('Message from unexpected source');
      return;
    }

    // Handle message
    const { type, data } = event.data;
    const handler = this.messageHandlers.get(type);
    
    if (handler) {
      handler(data);
    }
  }

  destroy(): void {
    window.removeEventListener('message', this.handleMessage);
  }
}

// Usage
const iframe = document.getElementById('third-party-widget') as HTMLIFrameElement;
const messenger = new IframeMessenger(iframe, 'https://third-party.com');

// Send message to iframe
messenger.sendMessage('init', {
  userId: '123',
  theme: 'dark'
});

// Receive message from iframe
messenger.onMessage('ready', (data) => {
  console.log('Widget is ready:', data);
});

messenger.onMessage('action', (data) => {
  console.log('User action:', data);
});

// Inside iframe (third-party code)
// Listen for messages from parent
window.addEventListener('message', (event) => {
  // Verify origin
  if (event.origin !== 'https://your-site.com') {
    return;
  }

  const { type, data } = event.data;
  
  if (type === 'init') {
    // Initialize widget
    initWidget(data);
    
    // Send ready message
    window.parent.postMessage(
      { type: 'ready', data: { version: '1.0' } },
      'https://your-site.com'
    );
  }
});

// Send message to parent
window.parent.postMessage(
  { type: 'action', data: { action: 'clicked' } },
  'https://your-site.com'
);
```

## CSP for Third-Party Scripts

### Content Security Policy Configuration

```typescript
// CSP for third-party scripts

// ❌ BAD: Allowing all scripts
app.use((req, res, next) => {
  res.setHeader(
    'Content-Security-Policy',
    "script-src *"  // Allows any script!
  );
  next();
});

// ✓ GOOD: Whitelist specific domains
app.use((req, res, next) => {
  res.setHeader(
    'Content-Security-Policy',
    "script-src 'self' https://cdn.example.com https://www.google-analytics.com"
  );
  next();
});

// ✓ BETTER: Use nonces for inline scripts, allowlist for external
app.use((req, res, next) => {
  const nonce = generateNonce();
  req.nonce = nonce;
  
  res.setHeader(
    'Content-Security-Policy',
    `script-src 'nonce-${nonce}' https://cdn.example.com https://www.google-analytics.com; ` +
    `connect-src 'self' https://api.example.com https://analytics.google.com; ` +
    `img-src 'self' data: https:; ` +
    `style-src 'self' 'unsafe-inline' https://cdn.example.com`
  );
  next();
});

// React: CSP-compliant third-party integration
const GoogleAnalytics: React.FC<{ trackingId: string }> = ({ trackingId }) => {
  useEffect(() => {
    // Load Google Analytics script
    const script = document.createElement('script');
    script.src = `https://www.googletagmanager.com/gtag/js?id=${trackingId}`;
    script.async = true;
    document.head.appendChild(script);

    // Initialize (inline script needs nonce)
    const initScript = document.createElement('script');
    initScript.setAttribute('nonce', getNonce());  // Get nonce from server
    initScript.textContent = `
      window.dataLayer = window.dataLayer || [];
      function gtag(){dataLayer.push(arguments);}
      gtag('js', new Date());
      gtag('config', '${trackingId}');
    `;
    document.head.appendChild(initScript);

    return () => {
      document.head.removeChild(script);
      document.head.removeChild(initScript);
    };
  }, [trackingId]);

  return null;
};
```

### CSP Reporting

```typescript
// Monitor CSP violations from third-party scripts

// Server-side: CSP with reporting
app.use((req, res, next) => {
  res.setHeader(
    'Content-Security-Policy',
    "script-src 'self' https://trusted-cdn.com; " +
    "report-uri /csp-violation-report"
  );
  next();
});

// CSP violation endpoint
app.post('/csp-violation-report', express.json({ type: 'application/csp-report' }), async (req, res) => {
  const report = req.body['csp-report'];
  
  // Log violation
  console.error('CSP Violation:', {
    blockedURI: report['blocked-uri'],
    violatedDirective: report['violated-directive'],
    documentURI: report['document-uri'],
    sourceFile: report['source-file']
  });

  // Check if it's a third-party script violation
  const blockedURL = new URL(report['blocked-uri']);
  const ourDomain = 'example.com';
  
  if (!blockedURL.hostname.endsWith(ourDomain)) {
    // Third-party script blocked
    await alertSecurityTeam({
      type: 'THIRD_PARTY_SCRIPT_BLOCKED',
      script: report['blocked-uri'],
      page: report['document-uri']
    });
  }

  res.status(204).send();
});
```

## Vendor Security Assessment

### Vetting Third-Party Vendors

```typescript
// Comprehensive vendor security assessment

class VendorSecurityAssessment {
  async assessVendor(vendorName: string): Promise<AssessmentResult> {
    const assessment = {
      basicInfo: await this.getBasicInfo(vendorName),
      securityPosture: await this.assessSecurityPosture(vendorName),
      dataHandling: await this.assessDataHandling(vendorName),
      compliance: await this.assessCompliance(vendorName),
      technicalSecurity: await this.assessTechnicalSecurity(vendorName),
      financialStability: await this.assessFinancialStability(vendorName),
      incidentHistory: await this.checkIncidentHistory(vendorName),
      contractual: await this.reviewContractTerms(vendorName)
    };

    return {
      ...assessment,
      overallScore: this.calculateScore(assessment),
      recommendation: this.makeRecommendation(assessment)
    };
  }

  private async assessSecurityPosture(vendorName: string): Promise<SecurityPosture> {
    return {
      certifications: await this.checkCertifications(vendorName),
      // ISO 27001, SOC 2, PCI DSS, etc.
      
      securityTeam: await this.checkSecurityTeam(vendorName),
      // Dedicated security team? Size? Experience?
      
      vulnerabilityProgram: await this.checkBugBounty(vendorName),
      // Bug bounty program? Responsible disclosure policy?
      
      incidentResponse: await this.checkIncidentResponse(vendorName),
      // Incident response plan? Communication protocol?
      
      penetrationTesting: await this.checkPenTesting(vendorName),
      // Regular penetration testing? Results available?
      
      securityUpdates: await this.checkUpdateCadence(vendorName)
      // How often are security updates released?
    };
  }

  private async assessDataHandling(vendorName: string): Promise<DataHandling> {
    return {
      dataAccess: await this.checkDataAccess(vendorName),
      // What data does the script access? (cookies, localStorage, forms, etc.)
      
      dataTransmission: await this.checkDataTransmission(vendorName),
      // Where is data sent? Encrypted? Third parties?
      
      dataRetention: await this.checkRetention(vendorName),
      // How long is data retained? Deletion procedures?
      
      dataLocation: await this.checkDataLocation(vendorName),
      // Where is data stored? (important for GDPR, etc.)
      
      encryption: await this.checkEncryption(vendorName)
      // Encryption at rest? In transit?
    };
  }

  private async assessCompliance(vendorName: string): Promise<Compliance> {
    return {
      gdpr: await this.checkGDPRCompliance(vendorName),
      ccpa: await this.checkCCPACompliance(vendorName),
      hipaa: await this.checkHIPAACompliance(vendorName),
      pciDss: await this.checkPCIDSSCompliance(vendorName),
      dpa: await this.checkDPA(vendorName),  // Data Processing Agreement
      privacyPolicy: await this.reviewPrivacyPolicy(vendorName),
      termsOfService: await this.reviewTOS(vendorName)
    };
  }

  private async assessTechnicalSecurity(vendorName: string): Promise<TechnicalSecurity> {
    const scriptUrl = await this.getScriptURL(vendorName);
    
    return {
      https: scriptUrl.startsWith('https://'),
      sri: await this.checkSRISupport(scriptUrl),
      csp: await this.checkCSPCompatibility(scriptUrl),
      codeQuality: await this.analyzeCode(scriptUrl),
      dependencies: await this.analyzeDependencies(scriptUrl),
      vulnerabilities: await this.scanForVulnerabilities(scriptUrl),
      updates: await this.checkUpdateFrequency(vendorName)
    };
  }

  private calculateScore(assessment: any): number {
    // Weighted scoring (0-100)
    const weights = {
      securityPosture: 0.25,
      dataHandling: 0.20,
      compliance: 0.20,
      technicalSecurity: 0.20,
      financialStability: 0.10,
      incidentHistory: 0.05
    };

    // Calculate weighted score
    let score = 0;
    for (const [category, weight] of Object.entries(weights)) {
      score += assessment[category].score * weight;
    }

    return Math.round(score);
  }

  private makeRecommendation(assessment: any): Recommendation {
    const score = this.calculateScore(assessment);
    
    if (score >= 80) {
      return {
        decision: 'APPROVED',
        conditions: [],
        notes: 'Vendor meets all security requirements'
      };
    } else if (score >= 60) {
      return {
        decision: 'APPROVED_WITH_CONDITIONS',
        conditions: this.generateConditions(assessment),
        notes: 'Vendor approved with additional safeguards'
      };
    } else if (score >= 40) {
      return {
        decision: 'REMEDIATION_REQUIRED',
        conditions: this.generateConditions(assessment),
        notes: 'Vendor must address security concerns before approval'
      };
    } else {
      return {
        decision: 'REJECTED',
        conditions: [],
        notes: 'Vendor does not meet minimum security requirements'
      };
    }
  }

  private generateConditions(assessment: any): string[] {
    const conditions: string[] = [];

    // Generate specific conditions based on assessment
    if (!assessment.technicalSecurity.sri) {
      conditions.push('Implement Subresource Integrity (SRI)');
    }

    if (!assessment.technicalSecurity.https) {
      conditions.push('Require HTTPS for all resources');
    }

    if (!assessment.compliance.dpa) {
      conditions.push('Execute Data Processing Agreement');
    }

    if (assessment.dataHandling.dataAccess.includes('forms')) {
      conditions.push('Limit access to form data via iframe sandboxing');
    }

    if (assessment.incidentHistory.incidents > 0) {
      conditions.push('Implement additional monitoring and alerting');
    }

    return conditions;
  }
}

// Usage
const assessor = new VendorSecurityAssessment();
const result = await assessor.assessVendor('Acme Analytics Co.');

console.log(`Vendor: ${result.basicInfo.name}`);
console.log(`Overall Score: ${result.overallScore}/100`);
console.log(`Recommendation: ${result.recommendation.decision}`);

if (result.recommendation.conditions.length > 0) {
  console.log('Conditions:');
  result.recommendation.conditions.forEach(condition => {
    console.log(`  - ${condition}`);
  });
}
```

## Implementation Examples

### Safe Third-Party Integration

```typescript
// Secure third-party script loader

class SecureThirdPartyLoader {
  private loadedScripts = new Set<string>();
  private trustedDomains = new Set<string>();

  constructor(trustedDomains: string[]) {
    this.trustedDomains = new Set(trustedDomains);
  }

  async loadScript(options: {
    src: string;
    integrity?: string;
    async?: boolean;
    defer?: boolean;
    sandbox?: boolean;
  }): Promise<void> {
    // 1. Verify domain is trusted
    const url = new URL(options.src);
    if (!this.isTrustedDomain(url.hostname)) {
      throw new Error(`Untrusted domain: ${url.hostname}`);
    }

    // 2. Check if already loaded
    if (this.loadedScripts.has(options.src)) {
      return;
    }

    // 3. Create script element
    const script = document.createElement('script');
    script.src = options.src;
    script.async = options.async !== false;
    script.defer = options.defer || false;
    
    // 4. Add SRI if provided
    if (options.integrity) {
      script.integrity = options.integrity;
      script.crossOrigin = 'anonymous';
    }

    // 5. Load in sandboxed iframe if requested
    if (options.sandbox) {
      await this.loadInSandbox(options.src);
      return;
    }

    // 6. Load script with error handling
    return new Promise((resolve, reject) => {
      script.onload = () => {
        this.loadedScripts.add(options.src);
        resolve();
      };

      script.onerror = () => {
        reject(new Error(`Failed to load script: ${options.src}`));
      };

      document.head.appendChild(script);
    });
  }

  private isTrustedDomain(domain: string): boolean {
    // Check exact match
    if (this.trustedDomains.has(domain)) {
      return true;
    }

    // Check subdomain match (*.example.com)
    for (const trusted of this.trustedDomains) {
      if (trusted.startsWith('*.')) {
        const baseDomain = trusted.slice(2);
        if (domain.endsWith(baseDomain)) {
          return true;
        }
      }
    }

    return false;
  }

  private async loadInSandbox(src: string): Promise<void> {
    return new Promise((resolve, reject) => {
      const iframe = document.createElement('iframe');
      iframe.sandbox.add('allow-scripts');
      iframe.style.display = 'none';
      
      iframe.onload = () => {
        const script = iframe.contentDocument!.createElement('script');
        script.src = src;
        script.onload = () => resolve();
        script.onerror = () => reject(new Error(`Failed to load in sandbox: ${src}`));
        iframe.contentDocument!.head.appendChild(script);
      };

      document.body.appendChild(iframe);
    });
  }

  async loadAnalytics(trackingId: string): Promise<void> {
    await this.loadScript({
      src: `https://www.googletagmanager.com/gtag/js?id=${trackingId}`,
      async: true
    });

    // Initialize with nonce
    const script = document.createElement('script');
    script.setAttribute('nonce', getNonce());
    script.textContent = `
      window.dataLayer = window.dataLayer || [];
      function gtag(){dataLayer.push(arguments);}
      gtag('js', new Date());
      gtag('config', '${trackingId}', {
        'anonymize_ip': true,
        'cookie_flags': 'SameSite=Strict;Secure'
      });
    `;
    document.head.appendChild(script);
  }

  async loadStripe(): Promise<void> {
    await this.loadScript({
      src: 'https://js.stripe.com/v3/',
      integrity: 'sha384-...',  // Get latest hash from Stripe
      async: true
    });
  }
}

// Usage
const loader = new SecureThirdPartyLoader([
  'www.googletagmanager.com',
  'www.google-analytics.com',
  'js.stripe.com',
  '*.cloudflare.com'
]);

// Load analytics
await loader.loadAnalytics('GA-TRACKING-ID');

// Load Stripe
await loader.loadStripe();

// ✗ This will fail (untrusted domain)
try {
  await loader.loadScript({
    src: 'https://unknown-cdn.com/script.js'
  });
} catch (error) {
  console.error('Failed to load untrusted script:', error);
}
```

### React: Third-Party Script Manager

```typescript
// React: Centralized third-party script management

import React, { createContext, useContext, useEffect, useState } from 'react';

interface ThirdPartyScript {
  id: string;
  src: string;
  integrity?: string;
  async?: boolean;
  sandbox?: boolean;
}

interface ThirdPartyContextType {
  loadScript: (script: ThirdPartyScript) => Promise<void>;
  isLoaded: (id: string) => boolean;
  trustedDomains: Set<string>;
}

const ThirdPartyContext = createContext<ThirdPartyContextType | undefined>(undefined);

export const ThirdPartyProvider: React.FC<{ 
  trustedDomains: string[];
  children: React.ReactNode;
}> = ({ trustedDomains, children }) => {
  const [loadedScripts, setLoadedScripts] = useState<Set<string>>(new Set());
  const domains = new Set(trustedDomains);

  const loadScript = async (script: ThirdPartyScript): Promise<void> => {
    // Check if already loaded
    if (loadedScripts.has(script.id)) {
      return;
    }

    // Verify domain
    const url = new URL(script.src);
    if (!domains.has(url.hostname)) {
      throw new Error(`Untrusted domain: ${url.hostname}`);
    }

    // Load script
    return new Promise((resolve, reject) => {
      const scriptElement = document.createElement('script');
      scriptElement.src = script.src;
      scriptElement.async = script.async !== false;
      
      if (script.integrity) {
        scriptElement.integrity = script.integrity;
        scriptElement.crossOrigin = 'anonymous';
      }

      scriptElement.onload = () => {
        setLoadedScripts(prev => new Set(prev).add(script.id));
        resolve();
      };

      scriptElement.onerror = () => {
        reject(new Error(`Failed to load script: ${script.id}`));
      };

      document.head.appendChild(scriptElement);
    });
  };

  const isLoaded = (id: string): boolean => {
    return loadedScripts.has(id);
  };

  return (
    <ThirdPartyContext.Provider value={{ loadScript, isLoaded, trustedDomains: domains }}>
      {children}
    </ThirdPartyContext.Provider>
  );
};

export const useThirdParty = () => {
  const context = useContext(ThirdPartyContext);
  if (!context) {
    throw new Error('useThirdParty must be used within ThirdPartyProvider');
  }
  return context;
};

// Hook for loading specific third-party scripts
export const useGoogleAnalytics = (trackingId: string) => {
  const { loadScript, isLoaded } = useThirdParty();
  const [initialized, setInitialized] = useState(false);

  useEffect(() => {
    if (isLoaded('google-analytics')) {
      return;
    }

    const initGA = async () => {
      await loadScript({
        id: 'google-analytics',
        src: `https://www.googletagmanager.com/gtag/js?id=${trackingId}`,
        async: true
      });

      // Initialize
      window.dataLayer = window.dataLayer || [];
      function gtag(...args: any[]) {
        window.dataLayer.push(args);
      }
      gtag('js', new Date());
      gtag('config', trackingId, {
        anonymize_ip: true,
        cookie_flags: 'SameSite=Strict;Secure'
      });

      setInitialized(true);
    };

    initGA();
  }, [trackingId, loadScript, isLoaded]);

  return initialized;
};

export const useStripe = () => {
  const { loadScript, isLoaded } = useThirdParty();
  const [stripe, setStripe] = useState<any>(null);

  useEffect(() => {
    if (isLoaded('stripe')) {
      setStripe(window.Stripe(process.env.REACT_APP_STRIPE_KEY));
      return;
    }

    const initStripe = async () => {
      await loadScript({
        id: 'stripe',
        src: 'https://js.stripe.com/v3/',
        integrity: 'sha384-...',  // Get from Stripe
        async: true
      });

      setStripe(window.Stripe(process.env.REACT_APP_STRIPE_KEY));
    };

    initStripe();
  }, [loadScript, isLoaded]);

  return stripe;
};

// App setup
function App() {
  return (
    <ThirdPartyProvider trustedDomains={[
      'www.googletagmanager.com',
      'www.google-analytics.com',
      'js.stripe.com'
    ]}>
      <YourApp />
    </ThirdPartyProvider>
  );
}

// Usage in components
function AnalyticsEnabledPage() {
  const gaInitialized = useGoogleAnalytics('GA-TRACKING-ID');
  
  return (
    <div>
      {gaInitialized && <p>Analytics tracking enabled</p>}
    </div>
  );
}

function CheckoutPage() {
  const stripe = useStripe();
  
  if (!stripe) {
    return <div>Loading payment form...</div>;
  }
  
  return <StripeCheckoutForm stripe={stripe} />;
}
```

## Common Mistakes

### 1. Loading Scripts Over HTTP

```html
<!-- ❌ BAD: HTTP (insecure) -->
<script src="http://cdn.example.com/library.js"></script>

<!-- ✓ GOOD: HTTPS -->
<script src="https://cdn.example.com/library.js"></script>
```

### 2. No Integrity Checks

```html
<!-- ❌ BAD: No SRI -->
<script src="https://cdn.example.com/library.js"></script>

<!-- ✓ GOOD: With SRI -->
<script
  src="https://cdn.example.com/library.js"
  integrity="sha384-..."
  crossorigin="anonymous"
></script>
```

### 3. Allowing Any Domain in CSP

```typescript
// ❌ BAD
res.setHeader('Content-Security-Policy', "script-src *");

// ✓ GOOD
res.setHeader('Content-Security-Policy', 
  "script-src 'self' https://trusted-cdn.com");
```

### 4. Not Sandboxing Third-Party Widgets

```html
<!-- ❌ BAD: Full access iframe -->
<iframe src="https://widget.example.com"></iframe>

<!-- ✓ GOOD: Sandboxed iframe -->
<iframe
  src="https://widget.example.com"
  sandbox="allow-scripts allow-same-origin"
></iframe>
```

### 5. Loading Too Many Third-Party Scripts

```typescript
// ❌ BAD: Loading everything
loadGoogleAnalytics();
loadFacebookPixel();
loadTwitterAnalytics();
loadLinkedInInsights();
loadHotjar();
loadIntercom();
loadFullStory();
// 20+ more scripts...

// ✓ GOOD: Only essential scripts
loadGoogleAnalytics();  // Essential for business
// Use server-side alternatives when possible
```

## Best Practices

### 1. Minimize Third-Party Dependencies

```typescript
// Evaluate necessity of each third-party script

class ThirdPartyAudit {
  auditScripts(): AuditResult[] {
    const results: AuditResult[] = [];
    const scripts = document.querySelectorAll('script[src]');

    scripts.forEach(script => {
      const src = script.getAttribute('src');
      if (!src) return;

      const isThirdParty = !src.startsWith(window.location.origin);
      
      if (isThirdParty) {
        results.push({
          src,
          necessary: this.evaluateNecessity(src),
          alternative: this.suggestAlternative(src),
          risk: this.assessRisk(src)
        });
      }
    });

    return results;
  }

  private evaluateNecessity(src: string): boolean {
    // Is this script absolutely necessary?
    // Could functionality be implemented server-side?
    // Could it be replaced with first-party solution?
    return false;  // Default: not necessary
  }

  private suggestAlternative(src: string): string | null {
    // Suggest alternatives
    const alternatives: Record<string, string> = {
      'analytics.js': 'Server-side analytics or Plausible',
      'livechat.js': 'Self-hosted chat solution',
      'maps.js': 'Leaflet with OpenStreetMap'
    };

    for (const [pattern, alternative] of Object.entries(alternatives)) {
      if (src.includes(pattern)) {
        return alternative;
      }
    }

    return null;
  }
}
```

### 2. Implement Defense in Depth

```typescript
// Multiple layers of protection

class ThirdPartyDefense {
  async loadSecurely(script: ThirdPartyScript): Promise<void> {
    // Layer 1: Domain whitelist
    this.verifyDomain(script.src);

    // Layer 2: HTTPS enforcement
    this.requireHTTPS(script.src);

    // Layer 3: Subresource Integrity
    this.requireSRI(script);

    // Layer 4: Content Security Policy
    this.verifyCSPCompliance(script);

    // Layer 5: Sandboxing (if widget)
    if (script.isWidget) {
      await this.loadInSandbox(script);
    } else {
      await this.loadScript(script);
    }

    // Layer 6: Monitoring
    this.monitorBehavior(script.id);
  }

  private monitorBehavior(scriptId: string): void {
    // Monitor what the script does
    const observer = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        if (entry.name.includes(scriptId)) {
          this.logActivity(scriptId, entry);
        }
      }
    });

    observer.observe({ entryTypes: ['resource', 'measure'] });
  }
}
```

### 3. Regular Security Audits

```typescript
// Automated third-party security audits

class ThirdPartySecurityAudit {
  async runAudit(): Promise<AuditReport> {
    const report: AuditReport = {
      timestamp: new Date(),
      scripts: [],
      issues: [],
      recommendations: []
    };

    // 1. Inventory all third-party scripts
    const scripts = this.inventoryScripts();

    // 2. Check each script
    for (const script of scripts) {
      const scriptReport = await this.auditScript(script);
      report.scripts.push(scriptReport);

      // 3. Identify issues
      if (!scriptReport.hasSRI) {
        report.issues.push({
          severity: 'HIGH',
          script: script.src,
          issue: 'Missing Subresource Integrity'
        });
      }

      if (!scriptReport.isHTTPS) {
        report.issues.push({
          severity: 'CRITICAL',
          script: script.src,
          issue: 'Loaded over HTTP'
        });
      }

      if (scriptReport.dataExfiltration) {
        report.issues.push({
          severity: 'HIGH',
          script: script.src,
          issue: 'Potential data exfiltration detected'
        });
      }
    }

    // 4. Generate recommendations
    report.recommendations = this.generateRecommendations(report);

    return report;
  }

  private async auditScript(script: HTMLScriptElement): Promise<ScriptAuditReport> {
    const src = script.getAttribute('src')!;
    const code = await this.fetchScript(src);

    return {
      src,
      isHTTPS: src.startsWith('https://'),
      hasSRI: script.hasAttribute('integrity'),
      cspCompliant: await this.checkCSPCompliance(src),
      dataAccess: this.analyzeDataAccess(code),
      externalRequests: this.analyzeExternalRequests(code),
      dataExfiltration: this.detectDataExfiltration(code),
      codeQuality: this.analyzeCodeQuality(code)
    };
  }
}

// Schedule regular audits
setInterval(async () => {
  const auditor = new ThirdPartySecurityAudit();
  const report = await auditor.runAudit();
  
  if (report.issues.some(i => i.severity === 'CRITICAL')) {
    await alertSecurityTeam(report);
  }
  
  await storeAuditReport(report);
}, 24 * 60 * 60 * 1000);  // Daily
```

### 4. Use Server-Side Alternatives

```typescript
// Server-side analytics instead of client-side

// ❌ Client-side Google Analytics
<script src="https://www.googletagmanager.com/gtag/js"></script>

// ✓ Server-side analytics
class ServerSideAnalytics {
  async trackPageView(req: Request): Promise<void> {
    await fetch('https://analytics-api.example.com/track', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        url: req.url,
        referrer: req.headers.referer,
        userAgent: req.headers['user-agent'],
        ip: req.ip,
        timestamp: new Date()
      })
    });
  }
}

// Benefits:
// - No third-party JavaScript on client
// - Better privacy
// - More control over data
// - Doesn't affect page load time
// - Not blocked by ad blockers
```

## When to Use/Not to Use

### When Third-Party Scripts Are Acceptable

**Use when:**
- Essential functionality (payment processing)
- Significant value (analytics, error tracking)
- Trusted vendor with good security practices
- Proper security measures implemented (SRI, CSP, sandboxing)

**Avoid when:**
- Functionality can be implemented in-house
- Server-side alternative available
- Vendor has poor security track record
- Script requests excessive permissions
- Privacy concerns for sensitive data

### Decision Framework

```typescript
// Decide whether to use third-party script

interface ThirdPartyDecision {
  use: boolean;
  conditions: string[];
  alternatives: string[];
}

function shouldUseThirdParty(
  scriptName: string,
  functionality: string
): ThirdPartyDecision {
  // 1. Is it essential?
  if (!isEssential(functionality)) {
    return {
      use: false,
      conditions: [],
      alternatives: ['Implement in-house', 'Remove feature']
    };
  }

  // 2. Can we do it server-side?
  if (canBeServerSide(functionality)) {
    return {
      use: false,
      conditions: [],
      alternatives: ['Server-side implementation']
    };
  }

  // 3. Is vendor trustworthy?
  const trustScore = assessVendorTrust(scriptName);
  if (trustScore < 70) {
    return {
      use: false,
      conditions: [],
      alternatives: ['Find alternative vendor', 'Build in-house']
    };
  }

  // 4. Can we secure it?
  const securityMeasures = [
    'Subresource Integrity',
    'Content Security Policy',
    'Sandboxed iframe',
    'Rate limiting',
    'Monitoring'
  ];

  return {
    use: true,
    conditions: securityMeasures,
    alternatives: []
  };
}
```

## Interview Questions

### Q1: What is Subresource Integrity (SRI) and why is it important?

**Answer:** SRI ensures that resources fetched from CDNs haven't been tampered with.

**How it works:**
```html
<script
  src="https://cdn.com/library.js"
  integrity="sha384-oqVuAfX..."
  crossorigin="anonymous"
></script>
```

Browser:
1. Downloads script
2. Computes hash of downloaded content
3. Compares with integrity hash
4. Executes ONLY if hashes match
5. Blocks if hashes don't match

**Why important:**
- **CDN compromise:** If CDN hacked, malicious code blocked
- **MITM protection:** Prevents modification during transmission
- **Supply chain security:** Detects unauthorized changes

**Without SRI:** Compromised CDN = all sites using script affected (British Airways attack)

**Limitation:** Hash must be updated when script updates (use automation)

### Q2: How do you secure third-party iframes?

**Answer:** Multi-layered approach:

**1. Sandbox attribute:**
```html
<iframe
  src="https://third-party.com"
  sandbox="allow-scripts allow-forms"
></iframe>
```

Limits what iframe can do:
- `allow-scripts`: JavaScript execution
- `allow-same-origin`: Access cookies/localStorage
- `allow-forms`: Form submission
- `allow-popups`: Open popups

Omit permissions iframe doesn't need.

**2. CSP frame-src:**
```
Content-Security-Policy: frame-src https://trusted-third-party.com
```
Only allows iframes from whitelisted domains.

**3. postMessage for communication:**
```typescript
// Verify origin before processing
window.addEventListener('message', (event) => {
  if (event.origin !== 'https://trusted-third-party.com') {
    return;  // Ignore
  }
  // Process message
});
```

**4. X-Frame-Options (server-side):**
```
X-Frame-Options: DENY
```
Prevents your site from being embedded in third-party iframes (reverse protection).

**5. Minimal permissions:**
- Only grant necessary permissions
- Test with most restrictive first
- Add permissions only if needed

### Q3: What risks do third-party scripts introduce?

**Answer:** Major risks:

**1. Code Injection:**
- Script executes with your site's privileges
- Can access DOM, cookies, localStorage
- Can modify page content
- Can steal sensitive data

**2. Supply Chain Attack:**
- Third-party service compromised
- All sites using script affected
- Example: British Airways Magecart (2018)

**3. Data Exfiltration:**
- Script sends data to third party
- Personal information, passwords, credit cards
- May violate privacy regulations (GDPR)

**4. Performance Impact:**
- Slow loading times
- Blocking resources
- Increased bandwidth

**5. Privacy Violations:**
- Tracking users without consent
- Sharing data with third parties
- Cookie violations

**6. Availability:**
- If third-party service down, features break
- Single point of failure

**7. Legal/Compliance:**
- GDPR violations (data processing without DPA)
- PCI DSS violations (credit card data)
- Liability for third-party actions

**Mitigation:** SRI, CSP, sandboxing, vendor vetting, monitoring.

### Q4: How do you evaluate whether to use a third-party script?

**Answer:** Systematic evaluation:

**1. Necessity:**
- Is functionality essential?
- Can it be built in-house?
- Server-side alternative available?

**2. Vendor assessment:**
- Reputation and track record
- Security certifications (SOC 2, ISO 27001)
- Incident history
- Response to vulnerabilities

**3. Technical security:**
- HTTPS delivery
- SRI support
- CSP compatibility
- Code quality
- Update frequency

**4. Data handling:**
- What data accessed?
- Where is data sent?
- Data retention policy
- GDPR/CCPA compliance

**5. Cost-benefit:**
- Value provided vs. risk introduced
- Alternatives comparison
- Total cost (licensing + security)

**Decision matrix:**
```
Essential + High Trust + Good Security = USE (with protections)
Essential + Low Trust = Find Alternative
Non-Essential = DON'T USE
```

**Document decision:** Maintain registry of third-party scripts with justification.

### Q5: What is Content Security Policy's role with third-party scripts?

**Answer:** CSP controls which external scripts can execute.

**Basic CSP:**
```
Content-Security-Policy: script-src 'self' https://trusted-cdn.com
```
Only allows scripts from your domain and trusted-cdn.com.

**Benefits:**

**1. Whitelist approach:**
- Explicitly allow trusted domains
- Block everything else by default
- Prevents injection of unauthorized scripts

**2. Nonce-based:**
```
Content-Security-Policy: script-src 'nonce-abc123'

<script nonce="abc123">/* allowed */</script>
<script>/* blocked */</script>
```
Only scripts with matching nonce execute.

**3. Block inline scripts:**
```
Content-Security-Policy: script-src 'self'
```
Blocks `<script>alert('xss')</script>` (XSS protection).

**4. Restrict connections:**
```
Content-Security-Policy: connect-src 'self' https://api.example.com
```
Controls where scripts can make requests.

**Limitations:**
- Doesn't prevent trusted scripts from being malicious
- Complex to configure correctly
- May break functionality if too strict

**Best practice:** CSP + SRI for defense in depth.

### Q6: How do you monitor third-party script behavior?

**Answer:** Multi-faceted monitoring:

**1. CSP Reporting:**
```
Content-Security-Policy: script-src 'self' https://trusted.com;
                         report-uri /csp-report
```
Alerts when unauthorized scripts blocked.

**2. Network Monitoring:**
```typescript
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (entry.initiatorType === 'script') {
      console.log('Script loaded:', entry.name);
      
      // Check if from third-party
      if (!entry.name.startsWith(window.location.origin)) {
        alertOnUnexpectedThirdParty(entry.name);
      }
    }
  }
});

observer.observe({ entryTypes: ['resource'] });
```

**3. DOM Mutation Observer:**
```typescript
const observer = new MutationObserver((mutations) => {
  mutations.forEach((mutation) => {
    mutation.addedNodes.forEach((node) => {
      if (node.nodeName === 'SCRIPT') {
        const src = (node as HTMLScriptElement).src;
        if (src && !isWhitelisted(src)) {
          console.warn('Unexpected script added:', src);
        }
      }
    });
  });
});

observer.observe(document, {
  childList: true,
  subtree: true
});
```

**4. Behavior Analysis:**
- Track what data scripts access (cookies, forms)
- Monitor external requests
- Detect data exfiltration attempts

**5. Regular Audits:**
```typescript
setInterval(async () => {
  const scripts = document.querySelectorAll('script[src]');
  scripts.forEach(async (script) => {
    const src = script.getAttribute('src');
    await auditScript(src);
  });
}, 24 * 60 * 60 * 1000);  // Daily
```

**Tools:**
- Google Tag Manager (debug mode)
- Request Map Generator
- Ghostery
- uBlock Origin

### Q7: What are the alternatives to client-side third-party scripts?

**Answer:** Several alternatives reduce risk:

**1. Server-Side Proxy:**
```typescript
// Instead of loading script client-side
// Proxy requests through your server

app.get('/api/analytics', async (req, res) => {
  // Forward to third-party analytics API server-side
  await fetch('https://analytics-api.com/track', {
    method: 'POST',
    body: JSON.stringify({
      page: req.headers.referer,
      userAgent: req.headers['user-agent']
    })
  });
  
  res.status(204).send();
});
```

Benefits: No client-side script, more control, better privacy.

**2. Self-Hosted:**
```bash
# Download and host analytics script yourself
# Update regularly (automation)

wget https://cdn.com/analytics.js -O public/analytics.js
```

Benefits: Control over code, no CDN dependency, faster loading.

**3. First-Party Alternatives:**
- Build functionality in-house
- Example: Simple analytics instead of Google Analytics
- Example: Custom chat instead of Intercom

**4. Progressive Enhancement:**
- Core functionality works without third-party
- Third-party enhances experience
- Graceful degradation if script fails

**5. Serverless Functions:**
```typescript
// Netlify/Vercel function instead of client-side script
export default async (req, res) => {
  // Process on server
  res.status(200).json({ success: true });
};
```

**Trade-offs:**
- Server-side: More server load, less real-time
- Self-hosted: Maintenance burden, update management
- In-house: Development time, ongoing maintenance
- Choose based on value vs. effort

### Q8: How do you handle third-party script failures gracefully?

**Answer:** Defensive programming for resilience:

**1. Feature Detection:**
```typescript
// Check if third-party loaded
if (typeof window.ThirdPartyLibrary !== 'undefined') {
  window.ThirdPartyLibrary.init();
} else {
  // Fallback or warn
  console.warn('Third-party library not loaded');
}
```

**2. Timeout Handling:**
```typescript
function loadScriptWithTimeout(src: string, timeout = 5000): Promise<void> {
  return Promise.race([
    loadScript(src),
    new Promise((_, reject) => 
      setTimeout(() => reject(new Error('Timeout')), timeout)
    )
  ]);
}

try {
  await loadScriptWithTimeout('https://cdn.com/script.js');
} catch (error) {
  console.error('Script failed to load:', error);
  // Continue without third-party feature
}
```

**3. Fallback Functionality:**
```typescript
async function initializeMap() {
  try {
    // Try Google Maps (third-party)
    await loadGoogleMaps();
    return new GoogleMap();
  } catch (error) {
    // Fallback to OpenStreetMap (self-hosted)
    return new OpenStreetMap();
  }
}
```

**4. Error Boundaries (React):**
```typescript
class ThirdPartyErrorBoundary extends React.Component {
  state = { hasError: false };

  static getDerivedStateFromError(error) {
    return { hasError: true };
  }

  componentDidCatch(error, errorInfo) {
    console.error('Third-party component failed:', error);
  }

  render() {
    if (this.state.hasError) {
      return <div>Feature temporarily unavailable</div>;
    }

    return this.props.children;
  }
}

// Wrap third-party components
<ThirdPartyErrorBoundary>
  <ThirdPartyWidget />
</ThirdPartyErrorBoundary>
```

**5. Monitoring & Alerting:**
```typescript
window.addEventListener('error', (event) => {
  if (event.filename?.includes('third-party-cdn.com')) {
    alertOps('Third-party script failed to load');
  }
});
```

**6. Progressive Enhancement:**
- Core features work without third-party
- Third-party enhances UX
- No hard dependency

**Best practice:** Assume third-party scripts will fail, plan accordingly.

## Key Takeaways

1. **Third-party scripts are high-risk** - Full access to your page context; CDN compromise affects all sites using script
2. **Use Subresource Integrity (SRI)** - Verify script integrity with hashes; prevents tampering during transmission or at source
3. **Minimize third-party dependencies** - Each script increases attack surface; prefer server-side alternatives when possible
4. **Sandbox third-party iframes** - Use sandbox attribute to limit permissions; only grant what's necessary
5. **Implement strict CSP** - Whitelist trusted domains only; use nonces for inline scripts; monitor violations
6. **Vet vendors thoroughly** - Security posture, incident history, data handling, compliance before using
7. **Monitor script behavior** - Track what scripts do (network, DOM, storage); detect unauthorized changes
8. **Plan for failures** - Graceful degradation when scripts fail; timeouts, fallbacks, error boundaries
9. **Regular security audits** - Automated scanning for issues; remove unnecessary scripts; update SRI hashes
10. **Defense in depth** - Multiple protection layers (SRI + CSP + sandboxing + monitoring + vendor vetting)

## Resources

### Tools and Services
- [SRI Hash Generator](https://www.srihash.org/)
- [Report URI](https://report-uri.com/) - CSP reporting and monitoring
- [Security Headers](https://securityheaders.com/) - Check security headers
- [Request Map](https://requestmap.webperf.tools/) - Visualize third-party requests

### Documentation
- [MDN: Subresource Integrity](https://developer.mozilla.org/en-US/docs/Web/Security/Subresource_Integrity)
- [MDN: iframe sandbox](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/iframe#attr-sandbox)
- [CSP Documentation](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP)

### Articles and Guides
- [Third-Party Script Security](https://web.dev/third-party-scripts/)
- [Magecart Attacks Explained](https://www.riskiq.com/what-is-magecart/)
- [Supply Chain Attacks](https://owasp.org/www-community/attacks/Supply_Chain_Attack)

### Standards and Best Practices
- [OWASP: Third Party JavaScript Management](https://cheatsheetseries.owasp.org/cheatsheets/Third_Party_Javascript_Management_Cheat_Sheet.html)
- [Content Security Policy Level 3](https://www.w3.org/TR/CSP3/)
- [Subresource Integrity W3C Recommendation](https://www.w3.org/TR/SRI/)
