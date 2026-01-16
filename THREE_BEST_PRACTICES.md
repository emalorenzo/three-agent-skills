# Three.js Best Practices

Comprehensive performance optimization and best practices guide for Three.js applications. Contains rules organized by priority and impact.

> **Source**: This guide incorporates official guidelines from the Three.js `llms` branch maintained by mrdoob.
> Current Three.js version: **@0.182.0**

## Rule Categories by Priority

| Priority | Category | Impact | Prefix |
|----------|----------|--------|--------|
| 0 | Modern Setup & Imports | FUNDAMENTAL | `setup-` |
| 1 | Memory Management & Dispose | CRITICAL | `memory-` |
| 2 | Render Loop Optimization | CRITICAL | `render-` |
| 3 | Geometry & Buffer Management | HIGH | `geometry-` |
| 4 | Material & Texture Optimization | HIGH | `material-` |
| 5 | Lighting & Shadows | MEDIUM-HIGH | `lighting-` |
| 6 | Scene Graph Organization | MEDIUM | `scene-` |
| 7 | Shader Best Practices (GLSL) | MEDIUM | `shader-` |
| 8 | TSL (Three.js Shading Language) | MEDIUM | `tsl-` |
| 9 | Loading & Assets | MEDIUM | `loading-` |
| 10 | Camera & Controls | LOW-MEDIUM | `camera-` |
| 11 | Debug & DevTools | LOW | `debug-` |

---

## 0. Modern Setup & Imports (FUNDAMENTAL)

These are the foundational patterns for modern Three.js development. Using outdated patterns will cause issues.

### setup-use-import-maps

**Use Import Maps instead of old CDN script tags.**

The old CDN pattern is outdated and causes issues with module resolution.

```html
<!-- WRONG - Outdated pattern (DO NOT USE) -->
<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>

<!-- CORRECT - Modern Import Maps pattern -->
<script type="importmap">
{
  "imports": {
    "three": "https://cdn.jsdelivr.net/npm/three@0.182.0/build/three.module.js",
    "three/addons/": "https://cdn.jsdelivr.net/npm/three@0.182.0/examples/jsm/"
  }
}
</script>
<script type="module">
import * as THREE from 'three';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
</script>
```

### setup-choose-renderer

**Choose the appropriate renderer for your use case.**

Three.js maintains two renderers with different capabilities.

**Use WebGLRenderer (default, mature):**
- Maximum browser compatibility
- Well-established, most examples use this
- Standard GLSL shaders

```javascript
import * as THREE from 'three';

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
```

**Use WebGPURenderer when you need:**
- Custom shaders using TSL (Three.js Shading Language)
- Compute shaders
- Advanced node-based materials

```javascript
import * as THREE from 'three/webgpu';

const renderer = new THREE.WebGPURenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
await renderer.init(); // Required for WebGPU
```

### setup-animation-loop

**Use renderer.setAnimationLoop instead of manual requestAnimationFrame.**

```javascript
// OUTDATED - Manual RAF
function animate() {
  requestAnimationFrame(animate);
  renderer.render(scene, camera);
}
animate();

// MODERN - setAnimationLoop (handles XR automatically)
function animate() {
  renderer.render(scene, camera);
}
renderer.setAnimationLoop(animate);
```

### setup-basic-scene-template

**Complete modern scene template.**

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <title>Three.js Scene</title>
  <style>body { margin: 0; }</style>
</head>
<body>
<script type="importmap">
{
  "imports": {
    "three": "https://cdn.jsdelivr.net/npm/three@0.182.0/build/three.module.js",
    "three/addons/": "https://cdn.jsdelivr.net/npm/three@0.182.0/examples/jsm/"
  }
}
</script>
<script type="module">
import * as THREE from 'three';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';

// Scene
const scene = new THREE.Scene();

// Camera
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
camera.position.z = 5;

// Renderer
const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
document.body.appendChild(renderer.domElement);

// Controls
const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;

// Lighting
const ambientLight = new THREE.AmbientLight(0x404040);
scene.add(ambientLight);

const directionalLight = new THREE.DirectionalLight(0xffffff, 1);
directionalLight.position.set(5, 5, 5);
scene.add(directionalLight);

// Mesh
const geometry = new THREE.BoxGeometry(1, 1, 1);
const material = new THREE.MeshStandardMaterial({ color: 0x00ff00 });
const cube = new THREE.Mesh(geometry, material);
scene.add(cube);

// Handle resize
window.addEventListener('resize', () => {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
});

// Animation loop
function animate() {
  cube.rotation.x += 0.01;
  cube.rotation.y += 0.01;
  controls.update();
  renderer.render(scene, camera);
}
renderer.setAnimationLoop(animate);
</script>
</body>
</html>
```

---

## 1. Memory Management & Dispose (CRITICAL)

Three.js does NOT automatically garbage collect GPU resources. Failing to dispose objects causes memory leaks that crash the browser.

### memory-dispose-geometry

**Always dispose geometries when removing objects from scene.**

Geometries allocate GPU buffer memory that persists until explicitly freed.

```javascript
// BAD - Memory leak
scene.remove(mesh);
mesh = null; // GPU buffers still allocated!

// GOOD - Proper cleanup
scene.remove(mesh);
mesh.geometry.dispose();
mesh = null;
```

### memory-dispose-material

**Always dispose materials and their associated textures.**

Materials contain shader programs and texture references that must be freed.

```javascript
// BAD - Leaks material and textures
scene.remove(mesh);
mesh.geometry.dispose();

// GOOD - Full cleanup
scene.remove(mesh);
mesh.geometry.dispose();
if (Array.isArray(mesh.material)) {
  mesh.material.forEach(mat => {
    disposeMaterial(mat);
  });
} else {
  disposeMaterial(mesh.material);
}

