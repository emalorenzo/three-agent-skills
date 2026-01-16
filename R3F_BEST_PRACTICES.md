# React Three Fiber Best Practices

Comprehensive guide for React Three Fiber (R3F) and the Poimandres ecosystem (@react-three/fiber, @react-three/drei, @react-three/postprocessing, @react-three/rapier, zustand, leva).

## Rule Categories by Priority

| Priority | Category | Impact | Prefix |
|----------|----------|--------|--------|
| 1 | Performance & Re-renders | CRITICAL | `perf-` |
| 2 | useFrame & Animation | CRITICAL | `frame-` |
| 3 | Component Patterns | HIGH | `component-` |
| 4 | Canvas & Setup | HIGH | `canvas-` |
| 5 | Drei Helpers | MEDIUM-HIGH | `drei-` |
| 6 | Loading & Suspense | MEDIUM-HIGH | `loading-` |
| 7 | State Management | MEDIUM | `state-` |
| 8 | Events & Interaction | MEDIUM | `events-` |
| 9 | Post-processing | MEDIUM | `postpro-` |
| 10 | Physics (Rapier) | LOW-MEDIUM | `physics-` |

---

## 1. Performance & Re-renders (CRITICAL)

React re-renders are the #1 performance killer in R3F. The render loop runs at 60fps - React reconciliation must not interfere.

### perf-never-set-state-in-useframe

**NEVER call setState inside useFrame.**

This triggers React re-renders 60 times per second, destroying performance.

```jsx
// BAD - Causes 60 re-renders per second
function BadComponent() {
  const [position, setPosition] = useState(0);

  useFrame(() => {
    setPosition(p => p + 0.01); // NEVER DO THIS
  });

  return <mesh position-x={position} />;
}

// GOOD - Mutate refs directly
function GoodComponent() {
  const meshRef = useRef();

  useFrame(() => {
    meshRef.current.position.x += 0.01;
  });

  return <mesh ref={meshRef} />;
}
```

### perf-isolate-state

**Isolate components that need React state.**

Components with state re-render. Keep them small and separate from 3D objects.

```jsx
// BAD - Entire scene re-renders when score changes
function Game() {
  const [score, setScore] = useState(0);

  return (
    <>
      <HUD score={score} />
      <Player />
      <Enemies count={100} /> {/* Re-renders when score changes! */}
      <Environment />
    </>
  );
}

// GOOD - Only HUD re-renders
function Game() {
  return (
    <>
      <HUD /> {/* Manages its own state or uses store */}
      <Player />
      <Enemies count={100} />
      <Environment />
    </>
  );
}

function HUD() {
  const score = useGameStore(state => state.score);
  return <Html><div>{score}</div></Html>;
}
```

### perf-zustand-selectors

**Use Zustand selectors to minimize re-renders.**

Subscribe only to the specific state slices you need.

```jsx
// BAD - Re-renders on ANY store change
function BadComponent() {
  const store = useGameStore(); // Subscribes to entire store
  return <mesh position-x={store.playerX} />;
}

// GOOD - Re-renders only when playerX changes
function GoodComponent() {
  const playerX = useGameStore(state => state.playerX);
  return <mesh position-x={playerX} />;
}

// BETTER - Use shallow equality for objects
import { shallow } from 'zustand/shallow';

function BetterComponent() {
  const { x, y, z } = useGameStore(
    state => ({ x: state.x, y: state.y, z: state.z }),
    shallow
  );
  return <mesh position={[x, y, z]} />;
}
```

### perf-transient-subscriptions

**Use transient subscriptions for continuous values.**

Zustand's subscribe function allows reading state without causing re-renders.

```jsx
// GOOD - No re-renders, direct mutation
function Player() {
  const meshRef = useRef();

  useEffect(() => {
    // Transient subscription - no re-renders
    const unsubscribe = useGameStore.subscribe(
      state => state.playerPosition,
      position => {
        meshRef.current.position.copy(position);
      }
    );
    return unsubscribe;
  }, []);

  return <mesh ref={meshRef} />;
}

// ALTERNATIVE - Use getState() in useFrame
function Player() {
  const meshRef = useRef();

  useFrame(() => {
    const { playerPosition } = useGameStore.getState();
    meshRef.current.position.copy(playerPosition);
  });

  return <mesh ref={meshRef} />;
}
```

### perf-memo-components

**Memoize expensive components.**

```jsx
// GOOD - Prevents re-renders if props haven't changed
const ExpensiveModel = memo(function ExpensiveModel({ url }) {
  const { scene } = useGLTF(url);
  return <primitive object={scene.clone()} />;
});

// With custom comparison
const OptimizedMesh = memo(
  function OptimizedMesh({ position, color }) {
    return (
      <mesh position={position}>
        <boxGeometry />
        <meshStandardMaterial color={color} />
      </mesh>
    );
  },
  (prev, next) => {
    return prev.color === next.color &&
           prev.position[0] === next.position[0] &&
           prev.position[1] === next.position[1] &&
           prev.position[2] === next.position[2];
  }
);
```

### perf-keys-for-lists

**Use stable keys for dynamic lists.**

