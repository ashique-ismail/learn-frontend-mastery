# Implement Promises from Scratch

## Overview

Implement a Promise from scratch to understand asynchronous JavaScript, the Promise/A+ specification, and chaining mechanisms.

## Basic Promise Implementation

```typescript
enum PromiseState {
  PENDING = 'pending',
  FULFILLED = 'fulfilled',
  REJECTED = 'rejected'
}

type Resolver<T> = (value: T | PromiseLike<T>) => void;
type Rejector = (reason?: any) => void;
type Executor<T> = (resolve: Resolver<T>, reject: Rejector) => void;

class MyPromise<T> {
  private state: PromiseState = PromiseState.PENDING;
  private value: T | undefined;
  private reason: any;
  private onFulfilledCallbacks: Array<(value: T) => void> = [];
  private onRejectedCallbacks: Array<(reason: any) => void> = [];

  constructor(executor: Executor<T>) {
    try {
      executor(this.resolve.bind(this), this.reject.bind(this));
    } catch (error) {
      this.reject(error);
    }
  }

  private resolve(value: T | PromiseLike<T>): void {
    if (this.state !== PromiseState.PENDING) return;

    // Handle Promise resolution
    if (value instanceof MyPromise) {
      value.then(this.resolve.bind(this), this.reject.bind(this));
      return;
    }

    this.state = PromiseState.FULFILLED;
    this.value = value as T;

    // Execute all callbacks
    this.onFulfilledCallbacks.forEach(callback => {
      callback(this.value!);
    });
  }

  private reject(reason?: any): void {
    if (this.state !== PromiseState.PENDING) return;

    this.state = PromiseState.REJECTED;
    this.reason = reason;

    this.onRejectedCallbacks.forEach(callback => {
      callback(this.reason);
    });
  }

  then<TResult1 = T, TResult2 = never>(
    onFulfilled?: ((value: T) => TResult1 | PromiseLike<TResult1>) | null,
    onRejected?: ((reason: any) => TResult2 | PromiseLike<TResult2>) | null
  ): MyPromise<TResult1 | TResult2> {
    return new MyPromise((resolve, reject) => {
      const handleFulfilled = (value: T) => {
        try {
          if (typeof onFulfilled === 'function') {
            const result = onFulfilled(value);
            resolve(result);
          } else {
            resolve(value as any);
          }
        } catch (error) {
          reject(error);
        }
      };

      const handleRejected = (reason: any) => {
        try {
          if (typeof onRejected === 'function') {
            const result = onRejected(reason);
            resolve(result);
          } else {
            reject(reason);
          }
        } catch (error) {
          reject(error);
        }
      };

      if (this.state === PromiseState.FULFILLED) {
        setTimeout(() => handleFulfilled(this.value!), 0);
      } else if (this.state === PromiseState.REJECTED) {
        setTimeout(() => handleRejected(this.reason), 0);
      } else {
        this.onFulfilledCallbacks.push(handleFulfilled);
        this.onRejectedCallbacks.push(handleRejected);
      }
    });
  }

  catch<TResult = never>(
    onRejected?: ((reason: any) => TResult | PromiseLike<TResult>) | null
  ): MyPromise<T | TResult> {
    return this.then(null, onRejected);
  }

  finally(onFinally?: (() => void) | null): MyPromise<T> {
    return this.then(
      value => {
        onFinally?.();
        return value;
      },
      reason => {
        onFinally?.();
        throw reason;
      }
    );
  }

  // Static methods
  static resolve<T>(value: T | PromiseLike<T>): MyPromise<T> {
    if (value instanceof MyPromise) {
      return value;
    }
    return new MyPromise(resolve => resolve(value));
  }

  static reject<T = never>(reason?: any): MyPromise<T> {
    return new MyPromise((_, reject) => reject(reason));
  }

  static all<T>(promises: Array<T | PromiseLike<T>>): MyPromise<T[]> {
    return new MyPromise((resolve, reject) => {
      const results: T[] = [];
      let completed = 0;

      if (promises.length === 0) {
        resolve(results);
        return;
      }

      promises.forEach((promise, index) => {
        MyPromise.resolve(promise).then(
          value => {
            results[index] = value;
            completed++;

            if (completed === promises.length) {
              resolve(results);
            }
          },
          reason => reject(reason)
        );
      });
    });
  }

  static race<T>(promises: Array<T | PromiseLike<T>>): MyPromise<T> {
    return new MyPromise((resolve, reject) => {
      promises.forEach(promise => {
        MyPromise.resolve(promise).then(resolve, reject);
      });
    });
  }

  static allSettled<T>(
    promises: Array<T | PromiseLike<T>>
  ): MyPromise<Array<{ status: 'fulfilled' | 'rejected'; value?: T; reason?: any }>> {
    return new MyPromise(resolve => {
      const results: any[] = [];
      let completed = 0;

      if (promises.length === 0) {
        resolve(results);
        return;
      }

      promises.forEach((promise, index) => {
        MyPromise.resolve(promise).then(
          value => {
            results[index] = { status: 'fulfilled', value };
            completed++;
            if (completed === promises.length) {
              resolve(results);
            }
          },
          reason => {
            results[index] = { status: 'rejected', reason };
            completed++;
            if (completed === promises.length) {
              resolve(results);
            }
          }
        );
      });
    });
  }

  static any<T>(promises: Array<T | PromiseLike<T>>): MyPromise<T> {
    return new MyPromise((resolve, reject) => {
      const errors: any[] = [];
      let rejected = 0;

      if (promises.length === 0) {
        reject(new AggregateError(errors, 'All promises were rejected'));
        return;
      }

      promises.forEach((promise, index) => {
        MyPromise.resolve(promise).then(
          value => resolve(value),
          reason => {
            errors[index] = reason;
            rejected++;
            if (rejected === promises.length) {
              reject(new AggregateError(errors, 'All promises were rejected'));
            }
          }
        );
      });
    });
  }
}

// Usage examples
const promise = new MyPromise<number>((resolve, reject) => {
  setTimeout(() => {
    resolve(42);
  }, 1000);
});

promise
  .then(value => {
    console.log('Value:', value);
    return value * 2;
  })
  .then(doubled => {
    console.log('Doubled:', doubled);
  })
  .catch(error => {
    console.error('Error:', error);
  })
  .finally(() => {
    console.log('Done');
  });
```

