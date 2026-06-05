# Rust to WebAssembly

## The Idea

**In plain English:** Rust to WebAssembly means writing code in the Rust programming language (a language known for speed and safety) and then converting it into WebAssembly (a compact, fast format that browsers can run directly) so your web page can do heavy tasks at near full computer speed.

**Real-world analogy:** Imagine a chef (Rust) who writes recipes in a very precise culinary language. To serve customers in a cafe (the browser), the recipes are printed onto a universal menu card (WebAssembly) that any waiter in any cafe around the world can read and execute without needing to know the original culinary language.

- The chef (Rust) = the programmer writing fast, safe code in the Rust language
- The universal menu card (WebAssembly) = the compiled .wasm file that the browser understands
- The waiter serving customers (the browser running the menu) = the browser executing the WebAssembly at near-native speed

---

## Introduction

Rust has emerged as the premier language for WebAssembly development, offering memory safety without garbage collection, zero-cost abstractions, and excellent tooling. The combination of Rust's performance characteristics and WebAssembly's portability makes it ideal for building high-performance web applications.

The Rust ecosystem provides powerful tools like `wasm-bindgen` for seamless JavaScript interop, `wasm-pack` for building and publishing packages, and `web-sys` for accessing Web APIs. This guide covers everything you need to know to build WebAssembly modules with Rust.

## Setting Up Rust for WebAssembly

### Installation

```bash
# Install Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Add WebAssembly target
rustup target add wasm32-unknown-unknown

# Install wasm-bindgen-cli
cargo install wasm-bindgen-cli

# Install wasm-pack (recommended)
curl https://rustwasm.github.io/wasm-pack/installer/init.sh -sSf | sh

# Install cargo-generate for templates
cargo install cargo-generate
```

### Project Setup

```bash
# Create new project with wasm-pack template
cargo generate --git https://github.com/rustwasm/wasm-pack-template

# Or create manually
cargo new --lib my-wasm-project
cd my-wasm-project
```

### Cargo.toml Configuration

```toml
[package]
name = "my-wasm-project"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib"]

[dependencies]
wasm-bindgen = "0.2"

[profile.release]
# Optimize for size
opt-level = "z"
lto = true
```

## Basic Rust to WebAssembly

### Hello World

```rust
// src/lib.rs
use wasm_bindgen::prelude::*;

// Export a function to JavaScript
#[wasm_bindgen]
pub fn greet(name: &str) -> String {
    format!("Hello, {}!", name)
}

// Export multiple functions
#[wasm_bindgen]
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}

#[wasm_bindgen]
pub fn fibonacci(n: u32) -> u32 {
    match n {
        0 => 0,
        1 => 1,
        _ => fibonacci(n - 1) + fibonacci(n - 2),
    }
}
```

### Building and Using

```bash
# Build with wasm-pack
wasm-pack build --target web

# Or build manually
cargo build --target wasm32-unknown-unknown --release
wasm-bindgen target/wasm32-unknown-unknown/release/my_wasm_project.wasm \
  --out-dir pkg \
  --target web
```

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <title>Rust WASM</title>
</head>
<body>
    <script type="module">
        import init, { greet, add, fibonacci } from './pkg/my_wasm_project.js';
        
        async function run() {
            await init();
            
            console.log(greet('World'));  // "Hello, World!"
            console.log(add(5, 10));      // 15
            console.log(fibonacci(10));   // 55
        }
        
        run();
    </script>
</body>
</html>
```

## Working with Complex Types

### Structs and Implementations

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub struct User {
    name: String,
    age: u32,
    email: String,
}

#[wasm_bindgen]
impl User {
    // Constructor
    #[wasm_bindgen(constructor)]
    pub fn new(name: String, age: u32, email: String) -> User {
        User { name, age, email }
    }
    
    // Getter methods
    #[wasm_bindgen(getter)]
    pub fn name(&self) -> String {
        self.name.clone()
    }
    
    #[wasm_bindgen(getter)]
    pub fn age(&self) -> u32 {
        self.age
    }
    
    #[wasm_bindgen(getter)]
    pub fn email(&self) -> String {
        self.email.clone()
    }
    
    // Setter methods
    #[wasm_bindgen(setter)]
    pub fn set_age(&mut self, age: u32) {
        self.age = age;
    }
    
    // Regular methods
    #[wasm_bindgen]
    pub fn greet(&self) -> String {
        format!("Hello, I'm {} and I'm {} years old", self.name, self.age)
    }
    
    #[wasm_bindgen]
    pub fn is_adult(&self) -> bool {
        self.age >= 18
    }
}
```

