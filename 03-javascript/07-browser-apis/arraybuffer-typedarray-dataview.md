# ArrayBuffer, TypedArrays, and DataView

## Overview

JavaScript's binary data primitives form a three-layer system:

- **`ArrayBuffer`** — a fixed-length raw byte storage. You cannot read or write it directly.
- **TypedArrays** (`Uint8Array`, `Float32Array`, etc.) — typed views into an `ArrayBuffer`.
- **`DataView`** — a low-level view for reading/writing mixed types at arbitrary byte offsets and controlling endianness.

These are essential for: WebAssembly memory, WebGL buffers, canvas pixel manipulation, binary file parsing, WebRTC data channels, Web Audio, and network protocol implementations.

---

## ArrayBuffer

```javascript
// Create a 16-byte buffer (zeroed)
const buffer = new ArrayBuffer(16);
console.log(buffer.byteLength); // 16

// You cannot read/write ArrayBuffer directly
// You must create a view over it

// Transfer ownership (detaches original, zero-copy move)
const { buffer: transferred } = new Uint8Array([1, 2, 3]);
const newBuffer = transferred.transfer(6); // resize to 6 bytes
// transferred is now detached (byteLength === 0)

// Check if detached
ArrayBuffer.isView(new Uint8Array()); // true — TypedArray is a view
```

---

## TypedArrays

Each TypedArray type specifies the element size, signedness, and float/int semantics:

| Type | Bytes | Range | Use case |
|---|---|---|---|
| `Int8Array` | 1 | −128 to 127 | Signed bytes |
| `Uint8Array` | 1 | 0–255 | Raw bytes, image data |
| `Uint8ClampedArray` | 1 | 0–255, clamped | Canvas pixel data |
| `Int16Array` | 2 | −32768 to 32767 | Audio samples |
| `Uint16Array` | 2 | 0–65535 | Unicode code units |
| `Int32Array` | 4 | ±2³¹ | Integer arithmetic |
| `Uint32Array` | 4 | 0–2³²−1 | Unsigned integers |
| `Float32Array` | 4 | ±3.4×10³⁸ | WebGL vertices |
| `Float64Array` | 8 | ±1.8×10³⁰⁸ | High-precision floats |
| `BigInt64Array` | 8 | ±2⁶³ | 64-bit integers |
| `BigUint64Array` | 8 | 0–2⁶⁴−1 | Unsigned 64-bit |

```javascript
// Create from length (zeroed buffer)
const u8 = new Uint8Array(4);      // [0, 0, 0, 0]

// Create from array
const floats = new Float32Array([1.5, 2.5, 3.5]);

// Create from existing ArrayBuffer
const buffer = new ArrayBuffer(8);
const ints = new Int32Array(buffer);        // 2 x 4-byte ints
const bytes = new Uint8Array(buffer);       // 8 bytes — SAME memory

// Modifying via one view affects the other (shared buffer)
ints[0] = 0x0102_0304;
console.log(bytes[0]); // 4 (little-endian: LSB first on x86)

// Create a slice of a buffer (offset, length in elements)
const partialView = new Uint8Array(buffer, 2, 4); // start at byte 2, 4 elements

// Copy data
const src  = new Uint8Array([1, 2, 3, 4]);
const dest = new Uint8Array(8);
dest.set(src, 2); // copy src into dest starting at offset 2
// dest: [0, 0, 1, 2, 3, 4, 0, 0]

// TypedArrays support most Array methods
const doubled = new Float32Array([1, 2, 3]).map(x => x * 2);
const sum = new Int32Array([1, 2, 3, 4]).reduce((a, b) => a + b, 0);
```

---

## DataView

Use `DataView` when you need to read/write mixed types or control byte order (endianness).