function disposeMaterial(material) {
  // Dispose all texture maps
  const textureKeys = [
    'map', 'lightMap', 'bumpMap', 'normalMap', 'specularMap',
    'envMap', 'alphaMap', 'aoMap', 'displacementMap',
    'emissiveMap', 'gradientMap', 'metalnessMap', 'roughnessMap'
  ];

  textureKeys.forEach(key => {
    if (material[key]) {
      material[key].dispose();
    }
  });

  material.dispose();
}
```

### memory-dispose-textures

**Dispose textures when no longer needed, especially dynamically created ones.**

Each texture consumes GPU memory proportional to its dimensions.

```javascript
// BAD - Texture never freed
const texture = new THREE.TextureLoader().load('image.jpg');
// ... later, texture is no longer used but still in GPU memory

// GOOD - Explicit disposal
texture.dispose();
texture = null;
```

### memory-dispose-render-targets

**Always dispose render targets (WebGLRenderTarget) when done.**

Render targets allocate framebuffer memory on the GPU.

```javascript
// BAD - Framebuffer leak
const renderTarget = new THREE.WebGLRenderTarget(1024, 1024);
// ... later, no longer needed

// GOOD - Proper cleanup
renderTarget.dispose();
renderTarget.texture.dispose();
```

### memory-dispose-recursive

**Use recursive disposal for complex scene hierarchies.**

```javascript
function disposeObject(obj) {
  if (obj.geometry) {
    obj.geometry.dispose();
  }

  if (obj.material) {
    if (Array.isArray(obj.material)) {
      obj.material.forEach(disposeMaterial);
    } else {
      disposeMaterial(obj.material);
    }
  }

  if (obj.children) {
    obj.children.forEach(disposeObject);
  }
}

// Usage
disposeObject(complexModel);
scene.remove(complexModel);
```

### memory-dispose-on-unmount

**In React/component frameworks, dispose in cleanup/unmount.**

```javascript
// React example
useEffect(() => {
  const geometry = new THREE.BoxGeometry();
  const material = new THREE.MeshBasicMaterial();
  const mesh = new THREE.Mesh(geometry, material);
  scene.add(mesh);

  return () => {
    scene.remove(mesh);
    geometry.dispose();
    material.dispose();
  };
}, []);
```

### memory-renderer-dispose

**Dispose the renderer when destroying the 3D view.**

```javascript
// BAD - WebGL context leak
function destroyView() {
  container.removeChild(renderer.domElement);
}

// GOOD - Full cleanup
function destroyView() {
  renderer.dispose();
  renderer.forceContextLoss();
  renderer.domElement = null;
  container.removeChild(renderer.domElement);
}
```

### memory-reuse-objects

**Reuse geometries and materials instead of creating new ones.**

```javascript
// BAD - Creates new geometry for each cube
function createCube(position) {
  const geometry = new THREE.BoxGeometry(1, 1, 1);
  const material = new THREE.MeshBasicMaterial({ color: 0xff0000 });
  const mesh = new THREE.Mesh(geometry, material);
  mesh.position.copy(position);
  return mesh;
}

// GOOD - Shared geometry and material
const sharedGeometry = new THREE.BoxGeometry(1, 1, 1);
const sharedMaterial = new THREE.MeshBasicMaterial({ color: 0xff0000 });

function createCube(position) {
  const mesh = new THREE.Mesh(sharedGeometry, sharedMaterial);
  mesh.position.copy(position);
  return mesh;
}
```

---

## 2. Render Loop Optimization (CRITICAL)

The render loop runs 60+ times per second. Small inefficiencies multiply into major performance issues.

### render-single-raf

**Use a single requestAnimationFrame loop.**

Multiple RAF loops cause redundant renders and timing issues.

```javascript
// BAD - Multiple RAF loops
function animateObjects() {
  requestAnimationFrame(animateObjects);
  objects.forEach(obj => obj.update());
}
function animateCamera() {
  requestAnimationFrame(animateCamera);
  camera.update();
}

// GOOD - Single unified loop
function animate() {
  requestAnimationFrame(animate);

  objects.forEach(obj => obj.update());
  camera.update();

  renderer.render(scene, camera);
}
```

### render-conditional

**Only render when something changes (on-demand rendering).**

For static or rarely-changing scenes, continuous rendering wastes resources.

```javascript
// BAD - Always rendering
function animate() {
  requestAnimationFrame(animate);
  renderer.render(scene, camera);
}

// GOOD - Render on demand
let needsRender = true;

function requestRender() {
  needsRender = true;
}

function animate() {
  requestAnimationFrame(animate);

  if (needsRender) {
    renderer.render(scene, camera);
    needsRender = false;
  }
}

// Call requestRender() when:
// - Camera moves
// - Objects change
// - Window resizes
controls.addEventListener('change', requestRender);
```

### render-delta-time

**Use delta time for frame-rate independent animation.**

```javascript
// BAD - Animation speed varies with frame rate
function animate() {
  requestAnimationFrame(animate);
  object.rotation.y += 0.01; // Fast on 144hz, slow on 30hz
  renderer.render(scene, camera);
}

// GOOD - Consistent speed regardless of frame rate
const clock = new THREE.Clock();

function animate() {
  requestAnimationFrame(animate);
  const delta = clock.getDelta();

  object.rotation.y += 1.0 * delta; // 1 radian per second
  renderer.render(scene, camera);
}
```

### render-avoid-allocations

**Never allocate objects inside the render loop.**

Garbage collection causes frame drops.

```javascript
// BAD - Creates new Vector3 every frame
function animate() {
  requestAnimationFrame(animate);

  const direction = new THREE.Vector3(); // Allocation!
  direction.subVectors(target.position, object.position);
  object.lookAt(target.position);

  renderer.render(scene, camera);
}

// GOOD - Reuse pre-allocated objects
const direction = new THREE.Vector3();