```javascript
// Using the struct from JavaScript
import init, { User } from './pkg/my_wasm_project.js';

async function run() {
    await init();
    
    // Create instance
    const user = new User('John Doe', 25, 'john@example.com');
    
    // Use getters
    console.log(user.name);    // "John Doe"
    console.log(user.age);     // 25
    
    // Use setter
    user.age = 26;
    
    // Call methods
    console.log(user.greet()); // "Hello, I'm John Doe and I'm 26 years old"
    console.log(user.is_adult()); // true
}
```

### Working with Arrays and Vectors

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn sum_array(arr: &[i32]) -> i32 {
    arr.iter().sum()
}

#[wasm_bindgen]
pub fn filter_positive(arr: Vec<i32>) -> Vec<i32> {
    arr.into_iter().filter(|&x| x > 0).collect()
}

#[wasm_bindgen]
pub fn sort_array(mut arr: Vec<i32>) -> Vec<i32> {
    arr.sort();
    arr
}

#[wasm_bindgen]
pub fn create_range(start: i32, end: i32) -> Vec<i32> {
    (start..end).collect()
}

// Working with byte arrays
#[wasm_bindgen]
pub fn process_bytes(data: &[u8]) -> Vec<u8> {
    data.iter().map(|&b| b.wrapping_add(1)).collect()
}
```

```javascript
// Using arrays from JavaScript
import init, { 
    sum_array, 
    filter_positive, 
    sort_array, 
    create_range,
    process_bytes 
} from './pkg/my_wasm_project.js';

async function run() {
    await init();
    
    // Int32Array
    const numbers = new Int32Array([1, 2, 3, 4, 5]);
    console.log(sum_array(numbers)); // 15
    
    // Regular arrays work too
    console.log(filter_positive([-5, 3, -2, 8, 0])); // [3, 8]
    console.log(sort_array([5, 2, 8, 1, 9])); // [1, 2, 5, 8, 9]
    console.log(create_range(0, 10)); // [0, 1, 2, ..., 9]
    
    // Byte arrays
    const bytes = new Uint8Array([1, 2, 3, 4, 5]);
    console.log(process_bytes(bytes)); // [2, 3, 4, 5, 6]
}
```

### Working with JsValue

```rust
use wasm_bindgen::prelude::*;
use serde::{Serialize, Deserialize};

// Struct that can be converted to/from JavaScript objects
#[derive(Serialize, Deserialize)]
pub struct Product {
    id: u32,
    name: String,
    price: f64,
    in_stock: bool,
}

#[wasm_bindgen]
pub fn process_product(js_value: JsValue) -> Result<JsValue, JsValue> {
    // Deserialize from JavaScript object
    let mut product: Product = serde_wasm_bindgen::from_value(js_value)
        .map_err(|e| JsValue::from_str(&e.to_string()))?;
    
    // Modify the product
    product.price *= 1.1; // 10% price increase
    
    // Serialize back to JavaScript object
    serde_wasm_bindgen::to_value(&product)
        .map_err(|e| JsValue::from_str(&e.to_string()))
}

#[wasm_bindgen]
pub fn create_product_list(count: u32) -> Result<JsValue, JsValue> {
    let products: Vec<Product> = (0..count)
        .map(|i| Product {
            id: i,
            name: format!("Product {}", i),
            price: (i as f64) * 9.99,
            in_stock: i % 2 == 0,
        })
        .collect();
    
    serde_wasm_bindgen::to_value(&products)
        .map_err(|e| JsValue::from_str(&e.to_string()))
}
```

```toml
# Add to Cargo.toml
[dependencies]
serde = { version = "1.0", features = ["derive"] }
serde_wasm_bindgen = "0.6"
```

```javascript
// Using complex objects
import init, { process_product, create_product_list } from './pkg/my_wasm_project.js';

