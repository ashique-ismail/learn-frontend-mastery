# Design a Multi-Tenant SaaS Frontend

## The Idea

**In plain English:** A multi-tenant SaaS (Software as a Service) is a single app that serves many different companies at once, where each company (called a "tenant") sees their own version with their own logo, colors, data, and settings — but they're all running on the same underlying code.

**Real-world analogy:** Think of a shopping mall where many different stores operate under one roof. Each store has its own name, decor, staff, and products — but they all share the same building, electricity, and parking lot.

- The mall building = the shared app code and servers
- Each individual store = each tenant (company) using the app
- Each store's unique decor and signage = per-tenant theming and branding
- The store's private stock room = each tenant's isolated data that other stores cannot access

---

## What Is Multi-Tenancy?

Multiple customers (tenants) share the same application codebase, but each experiences their own isolated, customised version. Tenants may be companies, organizations, or user groups.

---

## Tenant Identification

### Subdomain-Based (Most Common)

```
acme.myapp.com     → Tenant: Acme Corp
globex.myapp.com   → Tenant: Globex Inc
```

```ts
// Extract tenant from hostname on the client
function getTenantId(): string {
  const hostname = window.location.hostname;
  const subdomain = hostname.split('.')[0];
  // Map 'acme' to tenant config
  return subdomain;
}
```

### Path-Based

```
myapp.com/org/acme/dashboard
myapp.com/org/globex/settings
```

### JWT Claim

```ts
// After login, JWT contains tenant info
interface JwtPayload {
  sub: string;       // userId
  tenantId: string;  // 'acme'
  tenantPlan: 'starter' | 'pro' | 'enterprise';
  roles: string[];
}
```

---

## Per-Tenant Theming

### CSS Custom Properties (Recommended)

```ts
// Fetch tenant config on app load
async function loadTenantTheme(tenantId: string) {
  const theme = await api.getTenantTheme(tenantId);
  applyTheme(theme);
}

function applyTheme(theme: TenantTheme) {
  const root = document.documentElement;
  root.style.setProperty('--color-primary', theme.primaryColor);
  root.style.setProperty('--color-accent', theme.accentColor);
  root.style.setProperty('--logo-url', `url(${theme.logoUrl})`);
  root.style.setProperty('--font-family', theme.fontFamily);
}
```

```css
.button-primary {
  background: var(--color-primary, #3b82f6); /* fallback to default */
  font-family: var(--font-family, Inter, sans-serif);
}
```

### Theme Provider (React)

```tsx
interface TenantTheme {
  primaryColor: string;
  logoUrl: string;
  companyName: string;
}

const TenantContext = createContext<TenantTheme | null>(null);

export function TenantProvider({ children }: { children: ReactNode }) {
  const tenantId = getTenantId();
  const { data: theme } = useQuery({
    queryKey: ['tenant-theme', tenantId],
    queryFn: () => api.getTenantTheme(tenantId),
    staleTime: 5 * 60 * 1000, // cache for 5 minutes
  });

  if (!theme) return <LoadingScreen />;

  return (
    <TenantContext.Provider value={theme}>
      {children}
    </TenantContext.Provider>
  );
}
```

---

## Feature Flags per Tenant/Plan

```ts
interface FeatureFlags {
  advancedReporting: boolean;
  apiAccess: boolean;
  ssoIntegration: boolean;
  customDomain: boolean;
  auditLogs: boolean;
}

function useFeatureFlag(feature: keyof FeatureFlags): boolean {
  const { tenantPlan } = useAuth();
  const flags = PLAN_FEATURES[tenantPlan];
  return flags[feature] ?? false;
}

const PLAN_FEATURES: Record<string, FeatureFlags> = {
  starter: { advancedReporting: false, apiAccess: false, ssoIntegration: false, customDomain: false, auditLogs: false },
  pro: { advancedReporting: true, apiAccess: true, ssoIntegration: false, customDomain: false, auditLogs: false },
  enterprise: { advancedReporting: true, apiAccess: true, ssoIntegration: true, customDomain: true, auditLogs: true },
};
```

```tsx
// Usage
function ReportsPage() {
  const hasAdvancedReports = useFeatureFlag('advancedReporting');

  return (
    <div>
      <BasicReport />
      {hasAdvancedReports ? <AdvancedReport /> : <UpgradePrompt feature="Advanced Reports" />}
    </div>
  );
}
```

---

## Role-Based Access Control (RBAC)

```ts
// Roles are tenant-specific
type Role = 'admin' | 'manager' | 'member' | 'viewer';

interface Permission {
  resource: 'projects' | 'billing' | 'users' | 'settings';
  action: 'read' | 'write' | 'delete' | 'admin';
}

const ROLE_PERMISSIONS: Record<Role, Permission[]> = {
  admin: [
    { resource: 'projects', action: 'admin' },
    { resource: 'billing', action: 'admin' },
    { resource: 'users', action: 'admin' },
  ],
  manager: [
    { resource: 'projects', action: 'write' },
    { resource: 'users', action: 'read' },
  ],
  member: [
    { resource: 'projects', action: 'write' },
  ],
  viewer: [
    { resource: 'projects', action: 'read' },
  ],
};

function usePermission(resource: string, action: string): boolean {
  const { user } = useAuth();
  return hasPermission(user.role, resource, action);
}
```

---

## Tenant-Scoped Data Fetching

Every API request must be scoped to the tenant:

```ts
// All API calls include tenant context via:
// 1. Subdomain (handled by load balancer routing to tenant DB)
// 2. JWT claim (backend extracts tenant from token)
// 3. Request header (X-Tenant-ID header)

const apiClient = axios.create({
  baseURL: '/api',
});

apiClient.interceptors.request.use(config => {
  config.headers['Authorization'] = `Bearer ${getToken()}`;
  // Backend extracts tenantId from JWT — no need to send separately
  return config;
});
```

---

## White-Labelling Considerations

```ts
// Replace app branding with tenant branding
function useTenantBranding() {
  const { companyName, logoUrl, favicon } = useTenantTheme();

  useEffect(() => {
    document.title = companyName;
    document.querySelector('link[rel="icon"]')?.setAttribute('href', favicon);
  }, [companyName, favicon]);

  return { logoUrl, companyName };
}
```

---

## Onboarding Flow

```tsx
function OnboardingFlow() {
  const [step, setStep] = useState<'org' | 'plan' | 'invite' | 'done'>('org');

  return (
    <OnboardingLayout currentStep={step}>
      {step === 'org' && <OrgSetupStep onComplete={() => setStep('plan')} />}
      {step === 'plan' && <PlanSelectionStep onComplete={() => setStep('invite')} />}
      {step === 'invite' && <InviteTeamStep onComplete={() => setStep('done')} />}
      {step === 'done' && <Redirect to="/dashboard" />}
    </OnboardingLayout>
  );
}
```

---

## Interview Discussion Points

**Q: How do you prevent data leakage between tenants?**
Primary isolation is on the backend (row-level security in Postgres, separate schemas, or separate databases per tenant). The frontend treats tenantId from the JWT as authoritative — it never trusts client-side tenant selection for data access decisions.

**Q: How do you test multi-tenant features?**
Seed test data with multiple tenant configurations. Write integration tests that authenticate as users in different tenants. Test that tenant A cannot access tenant B's data even with valid tokens.

**Q: What's the risk with CSS custom property theming?**
Tenant-injected values could potentially exploit CSS injection if you're setting arbitrary string values. Sanitize and validate all theme values on the backend before serving them.
