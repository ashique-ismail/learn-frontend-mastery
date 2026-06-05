# C/C++ to WebAssembly

## The Idea

**In plain English:** C and C++ are older programming languages that computers understand very well and can run extremely fast. WebAssembly (WASM) is a special format that web browsers can run at near-native speed. A tool called Emscripten acts as a translator, converting C/C++ code into WebAssembly so that those fast programs can run inside a browser.

**Real-world analogy:** Imagine a master chef (C/C++ code) who only knows how to cook using a professional kitchen in France. A translation service (Emscripten) takes all of the chef's recipes and rewrites them so they work perfectly in a tiny portable camping kitchen (the browser). The food (the result) tastes the same and is made just as skillfully, just in a different setting.

- The master chef's original recipes = the C/C++ source code
- The translation service rewriting the recipes = Emscripten compiling to WebAssembly
- The portable camping kitchen = the web browser running WebAssembly

---

## Introduction

C and C++ were among the first languages to target WebAssembly, thanks to Emscripten, a powerful toolchain that compiles C/C++ code to WebAssembly. This enables developers to port existing C/C++ libraries and applications to the web, bringing decades of optimized code and algorithms to web browsers.

Emscripten not only compiles to WebAssembly but also provides a comprehensive runtime that emulates POSIX APIs, OpenGL, SDL, and other common C/C++ libraries, making it possible to run complex applications like game engines, image processors, and scientific computing tools in the browser.

## Emscripten Setup

### Installation

```bash
# Clone the Emscripten SDK
git clone https://github.com/emscripten-core/emsdk.git
cd emsdk

# Install the latest SDK tools
./emsdk install latest

# Activate the latest SDK
./emsdk activate latest

# Add to PATH (run each time or add to shell config)
source ./emsdk_env.sh

# Verify installation
emcc --version
```

### Environment Setup

```bash
# Add to .bashrc or .zshrc for permanent setup
echo 'source "/path/to/emsdk/emsdk_env.sh"' >> ~/.bashrc

# Or create an alias
alias emscripten='source /path/to/emsdk/emsdk_env.sh'
```

## Basic C to WebAssembly

### Hello World

```c
// hello.c
#include <stdio.h>
#include <emscripten/emscripten.h>

// Export function to JavaScript
EMSCRIPTEN_KEEPALIVE
int add(int a, int b) {
    return a + b;
}

EMSCRIPTEN_KEEPALIVE
void greet(const char* name) {
    printf("Hello, %s!\n", name);
}

int main() {
    printf("WebAssembly module loaded\n");
    return 0;
}
```

### Compilation

```bash
# Basic compilation
emcc hello.c -o hello.js

# With optimizations
emcc hello.c -O3 -o hello.js

# Output only WASM (no JS glue code)
emcc hello.c -O3 -o hello.wasm

# Standalone WASM with minimal runtime
emcc hello.c -O3 -o hello.js \
  -s STANDALONE_WASM=1 \
  -s EXPORTED_FUNCTIONS='["_add","_greet"]' \
  -s EXPORTED_RUNTIME_METHODS='["cwrap","ccall"]'

# For web (generates .html, .js, and .wasm)
emcc hello.c -o hello.html
```

### Using in JavaScript

```javascript
// Using the generated JavaScript
const Module = require('./hello.js');

Module.onRuntimeInitialized = () => {
    // Call exported functions
    const add = Module.cwrap('add', 'number', ['number', 'number']);
    console.log('5 + 3 =', add(5, 3)); // 8
    
    // Call with ccall for one-time use
    Module.ccall('greet', null, ['string'], ['World']);
};
```

```html
<!DOCTYPE html>
<html>
<head>
    <title>C to WASM</title>
</head>
<body>
    <h1>C to WebAssembly Demo</h1>
    <div id="output"></div>
    
    <script src="hello.js"></script>
    <script>
        Module.onRuntimeInitialized = () => {
            const add = Module.cwrap('add', 'number', ['number', 'number']);
            const result = add(10, 20);
            document.getElementById('output').textContent = `Result: ${result}`;
        };
    </script>
</body>
</html>
```

## Working with Memory

### Memory Allocation and Sharing

