# WebGL Basics

## The Idea

**In plain English:** WebGL is a tool built into your web browser that lets a webpage draw fast, detailed 2D and 3D graphics directly using your computer's graphics chip (GPU), the same chip used to run video games — no extra software needed.

**Real-world analogy:** Think of a movie studio shooting a scene on a green screen. The director writes a script describing where every actor and prop should stand (the geometry). A lighting crew then runs through every pixel of the final image and decides what color it should be based on the lights, shadows, and costumes (the coloring step). The finished frame is sent to the projector screen for the audience to see.

- The director's script listing positions = the vertex shader (tells the GPU where each point in your shape sits on screen)
- The lighting crew coloring each pixel = the fragment shader (tells the GPU what color to paint each tiny dot)
- The projector screen = the HTML canvas element (the surface in the browser where the final image appears)

---

## Introduction

WebGL (Web Graphics Library) is a JavaScript API for rendering interactive 2D and 3D graphics within any compatible web browser without plugins. Based on OpenGL ES, WebGL provides hardware-accelerated graphics rendering capabilities through the HTML5 canvas element.

## Getting Started

### Basic Setup

```html
<!DOCTYPE html>
<html>
<head>
    <title>WebGL Basics</title>
    <style>
        canvas { width: 100%; height: 100%; display: block; }
    </style>
</head>
<body>
    <canvas id="glCanvas"></canvas>
    <script src="webgl-app.js"></script>
</body>
</html>
```

```javascript
// webgl-app.js
const canvas = document.getElementById('glCanvas');
const gl = canvas.getContext('webgl');

if (!gl) {
    alert('WebGL not supported');
}

// Set canvas size
canvas.width = window.innerWidth;
canvas.height = window.innerHeight;

// Set viewport
gl.viewport(0, 0, canvas.width, canvas.height);

// Clear canvas
gl.clearColor(0.0, 0.0, 0.0, 1.0);
gl.clear(gl.COLOR_BUFFER_BIT);
```

## Shaders

### Vertex Shader

```glsl
// vertex-shader.glsl
attribute vec4 a_position;
attribute vec4 a_color;

varying vec4 v_color;

uniform mat4 u_matrix;

void main() {
    gl_Position = u_matrix * a_position;
    v_color = a_color;
}
```

### Fragment Shader

```glsl
// fragment-shader.glsl
precision mediump float;

varying vec4 v_color;

void main() {
    gl_FragColor = v_color;
}
```

### Shader Compilation

```javascript
function createShader(gl, type, source) {
    const shader = gl.createShader(type);
    gl.shaderSource(shader, source);
    gl.compileShader(shader);
    
    if (!gl.getShaderParameter(shader, gl.COMPILE_STATUS)) {
        console.error('Shader compile error:', gl.getShaderInfoLog(shader));
        gl.deleteShader(shader);
        return null;
    }
    
    return shader;
}

function createProgram(gl, vertexShader, fragmentShader) {
    const program = gl.createProgram();
    gl.attachShader(program, vertexShader);
    gl.attachShader(program, fragmentShader);
    gl.linkProgram(program);
    
    if (!gl.getProgramParameter(program, gl.LINK_STATUS)) {
        console.error('Program link error:', gl.getProgramInfoLog(program));
        gl.deleteProgram(program);
        return null;
    }
    
    return program;
}
```

## Drawing Triangles

```javascript
function drawTriangle(gl) {
    // Vertex positions
    const positions = new Float32Array([
        0.0,  0.5, 0.0,
       -0.5, -0.5, 0.0,
        0.5, -0.5, 0.0
    ]);
    
    // Create buffer
    const positionBuffer = gl.createBuffer();
    gl.bindBuffer(gl.ARRAY_BUFFER, positionBuffer);
    gl.bufferData(gl.ARRAY_BUFFER, positions, gl.STATIC_DRAW);
    
    // Create shaders
    const vertexShader = createShader(gl, gl.VERTEX_SHADER, vertexShaderSource);
    const fragmentShader = createShader(gl, gl.FRAGMENT_SHADER, fragmentShaderSource);
    
    // Create program
    const program = createProgram(gl, vertexShader, fragmentShader);
    gl.useProgram(program);
    
    // Get attribute location
    const positionLocation = gl.getAttribLocation(program, 'a_position');
    
    // Enable attribute
    gl.enableVertexAttribArray(positionLocation);
    gl.vertexAttribPointer(positionLocation, 3, gl.FLOAT, false, 0, 0);
    
    // Draw
    gl.drawArrays(gl.TRIANGLES, 0, 3);
}
```

## 3D Rendering

```javascript
// Matrix utilities
class Mat4 {
    static perspective(fov, aspect, near, far) {
        const f = 1.0 / Math.tan(fov / 2);
        return new Float32Array([
            f / aspect, 0, 0, 0,
            0, f, 0, 0,
            0, 0, (far + near) / (near - far), -1,
            0, 0, (2 * far * near) / (near - far), 0
        ]);
    }
    
    static rotateY(angle) {
        const c = Math.cos(angle);
        const s = Math.sin(angle);
        return new Float32Array([
            c, 0, s, 0,
            0, 1, 0, 0,
           -s, 0, c, 0,
            0, 0, 0, 1
        ]);
    }
}

function drawCube(gl, program, angle) {
    const projectionMatrix = Mat4.perspective(
        Math.PI / 4, canvas.width / canvas.height, 0.1, 100.0
    );
    const modelMatrix = Mat4.rotateY(angle);
    
    const uMatrix = gl.getUniformLocation(program, 'u_matrix');
    gl.uniformMatrix4fv(uMatrix, false, projectionMatrix);
    
    // Draw cube...
    gl.drawArrays(gl.TRIANGLES, 0, 36);
}
```

## Textures

```javascript
function loadTexture(gl, url) {
    const texture = gl.createTexture();
    const image = new Image();
    
    image.onload = () => {
        gl.bindTexture(gl.TEXTURE_2D, texture);
        gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA, gl.RGBA, gl.UNSIGNED_BYTE, image);
        gl.generateMipmap(gl.TEXTURE_2D);
    };
    
    image.src = url;
    return texture;
}
```

## Browser Support

| Browser | Version |
|---------|---------|
| Chrome | 9+ |
| Firefox | 4+ |
| Safari | 5.1+ |
| Edge | All |

## Common Mistakes

1. Not checking WebGL support
2. Forgetting to compile shaders
3. Not binding buffers
4. Incorrect attribute pointers
5. Missing viewport setup

## Best Practices

1. Check WebGL support
2. Compile shaders carefully
3. Use vertex buffer objects
4. Minimize state changes
5. Batch draw calls
6. Use texture atlases
7. Profile performance
8. Handle context loss

## Key Takeaways

- WebGL provides hardware-accelerated graphics
- Shaders control rendering pipeline
- Buffers store vertex data
- Matrices handle transformations
- Textures add visual detail
- Performance optimization is crucial

## Resources

- [WebGL Fundamentals](https://webglfundamentals.org/)
- [MDN: WebGL API](https://developer.mozilla.org/en-US/docs/Web/API/WebGL_API)
- [WebGL Specification](https://www.khronos.org/webgl/)
- [The Book of Shaders](https://thebookofshaders.com/)