```jsx
// BAD - Index as key causes unnecessary remounts
function Particles({ particles }) {
  return particles.map((p, i) => (
    <Particle key={i} position={p.position} /> // Index changes when array changes
  ));
}

// GOOD - Stable unique ID
function Particles({ particles }) {
  return particles.map(p => (
    <Particle key={p.id} position={p.position} />
  ));
}
```

### perf-avoid-inline-objects

**Avoid creating new objects/arrays in JSX.**

New references trigger re-renders and prop comparisons.

```jsx
// BAD - New array created every render
function BadMesh() {
  return <mesh position={[1, 2, 3]} />; // New array each render
}

// GOOD - Stable reference
const POSITION = [1, 2, 3];
function GoodMesh() {
  return <mesh position={POSITION} />;
}

// GOOD - Using individual props
function GoodMesh() {
  return <mesh position-x={1} position-y={2} position-z={3} />;
}

// GOOD - useMemo for computed values
function ComputedMesh({ x }) {
  const position = useMemo(() => [x, x * 2, x * 3], [x]);
  return <mesh position={position} />;
}
```

### perf-dispose-auto

**R3F auto-disposes by default - understand when to disable.**

R3F automatically disposes geometries, materials, and textures when components unmount.

```jsx
// Auto-dispose is ON by default (good!)
function MyMesh() {
  return (
    <mesh>
      <boxGeometry /> {/* Auto-disposed on unmount */}
      <meshStandardMaterial /> {/* Auto-disposed on unmount */}
    </mesh>
  );
}

// Disable for shared/reused resources
const sharedGeometry = new THREE.BoxGeometry();

function ReusedMesh() {
  return (
    <mesh geometry={sharedGeometry}>
      <meshStandardMaterial dispose={null} /> {/* Manual disposal */}
    </mesh>
  );
}

// Disable on primitive for external objects
function ImportedModel({ object }) {
  return <primitive object={object} dispose={null} />;
}
```

---

## 2. useFrame & Animation (CRITICAL)

useFrame is R3F's render loop hook. Misuse causes performance disasters.

### frame-priority

**Use priority parameter for execution order.**

Lower numbers run first. Default is 0.

```jsx
// Physics runs first, then animation, then camera
function Physics() {
  useFrame(() => {
    world.step();
  }, -100); // Runs first
}

function Animation() {
  useFrame(() => {
    updateAnimations();
  }, 0); // Default priority
}

function CameraFollow() {
  useFrame(() => {
    camera.lookAt(target);
  }, 100); // Runs last
}
```

### frame-delta-time

**Always use delta for frame-rate independent animation.**

```jsx
// BAD - Speed varies with frame rate
useFrame(() => {
  ref.current.rotation.y += 0.01;
});

// GOOD - Consistent speed
useFrame((state, delta) => {
  ref.current.rotation.y += 1 * delta; // 1 radian per second
});

// GOOD - Using clock for time-based effects
useFrame(({ clock }) => {
  ref.current.position.y = Math.sin(clock.elapsedTime) * 2;
});
```

### frame-conditional-subscription

**Disable useFrame when not needed.**

```jsx
// GOOD - Active only when needed
function OptionalAnimation({ active }) {
  useFrame(() => {
    // Animation logic
  }, active ? 0 : null); // null disables the subscription
}

// Alternative with early return
function ConditionalAnimation({ paused }) {
  useFrame((state, delta) => {
    if (paused) return;
    // Animation logic
  });
}
```

### frame-destructure-state

**Destructure only what you need from state.**

```jsx
// Use what you need
useFrame(({ clock, camera, pointer }) => {
  // clock.elapsedTime, clock.getDelta()
  // camera - the default camera
  // pointer - normalized mouse position (-1 to 1)
});

// Full state object contents:
// gl, scene, camera, raycaster, pointer, mouse, clock,
// viewport, size, set, get, invalidate, advance, events
```

### frame-render-on-demand

**Use invalidate() for on-demand rendering.**

```jsx
// Enable frameloop="demand" on Canvas
<Canvas frameloop="demand">
  <Scene />
</Canvas>

// Manually request render when needed
function OnDemandAnimation() {
  const { invalidate } = useThree();

  const handleClick = () => {
    // Trigger single render
    invalidate();
  };

  // Or invalidate in response to external events
  useEffect(() => {
    const unsubscribe = someStore.subscribe(() => {
      invalidate();
    });
    return unsubscribe;
  }, [invalidate]);
}
```

### frame-avoid-heavy-computation

**Move heavy computations outside useFrame.**

```jsx
// BAD - Expensive calculation every frame
useFrame(() => {
  const expensiveResult = heavyComputation(); // 60x per second!
  ref.current.position.copy(expensiveResult);
});

// GOOD - Compute less frequently
function SmartComponent() {
  const [targetPosition, setTargetPosition] = useState(new THREE.Vector3());

  // Update target occasionally
  useEffect(() => {
    const interval = setInterval(() => {
      setTargetPosition(heavyComputation());
    }, 100); // 10 times per second
    return () => clearInterval(interval);
  }, []);

  // Lerp to target every frame (cheap)
  useFrame(() => {
    ref.current.position.lerp(targetPosition, 0.1);
  });
}
```

---

## 3. Component Patterns (HIGH)

R3F's declarative nature requires specific patterns for Three.js integration.

### component-jsx-elements

**Use JSX elements for Three.js objects.**

