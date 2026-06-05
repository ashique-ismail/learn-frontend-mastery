# WebAssembly JavaScript Interop

## The Idea

**In plain English:** WebAssembly (WASM) and JavaScript are two different languages running inside your browser, and "interop" means making them talk to each other — passing data back and forth and calling each other's functions so you get the best of both worlds: JavaScript's ease and WebAssembly's raw speed.

**Real-world analogy:** Imagine a restaurant kitchen where the head chef (JavaScript) manages the whole operation, but for one specialized dish — a super-complex sauce — they hand off work to a master sous-chef (WebAssembly) who works in a separate prep station. The head chef writes the order on a slip and passes it through a small window, the sous-chef prepares the sauce using their own tools and workspace, then slides the finished dish back through the same window.

- The head chef (JavaScript) = the main programming language coordinating the app
- The sous-chef (WebAssembly) = the high-performance module doing heavy computation
- The small window between stations = the JS-WASM boundary where data is passed
- The shared prep counter = the shared memory both sides can read and write
- The order slip = function arguments passed from JavaScript to WebAssembly

---

## Introduction

WebAssembly's power comes not just from its performance but from its ability to seamlessly work with JavaScript. Understanding how data flows between JavaScript and WebAssembly, how to call functions across the boundary, and how to optimize these interactions is crucial for building high-performance web applications.

This guide covers the complete spectrum of JavaScript-WebAssembly interoperability, from basic function calls to advanced memory management, performance optimization, and real-world integration patterns.

## Function Calls

### Calling WASM from JavaScript

```javascript
// Basic function call
async function callWasmFunction() {
    const response = await fetch('module.wasm');
    const buffer = await response.arrayBuffer();
    const { instance } = await WebAssembly.instantiate(buffer);
    
    // Call exported function
    const result = instance.exports.add(5, 10);
    console.log('Result:', result); // 15
    
    // Call multiple functions
    const multiply = instance.exports.multiply(3, 4);
    const divide = instance.exports.divide(20, 5);
    console.log('Multiply:', multiply, 'Divide:', divide);
}
```

### Calling JavaScript from WASM

```wat
;; import.wat
(module
  ;; Import JavaScript functions
  (import "env" "log" (func $log (param i32)))
  (import "env" "random" (func $random (result f64)))
  (import "env" "now" (func $now (result f64)))
  
  (func $test
    ;; Call JavaScript log function
    i32.const 42
    call $log
    
    ;; Get random number
    call $random
    drop
    
    ;; Get current time
    call $now
    drop
  )
  
  (export "test" (func $test))
)
```

```javascript
// Provide JavaScript functions to WASM
const importObject = {
    env: {
        log: (value) => console.log('WASM says:', value),
        random: () => Math.random(),
        now: () => Date.now()
    }
};

async function loadWithImports() {
    const response = await fetch('import.wasm');
    const buffer = await response.arrayBuffer();
    const { instance } = await WebAssembly.instantiate(buffer, importObject);
    
    instance.exports.test();
}
```

## Memory Management

### Shared Memory Access

```javascript
// Creating shared memory
const memory = new WebAssembly.Memory({
    initial: 1,  // 64KB (1 page)
    maximum: 10, // 640KB max
    shared: false
});

const importObject = {
    env: {
        memory: memory
    }
};

// Load module with shared memory
const { instance } = await WebAssembly.instantiate(wasmBuffer, importObject);

// Access memory from JavaScript
const uint8View = new Uint8Array(memory.buffer);
const uint32View = new Uint32Array(memory.buffer);
const float32View = new Float32Array(memory.buffer);

// Write to memory
uint8View[0] = 255;
uint32View[0] = 0x12345678;
float32View[0] = 3.14159;

// Read from memory
console.log('Byte:', uint8View[0]);
console.log('Int:', uint32View[0]);
console.log('Float:', float32View[0]);
```

### Memory Helper Class

