# Three Agent Skills

AI coding agent skills for Three.js and React Three Fiber development.

Skills follow the [Agent Skills](https://agentskills.io/) specification.

## Installation

```bash
npx add-skill three-agent-skills
```

Or from GitHub directly:

```bash
npx add-skill emalorenzo/three-agent-skills
```

### List available skills

```bash
npx add-skill three-agent-skills --list
```

### Install specific skill

```bash
npx add-skill three-agent-skills --skill three-best-practices
npx add-skill three-agent-skills --skill r3f-best-practices
```

## Available Skills

### three-best-practices

Pure Three.js optimization guidelines. 100+ rules across 17 categories.

**Use when:**
- Writing Three.js code
- Optimizing WebGL/WebGPU performance
- Managing memory and disposal
- Writing shaders (GLSL or TSL)
- Building VR/AR experiences with WebXR
- Integrating physics engines
- Optimizing for mobile devices

**Categories:**
- Modern Setup & Imports (Import Maps, renderers)
- Memory Management & Dispose (CRITICAL)
- Render Loop Optimization (CRITICAL)
- Geometry & Buffer Management
- Material & Texture Optimization
- Lighting & Shadows
- Scene Graph Organization
- Shader Best Practices (GLSL)
- TSL - Three.js Shading Language (complete reference, post-processing, compute shaders)
- Loading & Assets (GLTF, DRACO, Meshopt, KTX2)
- Camera & Controls
- Animation System (Mixer, blending, morph targets, skeletal)
- Physics Integration (Rapier, Cannon-es)
- WebXR / VR / AR
- Audio (Spatial, HRTF)
- Mobile Optimization
- Production (Error handling, migration checklist)
- Debug & DevTools

### r3f-best-practices

React Three Fiber and Poimandres ecosystem best practices. 60+ rules across 11 categories.

**Use when:**
- Writing R3F components
- Optimizing R3F performance (re-renders)
- Using Drei helpers
- Managing state with Zustand
- Implementing physics or post-processing

**Ecosystem coverage:**
- @react-three/fiber
- @react-three/drei
- @react-three/postprocessing
- @react-three/rapier
- zustand
- leva

**Categories:**
- Performance & Re-renders (CRITICAL)
- useFrame & Animation (CRITICAL)
- Component Patterns
- Canvas & Setup
- Drei Helpers
- Loading & Suspense
- State Management
- Events & Interaction
- Post-processing
- Physics (Rapier)
- Leva (Debug GUI)

## Usage Examples

Once installed, the agent uses skills automatically based on context:

```
Review my entire Three.js app and give me a score from 1 to 100.
Explain what's good, what needs improvement, and provide actionable fixes.
```

```
Help me set up a Three.js scene with proper memory management
```

```
Review my R3F component for performance issues
```

```
How do I load GLTF models with Drei?
```

```
Optimize my shader for mobile devices
```

```
Set up WebXR for my VR experience
```

```
Implement GPU particles with compute shaders
```

```
Integrate Rapier physics with my Three.js scene
```

## Supported Agents

Works with any agent that supports the [Agent Skills](https://agentskills.io/) spec:

- Claude Code
- Cursor
- Codex
- OpenCode
- VS Code Copilot
- And more...

## Documentation

- Full Three.js guide: [THREE_BEST_PRACTICES.md](./THREE_BEST_PRACTICES.md)
- Full R3F guide: [R3F_BEST_PRACTICES.md](./R3F_BEST_PRACTICES.md)
- Creating skills: [CLAUDE.md](./CLAUDE.md)

## License

MIT