R3F automatically creates Three.js objects from lowercase JSX elements.

```jsx
// JSX elements map to THREE constructors
<mesh />           // new THREE.Mesh()
<boxGeometry />    // new THREE.BoxGeometry()
<meshStandardMaterial /> // new THREE.MeshStandardMaterial()
<ambientLight />   // new THREE.AmbientLight()
<group />          // new THREE.Group()

// Constructor arguments via args prop
<boxGeometry args={[2, 2, 2]} /> // new THREE.BoxGeometry(2, 2, 2)
<sphereGeometry args={[1, 32, 32]} />
<planeGeometry args={[10, 10, 1, 1]} />
```

### component-attach-prop

**Use attach for non-standard parent properties.**

```jsx
// Geometry and material auto-attach
<mesh>
  <boxGeometry />        {/* attach="geometry" (automatic) */}
  <meshStandardMaterial /> {/* attach="material" (automatic) */}
</mesh>

// Explicit attach for other properties
<mesh>
  <boxGeometry attach="geometry" />
  <meshStandardMaterial attach="material" />
</mesh>

// Array attachment with attach function
<mesh>
  <meshStandardMaterial attach={(parent, self) => {
    parent.material = [self]; // or parent.material.push(self)
    return () => { /* cleanup */ };
  }} />
</mesh>

// Shadow map attachment
<directionalLight>
  <orthographicCamera attach="shadow-camera" args={[-10, 10, 10, -10]} />
</directionalLight>
```

### component-primitive

**Use primitive for existing Three.js objects.**

```jsx
// For objects created outside React
const geometry = new THREE.BoxGeometry();
const material = new THREE.MeshStandardMaterial();
const mesh = new THREE.Mesh(geometry, material);

function ExternalObject() {
  return <primitive object={mesh} position={[0, 1, 0]} />;
}

// For loaded models
function Model() {
  const { scene } = useGLTF('/model.glb');
  return <primitive object={scene} />;
}

// Clone if reusing
function ReusableModel() {
  const { scene } = useGLTF('/model.glb');
  return <primitive object={scene.clone()} />;
}
```

### component-extend

**Use extend() for custom Three.js classes.**

```jsx
import { extend } from '@react-three/fiber';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';

// Register once at module level
extend({ OrbitControls, EffectComposer });

// Now use as JSX
function Scene() {
  const { camera, gl } = useThree();
  return (
    <>
      <orbitControls args={[camera, gl.domElement]} />
    </>
  );
}
```

### component-forwardref

**Use forwardRef for reusable components that need refs.**

```jsx
const Box = forwardRef(function Box({ color = 'orange', ...props }, ref) {
  return (
    <mesh ref={ref} {...props}>
      <boxGeometry />
      <meshStandardMaterial color={color} />
    </mesh>
  );
});

// Usage
function Scene() {
  const boxRef = useRef();

  useFrame(() => {
    boxRef.current.rotation.x += 0.01;
  });

  return <Box ref={boxRef} position={[0, 1, 0]} color="blue" />;
}
```

### component-dispose-null

**Set dispose={null} on shared or external resources.**

```jsx
// Shared geometry across components
const sharedGeo = new THREE.SphereGeometry(1, 32, 32);

function SharedSphere({ position }) {
  return (
    <mesh position={position} geometry={sharedGeo}>
      <meshStandardMaterial />
    </mesh>
  );
}

// External/loaded objects
function LoadedModel({ model }) {
  return <primitive object={model} dispose={null} />;
}
```

---

## 4. Canvas & Setup (HIGH)

Proper Canvas configuration is foundational.

### canvas-size-container

**Canvas fills its parent container.**

```jsx
// CSS - Parent must have dimensions
.canvas-container {
  width: 100%;
  height: 100vh;
  /* or specific dimensions */
  width: 800px;
  height: 600px;
}

// JSX
<div className="canvas-container">
  <Canvas>
    <Scene />
  </Canvas>
</div>
```

### canvas-camera-default

**Configure default camera via camera prop.**

```jsx
// Perspective camera (default)
<Canvas camera={{ position: [0, 5, 10], fov: 75, near: 0.1, far: 1000 }}>

// Orthographic camera
<Canvas orthographic camera={{ zoom: 50, position: [0, 0, 100] }}>

// Make camera manual (don't respond to resize)
<Canvas camera={{ manual: true }}>
```

### canvas-gl-config

**Configure WebGL context via gl prop.**

```jsx
<Canvas
  gl={{
    antialias: true,
    alpha: false,
    powerPreference: 'high-performance',
    stencil: false,
    depth: true,
  }}
  // Pixel ratio
  dpr={[1, 2]} // min 1, max 2
>
```

### canvas-shadows

**Enable shadows at Canvas level.**

```jsx
// Basic shadows
<Canvas shadows>

// Soft shadows (PCFSoft)
<Canvas shadows="soft">

// Custom shadow map type
<Canvas shadows={{ type: THREE.PCFSoftShadowMap }}>

// Then on lights and meshes
<directionalLight castShadow shadow-mapSize={[2048, 2048]} />
<mesh castShadow receiveShadow>
```

### canvas-frameloop

**Choose appropriate frameloop mode.**