```javascript
class WasmMemory {
    constructor(instance) {
        this.instance = instance;
        this.memory = instance.exports.memory;
        this.malloc = instance.exports.malloc;
        this.free = instance.exports.free;
    }
    
    // Get view of specific type
    getView(type) {
        const views = {
            'u8': Uint8Array,
            'u16': Uint16Array,
            'u32': Uint32Array,
            'i8': Int8Array,
            'i16': Int16Array,
            'i32': Int32Array,
            'f32': Float32Array,
            'f64': Float64Array
        };
        
        const ViewType = views[type];
        if (!ViewType) {
            throw new Error(`Unknown type: ${type}`);
        }
        
        return new ViewType(this.memory.buffer);
    }
    
    // Write array to memory
    writeArray(array, type = 'u8') {
        const view = this.getView(type);
        const bytesPerElement = view.BYTES_PER_ELEMENT;
        
        // Allocate memory
        const ptr = this.malloc(array.length * bytesPerElement);
        
        // Write data
        const offset = ptr / bytesPerElement;
        view.set(array, offset);
        
        return ptr;
    }
    
    // Read array from memory
    readArray(ptr, length, type = 'u8') {
        const view = this.getView(type);
        const bytesPerElement = view.BYTES_PER_ELEMENT;
        const offset = ptr / bytesPerElement;
        
        return Array.from(view.subarray(offset, offset + length));
    }
    
    // Write string to memory
    writeString(str) {
        const encoder = new TextEncoder();
        const bytes = encoder.encode(str + '\0'); // Null-terminated
        return this.writeArray(bytes, 'u8');
    }
    
    // Read string from memory
    readString(ptr) {
        const view = this.getView('u8');
        const bytes = [];
        
        for (let i = ptr; view[i] !== 0; i++) {
            bytes.push(view[i]);
        }
        
        const decoder = new TextDecoder();
        return decoder.decode(new Uint8Array(bytes));
    }
    
    // Free allocated memory
    freePtr(ptr) {
        if (this.free) {
            this.free(ptr);
        }
    }
}

// Usage
const wasmMem = new WasmMemory(instance);

// Write array
const numbers = [1, 2, 3, 4, 5];
const ptr = wasmMem.writeArray(numbers, 'i32');

// Process in WASM
instance.exports.processArray(ptr, numbers.length);

// Read back
const result = wasmMem.readArray(ptr, numbers.length, 'i32');
console.log('Result:', result);

// Clean up
wasmMem.freePtr(ptr);
```

### Memory Growth

```javascript
// Monitor and handle memory growth
class MemoryManager {
    constructor(memory) {
        this.memory = memory;
        this.views = new Map();
    }
    
    // Get current memory size
    getSize() {
        return this.memory.buffer.byteLength;
    }
    
    // Get number of pages
    getPages() {
        return this.getSize() / 65536; // 64KB per page
    }
    
    // Grow memory
    grow(pages) {
        try {
            const previousPages = this.memory.grow(pages);
            this.invalidateViews();
            return previousPages;
        } catch (error) {
            console.error('Memory growth failed:', error);
            return -1;
        }
    }
    
    // Invalidate cached views after growth
    invalidateViews() {
        this.views.clear();
    }
    
    // Get or create view
    getView(type) {
        if (!this.views.has(type)) {
            const view = this.createView(type);
            this.views.set(type, view);
        }
        return this.views.get(type);
    }
    
    createView(type) {
        const types = {
            'u8': Uint8Array,
            'u32': Uint32Array,
            'f32': Float32Array,
            'f64': Float64Array
        };
        
        const ViewType = types[type];
        return new ViewType(this.memory.buffer);
    }
}

// Usage
const memManager = new MemoryManager(memory);

console.log('Initial size:', memManager.getSize());
console.log('Initial pages:', memManager.getPages());

// Grow by 2 pages (128KB)
const previousPages = memManager.grow(2);
console.log('Previous pages:', previousPages);
console.log('New size:', memManager.getSize());
```

## Type Conversions

### Number Types