```c
// memory.c
#include <stdlib.h>
#include <string.h>
#include <emscripten/emscripten.h>

// Allocate memory accessible from JavaScript
EMSCRIPTEN_KEEPALIVE
int* create_array(int size) {
    return (int*)malloc(size * sizeof(int));
}

// Fill array with values
EMSCRIPTEN_KEEPALIVE
void fill_array(int* arr, int size, int value) {
    for (int i = 0; i < size; i++) {
        arr[i] = value + i;
    }
}

// Sum array elements
EMSCRIPTEN_KEEPALIVE
int sum_array(int* arr, int size) {
    int sum = 0;
    for (int i = 0; i < size; i++) {
        sum += arr[i];
    }
    return sum;
}

// Free allocated memory
EMSCRIPTEN_KEEPALIVE
void free_array(int* arr) {
    free(arr);
}

// Process byte array
EMSCRIPTEN_KEEPALIVE
void process_bytes(unsigned char* data, int length) {
    for (int i = 0; i < length; i++) {
        data[i] = (data[i] + 1) % 256;
    }
}
```

```javascript
// Working with memory from JavaScript
const Module = require('./memory.js');

Module.onRuntimeInitialized = () => {
    // Create array
    const size = 10;
    const arrayPtr = Module._create_array(size);
    
    // Fill array
    Module._fill_array(arrayPtr, size, 100);
    
    // Access memory directly
    const heap = Module.HEAP32;
    const offset = arrayPtr / 4; // HEAP32 is 4 bytes per element
    const array = heap.subarray(offset, offset + size);
    console.log('Array:', Array.from(array));
    
    // Sum array
    const sum = Module._sum_array(arrayPtr, size);
    console.log('Sum:', sum);
    
    // Process bytes
    const bytes = new Uint8Array([1, 2, 3, 4, 5]);
    const bytePtr = Module._malloc(bytes.length);
    Module.HEAPU8.set(bytes, bytePtr);
    Module._process_bytes(bytePtr, bytes.length);
    const result = Module.HEAPU8.subarray(bytePtr, bytePtr + bytes.length);
    console.log('Processed:', Array.from(result));
    
    // Clean up
    Module._free(bytePtr);
    Module._free_array(arrayPtr);
};
```

### String Handling

```c
// strings.c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <emscripten/emscripten.h>

EMSCRIPTEN_KEEPALIVE
char* string_to_upper(const char* str) {
    int len = strlen(str);
    char* result = (char*)malloc(len + 1);
    
    for (int i = 0; i < len; i++) {
        if (str[i] >= 'a' && str[i] <= 'z') {
            result[i] = str[i] - 32;
        } else {
            result[i] = str[i];
        }
    }
    result[len] = '\0';
    
    return result;
}

EMSCRIPTEN_KEEPALIVE
void free_string(char* str) {
    free(str);
}

EMSCRIPTEN_KEEPALIVE
int count_chars(const char* str, char c) {
    int count = 0;
    for (int i = 0; str[i] != '\0'; i++) {
        if (str[i] == c) {
            count++;
        }
    }
    return count;
}

EMSCRIPTEN_KEEPALIVE
char* concat_strings(const char* str1, const char* str2) {
    int len1 = strlen(str1);
    int len2 = strlen(str2);
    char* result = (char*)malloc(len1 + len2 + 1);
    
    strcpy(result, str1);
    strcat(result, str2);
    
    return result;
}
```

```javascript
// Using strings from JavaScript
Module.onRuntimeInitialized = () => {
    // Helper to convert JS string to C string
    function allocateString(str) {
        const len = Module.lengthBytesUTF8(str) + 1;
        const ptr = Module._malloc(len);
        Module.stringToUTF8(str, ptr, len);
        return ptr;
    }
    
    // Helper to convert C string to JS string
    function readString(ptr) {
        return Module.UTF8ToString(ptr);
    }
    
    // String to upper
    const input = 'hello world';
    const inputPtr = allocateString(input);
    const resultPtr = Module._string_to_upper(inputPtr);
    const result = readString(resultPtr);
    console.log('Upper:', result); // "HELLO WORLD"
    
    // Count characters
    const count = Module._count_chars(inputPtr, 'l'.charCodeAt(0));
    console.log('Count of "l":', count); // 3
    
    // Concat strings
    const str2Ptr = allocateString(' from C!');
    const concatPtr = Module._concat_strings(inputPtr, str2Ptr);
    const concat = readString(concatPtr);
    console.log('Concat:', concat); // "hello world from C!"
    
    // Clean up
    Module._free_string(resultPtr);
    Module._free_string(concatPtr);
    Module._free(inputPtr);
    Module._free(str2Ptr);
};
```

## C++ to WebAssembly

### Classes and Objects

