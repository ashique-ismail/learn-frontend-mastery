# WebAssembly Basics

## Introduction

WebAssembly (WASM) is a binary instruction format designed as a portable compilation target for programming languages, enabling deployment on the web for client and server applications. It provides a way to run code written in languages like C, C++, Rust, and others at near-native speed in web browsers.

WebAssembly is not meant to replace JavaScript but to complement it, handling computationally intensive tasks while JavaScript manages the DOM and user interface. Together, they form a powerful combination for building high-performance web applications.

## What is WebAssembly?

WebAssembly is:
- A **binary format** - Compact and fast to parse
- A **compilation target** - Not written by hand
- **Stack-based** - Virtual machine with its own instruction set
- **Safe** - Runs in a sandboxed environment
- **Fast** - Near-native execution speed
- **Portable** - Runs on all major browsers and platforms

### Key Benefits

1. **Performance** - 10-800x faster than JavaScript for compute-heavy tasks
2. **Language Flexibility** - Write in C, C++, Rust, Go, and more
3. **Code Reuse** - Port existing libraries to the web
4. **Security** - Memory-safe sandbox environment
5. **Compact** - Smaller file sizes than equivalent JavaScript

## WebAssembly Binary Format

### Text Format (WAT)

WebAssembly has a human-readable text format called WebAssembly Text (WAT):

```wat
;; simple.wat - Add two numbers
(module
  (func $add (param $a i32) (param $b i32) (result i32)
    local.get $a
    local.get $b
    i32.add
  )
  (export "add" (func $add))
)
```

### Compiling WAT to WASM

```bash
# Install wabt (WebAssembly Binary Toolkit)
# macOS
brew install wabt

# Linux
apt-get install wabt

# Compile WAT to WASM
wat2wasm simple.wat -o simple.wasm

# Decompile WASM to WAT
wasm2wat simple.wasm -o output.wat
```

### Loading WASM in JavaScript

```javascript
// Fetch and instantiate WebAssembly module
async function loadWasm() {
  // Fetch the .wasm file
  const response = await fetch('simple.wasm');
  const buffer = await response.arrayBuffer();
  
  // Instantiate the module
  const { instance } = await WebAssembly.instantiate(buffer);
  
  // Call exported function
  const result = instance.exports.add(5, 10);
  console.log('5 + 10 =', result); // 15
}

loadWasm();
```

## WebAssembly Fundamentals

### Data Types

WebAssembly supports four basic value types:

```wat
(module
  ;; i32 - 32-bit integer
  (func $int_demo (param $x i32) (result i32)
    local.get $x
    i32.const 10
    i32.add
  )
  
  ;; i64 - 64-bit integer
  (func $long_demo (param $x i64) (result i64)
    local.get $x
    i64.const 100
    i64.mul
  )
  
  ;; f32 - 32-bit float
  (func $float_demo (param $x f32) (result f32)
    local.get $x
    f32.const 2.5
    f32.mul
  )
  
  ;; f64 - 64-bit float
  (func $double_demo (param $x f64) (result f64)
    local.get $x
    f64.const 3.14159
    f64.div
  )
  
  (export "int_demo" (func $int_demo))
  (export "long_demo" (func $long_demo))
  (export "float_demo" (func $float_demo))
  (export "double_demo" (func $double_demo))
)
```

### Functions

```wat
;; functions.wat
(module
  ;; Function with parameters and return value
  (func $multiply (param $a i32) (param $b i32) (result i32)
    local.get $a
    local.get $b
    i32.mul
  )
  
  ;; Function with local variables
  (func $calculate (param $x i32) (result i32)
    (local $temp i32)
    
    ;; temp = x * 2
    local.get $x
    i32.const 2
    i32.mul
    local.set $temp
    
    ;; return temp + 10
    local.get $temp
    i32.const 10
    i32.add
  )
  
  ;; Function calling another function
  (func $square (param $n i32) (result i32)
    local.get $n
    local.get $n
    call $multiply
  )
  
  ;; Function with multiple return values (multi-value)
  (func $divmod (param $a i32) (param $b i32) (result i32 i32)
    local.get $a
    local.get $b
    i32.div_u    ;; quotient
    local.get $a
    local.get $b
    i32.rem_u    ;; remainder
  )
  
  (export "multiply" (func $multiply))
  (export "calculate" (func $calculate))
  (export "square" (func $square))
  (export "divmod" (func $divmod))
)
```