```javascript
class TypeConverter {
    // Integer conversions
    static toU8(value) {
        return (value >>> 0) & 0xFF;
    }
    
    static toU16(value) {
        return (value >>> 0) & 0xFFFF;
    }
    
    static toU32(value) {
        return value >>> 0;
    }
    
    static toI8(value) {
        return (value << 24) >> 24;
    }
    
    static toI16(value) {
        return (value << 16) >> 16;
    }
    
    static toI32(value) {
        return value | 0;
    }
    
    // Float conversions
    static toF32(value) {
        return Math.fround(value);
    }
    
    static toF64(value) {
        return +value;
    }
    
    // Boolean to i32
    static boolToI32(value) {
        return value ? 1 : 0;
    }
    
    // i32 to Boolean
    static i32ToBool(value) {
        return value !== 0;
    }
}

// Usage
const u8 = TypeConverter.toU8(-1);     // 255
const i8 = TypeConverter.toI8(255);    // -1
const f32 = TypeConverter.toF32(3.14); // Rounded to f32 precision
```

### Array Conversions

```javascript
class ArrayConverter {
    // JavaScript array to typed array
    static toTypedArray(array, type) {
        const types = {
            'u8': Uint8Array,
            'u16': Uint16Array,
            'u32': Uint32Array,
            'i8': Int8Array,
            'i16': Int16Array,
            'i32': Int32Array,
            'f32': Float32Array,
            'f64': Float64Array
        };
        
        const TypedArray = types[type];
        return new TypedArray(array);
    }
    
    // Typed array to JavaScript array
    static fromTypedArray(typedArray) {
        return Array.from(typedArray);
    }
    
    // Convert between typed arrays
    static convert(source, targetType) {
        const types = {
            'u8': Uint8Array,
            'u16': Uint16Array,
            'u32': Uint32Array,
            'i8': Int8Array,
            'i16': Int16Array,
            'i32': Int32Array,
            'f32': Float32Array,
            'f64': Float64Array
        };
        
        const TargetArray = types[targetType];
        return new TargetArray(source);
    }
}

// Usage
const jsArray = [1, 2, 3, 4, 5];
const u32Array = ArrayConverter.toTypedArray(jsArray, 'u32');
const f32Array = ArrayConverter.convert(u32Array, 'f32');
const backToJS = ArrayConverter.fromTypedArray(f32Array);
```

## Performance Optimization

### Minimizing Boundary Crossings

```javascript
// Bad: Multiple boundary crossings
function inefficientProcessing(instance, data) {
    let sum = 0;
    for (let i = 0; i < data.length; i++) {
        // Each iteration crosses the JS-WASM boundary
        sum += instance.exports.square(data[i]);
    }
    return sum;
}

// Good: Single boundary crossing
function efficientProcessing(instance, wasmMem, data) {
    // Write entire array to WASM memory once
    const ptr = wasmMem.writeArray(data, 'f32');
    
    // Process in WASM (single call)
    const result = instance.exports.processArray(ptr, data.length);
    
    // Clean up
    wasmMem.freePtr(ptr);
    
    return result;
}

// Even better: Work with shared memory
function optimalProcessing(instance, memory, data) {
    // Use pre-allocated shared memory
    const view = new Float32Array(memory.buffer);
    const offset = 0;
    
    // Copy to shared memory
    view.set(data, offset);
    
    // Process in place
    instance.exports.processInPlace(offset, data.length);
    
    // Read result
    return view[offset];
}
```

### Batch Operations

```javascript
class BatchProcessor {
    constructor(instance, memory) {
        this.instance = instance;
        this.memory = memory;
        this.batchSize = 1024;
    }
    
    // Process large dataset in batches
    processBatches(data, processFn) {
        const results = [];
        const view = new Float32Array(this.memory.buffer);
        const batches = Math.ceil(data.length / this.batchSize);
        
        for (let i = 0; i < batches; i++) {
            const start = i * this.batchSize;
            const end = Math.min(start + this.batchSize, data.length);
            const batch = data.slice(start, end);
            
            // Write batch
            view.set(batch, 0);
            
            // Process batch
            const result = processFn(0, batch.length);
            results.push(result);
        }
        
        return results;
    }
}

// Usage
const processor = new BatchProcessor(instance, memory);
const data = new Float32Array(10000);

console.time('batch processing');
const results = processor.processBatches(
    data,
    (offset, length) => instance.exports.sum(offset, length)
);
console.timeEnd('batch processing');
```

