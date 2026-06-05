# Dependency Vulnerabilities

## The Idea

**In plain English:** Your app is built on top of dozens of open-source packages written by others. A dependency vulnerability is a known security flaw in one of those packages — and if you haven't updated or patched it, attackers can exploit it to compromise your app.

**Real-world analogy:** Think of a building made from bricks sourced from a supplier. If the supplier later discovers a batch of bricks is structurally defective, every building using those bricks is at risk — regardless of how well the rest of the construction was done. You need to replace the bad bricks (update the package) before someone finds and exploits the weak wall.

- The building = your application
- The bricks from a supplier = third-party npm packages you depend on
- The defective batch = a package version with a known CVE (security flaw)
- Replacing the bad bricks = running `npm audit fix` or updating to a patched version

---

## Table of Contents
- [Introduction](#introduction)
- [Understanding Dependency Risks](#understanding-dependency-risks)
- [npm Audit Usage](#npm-audit-usage)
- [Snyk Integration](#snyk-integration)
- [Dependabot Setup](#dependabot-setup)
- [Security Updates Workflow](#security-updates-workflow)
- [Supply Chain Attacks](#supply-chain-attacks)
- [Package Integrity (SRI)](#package-integrity-sri)
- [Implementation Examples](#implementation-examples)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [When to Use/Not to Use](#when-to-usenot-to-use)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

**Dependency vulnerabilities** are security flaws in third-party packages that applications rely on. Modern web applications typically have hundreds or thousands of dependencies, creating a large attack surface. Managing these vulnerabilities is critical for application security.

### The Dependency Problem

```
┌─────────────────────────────────────────────────────────────┐
│           Modern Application Dependencies                    │
└─────────────────────────────────────────────────────────────┘

Your Application
      ├── react (16 dependencies)
      │     ├── loose-envify
      │     └── object-assign
      ├── lodash (0 dependencies)
      ├── axios (5 dependencies)
      │     ├── follow-redirects  ← Vulnerability!
      │     └── form-data
      └── express (31 dependencies)
            ├── body-parser
            ├── cookie
            └── ...

Total: ~3,000 files in node_modules
Risk: Any one can have vulnerabilities

┌──────────────────────────────────────────────────────────────┐
│  Real-World Impact                                           │
├──────────────────────────────────────────────────────────────┤
│  Event-Stream Attack (2018)                                  │
│  - Popular npm package compromised                           │
│  - 2 million weekly downloads                                │
│  - Stole Bitcoin from Copay wallet                           │
│                                                              │
│  Lodash Prototype Pollution (2019)                           │
│  - Affected millions of applications                         │
│  - CVE-2019-10744                                            │
│  - Critical severity                                         │
└──────────────────────────────────────────────────────────────┘
```

## Understanding Dependency Risks

### Types of Dependency Risks

```typescript
interface DependencyRisk {
  // 1. Known Vulnerabilities
  cve: {
    id: string;  // CVE-2021-23337
    severity: 'LOW' | 'MODERATE' | 'HIGH' | 'CRITICAL';
    description: string;
    patchVersion: string;
  };
  
  // 2. Unmaintained Packages
  maintenance: {
    lastUpdate: Date;
    openIssues: number;
    maintainerResponse: 'active' | 'slow' | 'abandoned';
  };
  
  // 3. Malicious Packages
  malicious: {
    typosquatting: boolean;  // react vs rea  ct
    backdoors: boolean;
    dataExfiltration: boolean;
  };
  
  // 4. License Issues
  license: {
    type: string;
    compatible: boolean;
    restrictions: string[];
  };
  
  // 5. Supply Chain Risks
  supplyChain: {
    publisherVerified: boolean;
    twoFactorAuth: boolean;
    dependencyCount: number;
  };
}
```

### Vulnerability Severity Levels

```
┌─────────────────────────────────────────────────────────────┐
│              CVSS Severity Scale                             │
└─────────────────────────────────────────────────────────────┘

CRITICAL (9.0-10.0)
  ├─ Remote Code Execution
  ├─ SQL Injection
  └─ Complete system compromise
  Action: Fix immediately

HIGH (7.0-8.9)
  ├─ Privilege escalation
  ├─ Significant data exposure
  └─ Authentication bypass
  Action: Fix within days

MODERATE (4.0-6.9)
  ├─ Denial of Service
  ├─ Information disclosure
  └─ Limited impact
  Action: Fix within weeks

LOW (0.1-3.9)
  ├─ Minor information leaks
  ├─ Limited scope
  └─ Difficult to exploit
  Action: Fix in next release
```

## npm Audit Usage

### Basic npm Audit Commands

```bash
# Run security audit
npm audit

# Show detailed report
npm audit --json

# Fix automatically (patches)
npm audit fix

# Fix everything (including breaking changes)
npm audit fix --force

# Audit only production dependencies
npm audit --production

# Audit specific severity
npm audit --audit-level=high

# Generate report in different formats
npm audit --json > audit-report.json
```

### Understanding npm Audit Output

```typescript
// npm audit JSON output structure
interface NpmAuditResult {
  auditReportVersion: 2;
  vulnerabilities: {
    [packageName: string]: {
      name: string;
      severity: 'info' | 'low' | 'moderate' | 'high' | 'critical';
      isDirect: boolean;
      via: Array<{
        source: string;
        name: string;
        dependency: string;
        title: string;
        url: string;
        severity: string;
        cwe: string[];
        cvss: {
          score: number;
          vectorString: string;
        };
        range: string;
      }>;
      effects: string[];
      range: string;
      nodes: string[];
      fixAvailable: boolean | {
        name: string;
        version: string;
        isSemVerMajor: boolean;
      };
    };
  };
  metadata: {
    vulnerabilities: {
      info: number;
      low: number;
      moderate: number;
      high: number;
      critical: number;
      total: number;
    };
    dependencies: {
      prod: number;
      dev: number;
      optional: number;
      peer: number;
      peerOptional: number;
      total: number;
    };
  };
}

// Parse and process audit results
class NpmAuditProcessor {
  async runAudit(): Promise<NpmAuditResult> {
    const { execSync } = require('child_process');
    const output = execSync('npm audit --json').toString();
    return JSON.parse(output);
  }

  async analyzeResults(auditResult: NpmAuditResult): Promise<AuditAnalysis> {
    const { vulnerabilities, metadata } = auditResult;
    
    const criticalVulns = this.filterBySeverity(vulnerabilities, 'critical');
    const highVulns = this.filterBySeverity(vulnerabilities, 'high');
    
    return {
      totalVulnerabilities: metadata.vulnerabilities.total,
      critical: criticalVulns,
      high: highVulns,
      fixableCount: this.countFixable(vulnerabilities),
      actionRequired: criticalVulns.length > 0 || highVulns.length > 0
    };
  }

  private filterBySeverity(
    vulnerabilities: any,
    severity: string
  ): VulnerabilityDetail[] {
    return Object.entries(vulnerabilities)
      .filter(([_, vuln]: any) => vuln.severity === severity)
      .map(([name, vuln]: any) => ({
        name,
        severity: vuln.severity,
        via: vuln.via,
        fixAvailable: vuln.fixAvailable
      }));
  }

  private countFixable(vulnerabilities: any): number {
    return Object.values(vulnerabilities).filter(
      (vuln: any) => vuln.fixAvailable
    ).length;
  }
}
```

### Automated Audit in CI/CD

```yaml
# .github/workflows/security-audit.yml
name: Security Audit

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  schedule:
    # Run daily at 2 AM
    - cron: '0 2 * * *'

jobs:
  audit:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run npm audit
        run: npm audit --audit-level=high
        continue-on-error: true
      
      - name: Generate audit report
        run: npm audit --json > audit-report.json
      
      - name: Upload audit report
        uses: actions/upload-artifact@v3
        with:
          name: audit-report
          path: audit-report.json
      
      - name: Check for critical vulnerabilities
        run: |
          CRITICAL=$(npm audit --json | jq '.metadata.vulnerabilities.critical')
          HIGH=$(npm audit --json | jq '.metadata.vulnerabilities.high')
          
          if [ "$CRITICAL" -gt 0 ]; then
            echo "❌ Found $CRITICAL critical vulnerabilities"
            exit 1
          fi
          
          if [ "$HIGH" -gt 5 ]; then
            echo "⚠️ Found $HIGH high vulnerabilities (threshold: 5)"
            exit 1
          fi
          
          echo "✅ Security audit passed"
```

### Custom Audit Script

```typescript
// scripts/security-audit.ts
import { execSync } from 'child_process';
import * as fs from 'fs';

class SecurityAuditor {
  async performAudit(): Promise<void> {
    console.log('🔍 Running security audit...\n');
    
    // 1. Run npm audit
    const auditResult = await this.runNpmAudit();
    
    // 2. Check severity thresholds
    const violations = this.checkThresholds(auditResult);
    
    // 3. Generate report
    await this.generateReport(auditResult, violations);
    
    // 4. Notify if needed
    if (violations.length > 0) {
      await this.notifyTeam(violations);
    }
    
    // 5. Exit with appropriate code
    process.exit(violations.length > 0 ? 1 : 0);
  }

  private async runNpmAudit(): Promise<any> {
    try {
      const output = execSync('npm audit --json').toString();
      return JSON.parse(output);
    } catch (error) {
      // npm audit exits with 1 if vulnerabilities found
      if (error.stdout) {
        return JSON.parse(error.stdout.toString());
      }
      throw error;
    }
  }

  private checkThresholds(auditResult: any): Violation[] {
    const violations: Violation[] = [];
    const thresholds = {
      critical: 0,  // No critical allowed
      high: 5,      // Max 5 high
      moderate: 20  // Max 20 moderate
    };

    const vulns = auditResult.metadata.vulnerabilities;

    if (vulns.critical > thresholds.critical) {
      violations.push({
        severity: 'CRITICAL',
        count: vulns.critical,
        threshold: thresholds.critical,
        message: `${vulns.critical} critical vulnerabilities found (threshold: ${thresholds.critical})`
      });
    }

    if (vulns.high > thresholds.high) {
      violations.push({
        severity: 'HIGH',
        count: vulns.high,
        threshold: thresholds.high,
        message: `${vulns.high} high vulnerabilities found (threshold: ${thresholds.high})`
      });
    }

    return violations;
  }

  private async generateReport(
    auditResult: any,
    violations: Violation[]
  ): Promise<void> {
    const report = {
      timestamp: new Date().toISOString(),
      summary: auditResult.metadata.vulnerabilities,
      violations,
      vulnerabilities: this.summarizeVulnerabilities(auditResult.vulnerabilities)
    };

    // Save to file
    fs.writeFileSync(
      'security-audit-report.json',
      JSON.stringify(report, null, 2)
    );

    // Console output
    console.log('📊 Audit Summary:');
    console.log(`  Critical: ${report.summary.critical}`);
    console.log(`  High: ${report.summary.high}`);
    console.log(`  Moderate: ${report.summary.moderate}`);
    console.log(`  Low: ${report.summary.low}`);
    console.log(`  Total: ${report.summary.total}\n`);

    if (violations.length > 0) {
      console.log('❌ Threshold violations:');
      violations.forEach(v => console.log(`  - ${v.message}`));
    } else {
      console.log('✅ All thresholds met');
    }
  }

  private summarizeVulnerabilities(vulnerabilities: any): any[] {
    return Object.entries(vulnerabilities).map(([name, vuln]: any) => ({
      package: name,
      severity: vuln.severity,
      fixAvailable: vuln.fixAvailable,
      via: vuln.via.map((v: any) => v.title)
    }));
  }

  private async notifyTeam(violations: Violation[]): Promise<void> {
    // Slack notification
    const message = {
      text: '🚨 Security Audit Failed',
      blocks: [
        {
          type: 'section',
          text: {
            type: 'mrkdwn',
            text: '*Security vulnerabilities detected*'
          }
        },
        {
          type: 'section',
          text: {
            type: 'mrkdwn',
            text: violations.map(v => `• ${v.message}`).join('\n')
          }
        }
      ]
    };

    await fetch(process.env.SLACK_WEBHOOK_URL!, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(message)
    });
  }
}

// Run audit
new SecurityAuditor().performAudit();
```

## Snyk Integration

### Setting Up Snyk

```bash
# Install Snyk CLI
npm install -g snyk

# Authenticate
snyk auth

# Test for vulnerabilities
snyk test

# Monitor project (continuous monitoring)
snyk monitor

# Test with specific severity threshold
snyk test --severity-threshold=high

# Generate HTML report
snyk test --json | snyk-to-html -o snyk-report.html
```

### Snyk Configuration

```json
// .snyk policy file
{
  "version": "v1.25.0",
  "ignore": {
    "SNYK-JS-LODASH-590103": [
      {
        "reason": "No fix available",
        "expires": "2024-12-31T00:00:00.000Z",
        "created": "2024-01-01T00:00:00.000Z"
      }
    ]
  },
  "patch": {},
  "exclude": {
    "global": [
      "node_modules/**"
    ]
  }
}
```

### Snyk in CI/CD

```yaml
# .github/workflows/snyk-security.yml
name: Snyk Security

on: [push, pull_request]

jobs:
  security:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Run Snyk to check for vulnerabilities
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high
      
      - name: Upload Snyk report
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: snyk.sarif
```

### Programmatic Snyk Usage

```typescript
// Use Snyk API
import axios from 'axios';

class SnykIntegration {
  private apiToken: string;
  private orgId: string;

  constructor() {
    this.apiToken = process.env.SNYK_API_TOKEN!;
    this.orgId = process.env.SNYK_ORG_ID!;
  }

  async testProject(projectPath: string): Promise<SnykTestResult> {
    const response = await axios.post(
      `https://snyk.io/api/v1/test/npm`,
      {
        encoding: 'plain',
        files: {
          target: {
            contents: require('fs').readFileSync(
              `${projectPath}/package.json`,
              'utf8'
            )
          },
          additional: [
            {
              contents: require('fs').readFileSync(
                `${projectPath}/package-lock.json`,
                'utf8'
              )
            }
          ]
        }
      },
      {
        headers: {
          'Authorization': `token ${this.apiToken}`,
          'Content-Type': 'application/json'
        }
      }
    );

    return response.data;
  }

  async getProjectIssues(projectId: string): Promise<Issue[]> {
    const response = await axios.post(
      `https://snyk.io/api/v1/org/${this.orgId}/project/${projectId}/issues`,
      {},
      {
        headers: {
          'Authorization': `token ${this.apiToken}`
        }
      }
    );

    return response.data.issues.vulnerabilities;
  }

  async ignoreIssue(
    orgId: string,
    issueId: string,
    reason: string,
    expires: Date
  ): Promise<void> {
    await axios.post(
      `https://snyk.io/api/v1/org/${orgId}/ignore/${issueId}`,
      {
        reason,
        expires: expires.toISOString()
      },
      {
        headers: {
          'Authorization': `token ${this.apiToken}`
        }
      }
    );
  }

  async generateReport(projectId: string): Promise<SecurityReport> {
    const issues = await this.getProjectIssues(projectId);
    
    const report = {
      timestamp: new Date().toISOString(),
      projectId,
      summary: {
        critical: issues.filter(i => i.severity === 'critical').length,
        high: issues.filter(i => i.severity === 'high').length,
        medium: issues.filter(i => i.severity === 'medium').length,
        low: issues.filter(i => i.severity === 'low').length
      },
      vulnerabilities: issues.map(i => ({
        id: i.id,
        title: i.title,
        severity: i.severity,
        package: i.packageName,
        version: i.version,
        fixedIn: i.fixedIn,
        upgradePath: i.upgradePath
      }))
    };

    return report;
  }
}
```

## Dependabot Setup

### Enabling Dependabot

```yaml
# .github/dependabot.yml
version: 2
updates:
  # npm dependencies
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "daily"  # or weekly, monthly
      time: "04:00"
      timezone: "America/New_York"
    
    # Security updates only
    open-pull-requests-limit: 10
    
    # Reviewers and assignees
    reviewers:
      - "security-team"
    assignees:
      - "lead-developer"
    
    # Labels for PRs
    labels:
      - "dependencies"
      - "security"
    
    # Grouping updates
    groups:
      development-dependencies:
        dependency-type: "development"
      production-dependencies:
        dependency-type: "production"
        update-types:
          - "patch"
          - "minor"
    
    # Version update strategy
    versioning-strategy: "increase"  # or lockfile-only, widen, increase-if-necessary
    
    # Ignore specific dependencies
    ignore:
      - dependency-name: "lodash"
        versions: ["4.17.x"]
      - dependency-name: "@types/*"
        update-types: ["version-update:semver-major"]
    
    # Commit message prefix
    commit-message:
      prefix: "chore"
      prefix-development: "chore(dev)"
      include: "scope"
    
    # Target branch
    target-branch: "develop"
    
    # Pull request limit
    open-pull-requests-limit: 5
    
    # Rebase strategy
    rebase-strategy: "auto"

  # GitHub Actions
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
```

### Dependabot Auto-Merge

```yaml
# .github/workflows/dependabot-auto-merge.yml
name: Dependabot Auto-Merge

on:
  pull_request:
    types: [opened, synchronize]

permissions:
  pull-requests: write
  contents: write

jobs:
  auto-merge:
    runs-on: ubuntu-latest
    if: ${{ github.actor == 'dependabot[bot]' }}
    
    steps:
      - name: Dependabot metadata
        id: metadata
        uses: dependabot/fetch-metadata@v1
        
      - name: Auto-merge patch updates
        if: ${{ steps.metadata.outputs.update-type == 'version-update:semver-patch' }}
        run: |
          gh pr merge --auto --squash "$PR_URL"
        env:
          PR_URL: ${{ github.event.pull_request.html_url }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Auto-approve minor updates
        if: ${{ steps.metadata.outputs.update-type == 'version-update:semver-minor' }}
        run: |
          gh pr review --approve "$PR_URL"
        env:
          PR_URL: ${{ github.event.pull_request.html_url }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## Security Updates Workflow

### Update Strategy

```typescript
// Dependency update strategy
class DependencyUpdateStrategy {
  async evaluateUpdate(
    packageName: string,
    currentVersion: string,
    newVersion: string
  ): Promise<UpdateDecision> {
    // 1. Check if security update
    const isSecurityUpdate = await this.isSecurityUpdate(packageName, newVersion);
    
    // 2. Determine update type
    const updateType = this.getUpdateType(currentVersion, newVersion);
    
    // 3. Check breaking changes
    const hasBreakingChanges = await this.checkBreakingChanges(
      packageName,
      currentVersion,
      newVersion
    );
    
    // 4. Make decision
    if (isSecurityUpdate) {
      return {
        action: 'UPDATE_IMMEDIATELY',
        priority: 'CRITICAL',
        runTests: true,
        requireReview: updateType === 'major'
      };
    }
    
    switch (updateType) {
      case 'patch':
        return {
          action: 'AUTO_UPDATE',
          priority: 'LOW',
          runTests: true,
          requireReview: false
        };
      
      case 'minor':
        return {
          action: 'UPDATE_WITH_REVIEW',
          priority: 'MEDIUM',
          runTests: true,
          requireReview: true
        };
      
      case 'major':
        return {
          action: hasBreakingChanges ? 'PLAN_MIGRATION' : 'UPDATE_WITH_TESTING',
          priority: 'LOW',
          runTests: true,
          requireReview: true,
          requireManualTesting: true
        };
      
      default:
        return {
          action: 'IGNORE',
          priority: 'NONE'
        };
    }
  }

  private getUpdateType(
    current: string,
    next: string
  ): 'major' | 'minor' | 'patch' {
    const [currentMajor, currentMinor, currentPatch] = current.split('.').map(Number);
    const [nextMajor, nextMinor, nextPatch] = next.split('.').map(Number);
    
    if (nextMajor > currentMajor) return 'major';
    if (nextMinor > currentMinor) return 'minor';
    return 'patch';
  }

  private async isSecurityUpdate(
    packageName: string,
    version: string
  ): Promise<boolean> {
    // Check npm security advisories
    const advisories = await this.getSecurityAdvisories(packageName);
    return advisories.some(a => a.patchedVersions.includes(version));
  }

  private async checkBreakingChanges(
    packageName: string,
    fromVersion: string,
    toVersion: string
  ): Promise<boolean> {
    // Check changelog for BREAKING CHANGE commits
    const changelog = await this.getChangelog(packageName, fromVersion, toVersion);
    return /BREAKING CHANGE/i.test(changelog);
  }
}
```

### Testing Updates

```typescript
// Automated testing for dependency updates
class DependencyUpdateTester {
  async testUpdate(
    packageName: string,
    newVersion: string
  ): Promise<TestResult> {
    console.log(`Testing update: ${packageName}@${newVersion}`);
    
    // 1. Create test branch
    await this.createTestBranch(packageName, newVersion);
    
    // 2. Update dependency
    await this.updateDependency(packageName, newVersion);
    
    // 3. Install dependencies
    await this.installDependencies();
    
    // 4. Run tests
    const testResults = await this.runTestSuite();
    
    // 5. Build project
    const buildSuccess = await this.buildProject();
    
    // 6. Run integration tests
    const integrationTests = await this.runIntegrationTests();
    
    // 7. Generate report
    return {
      package: packageName,
      version: newVersion,
      tests: testResults,
      build: buildSuccess,
      integration: integrationTests,
      passed: testResults.passed && buildSuccess && integrationTests.passed
    };
  }

  private async runTestSuite(): Promise<TestSuiteResult> {
    const { execSync } = require('child_process');
    
    try {
      execSync('npm test', { stdio: 'inherit' });
      return { passed: true, failures: [] };
    } catch (error) {
      return {
        passed: false,
        failures: this.parseTestFailures(error)
      };
    }
  }

  private async buildProject(): Promise<boolean> {
    try {
      execSync('npm run build', { stdio: 'inherit' });
      return true;
    } catch {
      return false;
    }
  }
}
```

## Supply Chain Attacks

### Detecting Malicious Packages

```typescript
// Supply chain security checks
class SupplyChainSecurityChecker {
  async checkPackage(packageName: string): Promise<SecurityAssessment> {
    return {
      typosquatting: await this.checkTyposquatting(packageName),
      maintainerTrust: await this.checkMaintainerTrust(packageName),
      recentChanges: await this.checkRecentChanges(packageName),
      downloadStats: await this.checkDownloadStats(packageName),
      codeAnalysis: await this.analyzeCode(packageName)
    };
  }

  private async checkTyposquatting(packageName: string): Promise<TyposquattingCheck> {
    // Check for similar package names
    const popularPackages = await this.getPopularPackages();
    const similar = popularPackages.filter(pkg => 
      this.levenshteinDistance(pkg, packageName) <= 2
    );
    
    if (similar.length > 0 && !popularPackages.includes(packageName)) {
      return {
        suspicious: true,
        similarTo: similar,
        warning: `Package name similar to popular packages: ${similar.join(', ')}`
      };
    }
    
    return { suspicious: false };
  }

  private async checkMaintainerTrust(packageName: string): Promise<MaintainerTrust> {
    const packageInfo = await this.getPackageInfo(packageName);
    const maintainers = packageInfo.maintainers;
    
    return {
      verifiedPublisher: maintainers.some(m => m.verified),
      twoFactorAuth: maintainers.some(m => m.twoFactorAuth),
      accountAge: maintainers.map(m => this.getAccountAge(m.created)),
      packageCount: maintainers.map(m => m.packageCount),
      reputation: this.calculateReputation(maintainers)
    };
  }

  private async checkRecentChanges(packageName: string): Promise<ChangeAnalysis> {
    const versions = await this.getVersionHistory(packageName);
    const recentVersion = versions[0];
    const previousVersion = versions[1];
    
    // Check for suspicious changes
    const suspiciousPatterns = [
      /eval\(/,
      /new Function\(/,
      /child_process/,
      /fs\.readFile/,
      /https?:\/\/\d+\.\d+\.\d+\.\d+/,  // IP addresses
      /atob|btoa/  // Base64 encoding/decoding
    ];
    
    const diff = await this.getVersionDiff(previousVersion, recentVersion);
    const matches = suspiciousPatterns.filter(pattern => pattern.test(diff));
    
    return {
      suspicious: matches.length > 0,
      patterns: matches.map(p => p.source),
      maintainerChanged: recentVersion.author !== previousVersion.author
    };
  }

  private async analyzeCode(packageName: string): Promise<CodeAnalysis> {
    const code = await this.downloadPackageCode(packageName);
    
    return {
      obfuscated: this.detectObfuscation(code),
      externalConnections: this.findExternalConnections(code),
      fileSystemAccess: this.findFileSystemAccess(code),
      environmentAccess: this.findEnvironmentAccess(code),
      dangerousFunctions: this.findDangerousFunctions(code)
    };
  }

  private detectObfuscation(code: string): boolean {
    // Check for common obfuscation patterns
    const obfuscationPatterns = [
      /\\x[0-9a-f]{2}/gi,  // Hex encoding
      /eval\s*\(/,  // eval usage
      /Function\s*\(/,  // Function constructor
      /_0x[0-9a-f]+/,  // Common obfuscator variable names
      /[\u0080-\uffff]/  // Non-ASCII characters
    ];
    
    return obfuscationPatterns.some(pattern => pattern.test(code));
  }

  private findExternalConnections(code: string): string[] {
    // Find HTTP/HTTPS connections
    const urlPattern = /https?:\/\/[^\s'"]+/g;
    const urls = code.match(urlPattern) || [];
    
    // Filter out common CDNs and registries
    const trustedDomains = [
      'npmjs.org',
      'registry.npmjs.org',
      'cdn.jsdelivr.net',
      'unpkg.com'
    ];
    
    return urls.filter(url => 
      !trustedDomains.some(domain => url.includes(domain))
    );
  }
}
```

### Package Integrity Verification

```typescript
// Verify package integrity
class PackageIntegrityVerifier {
  async verifyPackage(packageName: string, version: string): Promise<boolean> {
    // 1. Get package metadata
    const metadata = await this.getPackageMetadata(packageName, version);
    
    // 2. Download package
    const packageData = await this.downloadPackage(packageName, version);
    
    // 3. Verify checksum
    const computedHash = this.computeSHA512(packageData);
    
    if (computedHash !== metadata.dist.shasum) {
      throw new Error('Package integrity check failed: checksum mismatch');
    }
    
    // 4. Verify signature (if available)
    if (metadata.dist.signatures) {
      await this.verifySignature(packageData, metadata.dist.signatures);
    }
    
    return true;
  }

  private computeSHA512(data: Buffer): string {
    const crypto = require('crypto');
    return crypto.createHash('sha512').update(data).digest('hex');
  }

  private async verifySignature(
    data: Buffer,
    signatures: any[]
  ): Promise<boolean> {
    // Verify npm signature
    for (const signature of signatures) {
      const isValid = await this.verifyNpmSignature(data, signature);
      if (isValid) return true;
    }
    
    throw new Error('No valid signatures found');
  }
}
```

## Package Integrity (SRI)

### Subresource Integrity Implementation

```typescript
// Generate SRI hashes for dependencies
import crypto from 'crypto';
import fs from 'fs';

class SRIGenerator {
  generateSRIHash(filePath: string, algorithms: string[] = ['sha384']): string {
    const fileContent = fs.readFileSync(filePath);
    
    const hashes = algorithms.map(algorithm => {
      const hash = crypto
        .createHash(algorithm)
        .update(fileContent)
        .digest('base64');
      
      return `${algorithm}-${hash}`;
    });
    
    return hashes.join(' ');
  }

  generateForDirectory(directory: string): Map<string, string> {
    const hashes = new Map<string, string>();
    const files = this.getJavaScriptFiles(directory);
    
    files.forEach(file => {
      const hash = this.generateSRIHash(file);
      hashes.set(file, hash);
    });
    
    return hashes;
  }

  private getJavaScriptFiles(directory: string): string[] {
    const glob = require('glob');
    return glob.sync(`${directory}/**/*.{js,css}`);
  }
}

// Usage in HTML
/*
<script 
  src="https://cdn.example.com/library.js"
  integrity="sha384-oqVuAfXRKap7fdgcCY5uykM6+R9GqQ8K/uxy9rx7HNQlGYl1kPzQho1wx4JwY8wC"
  crossorigin="anonymous">
</script>
*/

// Webpack plugin for automatic SRI
class WebpackSRIPlugin {
  apply(compiler: any) {
    compiler.hooks.compilation.tap('WebpackSRIPlugin', (compilation: any) => {
      compilation.hooks.processAssets.tap(
        {
          name: 'WebpackSRIPlugin',
          stage: compiler.webpack.Compilation.PROCESS_ASSETS_STAGE_OPTIMIZE
        },
        (assets: any) => {
          const sriHashes = new Map<string, string>();
          
          Object.keys(assets).forEach(filename => {
            if (filename.endsWith('.js') || filename.endsWith('.css')) {
              const source = assets[filename].source();
              const hash = crypto
                .createHash('sha384')
                .update(source)
                .digest('base64');
              
              sriHashes.set(filename, `sha384-${hash}`);
            }
          });
          
          // Inject SRI into HTML
          compilation.hooks.processAssets.tap(
            'WebpackSRIPlugin-HTML',
            () => {
              Object.keys(assets).forEach(filename => {
                if (filename.endsWith('.html')) {
                  let content = assets[filename].source();
                  
                  sriHashes.forEach((hash, file) => {
                    const scriptRegex = new RegExp(
                      `<script[^>]*src=["']([^"']*${file})["']`,
                      'g'
                    );
                    content = content.replace(
                      scriptRegex,
                      `<script src="$1" integrity="${hash}" crossorigin="anonymous"`
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
        }
      );
    });
  }
}
```

## Common Mistakes

### 1. Ignoring npm Audit Warnings

```bash
# ❌ BAD: Ignoring audit output
npm install
# (vulnerabilities found but ignored)

# ✓ GOOD: Address vulnerabilities
npm audit
npm audit fix
# Review and fix remaining issues
```

### 2. Using Wildcard Version Ranges

```json
// ❌ BAD: Wildcard versions
{
  "dependencies": {
    "lodash": "*",
    "react": "^18.0.0"
  }
}

// ✓ GOOD: Pin exact versions for critical deps
{
  "dependencies": {
    "lodash": "4.17.21",
    "react": "18.2.0"
  }
}
```

### 3. Not Checking Dependency Licenses

```typescript
// ❌ BAD: Using packages without checking licenses

// ✓ GOOD: Verify licenses
// Use license-checker package
import licenseChecker from 'license-checker';

licenseChecker.init({
  start: '.',
  production: true
}, (err, packages) => {
  const problematic = Object.entries(packages)
    .filter(([_, info]) => {
      const license = info.licenses;
      return ['GPL', 'AGPL', 'LGPL'].some(l => 
        license.includes(l)
      );
    });
  
  if (problematic.length > 0) {
    console.error('Problematic licenses found:', problematic);
    process.exit(1);
  }
});
```

### 4. Trusting All npm Packages

```typescript
// ❌ BAD: Installing without verification
npm install random-package-from-internet

// ✓ GOOD: Verify before installing
// 1. Check npm page (downloads, maintenance)
// 2. Review GitHub repository
// 3. Check maintainers
// 4. Review recent changes
// 5. Run security scan
```

### 5. Not Using Lock Files

```bash
# ❌ BAD: Committing without lock file
git add package.json
git commit

# ✓ GOOD: Always commit lock files
git add package.json package-lock.json
git commit
# Ensures deterministic installs
```

## Best Practices

### 1. Dependency Minimization

```typescript
// Minimize dependency count
class DependencyMinimizer {
  async analyzeDependencies(): Promise<Analysis> {
    const deps = await this.getAllDependencies();
    
    return {
      directDeps: deps.filter(d => d.direct),
      transitiveDeps: deps.filter(d => !d.direct),
      unused: await this.findUnusedDependencies(),
      alternatives: await this.findLighterAlternatives(),
      recommendation: this.generateRecommendations(deps)
    };
  }

  private async findLighterAlternatives(): Promise<Alternative[]> {
    // Find lighter alternatives to heavy packages
    const heavyPackages = [
      { name: 'moment', alternative: 'date-fns', savings: '67KB' },
      { name: 'lodash', alternative: 'lodash-es', savings: 'tree-shakeable' },
      { name: 'axios', alternative: 'fetch API', savings: '13KB' }
    ];
    
    const installed = await this.getInstalledPackages();
    
    return heavyPackages.filter(pkg => 
      installed.includes(pkg.name)
    );
  }
}
```

### 2. Regular Security Audits

```typescript
// Schedule regular audits
class SecurityAuditScheduler {
  schedule(): void {
    // Daily npm audit
    this.scheduleDailyAudit();
    
    // Weekly dependency updates
    this.scheduleWeeklyUpdates();
    
    // Monthly comprehensive review
    this.scheduleMonthlyReview();
  }

  private scheduleDailyAudit(): void {
    cron.schedule('0 2 * * *', async () => {
      const result = await runNpmAudit();
      
      if (result.vulnerabilities.critical > 0) {
        await alertSecurityTeam(result);
      }
    });
  }

  private scheduleWeeklyUpdates(): void {
    cron.schedule('0 3 * * 0', async () => {
      await checkForUpdates();
      await createUpdatePRs();
    });
  }

  private scheduleMonthlyReview(): void {
    cron.schedule('0 9 1 * *', async () => {
      await comprehensiveSecurityReview();
      await generateMonthlyReport();
    });
  }
}
```

### 3. Automated Dependency Updates

```typescript
// Automated update workflow
class AutomatedDependencyUpdater {
  async updateDependencies(): Promise<void> {
    // 1. Check for updates
    const updates = await this.checkForUpdates();
    
    // 2. Filter by safety
    const safeUpdates = updates.filter(u => 
      u.type === 'patch' || (u.type === 'minor' && !u.breaking)
    );
    
    // 3. Test each update
    for (const update of safeUpdates) {
      const success = await this.testUpdate(update);
      
      if (success) {
        await this.applyUpdate(update);
        await this.commitUpdate(update);
      }
    }
    
    // 4. Create PR for manual review
    await this.createPR(safeUpdates);
  }

  private async testUpdate(update: Update): Promise<boolean> {
    // Create isolated test environment
    await this.createTestBranch();
    
    // Apply update
    await this.updatePackage(update.name, update.version);
    
    // Run tests
    const testsPass = await this.runTests();
    const buildsPass = await this.runBuild();
    
    // Cleanup
    await this.deleteTestBranch();
    
    return testsPass && buildsPass;
  }
}
```

### 4. Vulnerability Response Plan

```typescript
// Vulnerability response workflow
class VulnerabilityResponsePlan {
  async respondToVulnerability(vuln: Vulnerability): Promise<void> {
    // 1. Assess severity
    const assessment = this.assessSeverity(vuln);
    
    // 2. Determine action based on severity
    switch (assessment.severity) {
      case 'CRITICAL':
        await this.handleCritical(vuln);
        break;
      case 'HIGH':
        await this.handleHigh(vuln);
        break;
      case 'MODERATE':
        await this.handleModerate(vuln);
        break;
      case 'LOW':
        await this.handleLow(vuln);
        break;
    }
  }

  private async handleCritical(vuln: Vulnerability): Promise<void> {
    // Immediate action required
    await this.alertSecurityTeam('CRITICAL', vuln);
    await this.createHotfixBranch();
    
    if (vuln.fixAvailable) {
      await this.applyFix(vuln);
      await this.runEmergencyTests();
      await this.deployHotfix();
    } else {
      await this.implementWorkaround(vuln);
    }
    
    await this.notifyStakeholders();
  }

  private async handleHigh(vuln: Vulnerability): Promise<void> {
    // Fix within 24-48 hours
    await this.createJiraTicket('HIGH', vuln);
    await this.assignToSecurityTeam();
    
    if (vuln.fixAvailable) {
      await this.scheduleUpdate(vuln, '24h');
    } else {
      await this.investigateAlternatives(vuln);
    }
  }

  private async handleModerate(vuln: Vulnerability): Promise<void> {
    // Fix in next sprint
    await this.addToBacklog(vuln);
    await this.scheduleUpdate(vuln, '2 weeks');
  }

  private async handleLow(vuln: Vulnerability): Promise<void> {
    // Fix in next minor release
    await this.addToBacklog(vuln);
    await this.scheduleUpdate(vuln, '1 month');
  }
}
```

## When to Use/Not to Use

### When to Automate Updates

**Automate:**
- Patch updates (security fixes)
- Minor updates with good test coverage
- Development dependencies
- Well-maintained popular packages

**Manual review:**
- Major version updates
- Breaking changes
- Core dependencies
- Security-sensitive packages

### Dependency Selection Criteria

```typescript
// Evaluate package before adding
class PackageEvaluator {
  async shouldUsePackage(packageName: string): Promise<EvaluationResult> {
    const criteria = {
      downloads: await this.checkDownloads(packageName),
      maintenance: await this.checkMaintenance(packageName),
      security: await this.checkSecurity(packageName),
      license: await this.checkLicense(packageName),
      size: await this.checkSize(packageName),
      alternatives: await this.findAlternatives(packageName)
    };
    
    const score = this.calculateScore(criteria);
    
    return {
      recommended: score >= 70,
      score,
      criteria,
      reasoning: this.generateReasoning(criteria)
    };
  }

  private calculateScore(criteria: any): number {
    // Weighted scoring
    return (
      criteria.downloads.score * 0.2 +
      criteria.maintenance.score * 0.3 +
      criteria.security.score * 0.3 +
      criteria.license.score * 0.1 +
      criteria.size.score * 0.1
    );
  }
}
```

## Interview Questions

### Q1: What is the difference between npm audit and Snyk?

**Answer:**

**npm audit:**
- Built into npm (free, included)
- Checks only npm registry vulnerabilities
- Basic reporting
- `npm audit fix` for automatic fixes
- Good for quick checks

**Snyk:**
- Third-party service (paid features)
- Checks multiple sources (npm, NVD, Snyk database)
- Detailed reports with remediation advice
- Monitors continuously
- Integrates with CI/CD
- Provides fix PRs
- Supports multiple languages/ecosystems

**When to use each:**
- npm audit: Basic security checks, local development
- Snyk: Comprehensive security, enterprise, compliance requirements

**Best practice:** Use both - npm audit for quick checks, Snyk for comprehensive monitoring.

### Q2: Explain the concept of transitive dependencies and their security implications.

**Answer:** Transitive dependencies are dependencies of your dependencies (indirect dependencies).

**Example:**
```
Your app
  └── express
        ├── body-parser
        │     └── raw-body ← Transitive dependency
        └── cookie
```

**Security implications:**

**1. Large attack surface:**
- Average project: 10 direct deps, 1000+ transitive
- Vulnerability in any one affects your app

**2. Invisible vulnerabilities:**
- Don't directly control transitive deps
- Updates may introduce vulnerabilities

**3. Difficult to track:**
- Need tools to identify all dependencies
- `npm ls` shows full tree

**4. Update challenges:**
- Can't directly update transitive deps
- Must update parent package

**Mitigation:**
- Run `npm audit` (checks all dependencies)
- Use tools like Snyk (visualize dependency tree)
- Minimize direct dependencies
- npm `overrides` field to force specific versions:
```json
{
  "overrides": {
    "lodash": "4.17.21"
  }
}
```

### Q3: What is a supply chain attack and how do you prevent it?

**Answer:** Supply chain attack compromises a dependency to attack applications using it.

**Notable examples:**
- **event-stream (2018)**: Malicious code stealing Bitcoin
- **ua-parser-js (2021)**: Cryptominer and password stealer
- **colors/faker (2022)**: Maintainer intentionally broke packages

**Attack vectors:**

**1. Account compromise:**
- Attacker gains access to maintainer's npm account
- Publishes malicious version

**2. Malicious maintainer:**
- Legitimate package sold/transferred to malicious actor
- Typosquatting (lodash vs lodahs)

**3. Dependency confusion:**
- Internal package name same as public package
- npm installs malicious public version

**Prevention strategies:**

**1. Vet packages before installing:**
- Check maintainer reputation
- Review recent changes
- Check download stats
- Read source code

**2. Use package lock files:**
```bash
# Ensures exact versions
npm ci  # Instead of npm install
```

**3. Subresource Integrity (SRI):**
```html
<script 
  src="https://cdn.com/lib.js"
  integrity="sha384-..."
></script>
```

**4. Private registry for internal packages:**
- Prevents dependency confusion
- Control over supply chain

**5. Automated scanning:**
- Snyk, Socket.dev, npm audit
- Detect suspicious changes

**6. Minimal dependencies:**
- Fewer dependencies = smaller attack surface

### Q4: How do you handle a critical vulnerability with no fix available?

**Answer:** Multi-step approach when patch unavailable:

**1. Assess actual risk:**
```typescript
// Is the vulnerable code path actually used?
const analysis = {
  vulnerableFunction: 'parseXML',
  used: await checkIfFunctionUsed('parseXML'),
  exploitable: await assessExploitability(),
  impact: 'HIGH',
  likelihood: 'LOW'
};
```

**2. Implement workarounds:**
- Input validation before vulnerable function
- Disable vulnerable feature
- Use alternative library temporarily

**3. Patch locally:**
```bash
# Create patch file
npm install patch-package
npx patch-package package-name

# Commits patch to git
# Applied automatically on npm install
```

**4. Fork and fix:**
- Fork repository
- Fix vulnerability
- Use forked version
- Submit PR to upstream

**5. Find alternatives:**
- Replace package with safer alternative
- Implement functionality yourself

**6. Isolate risk:**
- Run in sandboxed environment
- Limit permissions
- Add monitoring/logging

**7. Monitor for fix:**
- Subscribe to security advisories
- Check daily for patches
- Have migration plan ready

**Example workflow:**
```typescript
// Temporary workaround
function safeParseXML(input: string) {
  // Validation before vulnerable function
  if (!isValidXML(input)) {
    throw new Error('Invalid XML');
  }
  
  // Sanitize input
  const sanitized = sanitizeXML(input);
  
  // Call vulnerable function with sanitized input
  return vulnerableLibrary.parseXML(sanitized);
}
```

### Q5: What is Dependabot and how does it help with security?

**Answer:** Dependabot is GitHub's automated dependency update tool.

**How it works:**
1. Monitors your dependencies
2. Checks for updates/security patches
3. Creates pull requests automatically
4. Runs your tests on PRs

**Configuration:**
```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "daily"
    open-pull-requests-limit: 10
```

**Benefits:**

**1. Automatic security updates:**
- Detects vulnerabilities immediately
- Creates PRs with patches
- Prioritizes security fixes

**2. Keeps dependencies current:**
- Regular minor/patch updates
- Prevents "version debt"

**3. Reduces manual work:**
- No need to manually check updates
- Automated testing via CI/CD

**4. Transparent:**
- Shows changelog in PR
- Easy to review changes

**Limitations:**

**1. Breaking changes:**
- Major versions require manual review
- May break tests

**2. PR volume:**
- Can create many PRs
- Use grouping and limits

**3. Test quality dependent:**
- Only as good as your tests
- False confidence if tests inadequate

**Best practices:**
- Set appropriate limits (5-10 PRs)
- Group related updates
- Auto-merge patch updates (after tests pass)
- Manual review for major versions
- Keep test coverage high

### Q6: How do you implement a security-first dependency management strategy?

**Answer:** Comprehensive strategy with multiple layers:

**1. Pre-installation vetting:**
```typescript
// Checklist before adding dependency
const packageVetting = {
  downloads: checkWeeklyDownloads(), // > 100k weekly
  maintenance: checkLastUpdate(), // < 6 months
  issues: checkOpenIssues(), // < 50 open issues
  maintainer: checkMaintainerRep(), // Verified, established
  license: checkLicense(), // Compatible
  security: runSecurityScan(), // No known vulns
  alternatives: evaluateAlternatives() // Lighter options?
};
```

**2. Continuous monitoring:**
- Daily: npm audit
- Weekly: Snyk scan
- Monthly: Comprehensive review

**3. Automated updates:**
```yaml
# Dependabot for security patches
# Auto-merge patch updates
# Manual review for minor/major
```

**4. Lock file discipline:**
```bash
# Always commit lock files
git add package-lock.json
# Use npm ci in CI/CD
npm ci
```

**5. Vulnerability response SLA:**
- Critical: 24 hours
- High: 1 week
- Moderate: 2 weeks
- Low: Next release

**6. Development practices:**
- Minimize dependencies
- Prefer standard library
- Use SRI for CDN resources
- Regular dependency audits

**7. Tooling:**
- npm audit (built-in)
- Snyk (comprehensive)
- Socket.dev (supply chain)
- Dependabot (automation)

**8. Governance:**
- Dependency approval process
- Security team review for critical deps
- Regular security training

### Q7: What is package-lock.json and why is it important for security?

**Answer:** package-lock.json ensures deterministic, reproducible installations.

**Purpose:**
- Locks exact versions of all dependencies (including transitive)
- Records integrity hashes
- Ensures same install across environments

**Security benefits:**

**1. Prevents unexpected updates:**
```bash
# Without lock file:
npm install  # Might get react@18.2.1 or 18.2.5

# With lock file:
npm ci  # Always gets react@18.2.0 (locked version)
```

**2. Integrity verification:**
```json
{
  "packages": {
    "node_modules/react": {
      "version": "18.2.0",
      "integrity": "sha512-/3IjMdb2L9QbBdWiW5e3P2/npwMBaU9mHCSCUzNln0ZCYbcfTsGbTJrU/kGemdH2IWmB2ioZ+zkxtmq6g09fGQ=="
    }
  }
}
```
If package modified, integrity check fails.

**3. Audit reproducibility:**
- Security audits based on exact versions
- Consistent results across team

**4. Supply chain protection:**
- Prevents dependency confusion attacks
- Malicious package can't sneak in via version range

**Best practices:**

**1. Always commit:**
```bash
git add package.json package-lock.json
git commit -m "Add dependency"
```

**2. Use npm ci in CI/CD:**
```bash
# package.json: "react": "^18.0.0"
npm install  # Might install 18.3.0
npm ci       # Installs exact version from lock file
```

**3. Keep lock file updated:**
```bash
npm update  # Updates lock file
npm audit fix  # Updates lock file
```

**4. Review lock file changes:**
- Check what versions changed in PRs
- Understand transitive dependency updates

**Warning:** Don't manually edit lock file - use npm commands.

### Q8: How do you balance dependency updates with application stability?

**Answer:** Strategic approach balancing security and stability:

**Update categorization:**

**1. Security patches (immediate):**
```typescript
if (vulnerability.severity === 'CRITICAL' || vulnerability.severity === 'HIGH') {
  updateImmediately();
  runFullTestSuite();
  deployASAP();
}
```

**2. Patch updates (automated):**
```yaml
# Auto-merge after tests pass
# lodash 4.17.20 → 4.17.21
if (updateType === 'patch') {
  automateUpdate();
}
```

**3. Minor updates (weekly batch):**
```yaml
# Group and review weekly
# react 18.2.0 → 18.3.0
if (updateType === 'minor') {
  scheduleForWeeklyReview();
}
```

**4. Major updates (planned):**
```yaml
# Plan migration, test thoroughly
# react 17.0.2 → 18.0.0
if (updateType === 'major') {
  createMigrationPlan();
  allocateDevelopmentTime();
}
```

**Risk mitigation:**

**1. Comprehensive test coverage:**
- Unit tests
- Integration tests
- End-to-end tests
- Ensures updates don't break functionality

**2. Staging environment:**
```bash
# Test updates in staging first
deploy-to-staging
run-smoke-tests
deploy-to-production
```

**3. Gradual rollout:**
- Deploy to 5% → 25% → 50% → 100%
- Monitor errors at each stage

**4. Rollback plan:**
```bash
# Quick rollback if issues
git revert <commit>
deploy
```

**5. Feature flags:**
```typescript
if (featureFlags.newDependencyVersion) {
  useNewVersion();
} else {
  useOldVersion();
}
```

**Decision framework:**
```typescript
const updateDecision = {
  patch: {
    action: 'AUTO_UPDATE',
    testing: 'AUTOMATED',
    review: 'NONE'
  },
  minor: {
    action: 'SCHEDULED_UPDATE',
    testing: 'FULL_SUITE',
    review: 'CODE_REVIEW'
  },
  major: {
    action: 'PLANNED_MIGRATION',
    testing: 'COMPREHENSIVE',
    review: 'ARCHITECTURE_REVIEW'
  }
};
```

**Best practice:** Automate low-risk updates, carefully plan high-risk ones. Never let fear of instability prevent security updates.

## Key Takeaways

1. **Run npm audit regularly** - Daily in CI/CD, immediately before deployments; address critical/high vulnerabilities promptly
2. **Commit lock files always** - Ensures deterministic installs, prevents supply chain attacks, enables integrity verification
3. **Vet packages before installing** - Check downloads, maintenance, maintainers, security history before adding dependencies
4. **Automate security updates** - Use Dependabot or similar for automatic patch updates; manual review for major changes
5. **Minimize dependencies** - Fewer dependencies reduce attack surface; prefer native solutions when possible
6. **Monitor continuously** - Security landscape changes; continuous monitoring catches new vulnerabilities quickly
7. **Use multiple tools** - npm audit for quick checks, Snyk for comprehensive scanning, Socket for supply chain
8. **Have response plan** - Define SLAs for different severity levels; critical vulnerabilities require immediate action
9. **Implement SRI for CDN** - Subresource Integrity prevents CDN compromise from affecting your application
10. **Stay informed** - Subscribe to security advisories, follow security researchers, learn from incidents

## Resources

### Tools
- [npm audit](https://docs.npmjs.com/cli/v8/commands/npm-audit) - Built-in security auditing
- [Snyk](https://snyk.io/) - Comprehensive security platform
- [Socket](https://socket.dev/) - Supply chain security
- [Dependabot](https://github.com/dependabot) - Automated dependency updates
- [OWASP Dependency-Check](https://owasp.org/www-project-dependency-check/) - Multi-language dependency checker

### Documentation
- [npm Security Best Practices](https://docs.npmjs.com/security-best-practices)
- [GitHub Security Advisories](https://github.com/advisories)
- [Node Security Platform](https://npmjs.com/advisories)

### Learning Resources
- [npm Security Blog](https://blog.npmjs.org/post/tags/security)
- [Snyk Learn](https://learn.snyk.io/)
- [Supply Chain Security Best Practices](https://slsa.dev/)

### Incident Reports
- [event-stream Incident](https://blog.npmjs.org/post/180565383195/details-about-the-event-stream-incident)
- [leftpad Incident](https://www.theregister.com/2016/03/23/npm_left_pad_chaos/)
- [ua-parser-js Compromise](https://github.com/advisories/GHSA-pjwm-rvh2-c87w)