async function run() {
    await init();
    
    // Pass JavaScript object to Rust
    const product = {
        id: 1,
        name: 'Laptop',
        price: 999.99,
        in_stock: true
    };
    
    const updated = process_product(product);
    console.log(updated); // price increased by 10%
    
    // Get array of objects from Rust
    const products = create_product_list(5);
    console.log(products); // Array of 5 products
}
```

## Accessing Web APIs

### Using web-sys

```toml
# Add to Cargo.toml
[dependencies]
web-sys = { version = "0.3", features = [
    "console",
    "Document",
    "Element",
    "HtmlElement",
    "Window",
] }
```

```rust
use wasm_bindgen::prelude::*;
use web_sys::{console, window, Document};

#[wasm_bindgen]
pub fn log_to_console(message: &str) {
    console::log_1(&message.into());
}

#[wasm_bindgen]
pub fn get_window_size() -> Result<JsValue, JsValue> {
    let window = window().ok_or("No window")?;
    let width = window.inner_width()?.as_f64().unwrap_or(0.0);
    let height = window.inner_height()?.as_f64().unwrap_or(0.0);
    
    let obj = js_sys::Object::new();
    js_sys::Reflect::set(&obj, &"width".into(), &width.into())?;
    js_sys::Reflect::set(&obj, &"height".into(), &height.into())?;
    
    Ok(obj.into())
}

#[wasm_bindgen]
pub fn create_element(tag: &str, text: &str) -> Result<(), JsValue> {
    let window = window().ok_or("No window")?;
    let document = window.document().ok_or("No document")?;
    
    let element = document.create_element(tag)?;
    element.set_text_content(Some(text));
    
    let body = document.body().ok_or("No body")?;
    body.append_child(&element)?;
    
    Ok(())
}

#[wasm_bindgen]
pub fn modify_dom() -> Result<(), JsValue> {
    let window = window().ok_or("No window")?;
    let document = window.document().ok_or("No document")?;
    
    // Get element by id
    let element = document
        .get_element_by_id("my-element")
        .ok_or("Element not found")?;
    
    // Modify element
    element.set_inner_html("<h1>Hello from Rust!</h1>");
    
    // Add class
    element.class_list().add_1("rust-modified")?;
    
    Ok(())
}
```

### Handling Events

```toml
# Add to Cargo.toml features
[dependencies]
web-sys = { version = "0.3", features = [
    "console",
    "Document",
    "Element",
    "EventTarget",
    "HtmlElement",
    "MouseEvent",
    "Window",
] }
```

```rust
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use web_sys::{console, window, EventTarget, MouseEvent};

#[wasm_bindgen]
pub fn setup_click_handler() -> Result<(), JsValue> {
    let window = window().ok_or("No window")?;
    let document = window.document().ok_or("No document")?;
    
    let button = document
        .get_element_by_id("my-button")
        .ok_or("Button not found")?;
    
    // Create closure for event handler
    let closure = Closure::wrap(Box::new(move |event: MouseEvent| {
        console::log_1(&"Button clicked!".into());
        console::log_1(&format!("X: {}, Y: {}", event.client_x(), event.client_y()).into());
    }) as Box<dyn FnMut(MouseEvent)>);
    
    // Add event listener
    button
        .add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
    
    // Forget the closure to keep it alive
    closure.forget();
    
    Ok(())
}

// Better pattern with proper cleanup
#[wasm_bindgen]
pub struct EventHandler {
    closure: Option<Closure<dyn FnMut(MouseEvent)>>,
}

#[wasm_bindgen]
impl EventHandler {
    #[wasm_bindgen(constructor)]
    pub fn new() -> Result<EventHandler, JsValue> {
        let closure = Closure::wrap(Box::new(move |event: MouseEvent| {
            console::log_1(&format!("Clicked at: {}, {}", 
                event.client_x(), 
                event.client_y()
            ).into());
        }) as Box<dyn FnMut(MouseEvent)>);
        
        let window = window().ok_or("No window")?;
        let document = window.document().ok_or("No document")?;
        let button = document
            .get_element_by_id("my-button")
            .ok_or("Button not found")?;
        
        button.add_event_listener_with_callback(
            "click",
            closure.as_ref().unchecked_ref()
        )?;
        
        Ok(EventHandler {
            closure: Some(closure),
        })
    }
    