```jsx
// Always render (default)
<Canvas frameloop="always">

// Render on demand - call invalidate() to render
<Canvas frameloop="demand">

// Never auto-render - call advance() manually
<Canvas frameloop="never">
```

### canvas-events

**Configure event handling.**

```jsx
<Canvas
  events={undefined} // Use default DOM events
  eventSource={document.body} // Event source element
  eventPrefix="offset" // 'offset' | 'client' | 'page' | 'layer' | 'screen'
>
```

### canvas-linear-flat

**Use linear/flat for correct color output.**

```jsx
// For physically correct lighting (sRGB workflow)
<Canvas linear flat>
  {/* Scene renders in linear color space */}
  {/* Apply tonemapping in post-processing */}
</Canvas>

// linear: Uses LinearSRGBColorSpace for textures
// flat: Disables automatic tonemapping
```

---

## 5. Drei Helpers (MEDIUM-HIGH)

@react-three/drei provides essential abstractions. Use them correctly.

### drei-use-gltf

**Use useGLTF for model loading with preloading.**

```jsx
import { useGLTF } from '@react-three/drei';

function Model() {
  const { scene, nodes, materials } = useGLTF('/model.glb');

  return <primitive object={scene} />;
}

// Preload for instant loading
useGLTF.preload('/model.glb');

// With Draco compression
function DracoModel() {
  const { scene } = useGLTF('/model.glb', '/draco/');
  return <primitive object={scene} />;
}
```

### drei-use-texture

**Use useTexture for texture loading.**

```jsx
import { useTexture } from '@react-three/drei';

function TexturedMesh() {
  // Single texture
  const texture = useTexture('/texture.png');

  // Multiple textures
  const [colorMap, normalMap, roughnessMap] = useTexture([
    '/color.png',
    '/normal.png',
    '/roughness.png'
  ]);

  // Object form
  const textures = useTexture({
    map: '/color.png',
    normalMap: '/normal.png',
    roughnessMap: '/roughness.png'
  });

  return (
    <mesh>
      <boxGeometry />
      <meshStandardMaterial {...textures} />
    </mesh>
  );
}

// Preload
useTexture.preload('/texture.png');
```

### drei-environment

**Use Environment for realistic lighting.**

```jsx
import { Environment } from '@react-three/drei';

// Preset environments
<Environment preset="city" /> // apartment, city, dawn, forest, lobby, night, park, studio, sunset, warehouse

// Custom HDR
<Environment files="/env.hdr" />

// Background visible
<Environment background preset="sunset" />

// Ground projection
<Environment ground={{ height: 15, radius: 60 }} preset="city" />
```

### drei-orbit-controls

**Use OrbitControls from Drei (not manual extend).**

```jsx
import { OrbitControls } from '@react-three/drei';

function Scene() {
  return (
    <>
      <OrbitControls
        enableDamping
        dampingFactor={0.05}
        minDistance={2}
        maxDistance={50}
        maxPolarAngle={Math.PI / 2}
      />
      <mesh>
        <boxGeometry />
        <meshStandardMaterial />
      </mesh>
    </>
  );
}
```

### drei-html

**Use Html for DOM overlays in 3D space.**

```jsx
import { Html } from '@react-three/drei';

function Label({ position, text }) {
  return (
    <Html
      position={position}
      center // Center the HTML element
      distanceFactor={10} // Scale with distance
      occlude // Hide when behind objects
      transform // Use CSS3D transforms
    >
      <div className="label">{text}</div>
    </Html>
  );
}
```

### drei-text

**Use Text for 3D text (troika-three-text).**

```jsx
import { Text, Text3D } from '@react-three/drei';

// 2D text in 3D space (SDF, very performant)
<Text
  color="black"
  fontSize={1}
  maxWidth={10}
  lineHeight={1}
  letterSpacing={0}
  textAlign="center"
  font="/fonts/Inter-Bold.woff"
  anchorX="center"
  anchorY="middle"
>
  Hello World
</Text>

// 3D extruded text
<Text3D
  font="/fonts/helvetiker_regular.typeface.json"
  size={1}
  height={0.2}
  bevelEnabled
  bevelSize={0.02}
>
  Hello
  <meshStandardMaterial color="orange" />
</Text3D>
```

### drei-instances

**Use Instances for optimized instancing.**

```jsx
import { Instances, Instance } from '@react-three/drei';

function Particles({ count = 1000 }) {
  return (
    <Instances limit={count} range={count}>
      <boxGeometry args={[0.1, 0.1, 0.1]} />
      <meshStandardMaterial />

      {Array.from({ length: count }, (_, i) => (
        <Instance
          key={i}
          position={[Math.random() * 10, Math.random() * 10, Math.random() * 10]}
          rotation={[Math.random(), Math.random(), 0]}
          color={`hsl(${Math.random() * 360}, 100%, 50%)`}
        />
      ))}
    </Instances>
  );
}
```

### drei-use-helper

**Use useHelper for debug visualization.**

```jsx
import { useHelper } from '@react-three/drei';
import { DirectionalLightHelper, BoxHelper } from 'three';

function DebugLight() {
  const lightRef = useRef();
  useHelper(lightRef, DirectionalLightHelper, 1, 'red');

  return <directionalLight ref={lightRef} />;
}

function DebugMesh() {
  const meshRef = useRef();
  useHelper(meshRef, BoxHelper, 'blue');

  return (
    <mesh ref={meshRef}>
      <boxGeometry />
      <meshStandardMaterial />
    </mesh>
  );
}
```