```javascript
const buffer = new ArrayBuffer(16);
const view   = new DataView(buffer);

// Write methods: setInt8, setUint8, setInt16, setUint16,
//               setInt32, setUint32, setFloat32, setFloat64, setBigInt64
view.setUint8(0, 0xFF);             // offset 0, value 255
view.setInt16(1, -300);             // offset 1, 2 bytes, big-endian (default)
view.setInt16(3, -300, true);       // offset 3, 2 bytes, LITTLE-endian
view.setFloat32(5, 3.14);           // offset 5, 4 bytes
view.setBigInt64(9, 9007199254740993n); // offset 9, 8 bytes

// Read methods: getInt8, getUint8, getInt16, ...
view.getUint8(0);                   // 255
view.getInt16(1);                   // -300
view.getInt16(1, true);             // different value — wrong endian!
view.getFloat32(5);                 // ~3.14

// Endianness matters for network protocols and file formats
// Network byte order = big-endian
// x86 CPUs = little-endian
// Always be explicit when interoperating
```

---

## Canvas Pixel Manipulation

```javascript
const canvas  = document.getElementById('canvas');
const ctx     = canvas.getContext('2d');
const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);

// imageData.data is a Uint8ClampedArray — [R, G, B, A, R, G, B, A, ...]
const data = imageData.data;

for (let i = 0; i < data.length; i += 4) {
  const r = data[i];
  const g = data[i + 1];
  const b = data[i + 2];
  // const a = data[i + 3]; // alpha

  // Grayscale: luminance formula
  const gray = 0.299 * r + 0.587 * g + 0.114 * b;
  data[i] = data[i + 1] = data[i + 2] = gray;
}

ctx.putImageData(imageData, 0, 0);

// Faster approach: 32-bit view reduces iterations 4×
const uint32 = new Uint32Array(imageData.data.buffer);
for (let i = 0; i < uint32.length; i++) {
  const pixel = uint32[i];
  const r = (pixel >> 0)  & 0xFF;
  const g = (pixel >> 8)  & 0xFF;
  const b = (pixel >> 16) & 0xFF;
  const a = (pixel >> 24) & 0xFF;
  const gray = (0.299 * r + 0.587 * g + 0.114 * b) | 0;
  uint32[i] = (a << 24) | (gray << 16) | (gray << 8) | gray;
}
```

---

## WebAssembly Memory

```javascript
// WASM linear memory is an ArrayBuffer
const memory = new WebAssembly.Memory({ initial: 1 }); // 1 page = 64 KB
const buffer = memory.buffer; // ArrayBuffer

// Read/write WASM memory from JS
const view = new Uint8Array(buffer);
view[0] = 42;

// After memory.grow(), buffer is replaced — must re-wrap
memory.grow(1); // grow by 1 page
const newView = new Uint8Array(memory.buffer); // old `view` is detached!

// Passing strings to WASM: encode to UTF-8 bytes
function writeString(memory, ptr, str) {
  const bytes = new TextEncoder().encode(str + '\0'); // null-terminated
  new Uint8Array(memory.buffer).set(bytes, ptr);
}

function readString(memory, ptr) {
  const bytes = new Uint8Array(memory.buffer, ptr);
  const end   = bytes.indexOf(0); // find null terminator
  return new TextDecoder().decode(bytes.subarray(0, end));
}
```

---

## Binary File Parsing

```javascript
// Parse a PNG file header
async function parsePngHeader(file) {
  const buffer  = await file.arrayBuffer();
  const view    = new DataView(buffer);
  const bytes   = new Uint8Array(buffer);

  // PNG signature: 8 bytes [137, 80, 78, 71, 13, 10, 26, 10]
  const PNG_SIG = [137, 80, 78, 71, 13, 10, 26, 10];
  const valid   = PNG_SIG.every((b, i) => bytes[i] === b);
  if (!valid) throw new Error('Not a PNG');

  // IHDR chunk starts at byte 8
  // Chunk length (4 bytes, big-endian)
  const chunkLength = view.getUint32(8);

  // Chunk type (4 bytes ASCII)
  const chunkType = String.fromCharCode(...bytes.subarray(12, 16));

  // Image dimensions in IHDR
  const width  = view.getUint32(16);
  const height = view.getUint32(20);
  const bitDepth    = view.getUint8(24);
  const colorType   = view.getUint8(25);

  return { width, height, bitDepth, colorType };
}
```

---

## WebAudio