### Memory Pooling

```javascript
class MemoryPool {
    constructor(instance, blockSize, maxBlocks) {
        this.instance = instance;
        this.blockSize = blockSize;
        this.maxBlocks = maxBlocks;
        this.freeBlocks = [];
        this.malloc = instance.exports.malloc;
        this.free = instance.exports.free;
        
        // Pre-allocate blocks
        for (let i = 0; i < maxBlocks; i++) {
            this.freeBlocks.push(this.malloc(blockSize));
        }
    }
    
    allocate() {
        if (this.freeBlocks.length === 0) {
            console.warn('Pool exhausted, allocating new block');
            return this.malloc(this.blockSize);
        }
        
        return this.freeBlocks.pop();
    }
    
    deallocate(ptr) {
        if (this.freeBlocks.length < this.maxBlocks) {
            this.freeBlocks.push(ptr);
        } else {
            this.free(ptr);
        }
    }
    
    cleanup() {
        // Free all pooled blocks
        this.freeBlocks.forEach(ptr => this.free(ptr));
        this.freeBlocks = [];
    }
}

// Usage
const pool = new MemoryPool(instance, 1024, 10);

// Allocate from pool
const ptr1 = pool.allocate();
const ptr2 = pool.allocate();

// Use memory
instance.exports.processData(ptr1, 1024);

// Return to pool
pool.deallocate(ptr1);
pool.deallocate(ptr2);

// Cleanup on exit
pool.cleanup();
```

## Complete Integration Example