### drei-bounds

**Use Bounds to fit camera to objects.**

```jsx
import { Bounds, useBounds } from '@react-three/drei';

function FitToView() {
  return (
    <Bounds fit clip observe margin={1.2}>
      <Model />
    </Bounds>
  );
}

// Manual control
function ManualBounds() {
  const bounds = useBounds();

  return (
    <mesh onClick={() => bounds.refresh().fit()}>
      <boxGeometry />
      <meshStandardMaterial />
    </mesh>
  );
}
```

### drei-center

**Use Center to center objects.**

```jsx
import { Center } from '@react-three/drei';

function CenteredModel() {
  return (
    <Center top> {/* Align to top */}
      <Model />
    </Center>
  );
}

// Options: top, bottom, left, right, front, back
// Or precise: <Center precise>
```

### drei-float

**Use Float for floating animation.**

```jsx
import { Float } from '@react-three/drei';

<Float
  speed={1}
  rotationIntensity={1}
  floatIntensity={1}
  floatingRange={[-0.1, 0.1]}
>
  <mesh>
    <boxGeometry />
    <meshStandardMaterial />
  </mesh>
</Float>
```

---

## 6. Loading & Suspense (MEDIUM-HIGH)

R3F integrates with React Suspense for loading states.

### loading-suspense

**Wrap async components in Suspense.**

```jsx
import { Suspense } from 'react';

function App() {
  return (
    <Canvas>
      <Suspense fallback={<LoadingFallback />}>
        <Model /> {/* Uses useGLTF internally */}
      </Suspense>
    </Canvas>
  );
}

function LoadingFallback() {
  return (
    <mesh>
      <boxGeometry />
      <meshBasicMaterial wireframe />
    </mesh>
  );
}
```

### loading-preload

**Preload assets outside render cycle.**

```jsx
import { useGLTF, useTexture } from '@react-three/drei';

// Preload at module level
useGLTF.preload('/model.glb');
useTexture.preload('/texture.png');

// Or preload multiple
useGLTF.preload(['/model1.glb', '/model2.glb']);
```

### loading-use-progress

**Use useProgress for loading progress UI.**

```jsx
import { useProgress, Html } from '@react-three/drei';

function Loader() {
  const { active, progress, errors, item, loaded, total } = useProgress();

  return (
    <Html center>
      <div className="loader">
        {progress.toFixed(0)}% loaded
        <br />
        Loading: {item}
      </div>
    </Html>
  );
}

function App() {
  return (
    <Canvas>
      <Suspense fallback={<Loader />}>
        <Scene />
      </Suspense>
    </Canvas>
  );
}
```

### loading-lazy-components

**Lazy load heavy scene components.**

```jsx
import { lazy, Suspense } from 'react';

const HeavyScene = lazy(() => import('./HeavyScene'));

function App() {
  const [showHeavy, setShowHeavy] = useState(false);

  return (
    <Canvas>
      <BasicScene />
      {showHeavy && (
        <Suspense fallback={null}>
          <HeavyScene />
        </Suspense>
      )}
    </Canvas>
  );
}
```

### loading-error-boundary

**Handle loading errors with error boundaries.**

```jsx
import { ErrorBoundary } from 'react-error-boundary';

function ModelErrorFallback({ error }) {
  return (
    <mesh>
      <boxGeometry />
      <meshBasicMaterial color="red" />
    </mesh>
  );
}

function SafeModel({ url }) {
  return (
    <ErrorBoundary FallbackComponent={ModelErrorFallback}>
      <Suspense fallback={null}>
        <Model url={url} />
      </Suspense>
    </ErrorBoundary>
  );
}
```

---

## 7. State Management (MEDIUM)

Zustand is the recommended state manager for R3F.

### state-zustand-store

**Create focused stores for game/app state.**

```jsx
import { create } from 'zustand';

const useGameStore = create((set, get) => ({
  // State
  score: 0,
  health: 100,
  position: new THREE.Vector3(),

  // Actions
  increaseScore: (amount) => set(state => ({ score: state.score + amount })),
  damage: (amount) => set(state => ({ health: Math.max(0, state.health - amount) })),
  setPosition: (pos) => set({ position: pos }),

  // Computed (use get() for derived state in actions)
  reset: () => set({ score: 0, health: 100, position: new THREE.Vector3() }),
}));
```

### state-avoid-objects-in-store

**Avoid storing Three.js objects directly in Zustand (or be careful).**

Three.js objects are mutable and don't trigger re-renders on internal changes.

```jsx
// PROBLEMATIC - Mutating Vector3 won't trigger re-renders
const useStore = create(set => ({
  position: new THREE.Vector3(), // Object reference stays same
  updatePosition: (x, y, z) => {
    const pos = get().position;
    pos.set(x, y, z); // Mutation, no re-render!
    set({ position: pos }); // Same reference, might not trigger
  }
}));

// BETTER - Store primitives
const useStore = create(set => ({
  positionX: 0,
  positionY: 0,
  positionZ: 0,
  setPosition: (x, y, z) => set({ positionX: x, positionY: y, positionZ: z })
}));

// OR - Create new object reference
const useStore = create(set => ({
  position: new THREE.Vector3(),
  setPosition: (x, y, z) => set({ position: new THREE.Vector3(x, y, z) })
}));
```

