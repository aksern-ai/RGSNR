# Renderer-Guided Sparse Neural Rendering

## Abstract

Modern neural rendering techniques such as super-resolution and frame generation attempt to reconstruct visual information that would otherwise require additional rendering computation. However, conventional approaches generally operate after the rendering pipeline has already produced a complete frame, leaving the game engine unaware of which parts of the image could safely be reconstructed.

This paper proposes **Renderer-Guided Sparse Neural Rendering (RGSNR)**, a rendering architecture in which the game engine explicitly communicates spatial rendering requirements to a neural reconstruction system. Instead of fully rendering every pixel of every frame, the renderer produces a sparse set of high-confidence regions, together with auxiliary information such as depth, motion vectors, object identifiers, and material information. A neural renderer reconstructs the remaining regions.

The central hypothesis is that game engines possess semantic and geometric information that can substantially reduce the uncertainty of neural reconstruction. By allowing the renderer to decide *where exact rendering is necessary* and *where neural reconstruction is acceptable*, RGSNR could potentially reduce rendering cost while maintaining perceptual quality.

---

## 1. Introduction

Real-time graphics traditionally follow a simple principle:

> if a pixel is visible, calculate it.

This approach is predictable and highly controllable, but expensive. Modern rendering pipelines increasingly rely on techniques such as temporal upscaling, ray tracing, variable rate shading, and frame generation to improve the performance-quality tradeoff.

Neural frame generation typically works by generating intermediate frames from previously rendered frames and motion information. Although this can significantly increase displayed frame rates, generated frames may contain artifacts in regions with complex motion, disocclusion, thin geometry, particles, or other difficult visual structures.

The problem is that the neural reconstruction system usually receives the final rendered image and auxiliary buffers, while the game engine itself already possesses significantly more information about the scene.

For example, the renderer knows:

* which objects are visible;
* object geometry;
* depth;
* motion vectors;
* material properties;
* object identifiers;
* camera movement;
* visibility changes;
* newly exposed regions;
* UI and overlay regions;
* regions requiring exact rendering.

This motivates a different approach.

Instead of asking:

> "How can a neural network reconstruct an entire frame?"

we ask:

> "Which parts of the frame actually need to be rendered exactly?"

---

## 2. Renderer-Guided Rendering

RGSNR divides each target frame into two categories:

1. **Explicitly rendered regions**
2. **Neurally reconstructed regions**

The game engine generates a spatial importance mask:

```text
R(x, y) ∈ {RENDER, RECONSTRUCT}
```

The mask may be binary initially, but a future implementation could use continuous importance values:

```text
R(x, y) ∈ [0, 1]
```

where higher values indicate that exact rendering is more important.

A simplified pipeline is:

```text
             Scene
               │
               ▼
        Game Engine / Renderer
               │
       ┌───────┴────────┐
       │                │
       ▼                ▼
Importance Mask     Scene Metadata
       │                │
       └───────┬────────┘
               ▼
       Sparse Rendering
               │
               ▼
      Partially Rendered Frame
               │
               ▼
       Neural Reconstruction
               │
               ▼
          Final Frame
```

The important distinction is that the neural network is not independently deciding what should exist.

The renderer provides structural constraints.

---

## 3. Sparse Rendering

Instead of rendering the entire frame, the GPU may render only selected regions.

For example:

```text
████████████████
██····██······██
██····██······██
████████████████
······████······
······████······
```

Here:

* `█` = explicitly rendered pixels or regions
* `·` = pixels reconstructed by the neural renderer

The exact spatial representation does not have to be individual pixels. Practical implementations could operate on tiles, blocks, variable-rate shading regions, or other hardware-supported primitives.

For example, a renderer might choose:

```text
128 × 128 frame

32 × 32 tiles

████████
██····██
██····██
████████
```

rather than constructing a completely irregular per-pixel rendering mask.

---

## 4. Why the Game Engine Should Participate

A conventional post-processing neural network must infer scene structure from the rendered output.

A game engine does not have this limitation.