function animate() {
  requestAnimationFrame(animate);

  direction.subVectors(target.position, object.position);
  object.lookAt(target.position);

  renderer.render(scene, camera);
}
```

### render-cache-computations

**Cache expensive computations outside the loop.**

```javascript
// BAD - Recalculating every frame
function animate() {
  requestAnimationFrame(animate);

  const boundingBox = new THREE.Box3().setFromObject(model);
  const center = boundingBox.getCenter(new THREE.Vector3());

  camera.lookAt(center);
  renderer.render(scene, camera);
}

// GOOD - Calculate once, update only when needed
let modelCenter = new THREE.Vector3();

function updateModelCenter() {
  const boundingBox = new THREE.Box3().setFromObject(model);
  boundingBox.getCenter(modelCenter);
}

function animate() {
  requestAnimationFrame(animate);
  camera.lookAt(modelCenter);
  renderer.render(scene, camera);
}
```

### render-frustum-culling

**Let Three.js handle frustum culling, ensure it's enabled.**

Objects outside camera view shouldn't be processed.

```javascript
// Frustum culling is ON by default
mesh.frustumCulled = true; // default

// Only disable when necessary (e.g., skinned meshes with animations)
skinnedMesh.frustumCulled = false;
```

### render-update-matrix-manual

**For static objects, disable automatic matrix updates.**

```javascript
// For objects that never move
staticMesh.matrixAutoUpdate = false;
staticMesh.updateMatrix(); // Call once after positioning

// For entire static scenes
scene.matrixAutoUpdate = false;
scene.updateMatrixWorld(true); // Call once
```

### render-pixel-ratio

**Limit pixel ratio on high-DPI displays.**

```javascript
// BAD - 4x pixels on retina displays (massive performance hit)
renderer.setPixelRatio(window.devicePixelRatio);

// GOOD - Cap at 2x for balance of quality and performance
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
```

### render-antialias-wisely

**Use antialiasing judiciously, consider alternatives.**

```javascript
// Antialiasing is expensive, especially on mobile
const renderer = new THREE.WebGLRenderer({
  antialias: !isMobile() // Disable on mobile
});

// Alternative: Use FXAA post-processing (cheaper)
// Or use SMAA for better quality at lower cost than MSAA
```

---

## 3. Geometry & Buffer Management (HIGH)

Geometry defines shape through vertex data. Efficient geometry means faster GPU processing.

### geometry-buffer-geometry

**Always use BufferGeometry, never legacy Geometry.**

BufferGeometry is the modern, performant format.

```javascript
// BAD - Legacy Geometry (deprecated, slow)
const geometry = new THREE.Geometry();

// GOOD - BufferGeometry
const geometry = new THREE.BufferGeometry();

// Built-in geometries are already BufferGeometry
const box = new THREE.BoxGeometry(1, 1, 1);
```

### geometry-merge-static

**Merge static geometries to reduce draw calls.**

Each mesh = 1 draw call. Fewer draw calls = better performance.

```javascript
// BAD - 1000 draw calls
for (let i = 0; i < 1000; i++) {
  const mesh = new THREE.Mesh(geometry, material);
  mesh.position.set(i, 0, 0);
  scene.add(mesh);
}

// GOOD - 1 draw call with merged geometry
import { mergeGeometries } from 'three/addons/utils/BufferGeometryUtils.js';

const geometries = [];
for (let i = 0; i < 1000; i++) {
  const geo = geometry.clone();
  geo.translate(i, 0, 0);
  geometries.push(geo);
}

const mergedGeometry = mergeGeometries(geometries);
const mesh = new THREE.Mesh(mergedGeometry, material);
scene.add(mesh);
```

### geometry-instanced-mesh

**Use InstancedMesh for many identical objects.**

Instancing renders multiple copies in a single draw call while allowing individual transforms.

```javascript
// BAD - 10000 individual meshes
for (let i = 0; i < 10000; i++) {
  const mesh = new THREE.Mesh(geometry, material);
  mesh.position.random().multiplyScalar(100);
  scene.add(mesh);
}

// GOOD - Single instanced mesh
const instancedMesh = new THREE.InstancedMesh(geometry, material, 10000);

const dummy = new THREE.Object3D();
for (let i = 0; i < 10000; i++) {
  dummy.position.random().multiplyScalar(100);
  dummy.updateMatrix();
  instancedMesh.setMatrixAt(i, dummy.matrix);
}
instancedMesh.instanceMatrix.needsUpdate = true;
scene.add(instancedMesh);
```

### geometry-lod

**Use Level of Detail (LOD) for complex models.**

Show simpler geometry when objects are far from camera.

```javascript
const lod = new THREE.LOD();

// High detail for close up
lod.addLevel(highDetailMesh, 0);

// Medium detail
lod.addLevel(mediumDetailMesh, 50);

// Low detail for far away
lod.addLevel(lowDetailMesh, 100);

scene.add(lod);
```

### geometry-index-buffer

**Use indexed geometry when possible.**

Indexed geometry shares vertices, reducing memory and improving cache efficiency.

```javascript
// Most built-in geometries are already indexed
const box = new THREE.BoxGeometry(1, 1, 1);
console.log(box.index); // Uint16Array or Uint32Array

// For custom geometry, set index
const indices = new Uint16Array([0, 1, 2, 2, 3, 0]);
geometry.setIndex(new THREE.BufferAttribute(indices, 1));
```

### geometry-vertex-count

**Minimize vertex count - use only what's needed.**

```javascript
// BAD - Too many segments for a simple plane
const plane = new THREE.PlaneGeometry(10, 10, 100, 100); // 10201 vertices

// GOOD - Minimal segments for flat surface
const plane = new THREE.PlaneGeometry(10, 10, 1, 1); // 4 vertices

// Use more segments only when needed (displacement, bending, etc.)
```

### geometry-attributes-typed

**Use appropriate typed arrays for attributes.**

```javascript
// Use Float32Array for positions, normals, uvs
geometry.setAttribute('position', new THREE.Float32BufferAttribute(positions, 3));

