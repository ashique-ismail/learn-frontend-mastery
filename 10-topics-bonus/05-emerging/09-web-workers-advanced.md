# Advanced Web Workers

## Introduction

Web Workers enable true multithreading in JavaScript, running scripts in background threads without blocking the main thread. Advanced patterns include worker pools, shared workers, and complex communication strategies for building high-performance web applications.

## Basic Web Worker

```javascript
// main.js
const worker = new Worker('worker.js');

worker.postMessage({ type: 'start', data: [1, 2, 3, 4, 5] });

worker.onmessage = (event) => {
    console.log('Result:', event.data);
};

worker.onerror = (error) => {
    console.error('Worker error:', error);
};
```

```javascript
// worker.js
self.onmessage = (event) => {
    const { type, data } = event.data;
    
    if (type === 'start') {
        const result = data.reduce((sum, num) => sum + num, 0);
        self.postMessage(result);
    }
};
```

## Worker Pool

```javascript
class WorkerPool {
    constructor(workerScript, poolSize = 4) {
        this.workers = [];
        this.taskQueue = [];
        this.availableWorkers = [];
        
        for (let i = 0; i < poolSize; i++) {
            const worker = new Worker(workerScript);
            worker.id = i;
            this.workers.push(worker);
            this.availableWorkers.push(worker);
            
            worker.onmessage = (event) => {
                this.handleWorkerComplete(worker, event.data);
            };
        }
    }
    
    execute(data) {
        return new Promise((resolve, reject) => {
            const task = { data, resolve, reject };
            
            if (this.availableWorkers.length > 0) {
                this.assignTask(task);
            } else {
                this.taskQueue.push(task);
            }
        });
    }
    
    assignTask(task) {
        const worker = this.availableWorkers.shift();
        worker.currentTask = task;
        worker.postMessage(task.data);
    }
    
    handleWorkerComplete(worker, result) {
        if (worker.currentTask) {
            worker.currentTask.resolve(result);
            worker.currentTask = null;
        }
        
        if (this.taskQueue.length > 0) {
            this.assignTask(this.taskQueue.shift());
        } else {
            this.availableWorkers.push(worker);
        }
    }
    
    terminate() {
        this.workers.forEach(w => w.terminate());
    }
}

// Usage
const pool = new WorkerPool('worker.js', 4);

const tasks = Array.from({ length: 100 }, (_, i) => pool.execute({ value: i }));
const results = await Promise.all(tasks);
console.log('All tasks complete:', results);
```

## Shared Workers

```javascript
// shared-worker.js
const connections = [];

self.onconnect = (event) => {
    const port = event.ports[0];
    connections.push(port);
    
    port.onmessage = (e) => {
        // Broadcast to all connections
        connections.forEach((p) => {
            if (p !== port) {
                p.postMessage(e.data);
            }
        });
    };
    
    port.start();
};
```

```javascript
// main.js - Using shared worker
const sharedWorker = new SharedWorker('shared-worker.js');

sharedWorker.port.onmessage = (event) => {
    console.log('Message from shared worker:', event.data);
};

sharedWorker.port.postMessage({ type: 'hello', from: 'tab1' });
```

## Transferable Objects

```javascript
// Transfer large data efficiently
const arrayBuffer = new ArrayBuffer(1024 * 1024); // 1MB

// Transfer ownership (zero-copy)
worker.postMessage(
    { buffer: arrayBuffer },
    [arrayBuffer] // Transferable list
);

// arrayBuffer is now unusable in main thread
console.log(arrayBuffer.byteLength); // 0
```

## Complex Worker Communication

```javascript
// worker-manager.js
class WorkerManager {
    constructor(workerScript) {
        this.worker = new Worker(workerScript);
        this.callbacks = new Map();
        this.messageId = 0;
        
        this.worker.onmessage = (event) => {
            const { id, result, error } = event.data;
            const callback = this.callbacks.get(id);
            
            if (callback) {
                if (error) {
                    callback.reject(error);
                } else {
                    callback.resolve(result);
                }
                this.callbacks.delete(id);
            }
        };
    }
    
    execute(method, ...args) {
        return new Promise((resolve, reject) => {
            const id = this.messageId++;
            this.callbacks.set(id, { resolve, reject });
            
            this.worker.postMessage({ id, method, args });
        });
    }
}

// advanced-worker.js
const methods = {
    async processData(data) {
        // Long running task
        return data.map(x => x * 2);
    },
    
    async calculate(a, b) {
        return a + b;
    }
};

self.onmessage = async (event) => {
    const { id, method, args } = event.data;
    
    try {
        const result = await methods[method](...args);
        self.postMessage({ id, result });
    } catch (error) {
        self.postMessage({ id, error: error.message });
    }
};

// Usage
const manager = new WorkerManager('advanced-worker.js');
const result = await manager.execute('processData', [1, 2, 3]);
```

## Worker with WebAssembly

```javascript
// wasm-worker.js
let wasmInstance;

self.onmessage = async (event) => {
    if (event.data.type === 'init') {
        const response = await fetch('module.wasm');
        const buffer = await response.arrayBuffer();
        const { instance } = await WebAssembly.instantiate(buffer);
        wasmInstance = instance;
        self.postMessage({ type: 'ready' });
    } else if (event.data.type === 'compute') {
        const result = wasmInstance.exports.compute(event.data.value);
        self.postMessage({ type: 'result', value: result });
    }
};
```

## Browser Support

| Feature | Chrome | Firefox | Safari | Edge |
|---------|--------|---------|--------|------|
| Web Workers | 4+ | 3.5+ | 4+ | 12+ |
| Shared Workers | 4+ | 29+ | 16+ | 79+ |
| Module Workers | 80+ | 114+ | 15+ | 80+ |

## Best Practices

1. Use worker pools for multiple tasks
2. Transfer large data when possible
3. Handle errors properly
4. Terminate workers when done
5. Avoid blocking worker thread
6. Use SharedArrayBuffer carefully
7. Profile worker performance
8. Consider fallbacks

## Key Takeaways

- Workers enable true parallelism
- Worker pools manage resources
- Transferable objects avoid copying
- Shared workers enable cross-tab communication
- Works great with WebAssembly
- Essential for CPU-intensive tasks

## Resources

- [MDN: Web Workers API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API)
- [HTML Living Standard: Workers](https://html.spec.whatwg.org/multipage/workers.html)
- [Using Web Workers](https://web.dev/workers-overview/)