```javascript
// Using the functions
async function useFunctions() {
  const response = await fetch('functions.wasm');
  const buffer = await response.arrayBuffer();
  const { instance } = await WebAssembly.instantiate(buffer);
  
  console.log('multiply(5, 3):', instance.exports.multiply(5, 3)); // 15
  console.log('calculate(10):', instance.exports.calculate(10)); // 30
  console.log('square(7):', instance.exports.square(7)); // 49
  
  // Multi-value return (not widely supported yet)
  // const [quotient, remainder] = instance.exports.divmod(17, 5);
}
```

### Memory

WebAssembly has linear memory that can be shared between JavaScript and WASM:

```wat
;; memory.wat
(module
  ;; Declare memory (1 page = 64KB)
  (memory $mem 1)
  
  ;; Export memory so JavaScript can access it
  (export "memory" (memory $mem))
  
  ;; Store a value in memory
  (func $store (param $offset i32) (param $value i32)
    local.get $offset
    local.get $value
    i32.store
  )
  
  ;; Load a value from memory
  (func $load (param $offset i32) (result i32)
    local.get $offset
    i32.load
  )
  
  ;; Fill memory with a value
  (func $fill (param $offset i32) (param $value i32) (param $size i32)
    local.get $offset
    local.get $value
    local.get $size
    memory.fill
  )
  
  (export "store" (func $store))
  (export "load" (func $load))
  (export "fill" (func $fill))
)
```

```javascript
// Working with memory
async function useMemory() {
  const response = await fetch('memory.wasm');
  const buffer = await response.arrayBuffer();
  const { instance } = await WebAssembly.instantiate(buffer);
  
  // Store value at offset 0
  instance.exports.store(0, 42);
  
  // Load value from offset 0
  const value = instance.exports.load(0);
  console.log('Loaded value:', value); // 42
  
  // Access memory directly from JavaScript
  const memory = instance.exports.memory;
  const memoryArray = new Uint8Array(memory.buffer);
  
  // Write from JavaScript
  memoryArray[4] = 100;
  memoryArray[5] = 200;
  
  // Read from WebAssembly
  const jsValue = instance.exports.load(4);
  console.log('Value at offset 4:', jsValue);
  
  // Fill memory with zeros
  instance.exports.fill(0, 0, 100);
}
```

### Tables

Tables store references to functions:

```wat
;; table.wat
(module
  ;; Define functions
  (func $add (param i32 i32) (result i32)
    local.get 0
    local.get 1
    i32.add
  )
  
  (func $subtract (param i32 i32) (result i32)
    local.get 0
    local.get 1
    i32.sub
  )
  
  (func $multiply (param i32 i32) (result i32)
    local.get 0
    local.get 1
    i32.mul
  )
  
  ;; Create a table of function references
  (table $funcs 3 funcref)
  
  ;; Initialize table with function references
  (elem (i32.const 0) $add $subtract $multiply)
  
  ;; Call function from table
  (func $call_indirect (param $index i32) (param $a i32) (param $b i32) (result i32)
    local.get $a
    local.get $b
    local.get $index
    call_indirect (param i32 i32) (result i32)
  )
  
  (export "call_indirect" (func $call_indirect))
)
```

```javascript
// Using tables
async function useTables() {
  const response = await fetch('table.wasm');
  const buffer = await response.arrayBuffer();
  const { instance } = await WebAssembly.instantiate(buffer);
  
  // Call different functions by index
  console.log('Add (index 0):', instance.exports.call_indirect(0, 10, 5)); // 15
  console.log('Subtract (index 1):', instance.exports.call_indirect(1, 10, 5)); // 5
  console.log('Multiply (index 2):', instance.exports.call_indirect(2, 10, 5)); // 50
}
```

## Importing and Exporting

### Importing JavaScript Functions