// Use Uint8Array or Uint16Array for indices when possible
// Uint16Array supports up to 65535 vertices
geometry.setIndex(new THREE.Uint16BufferAttribute(indices, 1));

// Use Uint8Array for colors if 256 levels is enough
geometry.setAttribute('color', new THREE.Uint8BufferAttribute(colors, 3, true));
```

### geometry-interleaved

**Consider interleaved buffers for better cache performance.**

```javascript
// Interleaved buffer: position + normal + uv per vertex (better cache locality)
const interleavedBuffer = new THREE.InterleavedBuffer(
  new Float32Array([
    // x, y, z, nx, ny, nz, u, v
    0, 0, 0, 0, 1, 0, 0, 0,
    1, 0, 0, 0, 1, 0, 1, 0,
    1, 1, 0, 0, 1, 0, 1, 1,
  ]),
  8 // stride
);

geometry.setAttribute('position', new THREE.InterleavedBufferAttribute(interleavedBuffer, 3, 0));
geometry.setAttribute('normal', new THREE.InterleavedBufferAttribute(interleavedBuffer, 3, 3));
geometry.setAttribute('uv', new THREE.InterleavedBufferAttribute(interleavedBuffer, 2, 6));
```

---

## 4. Material & Texture Optimization (HIGH)

Materials define appearance. Textures are often the largest memory consumers.

### material-reuse

**Reuse materials across multiple meshes.**

Each unique material = additional GPU state changes.

```javascript
// BAD - New material per mesh
meshes.forEach(mesh => {
  mesh.material = new THREE.MeshStandardMaterial({ color: 0xff0000 });
});

// GOOD - Shared material
const sharedMaterial = new THREE.MeshStandardMaterial({ color: 0xff0000 });
meshes.forEach(mesh => {
  mesh.material = sharedMaterial;
});
```

### material-simplest-sufficient

**Use the simplest material that achieves the desired effect.**

```javascript
// Performance (fastest to slowest):
// 1. MeshBasicMaterial - No lighting calculations
// 2. MeshLambertMaterial - Per-vertex lighting
// 3. MeshPhongMaterial - Per-pixel lighting, specular
// 4. MeshStandardMaterial - PBR, most realistic
// 5. MeshPhysicalMaterial - PBR + clearcoat, transmission, etc.

// BAD - Using PBR for a simple unlit object
const material = new THREE.MeshStandardMaterial({ color: 0xff0000 });

// GOOD - MeshBasicMaterial for unlit
const material = new THREE.MeshBasicMaterial({ color: 0xff0000 });
```

### material-texture-size-power-of-two

**Use power-of-two texture dimensions.**

Non-power-of-two textures can't use mipmapping efficiently.

```javascript
// BAD - 1000x600 texture (not power of two)
// GOOD - 1024x512 texture (power of two)

// Power of two: 1, 2, 4, 8, 16, 32, 64, 128, 256, 512, 1024, 2048, 4096
```

### material-texture-compression

**Use compressed texture formats (KTX2/Basis).**

Compressed textures use 4-8x less GPU memory and load faster.

```javascript
import { KTX2Loader } from 'three/addons/loaders/KTX2Loader.js';

const ktx2Loader = new KTX2Loader()
  .setTranscoderPath('basis/')
  .detectSupport(renderer);

ktx2Loader.load('texture.ktx2', (texture) => {
  material.map = texture;
  material.needsUpdate = true;
});
```

### material-texture-mipmaps

**Enable mipmaps for textures viewed at varying distances.**

```javascript
// Mipmaps are ON by default for power-of-two textures
texture.generateMipmaps = true;

// Use appropriate filtering
texture.minFilter = THREE.LinearMipmapLinearFilter; // Smooth
texture.magFilter = THREE.LinearFilter;

// For pixel art or when you want crisp textures
texture.minFilter = THREE.NearestFilter;
texture.magFilter = THREE.NearestFilter;
texture.generateMipmaps = false;
```

### material-texture-anisotropy

**Use anisotropic filtering for floor/wall textures.**

Improves texture quality at oblique angles.

```javascript
const maxAnisotropy = renderer.capabilities.getMaxAnisotropy();
texture.anisotropy = maxAnisotropy; // or a lower value like 4 or 8
```

### material-texture-atlas

**Use texture atlases to reduce texture binds.**

```javascript
// BAD - 10 textures = 10 binds
materials.forEach((mat, i) => {
  mat.map = textures[i];
});

// GOOD - 1 atlas texture, different UV regions
const atlas = textureLoader.load('atlas.png');
materials.forEach((mat, i) => {
  mat.map = atlas;
  // Adjust UVs per material/mesh to show correct region
});
```

### material-avoid-transparency

**Minimize transparent materials - they require sorting and multiple passes.**

```javascript
// BAD - Unnecessary transparency
material.transparent = true;
material.opacity = 1.0; // Fully opaque but still incurs transparency cost

// GOOD - Only use transparency when needed
material.transparent = false;

// If you need transparency, consider:
// - Using alphaTest instead of full transparency
// - Sorting transparent objects manually if depth issues occur
material.alphaTest = 0.5; // Cheaper than full transparency
```

### material-onbeforecompile

**Use onBeforeCompile for shader modifications instead of ShaderMaterial.**

Allows leveraging built-in material features while customizing.

```javascript
const material = new THREE.MeshStandardMaterial();

material.onBeforeCompile = (shader) => {
  shader.uniforms.time = { value: 0 };

  shader.vertexShader = shader.vertexShader.replace(
    '#include <begin_vertex>',
    `
    #include <begin_vertex>
    transformed.y += sin(position.x + time) * 0.5;
    `
  );

  material.userData.shader = shader;
};