```javascript
// Comprehensive WASM integration wrapper
class WasmModule {
    constructor() {
        this.instance = null;
        this.memory = null;
        this.memHelper = null;
        this.initialized = false;
    }
    
    async init(wasmPath, importObject = {}) {
        try {
            // Add memory to imports if not provided
            if (!importObject.env) {
                importObject.env = {};
            }
            
            if (!importObject.env.memory) {
                this.memory = new WebAssembly.Memory({
                    initial: 256,  // 16MB
                    maximum: 512   // 32MB
                });
                importObject.env.memory = this.memory;
            }
            
            // Add console logging
            importObject.env.log = this.log.bind(this);
            importObject.env.error = this.error.bind(this);
            
            // Load and instantiate
            const { instance } = await WebAssembly.instantiateStreaming(
                fetch(wasmPath),
                importObject
            );
            
            this.instance = instance;
            this.memory = instance.exports.memory || this.memory;
            this.memHelper = new WasmMemory(instance);
            this.initialized = true;
            
            console.log('WASM module initialized');
            return this;
        } catch (error) {
            console.error('Failed to initialize WASM:', error);
            throw error;
        }
    }
    
    // Logging functions
    log(ptr) {
        const message = this.memHelper.readString(ptr);
        console.log('[WASM]', message);
    }
    
    error(ptr) {
        const message = this.memHelper.readString(ptr);
        console.error('[WASM]', message);
    }
    
    // Type-safe function wrapper
    wrap(funcName, returnType, paramTypes) {
        const func = this.instance.exports[funcName];
        
        if (!func) {
            throw new Error(`Function ${funcName} not found`);
        }
        
        return (...args) => {
            if (args.length !== paramTypes.length) {
                throw new Error(`Expected ${paramTypes.length} arguments, got ${args.length}`);
            }
            
            // Convert arguments
            const convertedArgs = args.map((arg, i) => {
                return this.convertArg(arg, paramTypes[i]);
            });
            
            // Call function
            const result = func(...convertedArgs);
            
            // Convert result
            return this.convertResult(result, returnType);
        };
    }
    
    convertArg(value, type) {
        switch (type) {
            case 'i32':
            case 'u32':
                return value | 0;
            case 'i64':
            case 'u64':
                return BigInt(value);
            case 'f32':
                return Math.fround(value);
            case 'f64':
                return +value;
            case 'string':
                return this.memHelper.writeString(value);
            case 'array':
                return this.memHelper.writeArray(value);
            default:
                return value;
        }
    }
    
    convertResult(value, type) {
        switch (type) {
            case 'string':
                return this.memHelper.readString(value);
            case 'array':
                return this.memHelper.readArray(value);
            case 'bool':
                return value !== 0;
            default:
                return value;
        }
    }
    
    // Benchmarking utility
    benchmark(name, fn, iterations = 1000) {
        console.log(`\nBenchmarking: ${name}`);
        
        const start = performance.now();
        
        for (let i = 0; i < iterations; i++) {
            fn();
        }
        
        const end = performance.now();
        const total = end - start;
        const average = total / iterations;
        
        console.log(`Total: ${total.toFixed(2)}ms`);
        console.log(`Average: ${average.toFixed(4)}ms`);
        console.log(`Ops/sec: ${(1000 / average).toFixed(0)}`);
        
        return { total, average };
    }
    
    // Memory stats
    getMemoryStats() {
        const buffer = this.memory.buffer;
        const pages = buffer.byteLength / 65536;
        
        return {
            bufferSize: buffer.byteLength,
            pages: pages,
            sizeInMB: (buffer.byteLength / (1024 * 1024)).toFixed(2)
        };
    }
    
    // Cleanup
    destroy() {
        if (this.memHelper) {
            // Cleanup any allocated memory
        }
        
        this.instance = null;
        this.memory = null;
        this.memHelper = null;
        this.initialized = false;
    }
}

// Usage example
async function runExample() {
    const wasm = new WasmModule();
    
    await wasm.init('module.wasm', {
        env: {
            random: () => Math.random(),
            now: () => Date.now()
        }
    });
    
    // Wrap functions for type safety
    const add = wasm.wrap('add', 'i32', ['i32', 'i32']);
    const processString = wasm.wrap('processString', 'string', ['string']);
    const sumArray = wasm.wrap('sumArray', 'f32', ['array']);
    
    // Use wrapped functions
    console.log('Add:', add(5, 10));
    console.log('String:', processString('hello'));
    console.log('Sum:', sumArray([1, 2, 3, 4, 5]));
    
    // Benchmark
    wasm.benchmark('Addition', () => add(5, 10), 10000);
    
    // Memory stats
    console.log('Memory:', wasm.getMemoryStats());
    
    // Cleanup
    wasm.destroy();
}
```

## Error Handling

```javascript
class WasmError extends Error {
    constructor(message, code, details) {
        super(message);
        this.name = 'WasmError';
        this.code = code;
        this.details = details;
    }
}

class SafeWasmWrapper {
    constructor(instance) {
        this.instance = instance;
    }
    
    // Safe function call with error handling
    safeCall(funcName, ...args) {
        try {
            const func = this.instance.exports[funcName];
            
            if (!func) {
                throw new WasmError(
                    `Function ${funcName} not found`,
                    'FUNCTION_NOT_FOUND',
                    { funcName, available: Object.keys(this.instance.exports) }
                );
            }
            
            if (typeof func !== 'function') {
                throw new WasmError(
                    `${funcName} is not a function`,
                    'NOT_A_FUNCTION',
                    { funcName, type: typeof func }
                );
            }
            
            const result = func(...args);
            
            // Check for error codes (if WASM returns error codes)
            if (typeof result === 'number' && result < 0) {
                throw new WasmError(
                    'WASM function returned error',
                    'WASM_ERROR',
                    { funcName, errorCode: result, args }
                );
            }
            
            return { success: true, data: result };
        } catch (error) {
            console.error('Error calling WASM function:', error);
            
            return {
                success: false,
                error: error instanceof WasmError ? error : new WasmError(
                    error.message,
                    'UNKNOWN_ERROR',
                    { funcName, args, originalError: error }
                )
            };
        }
    }
    
    // Retry logic
    async retryCall(funcName, args, maxRetries = 3, delay = 100) {
        for (let i = 0; i < maxRetries; i++) {
            const result = this.safeCall(funcName, ...args);
            
            if (result.success) {
                return result;
            }
            
            if (i < maxRetries - 1) {
                console.log(`Retry ${i + 1}/${maxRetries}...`);
                await new Promise(resolve => setTimeout(resolve, delay));
            }
        }
        
        throw new WasmError(
            `Failed after ${maxRetries} retries`,
            'MAX_RETRIES_EXCEEDED',
            { funcName, args, maxRetries }
        );
    }
}
```