```javascript
// JavaScript side
const importObject = {
  env: {
    log: (value) => console.log('WASM says:', value),
    getCurrentTime: () => Date.now(),
    randomNumber: () => Math.random()
  }
};

async function loadWithImports() {
  const response = await fetch('imports.wasm');
  const buffer = await response.arrayBuffer();
  const { instance } = await WebAssembly.instantiate(buffer, importObject);
  
  instance.exports.test();
}
```

```wat
;; imports.wat - Using imported functions
(module
  ;; Import JavaScript functions
  (import "env" "log" (func $log (param i32)))
  (import "env" "getCurrentTime" (func $getCurrentTime (result f64)))
  (import "env" "randomNumber" (func $randomNumber (result f64)))
  
  ;; Use imported functions
  (func $test
    ;; Log a number
    i32.const 42
    call $log
    
    ;; Get current time
    call $getCurrentTime
    drop
    
    ;; Get random number
    call $randomNumber
    drop
  )
  
  (export "test" (func $test))
)
```

### Importing Memory

```javascript
// Create memory in JavaScript
const memory = new WebAssembly.Memory({ initial: 1, maximum: 10 });

const importObject = {
  env: {
    memory: memory
  }
};

async function loadWithMemory() {
  const response = await fetch('shared-memory.wasm');
  const buffer = await response.arrayBuffer();
  const { instance } = await WebAssembly.instantiate(buffer, importObject);
  
  // Both JavaScript and WASM can access the same memory
  const memoryArray = new Uint8Array(memory.buffer);
  memoryArray[0] = 42;
  
  const value = instance.exports.load(0);
  console.log('Value from WASM:', value); // 42
}
```

## Complete Example: Fibonacci

```wat
;; fibonacci.wat
(module
  ;; Recursive fibonacci
  (func $fib_recursive (param $n i32) (result i32)
    (if (result i32) (i32.le_u (local.get $n) (i32.const 1))
      (then
        local.get $n
      )
      (else
        (i32.add
          (call $fib_recursive
            (i32.sub (local.get $n) (i32.const 1))
          )
          (call $fib_recursive
            (i32.sub (local.get $n) (i32.const 2))
          )
        )
      )
    )
  )
  
  ;; Iterative fibonacci (more efficient)
  (func $fib_iterative (param $n i32) (result i32)
    (local $i i32)
    (local $a i32)
    (local $b i32)
    (local $temp i32)
    
    ;; Initialize
    (local.set $a (i32.const 0))
    (local.set $b (i32.const 1))
    (local.set $i (i32.const 0))
    
    ;; Loop
    (block $break
      (loop $continue
        ;; if i >= n, break
        (br_if $break (i32.ge_u (local.get $i) (local.get $n)))
        
        ;; temp = a + b
        (local.set $temp (i32.add (local.get $a) (local.get $b)))
        
        ;; a = b
        (local.set $a (local.get $b))
        
        ;; b = temp
        (local.set $b (local.get $temp))
        
        ;; i++
        (local.set $i (i32.add (local.get $i) (i32.const 1)))
        
        ;; continue loop
        (br $continue)
      )
    )
    
    local.get $a
  )
  
  (export "fib_recursive" (func $fib_recursive))
  (export "fib_iterative" (func $fib_iterative))
)
```

```javascript
// Using fibonacci
async function benchmarkFibonacci() {
  const response = await fetch('fibonacci.wasm');
  const buffer = await response.arrayBuffer();
  const { instance } = await WebAssembly.instantiate(buffer);
  
  // JavaScript implementation for comparison
  function fibJS(n) {
    let a = 0, b = 1;
    for (let i = 0; i < n; i++) {
      [a, b] = [b, a + b];
    }
    return a;
  }
  
  const n = 40;
  
  // Benchmark WebAssembly iterative
  console.time('WASM iterative');
  const wasmResult = instance.exports.fib_iterative(n);
  console.timeEnd('WASM iterative');
  console.log('Result:', wasmResult);
  
  // Benchmark JavaScript
  console.time('JavaScript');
  const jsResult = fibJS(n);
  console.timeEnd('JavaScript');
  console.log('Result:', jsResult);
  
  // WASM is typically 2-3x faster for this operation
}
```

## WebAssembly JavaScript API

### WebAssembly.instantiate

