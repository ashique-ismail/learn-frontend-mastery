# Tell Me About a Time You Optimized Performance

## What Interviewers Are Looking For

They want to see your diagnostic process — not just that you "made it faster," but that you measured first, identified the real bottleneck, applied targeted fixes, and verified improvement. The STAR format works well here.

---

## STAR Framework

**Situation** — Set the context (what was slow, why it mattered)
**Task** — Your responsibility in addressing it
**Action** — What you actually did (diagnostic steps + fixes)
**Result** — Measurable improvement

---

## Example Answer Structure

**Situation:**
"We had a React dashboard that showed financial data across 15-20 widgets. Users reported it felt sluggish — any interaction caused a noticeable freeze. Our Core Web Vitals were showing INP above 500ms."

**Task:**
"As the frontend lead, I was responsible for diagnosing and fixing the responsiveness issues before our enterprise launch."

**Action — Diagnose first:**
"I started with Chrome DevTools Performance tab and recorded a typical interaction. I saw several 200ms+ long tasks blocking the main thread. Then I used React DevTools Profiler — it showed that every widget was re-rendering on each state change, even widgets that had nothing to do with the update.

The root cause was our context structure — we had a single `AppContext` holding all state (user, theme, data, filters). Any change to filters caused all 20 widgets to re-render.

I applied three targeted fixes:
1. Split the single context into smaller, purpose-specific contexts (UserContext, ThemeContext, FilterContext) — widgets only subscribed to what they needed
2. Wrapped the expensive chart computations in `useMemo` with precise dependency arrays
3. Added `React.memo` to the widget components that were pure functions of their props

For the data-heavy tables, I discovered they were rendering thousands of rows — I replaced them with TanStack Virtual, only rendering visible rows."

**Result:**
"INP dropped from 500ms to under 80ms. The Profiler showed re-renders reduced from ~20 components per interaction to 2-3. We measured this in Sentry's performance monitoring before and after the release."

---

## Key Technical Vocabulary to Use

- **Profiling first, optimizing second** — "I measured before changing anything"
- **INP, LCP, CLS** — Core Web Vitals by name
- **Long tasks** — tasks blocking the main thread >50ms
- **Reconciliation** — React's process of diffing component trees
- **Virtualization** — rendering only visible items in a list
- **Memoization** — caching computation results

---

## Common Follow-Up Questions

**"How did you prioritize which issue to fix first?"**
"I ranked by impact — the context splitting affected all 20 widgets, so it had the highest multiplier. The table virtualization was specific to one component but caused the worst user-visible freeze."

**"Did any of your optimizations have downsides?"**
"Yes — `React.memo` adds a shallow comparison cost on every render, so it only pays off when re-renders are expensive. I profiled the overhead and confirmed the tradeoff was worth it for our heavier widgets, but I removed it from simpler ones."

**"What would you do differently?"**
"I'd establish a performance budget and monitoring earlier in the project — Sentry or web-vitals in CI so we catch regressions before they reach production."

---

## If You Don't Have a Great Real Example

**Angular version:**
"I profiled an Angular admin panel where the main table had 500+ rows with default change detection. Every route change triggered a full CD cycle checking all rows. I added `OnPush` to the table row components and used the `async` pipe with an Observable instead of property binding — CD now only triggers when the data Observable emits. Time-to-interactive dropped by 60%."

**CSS/Bundle version:**
"I ran webpack-bundle-analyzer and found we were importing Moment.js (600KB) and the full lodash package. Replacing Moment with date-fns (individual function imports) and switching to tree-shakable lodash-es reduced our main bundle by 40%, directly improving LCP by ~800ms."