    #[wasm_bindgen]
    pub fn cleanup(&mut self) {
        // Closure will be dropped and event listener removed
        self.closure = None;
    }
}
```

## Performance Optimization

### SIMD Operations

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn add_arrays_simd(a: &[f32], b: &[f32]) -> Vec<f32> {
    // Rust compiler can auto-vectorize this
    a.iter()
        .zip(b.iter())
        .map(|(x, y)| x + y)
        .collect()
}

#[wasm_bindgen]
pub fn dot_product(a: &[f32], b: &[f32]) -> f32 {
    a.iter()
        .zip(b.iter())
        .map(|(x, y)| x * y)
        .sum()
}

// Manual SIMD with portable_simd (nightly)
#[cfg(target_feature = "simd128")]
#[wasm_bindgen]
pub fn add_arrays_simd_manual(a: &[f32], b: &[f32]) -> Vec<f32> {
    use std::simd::*;
    
    let mut result = Vec::with_capacity(a.len());
    let lanes = 4;
    
    let chunks = a.len() / lanes;
    
    for i in 0..chunks {
        let offset = i * lanes;
        let va = f32x4::from_slice(&a[offset..offset + lanes]);
        let vb = f32x4::from_slice(&b[offset..offset + lanes]);
        let vc = va + vb;
        result.extend_from_slice(&vc.to_array());
    }
    
    // Handle remainder
    for i in (chunks * lanes)..a.len() {
        result.push(a[i] + b[i]);
    }
    
    result
}
```

### Memory Management

```rust
use wasm_bindgen::prelude::*;

// Efficient image processing with shared memory
#[wasm_bindgen]
pub struct ImageProcessor {
    width: u32,
    height: u32,
    data: Vec<u8>,
}

#[wasm_bindgen]
impl ImageProcessor {
    #[wasm_bindgen(constructor)]
    pub fn new(width: u32, height: u32) -> ImageProcessor {
        let size = (width * height * 4) as usize;
        ImageProcessor {
            width,
            height,
            data: vec![0; size],
        }
    }
    
    // Return pointer to internal buffer
    #[wasm_bindgen]
    pub fn buffer(&self) -> *const u8 {
        self.data.as_ptr()
    }
    
    // Get buffer length
    #[wasm_bindgen]
    pub fn buffer_len(&self) -> usize {
        self.data.len()
    }
    
    // Grayscale conversion
    #[wasm_bindgen]
    pub fn grayscale(&mut self) {
        for i in (0..self.data.len()).step_by(4) {
            let r = self.data[i] as f32;
            let g = self.data[i + 1] as f32;
            let b = self.data[i + 2] as f32;
            
            let gray = (r * 0.299 + g * 0.587 + b * 0.114) as u8;
            
            self.data[i] = gray;
            self.data[i + 1] = gray;
            self.data[i + 2] = gray;
        }
    }
    
    // Brightness adjustment
    #[wasm_bindgen]
    pub fn adjust_brightness(&mut self, amount: i32) {
        for i in (0..self.data.len()).step_by(4) {
            self.data[i] = (self.data[i] as i32 + amount).clamp(0, 255) as u8;
            self.data[i + 1] = (self.data[i + 1] as i32 + amount).clamp(0, 255) as u8;
            self.data[i + 2] = (self.data[i + 2] as i32 + amount).clamp(0, 255) as u8;
        }
    }
}
```

```javascript
// Using shared memory efficiently
import init, { ImageProcessor } from './pkg/my_wasm_project.js';

async function processImage(imageData) {
    await init();
    
    const processor = new ImageProcessor(imageData.width, imageData.height);
    
    // Get direct access to WASM memory
    const wasmMemory = new Uint8Array(
        (await init()).memory.buffer,
        processor.buffer(),
        processor.buffer_len()
    );
    
    // Copy image data to WASM memory
    wasmMemory.set(imageData.data);
    
    // Process in WASM
    processor.grayscale();
    processor.adjust_brightness(20);
    
    // Copy back to canvas
    imageData.data.set(wasmMemory);
}
```

## Error Handling