```javascript
// Method 1: From buffer
async function instantiateFromBuffer() {
  const response = await fetch('module.wasm');
  const buffer = await response.arrayBuffer();
  const { instance, module } = await WebAssembly.instantiate(buffer);
  return instance;
}

// Method 2: From module
async function instantiateFromModule() {
  const response = await fetch('module.wasm');
  const buffer = await response.arrayBuffer();
  const module = await WebAssembly.compile(buffer);
  const instance = await WebAssembly.instantiate(module);
  return instance;
}

// Method 3: Streaming (most efficient)
async function instantiateStreaming() {
  const { instance } = await WebAssembly.instantiateStreaming(
    fetch('module.wasm')
  );
  return instance;
}
```

### WebAssembly.compile

```javascript
// Compile module separately
async function compileModule() {
  const response = await fetch('module.wasm');
  const buffer = await response.arrayBuffer();
  
  // Compile to module
  const module = await WebAssembly.compile(buffer);
  
  // Can instantiate multiple times
  const instance1 = await WebAssembly.instantiate(module);
  const instance2 = await WebAssembly.instantiate(module);
  
  return { module, instance1, instance2 };
}

// Streaming compile
async function compileStreaming() {
  const module = await WebAssembly.compileStreaming(fetch('module.wasm'));
  return module;
}
```

### WebAssembly.validate

```javascript
// Check if bytes are valid WebAssembly
async function validateWasm() {
  const response = await fetch('module.wasm');
  const buffer = await response.arrayBuffer();
  const bytes = new Uint8Array(buffer);
  
  const isValid = WebAssembly.validate(bytes);
  console.log('Valid WebAssembly:', isValid);
  
  return isValid;
}
```

## Performance Comparison

```javascript
// Performance test suite
class WasmBenchmark {
  constructor() {
    this.wasmInstance = null;
  }
  
  async init() {
    const { instance } = await WebAssembly.instantiateStreaming(
      fetch('benchmark.wasm')
    );
    this.wasmInstance = instance;
  }
  
  // Matrix multiplication benchmark
  benchmarkMatrixMultiply(size) {
    const iterations = 100;
    
    // JavaScript implementation
    console.time('JS Matrix Multiply');
    for (let i = 0; i < iterations; i++) {
      this.matrixMultiplyJS(size);
    }
    console.timeEnd('JS Matrix Multiply');
    
    // WebAssembly implementation
    console.time('WASM Matrix Multiply');
    for (let i = 0; i < iterations; i++) {
      this.wasmInstance.exports.matrixMultiply(size);
    }
    console.timeEnd('WASM Matrix Multiply');
  }
  
  matrixMultiplyJS(size) {
    const a = Array(size).fill(0).map(() => Array(size).fill(1));
    const b = Array(size).fill(0).map(() => Array(size).fill(1));
    const c = Array(size).fill(0).map(() => Array(size).fill(0));
    
    for (let i = 0; i < size; i++) {
      for (let j = 0; j < size; j++) {
        for (let k = 0; k < size; k++) {
          c[i][j] += a[i][k] * b[k][j];
        }
      }
    }
    
    return c;
  }
  
  // Image processing benchmark
  benchmarkImageProcessing(width, height) {
    const pixels = width * height * 4;
    const imageData = new Uint8Array(pixels);
    
    // Fill with random data
    for (let i = 0; i < pixels; i++) {
      imageData[i] = Math.random() * 255;
    }
    
    // JavaScript grayscale
    console.time('JS Grayscale');
    this.grayscaleJS(imageData, width, height);
    console.timeEnd('JS Grayscale');
    
    // WebAssembly grayscale
    console.time('WASM Grayscale');
    const memory = this.wasmInstance.exports.memory;
    const wasmMemory = new Uint8Array(memory.buffer);
    wasmMemory.set(imageData);
    this.wasmInstance.exports.grayscale(width, height);
    console.timeEnd('WASM Grayscale');
  }
  
  grayscaleJS(imageData, width, height) {
    for (let i = 0; i < imageData.length; i += 4) {
      const gray = imageData[i] * 0.299 + 
                   imageData[i + 1] * 0.587 + 
                   imageData[i + 2] * 0.114;
      imageData[i] = imageData[i + 1] = imageData[i + 2] = gray;
    }
  }
}

// Usage
async function runBenchmarks() {
  const benchmark = new WasmBenchmark();
  await benchmark.init();
  
  console.log('Matrix Multiplication (100x100):');
  benchmark.benchmarkMatrixMultiply(100);
  
  console.log('\nImage Processing (1920x1080):');
  benchmark.benchmarkImageProcessing(1920, 1080);
}
```

