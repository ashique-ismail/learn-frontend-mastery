# Implement Promise.all, race, allSettled, any

## Why It's Asked

Implementing Promise combinators tests deep understanding of Promise mechanics, async state tracking, and edge cases like empty arrays and early rejection.

---

## `Promise.all`

Resolves when **all** promises resolve. Rejects immediately on **any** rejection.

```js
function promiseAll(promises) {
  return new Promise((resolve, reject) => {
    if (promises.length === 0) return resolve([]);

    const results = new Array(promises.length);
    let remaining = promises.length;

    promises.forEach((promise, index) => {
      Promise.resolve(promise).then(value => {
        results[index] = value;
        remaining--;
        if (remaining === 0) resolve(results);
      }).catch(reject);  // first rejection short-circuits
    });
  });
}

promiseAll([
  Promise.resolve(1),
  Promise.resolve(2),
  Promise.resolve(3),
]).then(console.log); // [1, 2, 3]

promiseAll([
  Promise.resolve(1),
  Promise.reject('error'),
]).catch(console.log); // 'error'
```

**Key details:**
- `results[index] = value` preserves input order regardless of resolution order
- `Promise.resolve(promise)` handles non-Promise values in the input array
- Empty array resolves immediately with `[]`

---

## `Promise.race`

Resolves/rejects with the **first** promise that settles (either way).

```js
function promiseRace(promises) {
  return new Promise((resolve, reject) => {
    for (const promise of promises) {
      Promise.resolve(promise).then(resolve).catch(reject);
    }
  });
}

// Note: empty promises array → pending forever (same as native)
```

**Use case:** Timeout pattern:
```js
function withTimeout(promise, ms) {
  const timeout = new Promise((_, reject) =>
    setTimeout(() => reject(new Error('Timeout')), ms)
  );
  return promiseRace([promise, timeout]);
}
```

---

## `Promise.allSettled`

Waits for **all** promises to settle. Never rejects. Returns array of `{ status, value/reason }` objects.

```js
function promiseAllSettled(promises) {
  return new Promise(resolve => {
    if (promises.length === 0) return resolve([]);

    const results = new Array(promises.length);
    let remaining = promises.length;

    promises.forEach((promise, index) => {
      Promise.resolve(promise)
        .then(value => {
          results[index] = { status: 'fulfilled', value };
        })
        .catch(reason => {
          results[index] = { status: 'rejected', reason };
        })
        .finally(() => {
          remaining--;
          if (remaining === 0) resolve(results);
        });
    });
  });
}

promiseAllSettled([
  Promise.resolve(1),
  Promise.reject('err'),
  Promise.resolve(3),
]).then(console.log);
// [
//   { status: 'fulfilled', value: 1 },
//   { status: 'rejected', reason: 'err' },
//   { status: 'fulfilled', value: 3 }
// ]
```

---

## `Promise.any`

Resolves with the **first fulfilled** promise. Only rejects if **all** promises reject, with an `AggregateError`.

```js
function promiseAny(promises) {
  return new Promise((resolve, reject) => {
    if (promises.length === 0) {
      return reject(new AggregateError([], 'All promises were rejected'));
    }

    const errors = new Array(promises.length);
    let rejectedCount = 0;

    promises.forEach((promise, index) => {
      Promise.resolve(promise)
        .then(resolve)  // first fulfillment wins
        .catch(reason => {
          errors[index] = reason;
          rejectedCount++;
          if (rejectedCount === promises.length) {
            reject(new AggregateError(errors, 'All promises were rejected'));
          }
        });
    });
  });
}
```

---

## Comparison

| Method | Resolves when | Rejects when | Returns |
|--------|---------------|--------------|---------|
| `all` | All fulfill | Any rejects | Array of values (in order) |
| `race` | First settles | First rejects | Single value/reason |
| `allSettled` | All settle | Never | Array of status objects |
| `any` | First fulfills | All reject | Single value / AggregateError |

---

## Common Interview Questions

**Q: Why does `Promise.all([])` resolve synchronously?**
The native implementation resolves microtask-asynchronously (in the next microtask), but conceptually: if there are zero promises, we immediately have all the results (an empty array).

**Q: Does `Promise.all` cancel other promises when one rejects?**
No. Promises have no cancellation mechanism. The other promises continue running — `Promise.all` just stops waiting for them and rejects its own returned promise.

**Q: What does `AggregateError` contain?**
An `errors` property that is an array of all rejection reasons, in the same order as the input promises.

**Q: What's the difference between `Promise.any` and `Promise.race`?**
`Promise.race` settles on the first *settled* promise (fulfilled or rejected). `Promise.any` only resolves on the first *fulfilled* promise — it ignores rejections unless all promises reject.