// Update in render loop
if (material.userData.shader) {
  material.userData.shader.uniforms.time.value = clock.getElapsedTime();
}
```

---

## 5. Lighting & Shadows (MEDIUM-HIGH)

Lighting is expensive. Shadows are very expensive. Use wisely.

### lighting-limit-lights

**Minimize number of lights, especially dynamic ones.**

```javascript
// BAD - Too many lights
for (let i = 0; i < 20; i++) {
  const light = new THREE.PointLight(0xffffff, 1);
  scene.add(light);
}

// GOOD - Few strategic lights + baked lighting
const ambient = new THREE.AmbientLight(0x404040);
const directional = new THREE.DirectionalLight(0xffffff, 1);
const fill = new THREE.HemisphereLight(0xffffbb, 0x080820, 0.5);
scene.add(ambient, directional, fill);
```

### lighting-bake-static

**Bake lighting for static scenes.**

Pre-computed lightmaps are essentially free at runtime.

```javascript
// Load baked lightmap
const lightMap = textureLoader.load('lightmap.png');
lightMap.channel = 1; // Use second UV set

material.lightMap = lightMap;
material.lightMapIntensity = 1;
```

### lighting-shadow-camera-tight

**Fit shadow camera tightly to the scene.**

Smaller shadow camera = better shadow resolution.

```javascript
// BAD - Huge shadow camera, wasted resolution
directionalLight.shadow.camera.left = -100;
directionalLight.shadow.camera.right = 100;
directionalLight.shadow.camera.top = 100;
directionalLight.shadow.camera.bottom = -100;

// GOOD - Tight fit to visible scene
directionalLight.shadow.camera.left = -10;
directionalLight.shadow.camera.right = 10;
directionalLight.shadow.camera.top = 10;
directionalLight.shadow.camera.bottom = -10;
directionalLight.shadow.camera.updateProjectionMatrix();
```

### lighting-shadow-map-size

**Choose appropriate shadow map resolution.**

```javascript
// Mobile / low-end: 512 or 1024
// Desktop: 1024 or 2048
// High quality: 2048 or 4096

directionalLight.shadow.mapSize.width = 1024;
directionalLight.shadow.mapSize.height = 1024;
```

### lighting-shadow-selective

**Only enable shadows where needed.**

```javascript
// Enable shadows selectively
importantMesh.castShadow = true;
importantMesh.receiveShadow = true;

// Disable for small or distant objects
smallDetail.castShadow = false;
smallDetail.receiveShadow = false;
```

### lighting-shadow-cascade

**Use Cascaded Shadow Maps (CSM) for large scenes.**

```javascript
import { CSM } from 'three/addons/csm/CSM.js';

const csm = new CSM({
  maxFar: 1000,
  cascades: 4,
  shadowMapSize: 1024,
  lightDirection: new THREE.Vector3(-1, -1, -1),
  camera: camera,
  parent: scene
});

// Update in render loop
csm.update();
```

### lighting-probe

**Use Light Probes for efficient environment lighting.**

```javascript
import { LightProbeGenerator } from 'three/addons/lights/LightProbeGenerator.js';

// Generate from environment map
const lightProbe = new THREE.LightProbe();
lightProbe.copy(LightProbeGenerator.fromCubeTexture(envMap));
scene.add(lightProbe);
```

---

## 6. Scene Graph Organization (MEDIUM)

A well-organized scene graph improves both performance and maintainability.

### scene-group-objects

**Use Groups to organize related objects.**

```javascript
// BAD - Flat structure
scene.add(wheel1);
scene.add(wheel2);
scene.add(wheel3);
scene.add(wheel4);
scene.add(carBody);

// GOOD - Hierarchical groups
const car = new THREE.Group();
const wheels = new THREE.Group();

wheels.add(wheel1, wheel2, wheel3, wheel4);
car.add(wheels);
car.add(carBody);

scene.add(car);

// Now you can transform the entire car
car.position.x = 10;
```

### scene-layers

**Use Layers for selective rendering.**

```javascript
// Define layers
const LAYER_DEFAULT = 0;
const LAYER_BLOOM = 1;
const LAYER_UI = 2;

// Assign objects to layers
glowingObject.layers.set(LAYER_BLOOM);
uiElement.layers.set(LAYER_UI);

// Camera sees specific layers
camera.layers.enable(LAYER_DEFAULT);
camera.layers.enable(LAYER_BLOOM);

// Render selectively
camera.layers.set(LAYER_BLOOM);
bloomComposer.render();
camera.layers.set(LAYER_DEFAULT);
renderer.render(scene, camera);
```

### scene-visible-toggle

**Use visible flag instead of add/remove for toggling.**

```javascript
// BAD - Adding/removing is expensive
function toggleObject(show) {
  if (show) {
    scene.add(object);
  } else {
    scene.remove(object);
  }
}

// GOOD - Toggle visibility
function toggleObject(show) {
  object.visible = show;
}
```

### scene-flatten-static

**Flatten static hierarchies for better performance.**

Deep hierarchies require more matrix multiplications.

```javascript
// If a group never transforms, consider flattening
// Before: Group -> Subgroup -> Mesh (3 levels)
// After: Mesh directly in scene (1 level)

// Use for static background elements
staticMesh.matrixAutoUpdate = false;
scene.add(staticMesh);
```

### scene-name-objects

**Name objects for debugging and selection.**

```javascript
mesh.name = 'player';
group.name = 'enemies';

// Find by name
const player = scene.getObjectByName('player');

// Traverse with names
scene.traverse((obj) => {
  if (obj.name.startsWith('enemy_')) {
    // Handle enemy
  }
});
```

### scene-object-pooling

**Use object pooling for frequently created/destroyed objects.**

```javascript
class BulletPool {
  constructor(size) {
    this.pool = [];
    this.active = [];

    const geometry = new THREE.SphereGeometry(0.1);
    const material = new THREE.MeshBasicMaterial({ color: 0xffff00 });

    for (let i = 0; i < size; i++) {
      const bullet = new THREE.Mesh(geometry, material);
      bullet.visible = false;
      this.pool.push(bullet);
      scene.add(bullet);
    }
  }

