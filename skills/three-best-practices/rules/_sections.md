# Rule Sections

## Priority 0: Modern Setup & Imports (FUNDAMENTAL)
- setup-use-import-maps
- setup-choose-renderer
- setup-animation-loop
- setup-basic-scene-template

## Priority 1: Memory Management & Dispose (CRITICAL)
- memory-dispose-geometry
- memory-dispose-material
- memory-dispose-textures
- memory-dispose-render-targets
- memory-dispose-recursive
- memory-dispose-on-unmount
- memory-renderer-dispose
- memory-reuse-objects

## Priority 2: Render Loop Optimization (CRITICAL)
- render-single-raf
- render-conditional
- render-delta-time
- render-avoid-allocations
- render-cache-computations
- render-frustum-culling
- render-update-matrix-manual
- render-pixel-ratio
- render-antialias-wisely

## Priority 3: Geometry & Buffer Management (HIGH)
- geometry-buffer-geometry
- geometry-merge-static
- geometry-instanced-mesh
- geometry-lod
- geometry-index-buffer
- geometry-vertex-count
- geometry-attributes-typed
- geometry-interleaved

## Priority 4: Material & Texture Optimization (HIGH)
- material-reuse
- material-simplest-sufficient
- material-texture-size-power-of-two
- material-texture-compression
- material-texture-mipmaps
- material-texture-anisotropy
- material-texture-atlas
- material-avoid-transparency
- material-onbeforecompile

## Priority 5: Lighting & Shadows (MEDIUM-HIGH)
- lighting-limit-lights
- lighting-bake-static
- lighting-shadow-camera-tight
- lighting-shadow-map-size
- lighting-shadow-selective
- lighting-shadow-cascade
- lighting-probe

## Priority 6: Scene Graph Organization (MEDIUM)
- scene-group-objects
- scene-layers
- scene-visible-toggle
- scene-flatten-static
- scene-name-objects
- scene-object-pooling

## Priority 7: Shader Best Practices GLSL (MEDIUM)
- shader-precision
- shader-avoid-branching
- shader-precompute-cpu
- shader-avoid-discard
- shader-texture-lod
- shader-uniform-arrays
- shader-varying-interpolation
- shader-chunk-injection

## Priority 8: TSL - Three.js Shading Language (MEDIUM)
- tsl-why-use
- tsl-setup-webgpu
- tsl-complete-reference
- tsl-material-slots
- tsl-node-materials
- tsl-basic-operations
- tsl-material-nodes
- tsl-functions
- tsl-conditionals
- tsl-textures
- tsl-post-processing
- tsl-compute-shaders
- tsl-glsl-to-tsl

## Priority 9: Loading & Assets (MEDIUM)
- loading-draco-compression
- loading-gltf-preferred
- gltf-loading-optimization
- loading-progress-feedback
- loading-async-await
- loading-lazy
- loading-cache-assets
- loading-dispose-unused

## Priority 10: Camera & Controls (LOW-MEDIUM)
- camera-near-far
- camera-fov
- camera-controls-damping
- camera-resize-handler
- camera-orbit-limits

## Priority 11: Animation System (MEDIUM)
- animation-system

## Priority 12: Physics Integration (MEDIUM)
- physics-integration

## Priority 13: WebXR / VR / AR (MEDIUM)
- webxr-setup

## Priority 14: Audio (LOW-MEDIUM)
- audio-spatial

## Priority 15: Optimization (HIGH)
- mobile-optimization
- raycasting-optimization

## Priority 16: Production (HIGH)
- error-handling-recovery
- migration-checklist

## Priority 17: Debug & DevTools (LOW)
- debug-stats
- debug-helpers
- debug-gui
- debug-renderer-info
- debug-conditional