## Advanced Patterns

### Promise Chaining

```typescript
// Sequential execution
function sequentialPromises() {
  return MyPromise.resolve(1)
    .then(value => {
      console.log('Step 1:', value);
      return value + 1;
    })
    .then(value => {
      console.log('Step 2:', value);
      return value + 1;
    })
    .then(value => {
      console.log('Step 3:', value);
      return value + 1;
    });
}

// Returning promises in then
function fetchUserData(userId: number) {
  return fetch(`/api/users/${userId}`)
    .then(response => response.json())
    .then(user => {
      // Return another promise
      return fetch(`/api/users/${user.id}/posts`);
    })
    .then(response => response.json())
    .then(posts => {
      console.log('Posts:', posts);
    });
}
```

### Error Propagation

```typescript
function errorPropagation() {
  return MyPromise.resolve()
    .then(() => {
      throw new Error('Step 1 failed');
    })
    .then(() => {
      // This won't execute
      console.log('Step 2');
    })
    .then(() => {
      // This won't execute either
      console.log('Step 3');
    })
    .catch(error => {
      // Error is caught here
      console.error('Caught:', error.message);
      return 'recovered';
    })
    .then(value => {
      // This executes after catch
      console.log('Recovered:', value);
    });
}
```

### Promise.all Patterns

```typescript
// Parallel execution
async function parallelFetch() {
  const [users, posts, comments] = await MyPromise.all([
    fetch('/api/users').then(r => r.json()),
    fetch('/api/posts').then(r => r.json()),
    fetch('/api/comments').then(r => r.json())
  ]);

  return { users, posts, comments };
}

// Fail-fast behavior
async function failFast() {
  try {
    const results = await MyPromise.all([
      MyPromise.resolve(1),
      MyPromise.reject('Error'),
      MyPromise.resolve(3)
    ]);
  } catch (error) {
    console.error('Failed:', error); // Catches 'Error'
  }
}
```

### Promise.allSettled Patterns

```typescript
// Handle partial failures
async function fetchMultiple() {
  const results = await MyPromise.allSettled([
    fetch('/api/users').then(r => r.json()),
    fetch('/api/posts').then(r => r.json()),
    fetch('/api/invalid').then(r => r.json())
  ]);

  results.forEach((result, index) => {
    if (result.status === 'fulfilled') {
      console.log(`Request ${index} succeeded:`, result.value);
    } else {
      console.error(`Request ${index} failed:`, result.reason);
    }
  });
}
```

### Promise.race Patterns