```rust
use wasm_bindgen::prelude::*;
use std::fmt;

// Custom error type
#[derive(Debug)]
pub enum MyError {
    InvalidInput(String),
    NotFound(String),
    CalculationError(String),
}

impl fmt::Display for MyError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            MyError::InvalidInput(msg) => write!(f, "Invalid input: {}", msg),
            MyError::NotFound(msg) => write!(f, "Not found: {}", msg),
            MyError::CalculationError(msg) => write!(f, "Calculation error: {}", msg),
        }
    }
}

impl std::error::Error for MyError {}

// Convert to JsValue for wasm_bindgen
impl From<MyError> for JsValue {
    fn from(error: MyError) -> Self {
        JsValue::from_str(&error.to_string())
    }
}

#[wasm_bindgen]
pub fn divide(a: f64, b: f64) -> Result<f64, MyError> {
    if b == 0.0 {
        Err(MyError::CalculationError("Division by zero".to_string()))
    } else {
        Ok(a / b)
    }
}

#[wasm_bindgen]
pub fn find_user(id: u32, users: JsValue) -> Result<JsValue, MyError> {
    // Complex error handling
    let users: Vec<serde_json::Value> = serde_wasm_bindgen::from_value(users)
        .map_err(|e| MyError::InvalidInput(e.to_string()))?;
    
    users
        .iter()
        .find(|u| u["id"].as_u64() == Some(id as u64))
        .map(|u| serde_wasm_bindgen::to_value(u).unwrap())
        .ok_or_else(|| MyError::NotFound(format!("User {} not found", id)))
}
```

```javascript
// Handling errors from JavaScript
import init, { divide, find_user } from './pkg/my_wasm_project.js';

async function run() {
    await init();
    
    try {
        const result = divide(10, 0);
        console.log(result);
    } catch (error) {
        console.error('Error:', error); // "Calculation error: Division by zero"
    }
    
    try {
        const users = [
            { id: 1, name: 'John' },
            { id: 2, name: 'Jane' }
        ];
        const user = find_user(3, users);
        console.log(user);
    } catch (error) {
        console.error('Error:', error); // "Not found: User 3 not found"
    }
}
```

## Complete Example: Game of Life

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub struct Universe {
    width: u32,
    height: u32,
    cells: Vec<u8>,
}

#[wasm_bindgen]
impl Universe {
    #[wasm_bindgen(constructor)]
    pub fn new(width: u32, height: u32) -> Universe {
        let cells = (0..width * height)
            .map(|_| if js_sys::Math::random() < 0.5 { 1 } else { 0 })
            .collect();
        
        Universe { width, height, cells }
    }
    
    #[wasm_bindgen]
    pub fn width(&self) -> u32 {
        self.width
    }
    
    #[wasm_bindgen]
    pub fn height(&self) -> u32 {
        self.height
    }
    
    #[wasm_bindgen]
    pub fn cells(&self) -> *const u8 {
        self.cells.as_ptr()
    }
    
    #[wasm_bindgen]
    pub fn tick(&mut self) {
        let mut next = self.cells.clone();
        
        for row in 0..self.height {
            for col in 0..self.width {
                let idx = self.get_index(row, col);
                let cell = self.cells[idx];
                let live_neighbors = self.live_neighbor_count(row, col);
                
                next[idx] = match (cell, live_neighbors) {
                    (1, x) if x < 2 => 0,
                    (1, 2) | (1, 3) => 1,
                    (1, x) if x > 3 => 0,
                    (0, 3) => 1,
                    (otherwise, _) => otherwise,
                };
            }
        }
        
        self.cells = next;
    }
    
    #[wasm_bindgen]
    pub fn toggle_cell(&mut self, row: u32, col: u32) {
        let idx = self.get_index(row, col);
        self.cells[idx] ^= 1;
    }
    
    fn get_index(&self, row: u32, col: u32) -> usize {
        (row * self.width + col) as usize
    }
    
    fn live_neighbor_count(&self, row: u32, col: u32) -> u8 {
        let mut count = 0;
        
        for delta_row in [self.height - 1, 0, 1].iter().cloned() {
            for delta_col in [self.width - 1, 0, 1].iter().cloned() {
                if delta_row == 0 && delta_col == 0 {
                    continue;
                }
                
                let neighbor_row = (row + delta_row) % self.height;
                let neighbor_col = (col + delta_col) % self.width;
                let idx = self.get_index(neighbor_row, neighbor_col);
                count += self.cells[idx];
            }
        }
        
        count
    }
}
```

```javascript
// Game of Life renderer
import init, { Universe } from './pkg/game_of_life.js';