### state-subscribeWithSelector

**Use subscribeWithSelector for fine-grained subscriptions.**

```jsx
import { create } from 'zustand';
import { subscribeWithSelector } from 'zustand/middleware';

const useStore = create(
  subscribeWithSelector((set) => ({
    score: 0,
    combo: 0,
    // ...
  }))
);

// Subscribe to specific changes
useEffect(() => {
  const unsub = useStore.subscribe(
    state => state.score,
    (score, prevScore) => {
      console.log('Score changed from', prevScore, 'to', score);
      playScoreSound();
    }
  );
  return unsub;
}, []);
```

### state-persist

**Persist state across sessions when needed.**

```jsx
import { persist } from 'zustand/middleware';

const useSettingsStore = create(
  persist(
    (set) => ({
      musicVolume: 0.8,
      sfxVolume: 1.0,
      setMusicVolume: (vol) => set({ musicVolume: vol }),
    }),
    {
      name: 'game-settings',
    }
  )
);
```

### state-separate-concerns

**Separate stores by concern.**

```jsx
// Game state
const useGameStore = create((set) => ({
  score: 0,
  level: 1,
  // ...
}));

// Player state
const usePlayerStore = create((set) => ({
  health: 100,
  inventory: [],
  // ...
}));

// UI state
const useUIStore = create((set) => ({
  isPaused: false,
  showInventory: false,
  // ...
}));
```

---

## 8. Events & Interaction (MEDIUM)

R3F provides a powerful event system built on react-three/fiber's raycaster.

### events-pointer-events

**Use pointer events on meshes.**

```jsx
function InteractiveMesh() {
  const [hovered, setHovered] = useState(false);

  return (
    <mesh
      onClick={(e) => {
        e.stopPropagation(); // Prevent event bubbling
        console.log('Clicked!', e.point); // World position
      }}
      onPointerOver={(e) => setHovered(true)}
      onPointerOut={(e) => setHovered(false)}
      onPointerMove={(e) => console.log(e.point)}
      onPointerDown={(e) => console.log('Down')}
      onPointerUp={(e) => console.log('Up')}
    >
      <boxGeometry />
      <meshStandardMaterial color={hovered ? 'hotpink' : 'orange'} />
    </mesh>
  );
}
```

### events-stop-propagation

**Use stopPropagation to prevent event bubbling.**

```jsx
// Events bubble through the scene graph
function Parent() {
  return (
    <group onClick={() => console.log('Parent clicked')}>
      <Child />
    </group>
  );
}

function Child() {
  return (
    <mesh onClick={(e) => {
      e.stopPropagation(); // Parent won't receive click
      console.log('Child clicked');
    }}>
      <boxGeometry />
      <meshStandardMaterial />
    </mesh>
  );
}
```

### events-cursor-pointer

**Change cursor on hover.**

```jsx
function HoverCursor() {
  const [hovered, setHovered] = useState(false);

  // Change cursor
  useEffect(() => {
    document.body.style.cursor = hovered ? 'pointer' : 'auto';
    return () => { document.body.style.cursor = 'auto'; };
  }, [hovered]);

  return (
    <mesh
      onPointerOver={() => setHovered(true)}
      onPointerOut={() => setHovered(false)}
    >
      <boxGeometry />
      <meshStandardMaterial />
    </mesh>
  );
}

// Or use useCursor from Drei
import { useCursor } from '@react-three/drei';

function HoverCursorDrei() {
  const [hovered, setHovered] = useState(false);
  useCursor(hovered);

  return (
    <mesh
      onPointerOver={() => setHovered(true)}
      onPointerOut={() => setHovered(false)}
    >
      <boxGeometry />
      <meshStandardMaterial />
    </mesh>
  );
}
```

### events-raycast-filter

**Filter raycasting with raycast prop or layers.**

```jsx
// Disable raycasting entirely
<mesh raycast={() => null}>
  <boxGeometry />
  <meshStandardMaterial />
</mesh>

// Use layers for selective raycasting
function SelectiveRaycast() {
  const meshRef = useRef();

  useEffect(() => {
    meshRef.current.layers.set(1); // Only layer 1
  }, []);

  return <mesh ref={meshRef} />;
}
```

### events-event-data

**Understand event data structure.**

```jsx
<mesh onClick={(event) => {
  // event contains:
  event.object;      // The mesh that was clicked
  event.eventObject; // The object that has the event handler
  event.point;       // World position of click (Vector3)
  event.distance;    // Distance from camera
  event.ray;         // The ray used for intersection
  event.camera;      // Camera used for raycasting
  event.uv;          // UV coordinates at intersection
  event.face;        // Face that was hit
  event.faceIndex;   // Index of hit face
  event.delta;       // Distance traveled since pointer down (for drag detection)
  event.stopPropagation(); // Stop bubbling
  event.nativeEvent; // Original DOM event
}}>
```

---

## 9. Post-processing (MEDIUM)

@react-three/postprocessing provides efficient post-processing effects.

