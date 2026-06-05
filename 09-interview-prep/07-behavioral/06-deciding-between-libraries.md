# How Do You Decide Between Two Libraries?

## The Idea

**In plain English:** When building a website or app, developers often use pre-built toolkits (called "libraries") written by others to avoid reinventing the wheel. Deciding between two libraries means figuring out which toolkit is the better fit for your specific project based on things like size, quality, and how well your team knows it.

**Real-world analogy:** Imagine you need to renovate your kitchen and you're choosing between two contractors. You compare their past work, how much they charge, how quickly they respond to problems, and whether your family has worked with them before — not just who has the flashier advertisement.

- The contractor = the library
- The renovation job = the feature you need to build
- Checking past work, pricing, and responsiveness = evaluating bundle size, maintenance activity, and documentation
- Your family's familiarity with a contractor = your team's existing experience with the library

---

## What Interviewers Want to See

Decision-making maturity: that you evaluate on technical merit, team fit, and long-term sustainability — not just GitHub stars or what's trending.

---

## Decision Framework

### 1. Does It Solve the Actual Problem?

Start with requirements, not libraries:
- What specific capability do we need?
- What constraints exist (bundle size, SSR, accessibility, browser support)?
- Do we actually need a library, or can we solve it with ~50 lines of custom code?

```
Problem: "We need a date picker"
Not: "Should we use react-datepicker or react-day-picker?"
But first: Does our design require calendar UI? Or just an <input type="date">?
```

### 2. Ecosystem Signals

| Signal | What to check |
|---|---|
| Maintenance | Last commit date, open issues response time, PR merge rate |
| Adoption | Weekly npm downloads (npmtrends.com), GitHub stars trend |
| Stability | Semantic version, breaking change frequency |
| Backing | Maintained by a company vs solo developer |
| Community | Documentation quality, Stack Overflow answers |

### 3. Technical Fit

- **TypeScript support**: Has proper types? First-party or DefinitelyTyped?
- **Bundle size**: Check bundlephobia.com. Gzipped size matters more than raw size.
- **Tree-shakable**: Can you import only what you use?
- **SSR compatible**: Does it use browser APIs unsafely?
- **Accessibility**: Does it follow WAI-ARIA patterns?
- **Peer dependencies**: Does it require a specific React/Angular version?

### 4. API Quality

Spend 30 minutes building the target use case with each library. Ask:
- Is the API intuitive?
- How does it handle your edge cases?
- How verbose is it for simple and complex cases?
- How easy is migration if we need to change later?

### 5. Team Fit

- Does the team already know one of these?
- How steep is the learning curve?
- Are there internal examples or conventions to follow?

---

## Concrete Example: TanStack Query vs SWR

**Requirements**: Server state management with caching, automatic refetch, loading/error states.

**Evaluation:**

| Dimension | TanStack Query | SWR |
|---|---|---|
| Bundle size | ~12 KB gzipped | ~4 KB gzipped |
| Features | Mutations, infinite queries, prefetching, devtools | Basic queries, mutation hooks |
| TypeScript | Excellent (inferred types) | Good |
| Community | Larger, more docs | Smaller but solid |
| API complexity | More complex for simple cases | Simpler for basic use |
| Framework support | React, Vue, Angular, Solid | React primarily |

**Decision**: If we're building a complex app with mutations, optimistic updates, and pagination → TanStack Query. If we're building a Next.js app with mostly read-heavy data and want the smallest footprint → SWR.

---

## Red Flags to Avoid

- "We should use X because it has more GitHub stars" — stars lag behind quality
- "We should build it ourselves" without estimating the maintenance cost of a custom solution
- Choosing based on a conference talk without evaluating fit for your specific context
- Not checking whether the library handles edge cases you know you'll hit (SSR, a11y, TypeScript strict mode)

---

## Example Answer

"When we were choosing between Zustand and Jotai for client state management, I started by defining what we actually needed: shared state across 5-6 pages, DevTools support, and minimal boilerplate. I spent a few hours building our main use case with both.

Zustand had a simpler API for our use case — the store felt more like a familiar pattern. Jotai's atomic model was more granular but also more complex for our team to reason about.

I checked bundle sizes (both small), TypeScript inference (both good), and maintenance status (both actively maintained by the same team, actually). The deciding factor was that two senior engineers on the team were already comfortable with Zustand's pattern from a previous project — onboarding cost mattered for our timeline. We picked Zustand."

---

## Common Follow-Up Questions

**"What if the team is split between two options?"**
"Build a small proof-of-concept with both, bring the team together to evaluate it, and focus the discussion on specific technical criteria rather than preferences. If still split, the tech lead makes the call and documents the reasoning."

**"Would you ever choose a less popular library?"**
"Yes, if it solves the problem better. For example, `react-aria` (Adobe) has lower npm download numbers than `headless-ui`, but its accessibility implementation is far more thorough. For a product with significant accessibility requirements, I'd choose technical quality over popularity."