  spawn(position, velocity) {
    const bullet = this.pool.pop();
    if (bullet) {
      bullet.position.copy(position);
      bullet.userData.velocity = velocity;
      bullet.visible = true;
      this.active.push(bullet);
    }
    return bullet;
  }

  release(bullet) {
    const index = this.active.indexOf(bullet);
    if (index > -1) {
      this.active.splice(index, 1);
      bullet.visible = false;
      this.pool.push(bullet);
    }
  }
}
```

---

## 7. Shader Best Practices (MEDIUM)

Custom shaders offer maximum control but require careful optimization.

### shader-precision

**Use appropriate precision qualifiers.**

```glsl
// Vertex shader - highp is usually fine
precision highp float;

// Fragment shader - mediump often sufficient
precision mediump float;

// Or be specific per variable
highp vec3 position;
mediump vec3 normal;
lowp vec3 color;
```

### shader-avoid-branching

**Minimize branching in shaders.**

GPU parallelism suffers from divergent branches.

```glsl
// BAD - Conditional branching
if (uv.x > 0.5) {
  color = texture2D(tex1, uv);
} else {
  color = texture2D(tex2, uv);
}

// GOOD - Use mix/step
float blend = step(0.5, uv.x);
color = mix(texture2D(tex2, uv), texture2D(tex1, uv), blend);
```

### shader-precompute-cpu

**Precompute values on CPU when possible.**

```javascript
// BAD - Computing in shader every frame
// uniform float time;
// vec3 dir = vec3(cos(time), sin(time), 0.0);

// GOOD - Compute on CPU, pass as uniform
const direction = new THREE.Vector3();
function animate() {
  direction.set(Math.cos(time), Math.sin(time), 0);
  material.uniforms.direction.value.copy(direction);
}
```

### shader-avoid-discard

**Avoid discard when possible - use alpha testing.**

`discard` prevents early-Z optimization.

```glsl
// BAD - Using discard
if (alpha < 0.5) discard;

// GOOD - Use alphaTest on material instead
// material.alphaTest = 0.5;
```

### shader-texture-lod

**Use textureLod for known mip levels.**

```glsl
// When you know the mip level needed
vec4 color = textureLod(sampler, uv, 2.0); // Use mip level 2

// For screen-space effects, calculate LOD explicitly
float lod = log2(max(textureSize.x, textureSize.y) / screenSize);
```

### shader-uniform-arrays

**Prefer uniform arrays over multiple uniforms.**

```glsl
// BAD - Many individual uniforms
uniform vec3 light0Position;
uniform vec3 light1Position;
uniform vec3 light2Position;

// GOOD - Uniform array
uniform vec3 lightPositions[3];
```

### shader-varying-interpolation

**Use flat interpolation when appropriate.**

```glsl
// Smooth interpolation (default)
varying vec3 vNormal;

// Flat interpolation - no interpolation, faster
flat varying int vId;
```

### shader-chunk-injection

**Use Three.js shader chunks for consistency.**

```javascript
// Inject custom code using chunk replacement
material.onBeforeCompile = (shader) => {
  shader.fragmentShader = shader.fragmentShader.replace(
    '#include <output_fragment>',
    `
    #include <output_fragment>
    gl_FragColor.rgb = pow(gl_FragColor.rgb, vec3(1.0/2.2)); // Custom gamma
    `
  );
};
```

---

## 8. Loading & Assets (MEDIUM)

Efficient loading improves user experience and reduces memory usage.

### loading-draco-compression

**Use Draco compression for large meshes.**

Draco can reduce geometry size by 90%+.

```javascript
import { DRACOLoader } from 'three/addons/loaders/DRACOLoader.js';
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';

const dracoLoader = new DRACOLoader();
dracoLoader.setDecoderPath('/draco/');

const gltfLoader = new GLTFLoader();
gltfLoader.setDRACOLoader(dracoLoader);
```

### loading-gltf-preferred

**Use glTF format for 3D models.**

glTF is the "JPEG of 3D" - widely supported, efficient.

```javascript
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';

const loader = new GLTFLoader();
loader.load('model.glb', (gltf) => {
  scene.add(gltf.scene);
});
```

### loading-progress-feedback

**Show loading progress to users.**

```javascript
const manager = new THREE.LoadingManager();

manager.onProgress = (url, loaded, total) => {
  const progress = (loaded / total) * 100;
  progressBar.style.width = `${progress}%`;
};

manager.onLoad = () => {
  hideLoadingScreen();
};

const textureLoader = new THREE.TextureLoader(manager);
const gltfLoader = new GLTFLoader(manager);
```

### loading-async-await

**Use async/await for cleaner loading code.**

```javascript
async function loadAssets() {
  const [model, texture, envMap] = await Promise.all([
    loadGLTF('model.glb'),
    loadTexture('texture.png'),
    loadEnvMap('env.hdr')
  ]);

  return { model, texture, envMap };
}

function loadGLTF(url) {
  return new Promise((resolve, reject) => {
    gltfLoader.load(url, resolve, undefined, reject);
  });
}
```

### loading-lazy

**Lazy load non-critical assets.**

```javascript
// Load essential assets first
await loadCriticalAssets();
startExperience();

// Load non-essential in background
loadBackgroundAssets().then(() => {
  enableOptionalFeatures();
});
```

### loading-cache-assets

**Enable caching and reuse loaded assets.**

```javascript
// Three.js has built-in caching via Cache
THREE.Cache.enabled = true;

