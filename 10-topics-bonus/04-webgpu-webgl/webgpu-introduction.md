# WebGPU Introduction

## Introduction

WebGPU is a modern GPU API for the web, providing low-level, high-performance access to GPU hardware. It's designed to expose modern GPU capabilities with better performance, lower overhead, and support for general-purpose GPU computing compared to WebGL.

## Getting Started

### Basic Setup

```javascript
// Check WebGPU support
if (!navigator.gpu) {
    throw new Error('WebGPU not supported');
}

// Request adapter and device
const adapter = await navigator.gpu.requestAdapter();
const device = await adapter.requestDevice();

// Get canvas context
const canvas = document.getElementById('gpuCanvas');
const context = canvas.getContext('webgpu');

// Configure canvas
const canvasFormat = navigator.gpu.getPreferredCanvasFormat();
context.configure({
    device,
    format: canvasFormat
});
```

## Render Pipeline

```javascript
// Create shader module
const shaderModule = device.createShaderModule({
    label: 'Triangle shader',
    code: `
        @vertex
        fn vertexMain(@location(0) position: vec2f) -> @builtin(position) vec4f {
            return vec4f(position, 0, 1);
        }
        
        @fragment
        fn fragmentMain() -> @location(0) vec4f {
            return vec4f(1, 0, 0, 1); // Red
        }
    `
});

// Create render pipeline
const pipeline = device.createRenderPipeline({
    label: 'Triangle pipeline',
    layout: 'auto',
    vertex: {
        module: shaderModule,
        entryPoint: 'vertexMain',
        buffers: [{
            arrayStride: 8,
            attributes: [{
                format: 'float32x2',
                offset: 0,
                shaderLocation: 0
            }]
        }]
    },
    fragment: {
        module: shaderModule,
        entryPoint: 'fragmentMain',
        targets: [{
            format: canvasFormat
        }]
    }
});
```

## Drawing

```javascript
// Create vertex buffer
const vertices = new Float32Array([
     0.0,  0.5,
    -0.5, -0.5,
     0.5, -0.5
]);

const vertexBuffer = device.createBuffer({
    label: 'Triangle vertices',
    size: vertices.byteLength,
    usage: GPUBufferUsage.VERTEX | GPUBufferUsage.COPY_DST
});

device.queue.writeBuffer(vertexBuffer, 0, vertices);

// Render
function render() {
    const encoder = device.createCommandEncoder();
    const pass = encoder.beginRenderPass({
        colorAttachments: [{
            view: context.getCurrentTexture().createView(),
            loadOp: 'clear',
            clearValue: { r: 0, g: 0, b: 0, a: 1 },
            storeOp: 'store'
        }]
    });
    
    pass.setPipeline(pipeline);
    pass.setVertexBuffer(0, vertexBuffer);
    pass.draw(3);
    pass.end();
    
    device.queue.submit([encoder.finish()]);
}

render();
```

## Compute Shaders

```javascript
// Compute shader for parallel processing
const computeShaderCode = `
    @group(0) @binding(0) var<storage, read> input: array<f32>;
    @group(0) @binding(1) var<storage, read_write> output: array<f32>;
    
    @compute @workgroup_size(64)
    fn main(@builtin(global_invocation_id) id: vec3u) {
        output[id.x] = input[id.x] * 2.0;
    }
`;

const computePipeline = device.createComputePipeline({
    label: 'Compute pipeline',
    layout: 'auto',
    compute: {
        module: device.createShaderModule({ code: computeShaderCode }),
        entryPoint: 'main'
    }
});

// Create buffers
const inputBuffer = device.createBuffer({
    size: data.byteLength,
    usage: GPUBufferUsage.STORAGE | GPUBufferUsage.COPY_DST
});

const outputBuffer = device.createBuffer({
    size: data.byteLength,
    usage: GPUBufferUsage.STORAGE | GPUBufferUsage.COPY_SRC
});

// Bind group
const bindGroup = device.createBindGroup({
    layout: computePipeline.getBindGroupLayout(0),
    entries: [
        { binding: 0, resource: { buffer: inputBuffer } },
        { binding: 1, resource: { buffer: outputBuffer } }
    ]
});

// Dispatch compute
const encoder = device.createCommandEncoder();
const pass = encoder.beginComputePass();
pass.setPipeline(computePipeline);
pass.setBindGroup(0, bindGroup);
pass.dispatchWorkgroups(Math.ceil(data.length / 64));
pass.end();
device.queue.submit([encoder.finish()]);
```

## Textures

```javascript
// Load and use textures
const texture = device.createTexture({
    size: [width, height],
    format: 'rgba8unorm',
    usage: GPUTextureUsage.TEXTURE_BINDING | GPUTextureUsage.COPY_DST
});

// Write image data
device.queue.writeTexture(
    { texture },
    imageData,
    { bytesPerRow: width * 4 },
    [width, height]
);

// Create sampler
const sampler = device.createSampler({
    magFilter: 'linear',
    minFilter: 'linear'
});
```

## Browser Support

| Browser | Status |
|---------|--------|
| Chrome | 113+ |
| Firefox | Experimental |
| Safari | Preview |
| Edge | 113+ |

## WebGPU vs WebGL

| Feature | WebGL | WebGPU |
|---------|-------|--------|
| Performance | Good | Excellent |
| Compute | Limited | Full support |
| Modern GPU | Partial | Full |
| API Design | OpenGL-based | Modern |

## Best Practices

1. Use compute shaders for parallel tasks
2. Batch operations
3. Minimize state changes
4. Profile GPU usage
5. Handle async operations
6. Validate shaders
7. Manage memory efficiently
8. Test across devices

## Key Takeaways

- WebGPU is the future of web graphics
- Better performance than WebGL
- Supports compute shaders
- Modern API design
- Currently in development
- Great for ML and graphics

## Resources

- [WebGPU Specification](https://www.w3.org/TR/webgpu/)
- [WebGPU Fundamentals](https://webgpufundamentals.org/)
- [MDN: WebGPU API](https://developer.mozilla.org/en-US/docs/Web/API/WebGPU_API)
- [GPU for the Web Community](https://www.w3.org/community/gpu/)
