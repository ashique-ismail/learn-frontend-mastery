# How Do You Handle Technical Debt?

## The Idea

**In plain English:** Technical debt is the hidden mess that builds up in a codebase when developers take shortcuts to ship things faster — like rushing to tidy your room by stuffing everything under the bed. Over time, that hidden mess slows everyone down and makes new work harder.

**Real-world analogy:** Imagine a restaurant kitchen where, during a rush, cooks leave dirty pots piled up, mise en place out of order, and labels missing from containers — just to keep serving tables. It works tonight, but the next shift walks into chaos.

- The dirty pots and unlabelled containers = messy, undocumented code
- The next shift struggling to find anything = new developers slowed down by confusing code
- Scheduling a deep clean on a quiet Monday = dedicating sprint time to paying off technical debt

---

## What Interviewers Want to Hear

That you treat tech debt as a business concern (not just a developer annoyance), that you can articulate the tradeoffs clearly to non-technical stakeholders, and that you have a practical strategy for addressing it without stopping feature delivery.

---

## Defining Technical Debt

Technical debt = the accumulated cost of shortcuts taken to ship faster. Like financial debt, it compounds — small debt accumulates interest (slower development, more bugs, higher onboarding cost).

Not all debt is bad — sometimes a quick solution is the right choice if you'll replace it soon. The problem is **unacknowledged** or **unmanaged** debt.

---

## Types (Important Distinction)

| Type | Description | Urgency |
|---|---|---|
| **Deliberate + prudent** | "We'll refactor this after launch" | Planned payback |
| **Inadvertent** | Discovered later: "we didn't know better" | Address as encountered |
| **Bit rot** | Dependencies outdated, patterns stale | Ongoing maintenance |
| **Accidental** | No design, written in a hurry | Track and schedule |

---

## Practical Strategies

### 1. Make It Visible

Invisible debt is unmanageable. Catalog it:

```
Tech Debt Log:
- [HIGH] Auth module uses jwt-decode v2 (EOL) — security risk
- [MED] UserProfile page: 800-line component, zero tests — fragile
- [LOW] Inconsistent error handling patterns across API layer
- [LOW] CSS color values hardcoded, not using design tokens
```

This moves the conversation from "everything is messy" to "here are 4 specific items with priorities."

### 2. Attach Business Value to Each Item

Engineers say: "The auth module is outdated."
Product hears: nothing actionable.

Reframe: "If we don't update the auth library by Q2, we lose SOC 2 compliance. That blocks enterprise sales. Here's the 3-day estimate to fix it."

The business value makes debt prioritizable against features.

### 3. The 20% Rule (or Boy Scout Rule)

Negotiate dedicated time — typically 10-20% of each sprint — for debt reduction. Frame it as "engineering health" not "cleanup." This prevents debt accumulation without halting feature work.

Or apply the **Boy Scout Rule**: leave the code cleaner than you found it. Refactoring in small increments during feature work, without dedicated tickets.

### 4. Strangler Fig Pattern for Large Refactors

Don't rewrite everything at once. Wrap the old system, incrementally replace pieces:

```
Sprint 1: New components alongside old, behind feature flag
Sprint 2: Route new traffic to new components
Sprint 3: Remove old components
```

This keeps the system working while replacing it incrementally.

---

## Example Answer (STAR)

**Situation:**
"We inherited an Angular 9 codebase with no tests, mixed state management patterns (some NgRx, some component state, some service state), and a bundle that took 8 seconds to load on mobile."

**Task:**
"As the new tech lead, I needed to improve the codebase without stopping a 6-month product roadmap."

**Action:**
"I documented all debt items in a shared spreadsheet with business impact and effort estimates. Three items had direct user impact: the bundle size, the broken form validation, and a memory leak causing app crashes.

I got buy-in from the product manager to dedicate 20% of each sprint to debt. In parallel:
- Fixed the bundle size (tree-shaking, lazy routes) — deployed in week 2, LCP dropped from 8s to 2.5s
- Fixed the memory leak — reduced crash reports by 90%
- Deferred the form validation rewrite to a dedicated sprint after launch

We also added a rule: any new feature must have tests. We didn't retroactively test the old code — just stopped accumulating new untested code."

**Result:**
"After 4 months, load time was down from 8s to 2.1s, crash reports dropped 90%, and new developers could read the code without a 2-hour guided tour. We never stopped shipping features — debt work was interleaved, not a separate phase."

---

## Common Follow-Up Questions

**"How do you convince stakeholders to prioritize debt?"**
"I translate it to business cost. 'The checkout flow has no tests. Last quarter, two bugs in checkout cost us 48 hours of engineering time to debug. Adding tests would cost 16 hours now and prevent the next incident.'"

**"What if you can't get time for debt reduction?"**
"I negotiate for the Boy Scout approach — no dedicated sprint, but we commit to small improvements during feature work. It's slower but prevents net accumulation."

**"When is it OK to take on new technical debt?"**
"When you're validating an idea quickly and plan to replace it. When a deadline is genuinely critical. The key is making it explicit and time-bounded — not debt by accident but debt by decision."