// Assets loaded with same URL are retrieved from cache
const texture1 = await loadTexture('shared.png');
const texture2 = await loadTexture('shared.png'); // From cache
```

### loading-dispose-unused

**Unload assets when changing scenes/levels.**

```javascript
function changeLevel(newLevel) {
  // Dispose current level assets
  currentLevel.traverse(disposeObject);
  scene.remove(currentLevel);

  // Load new level
  loadLevel(newLevel).then(level => {
    scene.add(level);
    currentLevel = level;
  });
}
```

---

## 9. Camera & Controls (LOW-MEDIUM)

Camera setup affects both visuals and performance.

### camera-near-far

**Set appropriate near/far planes.**

Tighter near/far improves depth buffer precision.

```javascript
// BAD - Huge range, depth precision issues
camera.near = 0.001;
camera.far = 100000;

// GOOD - Tight range for the actual scene
camera.near = 0.1;
camera.far = 1000;
camera.updateProjectionMatrix();
```

### camera-fov

**Choose appropriate field of view.**

```javascript
// Standard FOV
camera.fov = 75;

// Cinematic (less distortion)
camera.fov = 50;

// Wide angle (more distortion, larger view)
camera.fov = 90;

camera.updateProjectionMatrix();
```

### camera-controls-damping

**Use damping for smooth camera controls.**

```javascript
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;
controls.dampingFactor = 0.05;

// Must call update in render loop
function animate() {
  requestAnimationFrame(animate);
  controls.update(); // Required for damping
  renderer.render(scene, camera);
}
```

### camera-resize-handler

**Handle window resize properly.**

```javascript
function onResize() {
  const width = window.innerWidth;
  const height = window.innerHeight;

  camera.aspect = width / height;
  camera.updateProjectionMatrix();

  renderer.setSize(width, height);
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
}

window.addEventListener('resize', onResize);
```

### camera-orbit-limits

**Set appropriate limits on orbit controls.**

```javascript
controls.minDistance = 2;
controls.maxDistance = 50;

controls.minPolarAngle = 0; // Can look from above
controls.maxPolarAngle = Math.PI / 2; // Can't go below ground

controls.minAzimuthAngle = -Math.PI / 4;
controls.maxAzimuthAngle = Math.PI / 4;
```

---

## 10. Debug & DevTools (LOW)

Debugging tools help identify issues quickly.

### debug-stats

**Use Stats.js for performance monitoring.**

```javascript
import Stats from 'three/addons/libs/stats.module.js';

const stats = new Stats();
document.body.appendChild(stats.dom);

function animate() {
  requestAnimationFrame(animate);
  stats.begin();

  // ... render ...

  stats.end();
}
```

### debug-helpers

**Use built-in helpers during development.**

```javascript
// Axes helper
const axesHelper = new THREE.AxesHelper(5);
scene.add(axesHelper);

// Grid helper
const gridHelper = new THREE.GridHelper(10, 10);
scene.add(gridHelper);

// Camera helper (for debugging shadow cameras, etc.)
const cameraHelper = new THREE.CameraHelper(light.shadow.camera);
scene.add(cameraHelper);

// Box helper
const boxHelper = new THREE.BoxHelper(mesh, 0xffff00);
scene.add(boxHelper);
```

### debug-gui

**Use lil-gui for runtime parameter tweaking.**

```javascript
import GUI from 'lil-gui';

const gui = new GUI();

gui.add(mesh.position, 'x', -10, 10).name('Position X');
gui.add(mesh.position, 'y', -10, 10).name('Position Y');
gui.add(material, 'metalness', 0, 1).name('Metalness');
gui.addColor(material, 'color').name('Color');
```

### debug-renderer-info

**Check renderer info for debugging.**

```javascript
console.log('Renderer Info:', renderer.info);
// Shows: memory (geometries, textures), render (calls, triangles, points, lines)

// Reset counters per frame
renderer.info.reset();
```

### debug-conditional

**Remove debug code in production.**

```javascript
const DEBUG = process.env.NODE_ENV === 'development';

if (DEBUG) {
  scene.add(axesHelper);
  scene.add(gridHelper);
  document.body.appendChild(stats.dom);
}
```

---

## 11. TSL - Three.js Shading Language (MEDIUM)

TSL is the modern approach to shader creation in Three.js. It works with both WebGL and WebGPU backends.

### tsl-why-use

**Use TSL instead of onBeforeCompile hacks for custom materials.**

TSL benefits:
- Works with both WebGL and WebGPU backends
- No string manipulation or onBeforeCompile hacks
- Type-safe, composable shader nodes
- Automatic optimization
- Tree shaking support

```javascript
// OLD - onBeforeCompile (fragile, hard to maintain)
const material = new THREE.MeshStandardMaterial();
material.onBeforeCompile = (shader) => {
  shader.uniforms.detailMap = { value: detailMap };
  shader.fragmentShader = shader.fragmentShader.replace(
    '#include <map_fragment>',
    `#include <map_fragment>
     diffuseColor *= texture2D(detailMap, vMapUv * 10.0);`
  );
};

// NEW - TSL (clean, composable)
import { texture, uv } from 'three/tsl';

const detail = texture(detailMap, uv().mul(10));
const material = new THREE.MeshStandardNodeMaterial();
material.colorNode = texture(colorMap).mul(detail);
```

### tsl-setup-webgpu

**WebGPU setup for TSL.**

```html
<script type="importmap">
{
  "imports": {
    "three": "https://cdn.jsdelivr.net/npm/three@0.182.0/build/three.webgpu.js",
    "three/tsl": "https://cdn.jsdelivr.net/npm/three@0.182.0/build/three.tsl.js",
    "three/addons/": "https://cdn.jsdelivr.net/npm/three@0.182.0/examples/jsm/"
  }
}
</script>
<script type="module">
import * as THREE from 'three';
import { color, positionLocal, sin, time } from 'three/tsl';

const renderer = new THREE.WebGPURenderer({ antialias: true });
await renderer.init();

// Custom TSL material
const material = new THREE.MeshStandardNodeMaterial();
material.colorNode = color(0x00ff00).mul(sin(time).mul(0.5).add(0.5));
</script>
```

### tsl-node-materials

**Use NodeMaterial classes for TSL.**

```javascript
// Standard materials have Node equivalents
import * as THREE from 'three/webgpu';