### postpro-effect-composer

**Use EffectComposer from @react-three/postprocessing.**

```jsx
import { EffectComposer, Bloom, Vignette } from '@react-three/postprocessing';

function Scene() {
  return (
    <>
      <mesh>
        <boxGeometry />
        <meshStandardMaterial emissive="orange" emissiveIntensity={2} />
      </mesh>

      <EffectComposer>
        <Bloom luminanceThreshold={0.2} luminanceSmoothing={0.9} intensity={1} />
        <Vignette eskil={false} offset={0.1} darkness={1.1} />
      </EffectComposer>
    </>
  );
}
```

### postpro-common-effects

**Common post-processing effects.**

```jsx
import {
  EffectComposer,
  Bloom,
  ChromaticAberration,
  DepthOfField,
  Noise,
  Vignette,
  SMAA,
  ToneMapping,
  HueSaturation,
  BrightnessContrast,
  SSAO,
  Outline,
  SelectiveBloom
} from '@react-three/postprocessing';

// Bloom for glow
<Bloom intensity={1} luminanceThreshold={0.5} luminanceSmoothing={0.9} />

// Depth of Field
<DepthOfField focusDistance={0} focalLength={0.02} bokehScale={2} />

// Anti-aliasing
<SMAA />

// Screen-space ambient occlusion
<SSAO samples={31} radius={0.1} intensity={20} />

// Outline selected objects
<Outline selection={selectedRef} edgeStrength={3} />
```

### postpro-selective-bloom

**Use SelectiveBloom for optimized glow.**

```jsx
import { SelectiveBloom } from '@react-three/postprocessing';

function Scene() {
  const glowRef = useRef();

  return (
    <>
      {/* This mesh will glow */}
      <mesh ref={glowRef}>
        <sphereGeometry />
        <meshStandardMaterial emissive="orange" />
      </mesh>

      {/* This mesh won't glow */}
      <mesh position={[2, 0, 0]}>
        <boxGeometry />
        <meshStandardMaterial color="blue" />
      </mesh>

      <EffectComposer>
        <SelectiveBloom
          selection={glowRef}
          intensity={2}
          luminanceThreshold={0.1}
        />
      </EffectComposer>
    </>
  );
}
```

### postpro-custom-shader

**Create custom effects with shaders.**

```jsx
import { Effect } from 'postprocessing';
import { forwardRef, useMemo } from 'react';
import { Uniform } from 'three';

const fragmentShader = `
  uniform float intensity;

  void mainImage(const in vec4 inputColor, const in vec2 uv, out vec4 outputColor) {
    outputColor = vec4(inputColor.rgb * intensity, inputColor.a);
  }
`;

class CustomEffectImpl extends Effect {
  constructor({ intensity = 1.0 }) {
    super('CustomEffect', fragmentShader, {
      uniforms: new Map([['intensity', new Uniform(intensity)]])
    });
  }
}

const CustomEffect = forwardRef(({ intensity }, ref) => {
  const effect = useMemo(() => new CustomEffectImpl({ intensity }), [intensity]);
  return <primitive ref={ref} object={effect} />;
});

// Usage
<EffectComposer>
  <CustomEffect intensity={0.5} />
</EffectComposer>
```

### postpro-performance

**Optimize post-processing performance.**

```jsx
<EffectComposer
  multisampling={0} // Disable MSAA (use SMAA instead)
  frameBufferType={THREE.HalfFloatType} // Use half-float for better performance
>
  <SMAA /> {/* Cheaper than MSAA */}
  <Bloom mipmapBlur /> {/* mipmapBlur is more efficient */}
</EffectComposer>
```

---

## 10. Physics (Rapier) (LOW-MEDIUM)

@react-three/rapier provides physics simulation.

### physics-setup

**Basic physics setup with Rapier.**

```jsx
import { Physics, RigidBody } from '@react-three/rapier';

function Scene() {
  return (
    <Physics gravity={[0, -9.81, 0]} debug>
      {/* Dynamic body */}
      <RigidBody>
        <mesh>
          <boxGeometry />
          <meshStandardMaterial />
        </mesh>
      </RigidBody>

      {/* Static ground */}
      <RigidBody type="fixed">
        <mesh position={[0, -2, 0]}>
          <boxGeometry args={[10, 0.5, 10]} />
          <meshStandardMaterial />
        </mesh>
      </RigidBody>
    </Physics>
  );
}
```

### physics-body-types

**Understand RigidBody types.**

```jsx
// Dynamic - Affected by forces and collisions
<RigidBody type="dynamic">
  <mesh />
</RigidBody>

// Fixed - Immovable, infinite mass
<RigidBody type="fixed">
  <mesh />
</RigidBody>

// Kinematic - Moved programmatically, not by physics
<RigidBody type="kinematicPosition">
  <mesh />
</RigidBody>

// Kinematic velocity-based
<RigidBody type="kinematicVelocity">
  <mesh />
</RigidBody>
```

### physics-colliders

**Use appropriate colliders.**