## Browser Support

| Browser | Version | WebAssembly | Streaming | Threads |
|---------|---------|-------------|-----------|---------|
| Chrome | 57+ | ✓ | 61+ | 74+ |
| Firefox | 52+ | ✓ | 58+ | 79+ |
| Safari | 11+ | ✓ | 15+ | 15.2+ |
| Edge | 16+ | ✓ | 16+ | 79+ |

## Common Mistakes

1. **Not Using Streaming APIs**
```javascript
// Slower
const response = await fetch('module.wasm');
const buffer = await response.arrayBuffer();
const { instance } = await WebAssembly.instantiate(buffer);

// Faster - compilation starts while downloading
const { instance } = await WebAssembly.instantiateStreaming(
  fetch('module.wasm')
);
```

2. **Excessive Memory Copying**
```javascript
// Inefficient - copying data back and forth
function processData(data) {
  const memory = instance.exports.memory;
  const buffer = new Uint8Array(memory.buffer);
  buffer.set(data);
  instance.exports.process();
  return buffer.slice(0, data.length);
}

// Better - work directly with shared memory
const memory = instance.exports.memory;
const buffer = new Uint8Array(memory.buffer);
// Work directly with buffer, avoid copying
```

3. **Not Handling Errors**
```javascript
// Wrong - no error handling
const { instance } = await WebAssembly.instantiateStreaming(
  fetch('module.wasm')
);

// Correct
try {
  const { instance } = await WebAssembly.instantiateStreaming(
    fetch('module.wasm')
  );
} catch (error) {
  console.error('Failed to load WASM:', error);
}
```

## Best Practices

1. **Use Streaming APIs** - Compile while downloading
2. **Share Memory** - Avoid unnecessary copies between JS and WASM
3. **Profile Performance** - Measure before optimizing
4. **Cache Compiled Modules** - Store in IndexedDB
5. **Handle Errors Gracefully** - Provide fallbacks
6. **Use SIMD When Available** - For parallel operations
7. **Minimize JS-WASM Boundary Crossings** - Batch operations
8. **Consider Bundle Size** - WASM can be smaller than JS

## When to Use WebAssembly

Use WebAssembly for:
- Computationally intensive tasks
- Image/video processing
- Game engines
- Cryptography
- Scientific simulations
- Porting existing C/C++/Rust code

Avoid WebAssembly for:
- DOM manipulation
- Simple CRUD operations
- String processing (usually)
- Small computations where JS is fast enough

## Interview Questions

1. What is WebAssembly and why was it created?
2. What are the four basic value types in WebAssembly?
3. How does WebAssembly memory work?
4. What's the difference between WAT and WASM?
5. How do you share data between JavaScript and WebAssembly?
6. What are the performance benefits of WebAssembly?
7. When should you use WebAssembly vs JavaScript?
8. What is the WebAssembly streaming API?

## Key Takeaways

- WebAssembly is a binary instruction format for the web
- It runs at near-native speed in all major browsers
- WebAssembly complements JavaScript, not replaces it
- Four basic types: i32, i64, f32, f64
- Linear memory enables efficient data sharing
- Streaming APIs enable fast compilation
- Best for compute-intensive tasks
- Growing ecosystem with multiple language support

## Resources

- [WebAssembly Official Site](https://webassembly.org/)
- [MDN WebAssembly](https://developer.mozilla.org/en-US/docs/WebAssembly)
- [WebAssembly Specification](https://webassembly.github.io/spec/)
- [WABT - WebAssembly Binary Toolkit](https://github.com/WebAssembly/wabt)
- [WebAssembly Studio](https://webassembly.studio/)
- [Understanding WebAssembly Text Format](https://developer.mozilla.org/en-US/docs/WebAssembly/Understanding_the_text_format)