// Instead of MeshBasicMaterial -> MeshBasicNodeMaterial
// Instead of MeshStandardMaterial -> MeshStandardNodeMaterial
// Instead of MeshPhysicalMaterial -> MeshPhysicalNodeMaterial
// Instead of LineBasicMaterial -> LineBasicNodeMaterial
// Instead of SpriteMaterial -> SpriteNodeMaterial

const material = new THREE.MeshStandardNodeMaterial();
```

### tsl-basic-operations

**TSL basic operations and types.**

```javascript
import {
  float, int, vec2, vec3, vec4, color,  // Types
  uniform, texture, uv,                  // Inputs
  time, deltaTime,                       // Time
  positionLocal, positionWorld,          // Position
  normalLocal, normalWorld,              // Normals
  sin, cos, mix, step, smoothstep        // Math
} from 'three/tsl';

// Constants
const c = color(0xff0000);
const v = vec3(1, 2, 3);

// Uniforms (updatable values)
const myColor = uniform(new THREE.Color(0x0066ff));

// Operations use method chaining
const result = v.mul(2).add(1);  // (v * 2) + 1
const mixed = mix(colorA, colorB, 0.5);

// Swizzling
const xy = v.xy;    // vec2
const zyx = v.zyx;  // vec3 reversed
```

### tsl-material-nodes

**Key material node properties.**

```javascript
const material = new THREE.MeshStandardNodeMaterial();

// Basic
material.colorNode = color(0xff0000);           // Replace color
material.opacityNode = float(0.5);              // Replace opacity
material.positionNode = positionLocal.add(displacement); // Vertex displacement

// PBR
material.metalnessNode = float(0.5);
material.roughnessNode = float(0.5);
material.normalNode = customNormal;
material.emissiveNode = color(0x00ff00);

// Advanced
material.outputNode = customOutput;             // Final output
material.fragmentNode = customFragment;         // Replace fragment shader
```

### tsl-functions

**Creating TSL functions.**

```javascript
import { Fn, vec3, float } from 'three/tsl';

// TSL function with Fn()
const oscSine = Fn(([t = time]) => {
  return t.add(0.75).mul(Math.PI * 2).sin().mul(0.5).add(0.5);
});

// Usage
material.colorNode = vec3(oscSine(), 0, 0);

// Function with object parameters
const customColor = Fn(({ r, g, b }) => {
  return vec3(r, g, b);
});

material.colorNode = customColor({ r: 1, g: 0, b: 0 });
```

### tsl-conditionals

**Conditionals in TSL.**

```javascript
import { Fn, If, select, vec3, float } from 'three/tsl';

// Ternary (inline)
const result = select(value.greaterThan(1), 1.0, value);

// If-else (inside Fn)
const limitedPosition = Fn(({ position }) => {
  const limit = 10;
  const result = vec3(position);

  If(result.y.greaterThan(limit), () => {
    result.y.assign(limit);
  });

  return result;
});

material.positionNode = limitedPosition({ position: positionLocal });
```

### tsl-textures

**Working with textures in TSL.**

```javascript
import { texture, uv, triplanarTexture } from 'three/tsl';

// Basic texture
material.colorNode = texture(myTexture);

// With custom UVs
material.colorNode = texture(myTexture, uv().mul(2));

// Triplanar mapping (no UV seams)
material.colorNode = triplanarTexture(
  textureX, textureY, textureZ,
  float(1),      // scale
  positionLocal,
  normalLocal
);
```

### tsl-post-processing

**TSL post-processing effects.**

```javascript
import {
  pass, gaussianBlur, bloom, grayscale,
  fxaa, dof, ao
} from 'three/tsl';

// Create scene pass
const scenePass = pass(scene, camera);
const beauty = scenePass.getTextureNode();

// Chain effects
postProcessing.outputNode = grayscale(gaussianBlur(beauty, 4));

// Bloom
postProcessing.outputNode = bloom(beauty, 1, 0.4, 0.85);

// Depth of field
const depth = scenePass.getDepthNode();
postProcessing.outputNode = dof(beauty, depth, 0.1, 0.02, 2);
```

### tsl-glsl-to-tsl

**GLSL to TSL translation reference.**

| GLSL | TSL |
|------|-----|
| `position` | `positionGeometry` |
| `transformed` | `positionLocal` |
| `vWorldPosition` | `positionWorld` |
| `vUv` / `uv` | `uv()` |
| `vNormal` | `normalView` |
| `viewMatrix` | `cameraViewMatrix` |
| `modelMatrix` | `modelWorldMatrix` |
| `projectionMatrix` | `cameraProjectionMatrix` |
| `diffuseColor` | `material.colorNode` |
| `gl_FragColor` | `material.fragmentNode` |

---

## Quick Reference Card

### Fundamental (Modern Setup)

- [ ] Use Import Maps, not old CDN scripts
- [ ] Use `renderer.setAnimationLoop()` not manual RAF
- [ ] Choose WebGLRenderer (default) or WebGPURenderer (TSL/compute)
- [ ] Use `setPixelRatio(Math.min(devicePixelRatio, 2))`

### Critical (Always Do)

- [ ] Dispose geometries, materials, textures when removing objects
- [ ] Use delta time for frame-rate independent animation
- [ ] Reuse geometries and materials
- [ ] Use BufferGeometry only

### High Priority

- [ ] Merge static geometries
- [ ] Use InstancedMesh for many identical objects
- [ ] Compress textures (KTX2/Basis)
- [ ] Power-of-two texture dimensions
- [ ] Tight shadow camera bounds
- [ ] Limit light count

### Before Production

- [ ] Remove debug helpers
- [ ] Enable texture compression
- [ ] Verify no memory leaks
- [ ] Test on target devices
- [ ] Profile with Stats.js