```javascript
// Web Audio API uses Float32Array for audio samples
const ctx          = new AudioContext();
const bufferSize   = ctx.sampleRate * 2; // 2 seconds at sample rate
const audioBuffer  = ctx.createBuffer(2, bufferSize, ctx.sampleRate);

// Fill left and right channels with white noise
for (let ch = 0; ch < 2; ch++) {
  const channelData = audioBuffer.getChannelData(ch); // Float32Array
  for (let i = 0; i < bufferSize; i++) {
    channelData[i] = Math.random() * 2 - 1; // -1.0 to 1.0
  }
}

const source = ctx.createBufferSource();
source.buffer = audioBuffer;
source.connect(ctx.destination);
source.start();
```

---

## Blob and Fetch Interop

```javascript
// ArrayBuffer ↔ Blob
const buffer  = new Uint8Array([72, 101, 108, 108, 111]).buffer;
const blob    = new Blob([buffer], { type: 'text/plain' });
const recovered = await blob.arrayBuffer(); // back to ArrayBuffer

// Fetch binary data
const res    = await fetch('/api/data.bin');
const buf    = await res.arrayBuffer();
const floats = new Float32Array(buf);

// Send binary data
await fetch('/api/upload', {
  method: 'POST',
  body: new Uint8Array([1, 2, 3, 4]).buffer,
  headers: { 'Content-Type': 'application/octet-stream' },
});

// TextEncoder / TextDecoder
const encoder = new TextEncoder();         // always UTF-8
const bytes   = encoder.encode('Hello 🌍');  // Uint8Array

const decoder = new TextDecoder('utf-8');
const text    = decoder.decode(bytes);     // "Hello 🌍"

// TextDecoder with non-UTF-8 encoding (legacy pages)
const latin1 = new TextDecoder('iso-8859-1');
```

---

## Structured Clone and Transferables

```javascript
// Transferring ArrayBuffer to a Worker — zero-copy, ownership moves
const buf    = new ArrayBuffer(1024 * 1024); // 1 MB
const worker = new Worker('worker.js');

worker.postMessage({ buffer: buf }, [buf]); // transfer, not copy
// buf is now detached: buf.byteLength === 0

// In worker.js:
self.onmessage = ({ data: { buffer } }) => {
  const view = new Uint8Array(buffer);
  // process...
  self.postMessage({ result: view }, [view.buffer]); // transfer back
};

// SharedArrayBuffer — truly shared memory between threads
// Requires Cross-Origin-Isolation headers
const shared = new SharedArrayBuffer(4);
const shared32 = new Int32Array(shared);
Atomics.store(shared32, 0, 42);
Atomics.add(shared32, 0, 1);      // atomic increment
Atomics.load(shared32, 0);        // 43
Atomics.wait(shared32, 0, 43);    // block until value !== 43
Atomics.notify(shared32, 0, 1);   // wake 1 waiting thread
```

---

## Interview Questions

**Q: What's the difference between `ArrayBuffer`, `TypedArray`, and `DataView`?**  
A: `ArrayBuffer` is raw byte storage — you can't read it directly. `TypedArray` (e.g. `Uint8Array`) is a typed view that interprets bytes as same-type elements with a fixed stride. `DataView` is a flexible view that lets you read/write different types at arbitrary offsets and control endianness — necessary for binary protocol parsing.

**Q: Why is `Uint8ClampedArray` used for canvas pixel data instead of `Uint8Array`?**  
A: Clamping ensures that values written outside the 0–255 range are silently clipped (255 for overflow, 0 for underflow) instead of wrapping. This prevents pixel corruption when doing arithmetic on color channels.

**Q: What is a "transferable" and why does it matter for Web Workers?**  
A: Transferring an `ArrayBuffer` to a Worker moves ownership (zero-copy) — the buffer is detached in the sender and available in the receiver. Copying (default `postMessage`) serializes and deserializes the full data, which is slow for large buffers like video frames or audio.

**Q: When would you use `DataView` instead of a TypedArray?**  
A: When parsing binary formats with mixed field types (e.g. a PNG header has uint32 length, 4-byte ASCII type, then more uint32s), or when the byte order of the data differs from the host CPU endianness.