## Browser Support

All modern browsers support WebAssembly JavaScript interop as specified in the WebAssembly specification.

## Common Mistakes

1. **Not handling memory growth**
```javascript
// Wrong - views become invalid after memory growth
const view = new Uint8Array(memory.buffer);
memory.grow(1);
view[0] = 42; // Operating on detached buffer!

// Correct - recreate view after growth
let view = new Uint8Array(memory.buffer);
memory.grow(1);
view = new Uint8Array(memory.buffer);
view[0] = 42;
```

2. **Memory leaks**
```javascript
// Wrong - allocated but never freed
const ptr = instance.exports.malloc(1024);
instance.exports.process(ptr);
// Memory leaked!

// Correct - always free
const ptr = instance.exports.malloc(1024);
try {
    instance.exports.process(ptr);
} finally {
    instance.exports.free(ptr);
}
```

3. **Excessive boundary crossings**
```javascript
// Wrong - crossing boundary in loop
for (let i = 0; i < 1000000; i++) {
    result += instance.exports.calculate(i);
}

// Correct - batch processing in WASM
const ptr = writeArray(data);
const result = instance.exports.calculateBatch(ptr, data.length);
free(ptr);
```

## Best Practices

1. **Minimize Boundary Crossings** - Batch operations in WASM
2. **Use Shared Memory** - Avoid copying large data structures
3. **Handle Memory Carefully** - Track allocations and free memory
4. **Type Safety** - Create wrapper functions with type checking
5. **Error Handling** - Wrap calls in try-catch
6. **Profile Performance** - Measure and optimize hotpaths
7. **Memory Pooling** - Reuse allocations when possible
8. **Documentation** - Document memory ownership

## When to Optimize Interop

Optimize interop when:
- Profiling shows boundary crossing overhead
- Processing large datasets
- High-frequency function calls
- Memory allocation is a bottleneck
- Converting between types is expensive

## Interview Questions

1. How do you share memory between JavaScript and WebAssembly?
2. What happens to typed array views when memory grows?
3. How do you minimize performance overhead at the JS-WASM boundary?
4. What are the trade-offs of different memory management strategies?
5. How do you handle errors across the JS-WASM boundary?
6. What is memory pooling and when should you use it?
7. How do you debug memory leaks in WASM applications?
8. What are the best practices for type conversions?

## Key Takeaways

- JavaScript and WebAssembly share linear memory
- Minimize boundary crossings for better performance
- Memory views become invalid when memory grows
- Always free allocated memory to prevent leaks
- Batch operations reduce interop overhead
- Memory pooling improves allocation performance
- Type-safe wrappers prevent runtime errors
- Proper error handling is crucial for reliability

## Resources

- [MDN: WebAssembly JavaScript Interface](https://developer.mozilla.org/en-US/docs/WebAssembly/Using_the_JavaScript_API)
- [WebAssembly Memory](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/Memory)
- [WebAssembly Performance](https://web.dev/webassembly-performance/)
- [Emscripten Interacting with Code](https://emscripten.org/docs/porting/connecting_cpp_and_javascript/Interacting-with-code.html)
- [wasm-bindgen Guide](https://rustwasm.github.io/docs/wasm-bindgen/)