Consider a character's hand moving rapidly across the screen.

A conventional neural reconstruction system might observe:

```text
previous frame: hand at X
current frame:  hand at Y
```

and infer the intermediate state.

The renderer can instead provide:

```text
object_id = CHARACTER_HAND
motion_vector = ...
depth = ...
geometry = ...
visibility = ...
material = ...
```

The neural renderer therefore receives information that is unavailable from RGB pixels alone.

This could be particularly useful for:

* disocclusion;
* thin geometry;
* particles;
* transparent objects;
* rapidly moving objects;
* reflections;
* foliage;
* complex lighting;
* camera cuts.

The engine could simply declare certain regions as mandatory exact rendering:

```text
REGION 0:
    mode = EXACT

REGION 1:
    mode = NEURAL

REGION 2:
    mode = EXACT

REGION 3:
    mode = NEURAL
```

This turns the neural renderer into a cooperative component of the rendering pipeline rather than a black-box post-processing effect.

---

## 5. Adaptive Rendering Budget

The rendering mask can also be dynamically adjusted.

For a visually simple scene:

```text
exact rendering:       30%
neural reconstruction: 70%
```

For a complex scene:

```text
exact rendering:       70%
neural reconstruction: 30%
```

For extremely difficult scenes:

```text
exact rendering:       100%
neural reconstruction:  0%
```

This allows the system to degrade gracefully.

Instead of forcing the neural renderer to reconstruct difficult content, the engine can simply spend additional rendering budget where necessary.

---

## 6. Temporal Reconstruction

RGSNR naturally extends to temporal rendering.

The neural renderer may receive:

```text
Frame(t-1)
Frame(t)
SparseRender(t+1)

Depth(t)
MotionVectors(t)
ObjectIDs(t)
ImportanceMask(t+1)
```

and produce:

```text
Frame(t+1)
```

The difference from conventional frame generation is that the target frame is not necessarily generated entirely from previous frames.

A portion of it already exists as newly rendered information.

The network therefore performs **completion rather than pure frame synthesis**.

Conceptually:

```text
FULL FRAME GENERATION

A ──────► Neural Network ──────► B


SPARSE NEURAL RENDERING

A ──────────────┐
                │
Sparse B ───────┼──► Neural Network ──► Complete B
                │
Scene metadata ─┘
```

---

## 7. Importance Mask Generation

The importance mask could initially be generated using explicit engine heuristics.

For example:

```text
importance =
    geometry_complexity
  + motion_magnitude
  + depth_discontinuity
  + object_priority
  + visibility_change
  + temporal_instability
```

Objects could additionally expose rendering priorities:

```cpp
renderer.set_neural_priority(object, HIGH);
renderer.set_neural_priority(object, LOW);
```

A future system could learn the mask itself.

This creates a two-stage optimization problem:

```text
Scene
  │
  ▼
Importance Predictor
  │
  ▼
Rendering Allocation
  │
  ├── Exact rendering
  │
  └── Neural reconstruction
```

The objective would be to minimize rendering cost while keeping reconstruction error below a perceptual threshold.

---

## 8. Optimization Objective

A simplified objective can be expressed as:

```text
minimize:

    C_render(M) + λ C_neural(M)

subject to:

    D(F_generated, F_ground_truth) < ε
```

where:

* `M` is the rendering mask;
* `C_render` is the cost of explicitly rendering selected regions;
* `C_neural` is the cost of neural reconstruction;
* `D` is a visual or perceptual distortion metric;
* `ε` is the maximum acceptable reconstruction error;
* `λ` controls the relative computational cost.

The system therefore searches for the cheapest combination of exact rendering and neural reconstruction that remains visually acceptable.

---

## 9. Potential Hardware Implementation

RGSNR could potentially integrate with existing GPU concepts such as:

* variable rate shading;
* tile-based rendering;
* compute shaders;
* tensor/matrix acceleration;
* motion vector generation;
* hardware ray tracing;
* temporal accumulation.

A hypothetical pipeline could look like:

