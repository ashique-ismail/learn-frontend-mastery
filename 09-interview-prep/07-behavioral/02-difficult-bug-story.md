# Describe a Difficult Bug You Fixed

## What Interviewers Want to See

Systematic debugging methodology, not luck. They want evidence that you can isolate root causes under pressure rather than guessing at fixes until something works.

---

## STAR Framework for Bug Stories

**Situation** — What was the context? Why did it matter?
**Task** — Your responsibility in fixing it
**Action** — Your diagnostic process, step by step
**Result** — Fix, prevention, and lessons learned

The **Action** section is most important — walk through your reasoning, not just the solution.

---

## Example Answer: Race Condition in Production

**Situation:**
"We had a critical bug in production where a small percentage of users (~2%) saw their profile data replaced with another user's data after saving changes. It was intermittent — couldn't reproduce in development. This was a data privacy issue and a P0 incident."

**Task:**
"I was the senior frontend engineer on call. I needed to identify the root cause quickly — we'd had complaints for 3 days without finding the cause."

**Action:**
"First, I looked for patterns in the reports:
- Only users who saved profile changes while navigating between pages
- Always happened when two API calls were in flight simultaneously
- Sentry showed both requests completing successfully

That pointed to a race condition. I looked at our data fetching code for the profile page:

```ts
// In useEffect (the buggy version)
useEffect(() => {
  if (userId) {
    fetchUserProfile(userId).then(data => setUserData(data));
  }
}, [userId]);
```

The problem: when the user navigated from profile A to profile B quickly, profile A's fetch could complete after profile B's fetch. The `setUserData` for profile A would overwrite profile B's data.

I confirmed this by adding `console.log` timestamps in development and rapidly changing the `userId` — I reproduced it within 2 minutes.

The fix:
```ts
useEffect(() => {
  let isCancelled = false;
  fetchUserProfile(userId).then(data => {
    if (!isCancelled) setUserData(data);
  });
  return () => { isCancelled = true; };
}, [userId]);
```

I also added an AbortController to actually cancel the in-flight request (saves bandwidth)."

**Result:**
"We deployed the fix within 2 hours of identifying the root cause. The race condition bug rate dropped to zero. I wrote a team document about the pattern and added it to our code review checklist — 'check all useEffect data fetching for cancellation.' We've not had this class of bug since."

---

## Example Answer: Memory Leak Causing Tab Crash

**Situation:**
"Our Angular dashboard would crash Chrome tabs after 2-3 hours of use. Users reported it only on the data-intensive analytics pages."

**Action:**
"I used Chrome DevTools Memory tab — took heap snapshots every 30 minutes. Memory grew linearly without ever decreasing, even after navigating away from the analytics page. That told me something was keeping references alive that shouldn't be.

The Retainer tree in the heap snapshot showed hundreds of `Subscription` objects accumulating. I traced them to an analytics service that subscribed to an interval Observable and never unsubscribed:

```ts
// Bug: no takeUntil or unsubscribe
this.interval(5000).pipe(
  switchMap(() => this.api.getMetrics())
).subscribe(data => this.metrics.next(data));
```

When users navigated away, Angular destroyed the component but not the service's subscriptions (the service was `providedIn: 'root'`). Each navigation created new subscriptions while the old ones kept running.

Fix: added `takeUntilDestroyed()` (Angular 16+) to all service subscriptions, and added `ngOnDestroy` to explicitly unsubscribe where needed."

**Result:**
"Memory growth stopped. Tab crashes disappeared. As a prevention measure, I added an ESLint rule to warn on `.subscribe()` without a clear unsubscription strategy."

---

## Key Diagnostic Vocabulary

- **Reproduce reliably first** — can't fix what you can't reproduce
- **Narrow the scope** — binary search the codebase, not random guessing
- **Separate symptoms from root causes** — the crash was a symptom; the subscription leak was the cause
- **Verify the fix** — don't declare victory without proof the issue is gone
- **Prevent recurrence** — a fix without a prevention mechanism will happen again

---

## Common Follow-Up Questions

**"How long did it take you to find this bug?"**
Be honest. "It took 6 hours to find but 30 minutes to fix" is a legitimate and impressive answer — it shows the debugging work was the hard part.

**"How could the bug have been caught earlier?"**
"A memory profiling step in our staging environment would have caught this before production. We added periodic heap snapshot checks to our performance testing suite."

**"What was the hardest part?"**
"The intermittent nature — bugs you can't reliably reproduce are the hardest. I had to think carefully about what conditions could cause it rather than just running the code and observing."
