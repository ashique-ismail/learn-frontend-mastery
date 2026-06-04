# Promises and Async/Await - JavaScript Interview Questions

## Table of Contents
- [Core Concepts](#core-concepts)
- [Common Interview Questions](#common-interview-questions)
- [Advanced Questions](#advanced-questions)
- [Error Handling Patterns](#error-handling-patterns)
- [What Interviewers Look For](#what-interviewers-look-for)
- [Red Flags to Avoid](#red-flags-to-avoid)
- [Key Takeaways](#key-takeaways)

## Core Concepts

### What is a Promise?

A Promise is an object representing the eventual completion or failure of an asynchronous operation. It's a proxy for a value that may not be known when the promise is created.

**Promise States:**
1. **Pending**: Initial state, neither fulfilled nor rejected
2. **Fulfilled**: Operation completed successfully
3. **Rejected**: Operation failed

**Key Characteristics:**
- Promises are immutable once settled (fulfilled or rejected)
- Promises can only settle once
- Promises are eager (execute immediately when created)
- Promises enable better error handling than callbacks

### Async/Await

Async/await is syntactic sugar built on top of Promises, making asynchronous code look and behave more like synchronous code.

**Key Points:**
- `async` functions always return a Promise
- `await` can only be used inside `async` functions
- `await` pauses execution until Promise resolves
- Error handling uses try/catch blocks

## Common Interview Questions

### Question 1: What is a Promise and How Does It Work?

**Expected Answer:**

A Promise is an object that represents the eventual result of an asynchronous operation. It acts as a placeholder for a value that will be available in the future.

```javascript
// Creating a Promise
const promise = new Promise((resolve, reject) => {
  // Asynchronous operation
  const success = true;
  
  if (success) {
    resolve('Operation successful!');
  } else {
    reject('Operation failed!');
  }
});

// Consuming a Promise
promise
  .then(result => console.log(result))
  .catch(error => console.error(error))
  .finally(() => console.log('Cleanup'));
```

**Real-World Example:**
```javascript
function fetchUserData(userId) {
  return new Promise((resolve, reject) => {
    // Simulate API call
    setTimeout(() => {
      if (userId > 0) {
        resolve({
          id: userId,
          name: 'John Doe',
          email: 'john@example.com'
        });
      } else {
        reject(new Error('Invalid user ID'));
      }
    }, 1000);
  });
}

fetchUserData(1)
  .then(user => {
    console.log('User:', user);
    return fetchUserData(2); // Chain another promise
  })
  .then(user => console.log('Second user:', user))
  .catch(error => console.error('Error:', error.message));
```

**What Interviewers Look For:**
- Understanding of the three states
- Knowledge of resolve/reject functions
- Proper error handling with catch
- Understanding of Promise chaining

### Question 2: Explain Promise Chaining

**Expected Answer:**

Promise chaining allows you to perform a sequence of asynchronous operations where each operation starts when the previous one completes.

```javascript
// Basic chaining
fetch('https://api.example.com/user/1')
  .then(response => response.json())
  .then(user => {
    console.log('User:', user);
    return fetch(`https://api.example.com/posts?userId=${user.id}`);
  })
  .then(response => response.json())
  .then(posts => {
    console.log('User posts:', posts);
  })
  .catch(error => {
    console.error('Error in chain:', error);
  });
```

**Important Rules:**
1. Always return a value or Promise from `.then()`
2. If you don't return, next `.then()` receives `undefined`
3. Errors propagate down the chain until caught
4. You can return a new Promise to continue the chain

**Common Mistake:**
```javascript
// WRONG - Nesting instead of chaining
fetch('/api/user')
  .then(response => {
    return response.json().then(user => {
      return fetch(`/api/posts?userId=${user.id}`).then(response => {
        return response.json().then(posts => {
          console.log(posts);
        });
      });
    });
  });

// RIGHT - Proper chaining
fetch('/api/user')
  .then(response => response.json())
  .then(user => fetch(`/api/posts?userId=${user.id}`))
  .then(response => response.json())
  .then(posts => console.log(posts))
  .catch(error => console.error(error));
```

### Question 3: What's the Difference Between Promise.all, Promise.race, Promise.allSettled, and Promise.any?

**Expected Answer:**

These are Promise combinators that handle multiple promises differently.

**Promise.all() - Wait for all to succeed:**
```javascript
const promise1 = Promise.resolve(3);
const promise2 = new Promise(resolve => setTimeout(() => resolve('foo'), 100));
const promise3 = fetch('/api/data').then(r => r.json());

Promise.all([promise1, promise2, promise3])
  .then(([result1, result2, result3]) => {
    console.log(result1, result2, result3);
    // All resolved values in order
  })
  .catch(error => {
    // Rejects if ANY promise rejects
    console.error('One of the promises failed:', error);
  });
```

**Use Case:** When you need all operations to succeed (e.g., loading multiple required resources).

**Promise.race() - First to settle wins:**
```javascript
const slow = new Promise(resolve => setTimeout(() => resolve('slow'), 2000));
const fast = new Promise(resolve => setTimeout(() => resolve('fast'), 100));

Promise.race([slow, fast])
  .then(result => console.log(result)) // 'fast'
  .catch(error => console.error(error));

// Practical: Timeout implementation
function fetchWithTimeout(url, timeout = 5000) {
  return Promise.race([
    fetch(url),
    new Promise((_, reject) => 
      setTimeout(() => reject(new Error('Timeout')), timeout)
    )
  ]);
}
```

**Use Case:** Timeouts, racing multiple sources, first available response.

**Promise.allSettled() - Wait for all, regardless of outcome:**
```javascript
const promises = [
  fetch('/api/users'),
  fetch('/api/posts'),
  fetch('/api/invalid-endpoint') // Will fail
];

Promise.allSettled(promises)
  .then(results => {
    results.forEach((result, index) => {
      if (result.status === 'fulfilled') {
        console.log(`Promise ${index} succeeded:`, result.value);
      } else {
        console.log(`Promise ${index} failed:`, result.reason);
      }
    });
  });

// Result format:
// [
//   { status: 'fulfilled', value: response1 },
//   { status: 'fulfilled', value: response2 },
//   { status: 'rejected', reason: error }
// ]
```

**Use Case:** When you want results from all operations regardless of failures (e.g., batch operations with error reporting).

**Promise.any() - First to fulfill wins:**
```javascript
const promises = [
  fetch('https://slow-api.com/data'),
  fetch('https://fast-api.com/data'),
  fetch('https://medium-api.com/data')
];

Promise.any(promises)
  .then(firstSuccess => {
    console.log('First successful response:', firstSuccess);
  })
  .catch(error => {
    // Only rejects if ALL promises reject
    console.error('All promises rejected:', error);
  });
```

**Use Case:** Multiple fallback sources, fastest successful response.

**Comparison Table:**
```javascript
// Summary
Promise.all([p1, p2, p3])
// Resolves: When ALL fulfill
// Rejects: When ANY rejects
// Returns: Array of all results

Promise.race([p1, p2, p3])
// Resolves: When FIRST settles (fulfill or reject)
// Rejects: If first to settle rejects
// Returns: First settled result

Promise.allSettled([p1, p2, p3])
// Resolves: When ALL settle
// Rejects: Never
// Returns: Array of {status, value/reason}

Promise.any([p1, p2, p3])
// Resolves: When FIRST fulfills
// Rejects: When ALL reject
// Returns: First fulfilled result
```

### Question 4: Async/Await vs Promises - When to Use Each?

**Expected Answer:**

Async/await is syntactic sugar over Promises, making async code more readable and easier to reason about.

**Promise Syntax:**
```javascript
function getUserWithPosts(userId) {
  return fetch(`/api/users/${userId}`)
    .then(response => response.json())
    .then(user => {
      return fetch(`/api/posts?userId=${user.id}`)
        .then(response => response.json())
        .then(posts => {
          return { user, posts };
        });
    })
    .catch(error => {
      console.error('Error:', error);
      throw error;
    });
}
```

**Async/Await Syntax:**
```javascript
async function getUserWithPosts(userId) {
  try {
    const userResponse = await fetch(`/api/users/${userId}`);
    const user = await userResponse.json();
    
    const postsResponse = await fetch(`/api/posts?userId=${user.id}`);
    const posts = await postsResponse.json();
    
    return { user, posts };
  } catch (error) {
    console.error('Error:', error);
    throw error;
  }
}
```

**When to Use Promises:**
1. **Parallel operations:**
```javascript
// Better with Promise.all
Promise.all([
  fetch('/api/users'),
  fetch('/api/posts'),
  fetch('/api/comments')
]).then(([users, posts, comments]) => {
  // Process results
});
```

2. **Complex chains with branching:**
```javascript
fetchUser(id)
  .then(user => {
    if (user.isPremium) {
      return fetchPremiumContent();
    }
    return fetchRegularContent();
  })
  .then(content => process(content));
```

**When to Use Async/Await:**
1. **Sequential operations:**
```javascript
async function processOrder(orderId) {
  const order = await fetchOrder(orderId);
  const payment = await processPayment(order);
  const shipment = await scheduleShipment(order, payment);
  return shipment;
}
```

2. **Better error handling:**
```javascript
async function complexOperation() {
  try {
    const step1 = await operation1();
    const step2 = await operation2(step1);
    const step3 = await operation3(step2);
    return step3;
  } catch (error) {
    // Handle errors from any step
    console.error('Operation failed:', error);
    throw error;
  }
}
```

3. **Conditional logic:**
```javascript
async function smartFetch(url) {
  try {
    const response = await fetch(url);
    
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}`);
    }
    
    const contentType = response.headers.get('content-type');
    if (contentType.includes('application/json')) {
      return await response.json();
    } else {
      return await response.text();
    }
  } catch (error) {
    console.error('Fetch failed:', error);
    throw error;
  }
}
```

### Question 5: How Do You Handle Errors in Promises and Async/Await?

**Expected Answer:**

**Promise Error Handling:**
```javascript
// Method 1: .catch() at the end
fetch('/api/data')
  .then(response => response.json())
  .then(data => processData(data))
  .catch(error => {
    console.error('Error:', error);
    // Handle error
  });

// Method 2: .catch() in the middle (recovery)
fetch('/api/data')
  .then(response => response.json())
  .catch(error => {
    console.error('Fetch failed, using fallback');
    return getFallbackData(); // Recover from error
  })
  .then(data => processData(data));

// Method 3: Second argument to .then()
fetch('/api/data')
  .then(
    response => response.json(),
    error => {
      console.error('Fetch error:', error);
      return null;
    }
  );
```

**Async/Await Error Handling:**
```javascript
// Method 1: Try/catch block
async function fetchData() {
  try {
    const response = await fetch('/api/data');
    const data = await response.json();
    return data;
  } catch (error) {
    console.error('Error:', error);
    throw error; // Re-throw or handle
  }
}

// Method 2: Try/catch with specific error handling
async function robustFetch(url) {
  try {
    const response = await fetch(url);
    
    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`);
    }
    
    return await response.json();
  } catch (error) {
    if (error instanceof TypeError) {
      console.error('Network error:', error);
    } else if (error instanceof SyntaxError) {
      console.error('Invalid JSON:', error);
    } else {
      console.error('Unknown error:', error);
    }
    
    throw error;
  }
}

// Method 3: Multiple try/catch blocks
async function multiStepProcess() {
  let step1Result;
  
  try {
    step1Result = await step1();
  } catch (error) {
    console.error('Step 1 failed:', error);
    throw error;
  }
  
  try {
    const step2Result = await step2(step1Result);
    return step2Result;
  } catch (error) {
    console.error('Step 2 failed:', error);
    // Maybe retry or use fallback
    return await fallbackOperation();
  }
}

// Method 4: Higher-order function for error handling
function asyncHandler(fn) {
  return async function(...args) {
    try {
      return await fn(...args);
    } catch (error) {
      console.error('Async error:', error);
      // Centralized error handling
      throw error;
    }
  };
}

const safeFetch = asyncHandler(async (url) => {
  const response = await fetch(url);
  return response.json();
});
```

## Advanced Questions

### Question 6: Implement Promise.all from Scratch

**Interview Question:** "Can you implement your own version of Promise.all?"

**Solution:**
```javascript
function promiseAll(promises) {
  return new Promise((resolve, reject) => {
    // Handle edge cases
    if (!Array.isArray(promises)) {
      return reject(new TypeError('Argument must be an array'));
    }
    
    if (promises.length === 0) {
      return resolve([]);
    }
    
    const results = [];
    let completedCount = 0;
    
    promises.forEach((promise, index) => {
      // Ensure it's a promise
      Promise.resolve(promise)
        .then(value => {
          results[index] = value;
          completedCount++;
          
          // All promises completed
          if (completedCount === promises.length) {
            resolve(results);
          }
        })
        .catch(error => {
          // Reject on first error
          reject(error);
        });
    });
  });
}

// Test
const p1 = Promise.resolve(1);
const p2 = new Promise(resolve => setTimeout(() => resolve(2), 100));
const p3 = Promise.resolve(3);

promiseAll([p1, p2, p3])
  .then(results => console.log(results)) // [1, 2, 3]
  .catch(error => console.error(error));
```

**Follow-up:** Implement Promise.race:
```javascript
function promiseRace(promises) {
  return new Promise((resolve, reject) => {
    if (!Array.isArray(promises)) {
      return reject(new TypeError('Argument must be an array'));
    }
    
    if (promises.length === 0) {
      return; // Never settles
    }
    
    promises.forEach(promise => {
      Promise.resolve(promise)
        .then(resolve)
        .catch(reject);
    });
  });
}
```

### Question 7: Async Iteration and Sequencing

**Interview Question:** "How do you process an array of async operations sequentially?"

**Problem:**
```javascript
const urls = ['/api/user/1', '/api/user/2', '/api/user/3'];

// WRONG - All fire simultaneously
urls.forEach(async (url) => {
  const response = await fetch(url);
  console.log(await response.json());
});
```

**Solution 1: For...of loop**
```javascript
async function processSequentially(urls) {
  for (const url of urls) {
    const response = await fetch(url);
    const data = await response.json();
    console.log(data);
  }
}
```

**Solution 2: Reduce**
```javascript
async function processSequentially(urls) {
  return urls.reduce(async (previousPromise, url) => {
    await previousPromise;
    const response = await fetch(url);
    return response.json();
  }, Promise.resolve());
}
```

**Solution 3: Recursive**
```javascript
async function processSequentially(urls, index = 0) {
  if (index >= urls.length) return;
  
  const response = await fetch(urls[index]);
  const data = await response.json();
  console.log(data);
  
  return processSequentially(urls, index + 1);
}
```

**Solution 4: Custom sequential executor**
```javascript
async function sequential(tasks) {
  const results = [];
  
  for (const task of tasks) {
    const result = await task();
    results.push(result);
  }
  
  return results;
}

// Usage
const tasks = urls.map(url => () => fetch(url).then(r => r.json()));
const results = await sequential(tasks);
```

**Parallel with concurrency limit:**
```javascript
async function parallelLimit(tasks, limit) {
  const results = [];
  const executing = [];
  
  for (const [index, task] of tasks.entries()) {
    const promise = Promise.resolve().then(() => task());
    results.push(promise);
    
    if (limit <= tasks.length) {
      const execute = promise.then(() => {
        executing.splice(executing.indexOf(execute), 1);
      });
      executing.push(execute);
      
      if (executing.length >= limit) {
        await Promise.race(executing);
      }
    }
  }
  
  return Promise.all(results);
}

// Execute max 3 requests at a time
const tasks = urls.map(url => () => fetch(url).then(r => r.json()));
const results = await parallelLimit(tasks, 3);
```

### Question 8: Promise Anti-patterns and Common Mistakes

**Anti-pattern 1: The Promisor (unnecessary Promise wrapping)**
```javascript
// WRONG
async function getUser(id) {
  return new Promise(async (resolve, reject) => {
    try {
      const response = await fetch(`/api/users/${id}`);
      const user = await response.json();
      resolve(user);
    } catch (error) {
      reject(error);
    }
  });
}

// RIGHT
async function getUser(id) {
  const response = await fetch(`/api/users/${id}`);
  return response.json();
}
```

**Anti-pattern 2: The Ghost Promise (not returning)**
```javascript
// WRONG
function processData() {
  fetch('/api/data')
    .then(response => response.json())
    .then(data => console.log(data));
  // Function returns undefined, not a Promise!
}

// RIGHT
function processData() {
  return fetch('/api/data')
    .then(response => response.json())
    .then(data => console.log(data));
}
```

**Anti-pattern 3: Broken chain**
```javascript
// WRONG
function getData() {
  return fetch('/api/data')
    .then(response => {
      response.json(); // Not returning!
    })
    .then(data => {
      console.log(data); // undefined
    });
}

// RIGHT
function getData() {
  return fetch('/api/data')
    .then(response => response.json())
    .then(data => console.log(data));
}
```

**Anti-pattern 4: Using async without await**
```javascript
// WRONG - Unnecessary async
async function multiplyBy2(x) {
  return x * 2; // No async operation
}

// RIGHT
function multiplyBy2(x) {
  return x * 2;
}
```

**Anti-pattern 5: Not handling errors**
```javascript
// WRONG
async function fetchData() {
  const response = await fetch('/api/data');
  return response.json();
  // Errors will silently fail
}

// RIGHT
async function fetchData() {
  try {
    const response = await fetch('/api/data');
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}`);
    }
    return response.json();
  } catch (error) {
    console.error('Fetch failed:', error);
    throw error;
  }
}
```

## Error Handling Patterns

### Pattern 1: Retry Logic
```javascript
async function fetchWithRetry(url, options = {}, maxRetries = 3) {
  let lastError;
  
  for (let i = 0; i < maxRetries; i++) {
    try {
      const response = await fetch(url, options);
      if (!response.ok) {
        throw new Error(`HTTP ${response.status}`);
      }
      return response;
    } catch (error) {
      lastError = error;
      console.log(`Attempt ${i + 1} failed, retrying...`);
      
      // Exponential backoff
      await new Promise(resolve => 
        setTimeout(resolve, Math.pow(2, i) * 1000)
      );
    }
  }
  
  throw lastError;
}
```

### Pattern 2: Fallback Chain
```javascript
async function fetchWithFallbacks(urls) {
  for (const url of urls) {
    try {
      const response = await fetch(url);
      if (response.ok) {
        return await response.json();
      }
    } catch (error) {
      console.log(`Failed to fetch from ${url}, trying next...`);
    }
  }
  
  throw new Error('All endpoints failed');
}

// Usage
const data = await fetchWithFallbacks([
  'https://primary-api.com/data',
  'https://backup-api.com/data',
  'https://fallback-api.com/data'
]);
```

### Pattern 3: Timeout with Cleanup
```javascript
function withTimeout(promise, timeoutMs) {
  let timeoutId;
  
  const timeoutPromise = new Promise((_, reject) => {
    timeoutId = setTimeout(() => {
      reject(new Error(`Operation timed out after ${timeoutMs}ms`));
    }, timeoutMs);
  });
  
  return Promise.race([promise, timeoutPromise])
    .finally(() => clearTimeout(timeoutId));
}

// Usage
try {
  const data = await withTimeout(
    fetch('/api/slow-endpoint'),
    5000
  );
} catch (error) {
  console.error('Request timed out');
}
```

### Pattern 4: Cancellable Promises
```javascript
function makeCancellable(promise) {
  let cancelled = false;
  
  const wrappedPromise = new Promise((resolve, reject) => {
    promise
      .then(value => {
        if (!cancelled) {
          resolve(value);
        }
      })
      .catch(error => {
        if (!cancelled) {
          reject(error);
        }
      });
  });
  
  return {
    promise: wrappedPromise,
    cancel() {
      cancelled = true;
    }
  };
}

// Usage
const { promise, cancel } = makeCancellable(fetch('/api/data'));

// Cancel if needed
setTimeout(() => cancel(), 1000);

try {
  const data = await promise;
} catch (error) {
  // Won't trigger if cancelled
}
```

## What Interviewers Look For

### 1. Fundamental Understanding
- Clear explanation of Promise states
- Understanding of async/await transformation
- Knowledge of microtask queue
- Difference between Promise and callback

### 2. Practical Skills
- Proper error handling
- Promise chaining vs async/await choice
- Parallel vs sequential execution
- Appropriate use of combinators

### 3. Problem-Solving
- Can implement Promise patterns from scratch
- Handles edge cases
- Understands timing and execution order
- Can debug async issues

### 4. Best Practices
- Always handles errors
- Returns Promises from functions
- Avoids anti-patterns
- Uses appropriate async patterns

### 5. Advanced Knowledge
- Understands Promise internals
- Can implement custom Promise utilities
- Knows about cancellation and cleanup
- Familiar with async iteration

## Red Flags to Avoid

### 1. Misunderstanding Basics
- Confusing Promises with callbacks
- Not knowing the three states
- Thinking await makes code synchronous

### 2. Poor Error Handling
- No try/catch with async/await
- No .catch() with Promises
- Silently swallowing errors

### 3. Common Mistakes
- Not returning Promises
- Breaking Promise chains
- Using async without await unnecessarily
- forEach with async functions

### 4. Performance Issues
- Running sequential when parallel is possible
- Not using Promise.all for independent operations
- Creating unnecessary Promise wrappers

### 5. Code Quality
- Callback hell with Promises (nested .then())
- Not handling all Promise states
- Inconsistent error handling

## Key Takeaways

### Essential Concepts
1. **Promises have three states**: pending, fulfilled, rejected
2. **Async/await is syntactic sugar**: It's built on Promises
3. **Always handle errors**: Use .catch() or try/catch
4. **Return Promises**: Don't break the chain
5. **Choose the right tool**: Promise.all for parallel, for...of for sequential

### Best Practices
1. **Prefer async/await** for readability
2. **Use Promise.all** for parallel operations
3. **Handle all error cases** explicitly
4. **Return Promises** from async functions
5. **Avoid nesting** Promises (Promise hell)

### Common Patterns
1. **Retry logic** with exponential backoff
2. **Timeout** with Promise.race
3. **Sequential processing** with for...of
4. **Parallel with limit** for controlled concurrency
5. **Fallback chains** for resilience

### Interview Success Tips
1. **Start simple**: Explain basics before advanced topics
2. **Show error handling**: Always include try/catch
3. **Discuss trade-offs**: Parallel vs sequential, Promise vs async/await
4. **Write clean code**: Avoid anti-patterns
5. **Test edge cases**: Empty arrays, single item, all failures
6. **Explain execution order**: Understand microtask queue

Remember: Promises are fundamental to modern JavaScript. Showing both theoretical knowledge and practical application of async patterns will demonstrate your expertise in handling asynchronous operations effectively.