```text
Raster / RT
    │
    ├── exact tiles ──────────────┐
    │                             │
    └── metadata ───────────────┐ │
                                ▼ ▼
                         Neural Reconstruction
                                │
                                ▼
                           Final Frame
```

Future GPU architectures could theoretically expose hardware support for sparse neural reconstruction directly inside the graphics pipeline.

---

## 10. Advantages

The proposed architecture has several potential advantages over purely post-process neural frame generation.

### 10.1 Better scene awareness

The engine can explicitly communicate geometry, visibility, depth, and object information.

### 10.2 Adaptive computational cost

Rendering effort can be concentrated on perceptually important regions.

### 10.3 Reduced hallucination

The neural network does not need to synthesize information that the renderer could cheaply provide directly.

### 10.4 Graceful degradation

If reconstruction confidence becomes low, the renderer can simply increase the exact-rendering region.

### 10.5 Engine-specific optimization

Game engines can expose semantic information unavailable to generic image-processing systems.

---

## 11. Limitations

RGSNR is not expected to eliminate the fundamental challenges of neural rendering.

Potential problems include:

* neural inference latency;
* temporal instability;
* disocclusion;
* transparency;
* particles;
* highly detailed textures;
* reflections;
* reconstruction errors;
* memory bandwidth;
* synchronization between graphics and neural workloads.

There is also a major engineering challenge: existing game engines and graphics APIs are not designed around this architecture.

A practical implementation would require cooperation between:

```text
Game Engine
      +
Graphics API
      +
GPU Driver
      +
GPU Hardware
      +
Neural Reconstruction Model
```

This makes RGSNR significantly more difficult to deploy than a conventional post-processing technique.

---

## 12. Experimental Proposal

A prototype implementation could be developed using an existing game engine or custom renderer.

The experiment would compare:

1. Full-resolution rendering
2. Conventional temporal upscaling
3. Conventional frame generation
4. Sparse rendering without neural reconstruction
5. Renderer-Guided Sparse Neural Rendering

Metrics should include:

* FPS;
* GPU utilization;
* rendering time;
* neural inference time;
* power consumption;
* VRAM usage;
* PSNR;
* SSIM;
* LPIPS;
* temporal stability;
* perceptual quality.

An especially important experiment would measure the relationship between:

```text
percentage of explicitly rendered pixels
          ↓
reconstruction quality
          ↓
total rendering cost
```

This would determine whether sparse rendering actually provides a useful quality/performance tradeoff.

---

## 13. Future Work

Several extensions are possible.

### Learned importance masks

A neural network could predict which regions should be rendered exactly.

### Multi-level reconstruction

Instead of only:

```text
EXACT / NEURAL
```

the system could support:

```text
EXACT
HIGH QUALITY NEURAL
LOW QUALITY NEURAL
TEMPORAL REUSE
```

### Semantic rendering priorities

Game engines could mark objects according to importance:

```text
player       = 1.0
weapon       = 0.9
UI           = 1.0
background   = 0.3
far foliage  = 0.1
```

### Hardware acceleration

Future GPUs could expose native sparse neural rendering primitives.

### Joint renderer-model optimization

The renderer and neural model could be trained together so that the neural network learns specifically which information the renderer can cheaply provide.

---

## 14. Conclusion

Renderer-Guided Sparse Neural Rendering proposes a shift in how neural rendering is integrated into real-time graphics.

Instead of treating neural reconstruction as a post-processing step applied after a complete frame has been rendered, the proposed architecture allows the game engine to participate directly in deciding what should be rendered exactly and what can be reconstructed.

The key observation is simple:

> **the renderer already knows what is happening in the scene.**

A neural network does not necessarily need to reconstruct an entire frame if the graphics pipeline can provide a sparse set of trustworthy visual information and explicit scene metadata.

This could potentially enable a new class of rendering systems in which GPU computation is allocated dynamically according to visual importance rather than uniformly across every pixel.

The long-term goal is not simply to generate more frames.

It is to **avoid rendering pixels that do not need to be rendered in the first place.**