```typescript
// Timeout implementation
function timeout<T>(promise: Promise<T>, ms: number): Promise<T> {
  const timeoutPromise = new MyPromise<T>((_, reject) => {
    setTimeout(() => reject(new Error('Timeout')), ms);
  });

  return MyPromise.race([promise, timeoutPromise]) as Promise<T>;
}

// Usage
timeout(fetch('/api/slow'), 5000)
  .then(response => console.log('Success'))
  .catch(error => console.error('Timeout or error'));
```

## Testing

```typescript
describe('MyPromise', () => {
  describe('Basic functionality', () => {
    it('should resolve with value', (done) => {
      const promise = new MyPromise<number>(resolve => {
        resolve(42);
      });

      promise.then(value => {
        expect(value).toBe(42);
        done();
      });
    });

    it('should reject with reason', (done) => {
      const promise = new MyPromise((_, reject) => {
        reject('Error');
      });

      promise.catch(reason => {
        expect(reason).toBe('Error');
        done();
      });
    });

    it('should handle async resolution', (done) => {
      const promise = new MyPromise<number>(resolve => {
        setTimeout(() => resolve(42), 10);
      });

      promise.then(value => {
        expect(value).toBe(42);
        done();
      });
    });
  });

  describe('Chaining', () => {
    it('should chain then calls', (done) => {
      MyPromise.resolve(1)
        .then(value => value + 1)
        .then(value => value * 2)
        .then(value => {
          expect(value).toBe(4);
          done();
        });
    });

    it('should propagate errors through chain', (done) => {
      MyPromise.resolve()
        .then(() => {
          throw new Error('Test error');
        })
        .then(() => {
          fail('Should not reach here');
        })
        .catch(error => {
          expect(error.message).toBe('Test error');
          done();
        });
    });
  });

  describe('Static methods', () => {
    it('Promise.all should resolve when all resolve', async () => {
      const result = await MyPromise.all([
        MyPromise.resolve(1),
        MyPromise.resolve(2),
        MyPromise.resolve(3)
      ]);

      expect(result).toEqual([1, 2, 3]);
    });

    it('Promise.all should reject when one rejects', async () => {
      try {
        await MyPromise.all([
          MyPromise.resolve(1),
          MyPromise.reject('Error'),
          MyPromise.resolve(3)
        ]);
        fail('Should have rejected');
      } catch (error) {
        expect(error).toBe('Error');
      }
    });

    it('Promise.race should resolve with first', async () => {
      const result = await MyPromise.race([
        new MyPromise(resolve => setTimeout(() => resolve(1), 100)),
        new MyPromise(resolve => setTimeout(() => resolve(2), 50)),
        new MyPromise(resolve => setTimeout(() => resolve(3), 150))
      ]);

      expect(result).toBe(2);
    });

    it('Promise.allSettled should wait for all', async () => {
      const results = await MyPromise.allSettled([
        MyPromise.resolve(1),
        MyPromise.reject('Error'),
        MyPromise.resolve(3)
      ]);

      expect(results).toEqual([
        { status: 'fulfilled', value: 1 },
        { status: 'rejected', reason: 'Error' },
        { status: 'fulfilled', value: 3 }
      ]);
    });
  });
});
```

## Key Takeaways

1. **Promise States**: Three states - pending, fulfilled, rejected. Can only transition once.

2. **Executor Function**: Runs synchronously, receives resolve and reject functions.

3. **Then Chaining**: Returns new promise, enabling chaining and transformation.

4. **Error Handling**: Errors propagate down chain until caught by catch handler.

5. **Callback Arrays**: Store callbacks when promise is pending, execute when settled.

6. **Async Resolution**: Use setTimeout to ensure then callbacks execute asynchronously.

7. **Promise.all**: Resolves when all resolve, rejects when any rejects.

8. **Promise.race**: Settles with first settled promise result.

9. **Finally Handler**: Executes regardless of outcome, doesn't receive arguments.

10. **Type Safety**: Use TypeScript generics for type-safe promise operations.

## Interview Talking Points

- Explain the three promise states
- Describe the microtask queue and event loop
- Discuss error propagation in promise chains
- Compare Promise.all vs Promise.allSettled vs Promise.race vs Promise.any
- Explain how then returns a new promise
- Describe callback registration for pending promises
- Discuss thenable objects and promise resolution
- Explain finally implementation
- Compare promises vs callbacks vs async/await
- Discuss memory considerations with promise chains