```cpp
// calculator.cpp
#include <emscripten/bind.h>
#include <string>
#include <vector>

using namespace emscripten;

class Calculator {
private:
    double value;
    
public:
    Calculator() : value(0) {}
    
    Calculator(double initial) : value(initial) {}
    
    double getValue() const {
        return value;
    }
    
    void setValue(double v) {
        value = v;
    }
    
    Calculator& add(double x) {
        value += x;
        return *this;
    }
    
    Calculator& subtract(double x) {
        value -= x;
        return *this;
    }
    
    Calculator& multiply(double x) {
        value *= x;
        return *this;
    }
    
    Calculator& divide(double x) {
        if (x != 0) {
            value /= x;
        }
        return *this;
    }
    
    Calculator& reset() {
        value = 0;
        return *this;
    }
};

// Bind C++ class to JavaScript
EMSCRIPTEN_BINDINGS(calculator_module) {
    class_<Calculator>("Calculator")
        .constructor<>()
        .constructor<double>()
        .property("value", &Calculator::getValue, &Calculator::setValue)
        .function("add", &Calculator::add)
        .function("subtract", &Calculator::subtract)
        .function("multiply", &Calculator::multiply)
        .function("divide", &Calculator::divide)
        .function("reset", &Calculator::reset);
}
```

```bash
# Compile with embind
em++ calculator.cpp -O3 -o calculator.js \
  -lembind \
  -s MODULARIZE=1 \
  -s EXPORT_ES6=1
```

```javascript
// Using C++ class from JavaScript
import createModule from './calculator.js';

async function run() {
    const Module = await createModule();
    
    // Create instance
    const calc = new Module.Calculator();
    
    // Chain operations
    calc.add(10)
        .multiply(2)
        .subtract(5);
    
    console.log('Result:', calc.value); // 15
    
    // Create with initial value
    const calc2 = new Module.Calculator(100);
    console.log('Initial:', calc2.value); // 100
    
    calc2.divide(4);
    console.log('After division:', calc2.value); // 25
    
    // Clean up
    calc.delete();
    calc2.delete();
}

run();
```

### STL Containers

```cpp
// containers.cpp
#include <emscripten/bind.h>
#include <vector>
#include <map>
#include <string>
#include <algorithm>

using namespace emscripten;

// Vector operations
std::vector<int> createRange(int start, int end) {
    std::vector<int> result;
    for (int i = start; i < end; i++) {
        result.push_back(i);
    }
    return result;
}

std::vector<int> filterPositive(const std::vector<int>& vec) {
    std::vector<int> result;
    std::copy_if(vec.begin(), vec.end(), std::back_inserter(result),
                 [](int x) { return x > 0; });
    return result;
}

int sumVector(const std::vector<int>& vec) {
    int sum = 0;
    for (int x : vec) {
        sum += x;
    }
    return sum;
}

// Map operations
std::map<std::string, int> createMap() {
    return {
        {"one", 1},
        {"two", 2},
        {"three", 3}
    };
}

// Bind to JavaScript
EMSCRIPTEN_BINDINGS(containers_module) {
    register_vector<int>("VectorInt");
    register_map<std::string, int>("MapStringInt");
    
    function("createRange", &createRange);
    function("filterPositive", &filterPositive);
    function("sumVector", &sumVector);
    function("createMap", &createMap);
}
```

```javascript
// Using STL containers
const Module = await createModule();

// Vector operations
const vec = Module.createRange(0, 10);
console.log('Size:', vec.size()); // 10
console.log('First:', vec.get(0)); // 0
console.log('Last:', vec.get(9));  // 9

// Filter
const filtered = Module.filterPositive([-5, 3, -2, 8, 0]);
console.log('Filtered size:', filtered.size()); // 2

// Sum
const sum = Module.sumVector(vec);
console.log('Sum:', sum); // 45

// Map operations
const map = Module.createMap();
console.log('Size:', map.size()); // 3
console.log('Value:', map.get('two')); // 2

// Clean up
vec.delete();
filtered.delete();
map.delete();
```

## Image Processing Example

