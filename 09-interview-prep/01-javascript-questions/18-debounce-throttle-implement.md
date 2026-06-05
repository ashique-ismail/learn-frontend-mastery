# Implement Debounce & Throttle

## The Idea

**In plain English:** Debounce and throttle are two techniques that control how often a function (a set of instructions the computer runs) is allowed to execute when something triggers it repeatedly — like typing or scrolling. Debounce waits for the action to stop before running, while throttle runs at most once every set amount of time no matter how many triggers happen.

**Real-world analogy:** Imagine a bathroom hand-dryer with a motion sensor. When you wave your hands in front of it, it decides whether to turn on based on a rule.

- Debounce version: the dryer only turns on after you stop waving for 2 seconds — this is like debounce, which waits for a pause before acting.
- Throttle version: the dryer can only turn on once every 5 seconds, no matter how many times you wave — this is like throttle, which limits how often the action can run.
- The waving = the repeated events (typing, scrolling) that keep triggering the function
- The dryer turning on = the function (the code) that actually runs
- The 2-second pause / 5-second limit = the delay or interval that controls when execution is allowed

---

## Why This Comes Up in Interviews

Both patterns limit the rate of function execution. Interviewers ask you to implement them from scratch to test your closure knowledge, timer management, and understanding of when each applies.

---

## Debounce

**Concept:** Wait until a burst of calls stops before executing. Only the *last* call in a rapid sequence fires.

**Use cases:** search-as-you-type, resize handler, autosave, form validation on input.

### Basic Implementation

```js
function debounce(fn, delay) {
  let timer = null;
  return function(...args) {
    clearTimeout(timer);
    timer = setTimeout(() => {
      fn.apply(this, args);
    }, delay);
  };
}

const search = debounce((query) => fetchResults(query), 300);
input.addEventListener('input', (e) => search(e.target.value));
```

### With Leading Edge Option

```js
function debounce(fn, delay, { leading = false } = {}) {
  let timer = null;
  return function(...args) {
    const callNow = leading && !timer;
    clearTimeout(timer);
    timer = setTimeout(() => {
      timer = null;
      if (!leading) fn.apply(this, args);
    }, delay);
    if (callNow) fn.apply(this, args);
  };
}
```

`leading: true` fires immediately on first call, then ignores calls until quiet.

### React Hook

```js
function useDebounce(value, delay) {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(id);
  }, [value, delay]);
  return debounced;
}
```

---

## Throttle

**Concept:** Execute at most once per interval, no matter how many calls arrive. Guaranteed minimum call rate.

**Use cases:** scroll event handlers, mousemove, rate-limited API calls, game loop input.

### Timestamp-Based (Recommended)

```js
function throttle(fn, interval) {
  let lastTime = 0;
  return function(...args) {
    const now = performance.now();
    if (now - lastTime >= interval) {
      lastTime = now;
      fn.apply(this, args);
    }
  };
}

const onScroll = throttle(() => updateScrollIndicator(), 100);
window.addEventListener('scroll', onScroll);
```

### Timer-Based (Trailing Edge)

```js
function throttle(fn, interval) {
  let timer = null;
  return function(...args) {
    if (timer) return;
    timer = setTimeout(() => {
      fn.apply(this, args);
      timer = null;
    }, interval);
  };
}
```

### With Both Leading and Trailing

```js
function throttle(fn, interval) {
  let lastTime = 0;
  let timer = null;
  return function(...args) {
    const now = performance.now();
    const remaining = interval - (now - lastTime);
    if (remaining <= 0) {
      clearTimeout(timer);
      timer = null;
      lastTime = now;
      fn.apply(this, args);
    } else if (!timer) {
      timer = setTimeout(() => {
        lastTime = performance.now();
        timer = null;
        fn.apply(this, args);
      }, remaining);
    }
  };
}
```

---

## Key Differences

| | Debounce | Throttle |
|---|---|---|
| Fires when | After quiet period | At most once per interval |
| Best for | Bursty events where only last matters | Continuous events needing regular sampling |
| Example | Search input | Scroll position tracking |
| Risk of never firing | Yes (if calls keep coming) | No |

---

## Common Interview Follow-Ups

**Q: How do you cancel a debounced call?**
```js
function debounce(fn, delay) {
  let timer = null;
  function debounced(...args) {
    clearTimeout(timer);
    timer = setTimeout(() => fn.apply(this, args), delay);
  }
  debounced.cancel = () => clearTimeout(timer);
  return debounced;
}
const search = debounce(fetchResults, 300);
search.cancel(); // cancel pending call
```

**Q: Can debounce cause a call to never fire?**
Yes — if events keep firing within the delay window (e.g. continuous typing), the debounced function never executes. This is why autosave UIs combine debounce with a max-wait option.

**Q: Which does lodash `_.debounce` use by default — leading or trailing?**
Trailing edge (executes after the wait period ends), with optional `leading` and `trailing` flags.

**Q: What happens to `this` inside a debounced method?**
Arrow functions in the setTimeout capture the outer `this` at definition time. Using `fn.apply(this, args)` correctly forwards the `this` from the call site. Arrow function debounced wrappers would capture the wrong context.
