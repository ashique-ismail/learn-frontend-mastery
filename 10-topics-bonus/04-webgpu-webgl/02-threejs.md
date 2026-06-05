# Three.js 3D Graphics

## The Idea

**In plain English:** Three.js is a JavaScript library that lets you build and display 3D worlds inside a web browser, without needing to know the complicated low-level graphics code underneath. Think of it as a toolbox full of pre-built pieces — shapes, lights, cameras — that you snap together to create interactive 3D experiences.

**Real-world analogy:** Imagine setting up a scene for a school play. You have a stage, props, spotlights, and a director deciding what the audience sees from a specific seat.

- The stage = the Three.js `Scene` (the container that holds everything)
- The props and set pieces = `Mesh` objects (the 3D shapes like cubes and spheres)
- The spotlights = `Light` objects (they illuminate the scene so things are visible)
- The director's viewpoint from a seat = the `Camera` (it determines what angle the browser renders the scene from)

---

## Introduction

Three.js is a powerful JavaScript library that makes creating 3D graphics on the web accessible by abstracting WebGL complexity. It provides an intuitive API for building 3D scenes, adding objects, lights, cameras, and animations with significantly less code than raw WebGL.

## Getting Started

### Installation

```bash
npm install three
```

```html
<!-- Or CDN -->
<script src="https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.min.js"></script>
```

### Basic Scene Setup

```javascript
import * as THREE from 'three';

// Scene
const scene = new THREE.Scene();

// Camera
const camera = new THREE.PerspectiveCamera(
    75,                                    // FOV
    window.innerWidth / window.innerHeight, // Aspect ratio
    0.1,                                   // Near plane
    1000                                   // Far plane
);
camera.position.z = 5;

// Renderer
const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

// Handle resize
window.addEventListener('resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
});
```

## Creating Objects

### Basic Geometry

```javascript
// Cube
const geometry = new THREE.BoxGeometry(1, 1, 1);
const material = new THREE.MeshBasicMaterial({ color: 0x00ff00 });
const cube = new THREE.Mesh(geometry, material);
scene.add(cube);

// Sphere
const sphereGeometry = new THREE.SphereGeometry(1, 32, 32);
const sphereMaterial = new THREE.MeshStandardMaterial({ 
    color: 0xff0000,
    metalness: 0.5,
    roughness: 0.5
});
const sphere = new THREE.Mesh(sphereGeometry, sphereMaterial);
sphere.position.x = 3;
scene.add(sphere);
```

## Lighting

```javascript
// Ambient light
const ambientLight = new THREE.AmbientLight(0x404040, 0.5);
scene.add(ambientLight);

// Directional light
const directionalLight = new THREE.DirectionalLight(0xffffff, 1);
directionalLight.position.set(5, 5, 5);
scene.add(directionalLight);

// Point light
const pointLight = new THREE.PointLight(0xff0000, 1, 100);
pointLight.position.set(0, 2, 0);
scene.add(pointLight);

// Spotlight
const spotLight = new THREE.SpotLight(0xffffff, 1);
spotLight.position.set(10, 10, 10);
spotLight.castShadow = true;
scene.add(spotLight);
```

## Materials

```javascript
// Standard material (PBR)
const standardMaterial = new THREE.MeshStandardMaterial({
    color: 0x00ff00,
    metalness: 0.7,
    roughness: 0.3
});

// Physical material
const physicalMaterial = new THREE.MeshPhysicalMaterial({
    color: 0xffffff,
    metalness: 0.9,
    roughness: 0.1,
    clearcoat: 1.0,
    clearcoatRoughness: 0.1
});

// Textured material
const textureLoader = new THREE.TextureLoader();
const texture = textureLoader.load('texture.jpg');
const texturedMaterial = new THREE.MeshStandardMaterial({
    map: texture
});
```

## Animation

```javascript
function animate() {
    requestAnimationFrame(animate);
    
    // Rotate cube
    cube.rotation.x += 0.01;
    cube.rotation.y += 0.01;
    
    // Move sphere
    sphere.position.y = Math.sin(Date.now() * 0.001) * 2;
    
    renderer.render(scene, camera);
}

animate();
```

## Controls