```c
// image_processor.c
#include <stdlib.h>
#include <math.h>
#include <emscripten/emscripten.h>

typedef struct {
    unsigned char* data;
    int width;
    int height;
} Image;

EMSCRIPTEN_KEEPALIVE
Image* create_image(int width, int height) {
    Image* img = (Image*)malloc(sizeof(Image));
    img->width = width;
    img->height = height;
    img->data = (unsigned char*)malloc(width * height * 4);
    return img;
}

EMSCRIPTEN_KEEPALIVE
unsigned char* get_image_data(Image* img) {
    return img->data;
}

EMSCRIPTEN_KEEPALIVE
void grayscale(Image* img) {
    int size = img->width * img->height * 4;
    
    for (int i = 0; i < size; i += 4) {
        unsigned char r = img->data[i];
        unsigned char g = img->data[i + 1];
        unsigned char b = img->data[i + 2];
        
        unsigned char gray = (unsigned char)(0.299 * r + 0.587 * g + 0.114 * b);
        
        img->data[i] = gray;
        img->data[i + 1] = gray;
        img->data[i + 2] = gray;
    }
}

EMSCRIPTEN_KEEPALIVE
void brightness(Image* img, int amount) {
    int size = img->width * img->height * 4;
    
    for (int i = 0; i < size; i += 4) {
        int r = img->data[i] + amount;
        int g = img->data[i + 1] + amount;
        int b = img->data[i + 2] + amount;
        
        img->data[i] = r < 0 ? 0 : (r > 255 ? 255 : r);
        img->data[i + 1] = g < 0 ? 0 : (g > 255 ? 255 : g);
        img->data[i + 2] = b < 0 ? 0 : (b > 255 ? 255 : b);
    }
}

EMSCRIPTEN_KEEPALIVE
void blur(Image* img, int radius) {
    int width = img->width;
    int height = img->height;
    unsigned char* temp = (unsigned char*)malloc(width * height * 4);
    
    for (int y = 0; y < height; y++) {
        for (int x = 0; x < width; x++) {
            int r = 0, g = 0, b = 0, count = 0;
            
            for (int dy = -radius; dy <= radius; dy++) {
                for (int dx = -radius; dx <= radius; dx++) {
                    int nx = x + dx;
                    int ny = y + dy;
                    
                    if (nx >= 0 && nx < width && ny >= 0 && ny < height) {
                        int idx = (ny * width + nx) * 4;
                        r += img->data[idx];
                        g += img->data[idx + 1];
                        b += img->data[idx + 2];
                        count++;
                    }
                }
            }
            
            int idx = (y * width + x) * 4;
            temp[idx] = r / count;
            temp[idx + 1] = g / count;
            temp[idx + 2] = b / count;
            temp[idx + 3] = img->data[idx + 3];
        }
    }
    
    // Copy back
    for (int i = 0; i < width * height * 4; i++) {
        img->data[i] = temp[i];
    }
    
    free(temp);
}

EMSCRIPTEN_KEEPALIVE
void free_image(Image* img) {
    if (img) {
        if (img->data) {
            free(img->data);
        }
        free(img);
    }
}
```

```javascript
// Using image processor
class ImageProcessor {
    constructor(module) {
        this.module = module;
        this.imagePtr = null;
    }
    
    loadImage(imageData) {
        const { width, height } = imageData;
        
        // Create image in WASM
        this.imagePtr = this.module._create_image(width, height);
        
        // Get pointer to data
        const dataPtr = this.module._get_image_data(this.imagePtr);
        
        // Copy image data to WASM memory
        const wasmData = new Uint8ClampedArray(
            this.module.HEAPU8.buffer,
            dataPtr,
            width * height * 4
        );
        wasmData.set(imageData.data);
        
        return { width, height, dataPtr };
    }
    
    getImageData(width, height, dataPtr) {
        const wasmData = new Uint8ClampedArray(
            this.module.HEAPU8.buffer,
            dataPtr,
            width * height * 4
        );
        
        return new ImageData(
            new Uint8ClampedArray(wasmData),
            width,
            height
        );
    }
    
    grayscale(imageData) {
        const { width, height, dataPtr } = this.loadImage(imageData);
        this.module._grayscale(this.imagePtr);
        return this.getImageData(width, height, dataPtr);
    }
    
    brightness(imageData, amount) {
        const { width, height, dataPtr } = this.loadImage(imageData);
        this.module._brightness(this.imagePtr, amount);
        return this.getImageData(width, height, dataPtr);
    }
    
    blur(imageData, radius) {
        const { width, height, dataPtr } = this.loadImage(imageData);
        this.module._blur(this.imagePtr, radius);
        return this.getImageData(width, height, dataPtr);
    }
    
    cleanup() {
        if (this.imagePtr) {
            this.module._free_image(this.imagePtr);
            this.imagePtr = null;
        }
    }
}

// Usage with canvas
async function processImage() {
    const Module = await createModule();
    const processor = new ImageProcessor(Module);
    
    const canvas = document.getElementById('canvas');
    const ctx = canvas.getContext('2d');
    
    // Load image
    const img = new Image();
    img.onload = () => {
        canvas.width = img.width;
        canvas.height = img.height;
        ctx.drawImage(img, 0, 0);
        
        // Get image data
        const imageData = ctx.getImageData(0, 0, img.width, img.height);
        
        // Process
        console.time('grayscale');
        const processed = processor.grayscale(imageData);
        console.timeEnd('grayscale');
        
        // Display
        ctx.putImageData(processed, 0, 0);
        
        // Cleanup
        processor.cleanup();
    };
    img.src = 'image.jpg';
}
```