class GameOfLife {
    constructor(canvas, width, height, cellSize) {
        this.canvas = canvas;
        this.ctx = canvas.getContext('2d');
        this.cellSize = cellSize;
        this.universe = null;
        this.animationId = null;
        
        canvas.width = width * cellSize;
        canvas.height = height * cellSize;
        
        init().then(() => {
            this.universe = new Universe(width, height);
            this.render();
        });
    }
    
    render() {
        const width = this.universe.width();
        const height = this.universe.height();
        const cellsPtr = this.universe.cells();
        const cells = new Uint8Array(
            (await init()).memory.buffer,
            cellsPtr,
            width * height
        );
        
        this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);
        
        for (let row = 0; row < height; row++) {
            for (let col = 0; col < width; col++) {
                const idx = row * width + col;
                
                this.ctx.fillStyle = cells[idx] === 1 ? '#000' : '#fff';
                this.ctx.fillRect(
                    col * this.cellSize,
                    row * this.cellSize,
                    this.cellSize,
                    this.cellSize
                );
            }
        }
    }
    
    tick() {
        this.universe.tick();
        this.render();
    }
    
    start() {
        const loop = () => {
            this.tick();
            this.animationId = requestAnimationFrame(loop);
        };
        loop();
    }
    
    stop() {
        cancelAnimationFrame(this.animationId);
        this.animationId = null;
    }
}

// Usage
const canvas = document.getElementById('game-of-life');
const game = new GameOfLife(canvas, 64, 64, 8);
game.start();
```

## Browser Support

Rust to WebAssembly works in all browsers that support WebAssembly (see wasm-basics.md).

## Common Mistakes

1. **Forgetting to add wasm target**
```bash
# Will fail without wasm32 target
cargo build --target wasm32-unknown-unknown

# Install target first
rustup target add wasm32-unknown-unknown
```

2. **Not using wasm-bindgen correctly**
```rust
// Wrong - this won't be exported
pub fn my_function() -> i32 {
    42
}

// Correct - needs #[wasm_bindgen]
#[wasm_bindgen]
pub fn my_function() -> i32 {
    42
}
```

3. **Memory leaks with closures**
```rust
// Wrong - closure is forgotten and never cleaned up
closure.forget();

// Better - store closure and clean up properly
pub struct Handler {
    closure: Option<Closure<dyn FnMut()>>,
}
```

## Best Practices

1. **Use wasm-pack** - Simplifies build and publishing
2. **Leverage Rust's Type System** - Catch errors at compile time
3. **Minimize JS-Rust Boundary Crossings** - Batch operations
4. **Use Shared Memory** - Avoid copying large data structures
5. **Profile Performance** - Use browser devtools and cargo flamegraph
6. **Handle Errors Properly** - Use Result types
7. **Document Public APIs** - Use Rust doc comments
8. **Optimize Build Size** - Use release mode with LTO

## When to Use Rust for WASM

Use Rust for WebAssembly when:
- Performance is critical
- You need memory safety without GC
- Porting existing Rust code
- Building compute-intensive applications
- Need strong type safety
- Building game engines or simulations

## Interview Questions

1. What is wasm-bindgen and why is it needed?
2. How do you pass complex types between Rust and JavaScript?
3. What is the difference between wasm32-unknown-unknown and wasm32-wasi?
4. How do you handle memory in Rust WebAssembly?
5. What is wasm-pack and what does it do?
6. How do you access Web APIs from Rust?
7. What are the benefits of using Rust over C/C++ for WebAssembly?
8. How do you handle errors in Rust WebAssembly?

## Key Takeaways

- Rust is the premier language for WebAssembly development
- wasm-bindgen enables seamless JavaScript interop
- wasm-pack simplifies building and publishing
- web-sys provides access to Web APIs
- Memory can be shared efficiently between Rust and JS
- Rust's type system catches errors at compile time
- Performance is near-native for compute-intensive tasks
- Excellent tooling and ecosystem support

## Resources

- [Rust and WebAssembly Book](https://rustwasm.github.io/docs/book/)
- [wasm-bindgen Guide](https://rustwasm.github.io/wasm-bindgen/)
- [wasm-pack Documentation](https://rustwasm.github.io/wasm-pack/)
- [web-sys Documentation](https://rustwasm.github.io/wasm-bindgen/api/web_sys/)
- [Rust WASM Examples](https://github.com/rustwasm/wasm-bindgen/tree/main/examples)