```javascript
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;
controls.dampingFactor = 0.05;
controls.minDistance = 2;
controls.maxDistance = 10;

function animate() {
    requestAnimationFrame(animate);
    controls.update();
    renderer.render(scene, camera);
}
```

## Loading 3D Models

```javascript
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader';

const loader = new GLTFLoader();

loader.load(
    'model.gltf',
    (gltf) => {
        scene.add(gltf.scene);
        gltf.scene.scale.set(2, 2, 2);
    },
    (progress) => {
        console.log((progress.loaded / progress.total * 100) + '% loaded');
    },
    (error) => {
        console.error('Error loading model:', error);
    }
);
```

## Raycasting (Mouse Interaction)

```javascript
const raycaster = new THREE.Raycaster();
const mouse = new THREE.Vector2();

window.addEventListener('click', (event) => {
    mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
    mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;
    
    raycaster.setFromCamera(mouse, camera);
    const intersects = raycaster.intersectObjects(scene.children);
    
    if (intersects.length > 0) {
        intersects[0].object.material.color.set(0xff0000);
    }
});
```

## Post-Processing

```javascript
import { EffectComposer } from 'three/examples/jsm/postprocessing/EffectComposer';
import { RenderPass } from 'three/examples/jsm/postprocessing/RenderPass';
import { UnrealBloomPass } from 'three/examples/jsm/postprocessing/UnrealBloomPass';

const composer = new EffectComposer(renderer);
const renderPass = new RenderPass(scene, camera);
composer.addPass(renderPass);

const bloomPass = new UnrealBloomPass(
    new THREE.Vector2(window.innerWidth, window.innerHeight),
    1.5,  // strength
    0.4,  // radius
    0.85  // threshold
);
composer.addPass(bloomPass);

function animate() {
    requestAnimationFrame(animate);
    composer.render();
}
```

## Shadows

```javascript
// Enable shadows
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;

// Light casts shadows
directionalLight.castShadow = true;
directionalLight.shadow.mapSize.width = 2048;
directionalLight.shadow.mapSize.height = 2048;

// Object casts shadows
cube.castShadow = true;

// Object receives shadows
const ground = new THREE.Mesh(
    new THREE.PlaneGeometry(10, 10),
    new THREE.MeshStandardMaterial({ color: 0x808080 })
);
ground.receiveShadow = true;
ground.rotation.x = -Math.PI / 2;
scene.add(ground);
```

## Performance Optimization

```javascript
// Use BufferGeometry
const geometry = new THREE.BufferGeometry();
const vertices = new Float32Array([...]);
geometry.setAttribute('position', new THREE.BufferAttribute(vertices, 3));

// Dispose unused objects
geometry.dispose();
material.dispose();
texture.dispose();

// Level of Detail (LOD)
const lod = new THREE.LOD();
lod.addLevel(highDetailMesh, 0);
lod.addLevel(mediumDetailMesh, 50);
lod.addLevel(lowDetailMesh, 100);
scene.add(lod);

// Instance rendering
const instancedMesh = new THREE.InstancedMesh(geometry, material, 1000);
for (let i = 0; i < 1000; i++) {
    const matrix = new THREE.Matrix4();
    matrix.setPosition(
        Math.random() * 100 - 50,
        Math.random() * 100 - 50,
        Math.random() * 100 - 50
    );
    instancedMesh.setMatrixAt(i, matrix);
}
scene.add(instancedMesh);
```

## Browser Support

Three.js works in all browsers that support WebGL (all modern browsers).

## Best Practices

1. Reuse geometries and materials
2. Dispose unused objects
3. Use BufferGeometry
4. Limit draw calls
5. Optimize textures
6. Use instancing for many objects
7. Implement LOD
8. Profile performance

## Key Takeaways

- Three.js simplifies 3D web graphics
- Built on top of WebGL
- Rich ecosystem of examples
- Supports various file formats
- Easy animation system
- Great documentation
- Active community

## Resources

- [Three.js Official Site](https://threejs.org/)
- [Three.js Documentation](https://threejs.org/docs/)
- [Three.js Examples](https://threejs.org/examples/)
- [Three.js Journey](https://threejs-journey.com/)
- [Discover Three.js](https://discoverthreejs.com/)