## Optimization Flags

```bash
# Development build (fast compilation, debug info)
emcc code.c -O0 -g -o output.js

# Production build (optimized for performance)
emcc code.c -O3 -o output.js

# Optimize for size
emcc code.c -Os -o output.js
emcc code.c -Oz -o output.js  # More aggressive size optimization

# Link-time optimization
emcc code.c -O3 --llvm-lto 3 -o output.js

# Enable SIMD
emcc code.c -O3 -msimd128 -o output.js

# Optimize for modern browsers only
emcc code.c -O3 -o output.js \
  -s ENVIRONMENT='web' \
  -s EXPORT_ES6=1 \
  -s MODULARIZE=1

# Minimize JavaScript glue code
emcc code.c -O3 -o output.js \
  -s MINIMAL_RUNTIME=1 \
  -s FILESYSTEM=0
```

## Debugging

```bash
# Build with source maps
emcc code.c -O0 -g -o output.js \
  -s ASSERTIONS=1 \
  -s SAFE_HEAP=1

# Enable runtime checks
emcc code.c -O0 -g -o output.js \
  -s ASSERTIONS=2 \
  -s STACK_OVERFLOW_CHECK=2

# Generate HTML with debugging UI
emcc code.c -O0 -g -o output.html
```

## Browser Support

Same as WebAssembly basics - all modern browsers support Emscripten-compiled code.

## Common Mistakes

1. **Not exporting functions**
```c
// Wrong - function not exported
int add(int a, int b) {
    return a + b;
}

// Correct - use EMSCRIPTEN_KEEPALIVE
EMSCRIPTEN_KEEPALIVE
int add(int a, int b) {
    return a + b;
}
```

2. **Memory leaks**
```c
// Wrong - malloc without free
char* create_string() {
    return (char*)malloc(100);
}

// Correct - provide cleanup function
EMSCRIPTEN_KEEPALIVE
void free_string(char* str) {
    free(str);
}
```

3. **Not handling async operations**
```javascript
// Wrong - Module not ready
const result = Module._add(5, 3);

// Correct - wait for module
Module.onRuntimeInitialized = () => {
    const result = Module._add(5, 3);
};
```

## Best Practices

1. **Use KEEPALIVE for exports** - Prevent function removal
2. **Manage memory carefully** - Provide free functions
3. **Use embind for C++** - Better type safety
4. **Optimize build flags** - Choose right optimization level
5. **Profile performance** - Use browser devtools
6. **Minimize JS glue code** - Use appropriate flags
7. **Handle strings properly** - Use UTF8ToString/stringToUTF8
8. **Clean up resources** - Call delete() on objects

## When to Use C/C++ for WASM

Use C/C++ when:
- Porting existing C/C++ libraries
- Need access to specific C/C++ libraries
- Working with legacy codebases
- Team has C/C++ expertise
- Need maximum performance

Consider Rust when:
- Starting new project
- Want memory safety
- Prefer modern tooling
- Need better error messages

## Interview Questions

1. What is Emscripten and what does it do?
2. How do you export C functions to JavaScript?
3. What is EMSCRIPTEN_KEEPALIVE?
4. How do you pass strings between C and JavaScript?
5. What is embind and when should you use it?
6. How do you handle memory in Emscripten?
7. What are the common optimization flags?
8. How do you debug Emscripten code?

## Key Takeaways

- Emscripten compiles C/C++ to WebAssembly
- EMSCRIPTEN_KEEPALIVE exports functions to JavaScript
- Memory must be managed carefully between C and JS
- embind provides better C++ integration
- Optimization flags significantly impact performance and size
- Existing C/C++ libraries can be ported to the web
- Comprehensive runtime emulates POSIX APIs
- Excellent for porting legacy applications

## Resources

- [Emscripten Documentation](https://emscripten.org/docs/)
- [Emscripten Tutorial](https://emscripten.org/docs/getting_started/Tutorial.html)
- [Embind Documentation](https://emscripten.org/docs/porting/connecting_cpp_and_javascript/embind.html)
- [Emscripten Examples](https://github.com/emscripten-core/emscripten/tree/main/tests)
- [WebAssembly Studio](https://webassembly.studio/)