```jsx
// Auto-collider from mesh
<RigidBody colliders="hull"> {/* convex hull */}
  <mesh />
</RigidBody>

<RigidBody colliders="cuboid"> {/* box */}
  <mesh />
</RigidBody>

<RigidBody colliders="ball"> {/* sphere */}
  <mesh />
</RigidBody>

<RigidBody colliders="trimesh"> {/* exact mesh shape (expensive) */}
  <mesh />
</RigidBody>

// Manual colliders
import { CuboidCollider, BallCollider, CapsuleCollider } from '@react-three/rapier';

<RigidBody colliders={false}>
  <CuboidCollider args={[0.5, 0.5, 0.5]} />
  <mesh>
    <boxGeometry />
    <meshStandardMaterial />
  </mesh>
</RigidBody>
```

### physics-events

**Handle collision events.**

```jsx
<RigidBody
  onCollisionEnter={({ manifold, target, other }) => {
    console.log('Collision with', other.rigidBodyObject.name);
    console.log('Contact point', manifold.solverContactPoint(0));
  }}
  onCollisionExit={({ target, other }) => {
    console.log('Collision ended with', other.rigidBodyObject.name);
  }}
  onIntersectionEnter={({ target, other }) => {
    // For sensor colliders
    console.log('Entered trigger zone');
  }}
>
  <mesh name="player">
    <boxGeometry />
    <meshStandardMaterial />
  </mesh>
</RigidBody>
```

### physics-api-ref

**Use ref for physics API access.**

```jsx
import { RigidBody } from '@react-three/rapier';

function Player() {
  const rigidBodyRef = useRef();

  const jump = () => {
    rigidBodyRef.current.applyImpulse({ x: 0, y: 10, z: 0 }, true);
  };

  const move = (direction) => {
    rigidBodyRef.current.setLinvel({ x: direction.x * 5, y: 0, z: direction.z * 5 });
  };

  useFrame(() => {
    const position = rigidBodyRef.current.translation();
    const velocity = rigidBodyRef.current.linvel();
    // Use position and velocity
  });

  return (
    <RigidBody ref={rigidBodyRef}>
      <mesh>
        <capsuleGeometry args={[0.5, 1]} />
        <meshStandardMaterial />
      </mesh>
    </RigidBody>
  );
}
```

### physics-performance

**Optimize physics performance.**

```jsx
<Physics
  gravity={[0, -9.81, 0]}
  timeStep="vary" // or fixed like 1/60
  updatePriority={-50} // Run before rendering
  interpolate={true} // Smooth motion between physics steps
>
  {/* Use simple colliders instead of trimesh */}
  <RigidBody colliders="cuboid">
    <mesh />
  </RigidBody>

  {/* Disable collision detection when not needed */}
  <RigidBody sensor>
    <CuboidCollider args={[5, 5, 5]} />
  </RigidBody>
</Physics>
```

---

## 11. Leva (Debug GUI)

Leva provides a modern GUI for tweaking parameters.

### leva-basic

**Basic Leva usage.**

```jsx
import { useControls } from 'leva';

function ControlledMesh() {
  const { position, color, wireframe } = useControls({
    position: { value: [0, 0, 0], step: 0.1 },
    color: '#ff0000',
    wireframe: false
  });

  return (
    <mesh position={position}>
      <boxGeometry />
      <meshStandardMaterial color={color} wireframe={wireframe} />
    </mesh>
  );
}
```

### leva-folders

**Organize controls with folders.**

```jsx
const { ambient, directional } = useControls('Lighting', {
  ambient: { value: 0.5, min: 0, max: 1, step: 0.1 },
  directional: { value: 1, min: 0, max: 2, step: 0.1 }
});

// Nested folders
const materialProps = useControls('Material', {
  color: '#ffffff',

  'PBR Properties': folder({
    metalness: { value: 0.5, min: 0, max: 1 },
    roughness: { value: 0.5, min: 0, max: 1 }
  })
});
```

### leva-conditional

**Hide Leva in production.**

```jsx
import { Leva } from 'leva';

function App() {
  return (
    <>
      <Leva hidden={process.env.NODE_ENV === 'production'} />
      <Canvas>
        <Scene />
      </Canvas>
    </>
  );
}
```

---

## Quick Reference Card

### Critical (Always Do)

- [ ] NEVER use setState in useFrame
- [ ] Use Zustand selectors (not entire store)
- [ ] Use refs for animation, not state
- [ ] Avoid inline objects/arrays in JSX
- [ ] Use delta time for animations
- [ ] Wrap async components in Suspense

### High Priority

- [ ] Memoize expensive components
- [ ] Use stable keys for dynamic lists
- [ ] Preload assets with useGLTF.preload
- [ ] Configure Canvas shadows properly
- [ ] Use dispose={null} for shared resources

### Poimandres Ecosystem

- [ ] Drei: useGLTF, useTexture, Environment, OrbitControls
- [ ] Zustand: Selectors, transient subscriptions
- [ ] Postprocessing: EffectComposer with SMAA
- [ ] Rapier: Simple colliders, collision events
- [ ] Leva: Hidden in production

### Common Patterns

```jsx
// Animation without re-renders
const ref = useRef();
useFrame((state, delta) => {
  ref.current.rotation.y += delta;
});

// Zustand without re-renders
useFrame(() => {
  const { value } = useStore.getState();
  ref.current.position.x = value;
});

// Event handling
<mesh
  onClick={(e) => {
    e.stopPropagation();
    handleClick(e.point);
  }}
/>
```
