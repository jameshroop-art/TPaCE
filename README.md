# TPaCE

This repository now defines the target specification for a **low-overhead, high-fidelity character and environment system** centered on a dual-skeleton anatomical framework, advanced deformation, local AI material generation, and Vulkan-first runtime performance constraints.

## 1) Anatomical Substructure Assembly Vector Framework

### Dual-Skeleton System

```text
                             [CANONICAL MESH TOPOLOGY]
                                           │
             ┌─────────────────────────────┴─────────────────────────────┐
             ▼                                                           ▼
    [PRIMARY SKELETON]                                         [SECONDARY SKELETON]
      - Pose Kinematics                                          - Anatomical Landmarks
      - Kinematic Chains                                         - Bone Surface Sliding
      - Joint Excursion Limits                                   - Muscle Origin/Insertion
             │                                                           │
             └─────────────────────────────┬─────────────────────────────┘
                                           ▼
                             [RBF DUAL-MATRIX DEFORMATION]
                                           │
             ┌─────────────────────────────┼─────────────────────────────┐
             ▼                             ▼                             ▼
      [TWIST CHAINS]             [XPBD MUSCLE SOLVER]       [PELVIC Q-ANGLE SYSTEM]
        - Radial Distribution       - Volumetric Flexion      - Valgus/Varus Shift
        - Shear Suppression         - Tendon Contraction      - Weight Shift Compensation
```

### Primary Joint Excursion Constraints

- **Axial spine (C1-L5)**
  - Pitch: `[-45.0°, +75.0°]`
  - Yaw: `[-30.0°, +30.0°]`
- **Glenohumeral joint**
  - Elevation: `[-60.0°, +180.0°]`
  - Rotation: `[-90.0°, +90.0°]`
- **Femoroacetabular joint**
  - Flexion/Extension: `[-30.0°, +120.0°]`
  - Abduction/Adduction: `[-45.0°, +45.0°]`

### Secondary Osteological Tracking Constraints

- **Scapular sliding plane**: constrained to thoracic cage ellipsoid via radial distance constraint.
- **Patellar ligament distance**:

  `|P_patella - P_tibial_tuberosity| = L_tendon ± 0.05 mm`

- **Pelvic Q-angle compensation**:

  `θ_q = θ_static + f(Abduction)`

### Canonical Topology Auditing

- **Quads-only target**: `99.85%` quads, `0` non-manifold edges.
- Pole points in non-deforming regions only, max valence `5`.
- Anatomical edge flow:
  - Periorbital/perioral concentric loops.
  - Deltoid-pectoral loops aligned with fiber vectors.
  - Popliteal/antecubital high-density diamond quads for sharp flexion.

## 2) RBF Pose Drivers and Dynamic Tissue Deformation

### RBF Kernel

`Φ(r) = e^{-(εr)^2}`

Where `r` is angular distance in quaternion pose space.

### Twist Distribution

Forearm pronation/supination is distributed over an 8-node chain:

`Δθ_k = θ_total × (k/N)^1.35`

This suppresses candy-wrapper volume collapse.

### XPBD Volume Preservation

`C_vol(V) = (V_current - V_rest) / V_rest = 0`

- Stiffness: `α = 1.0 × 10^-6 Pa^-1`
- Targets: quadriceps, biceps, gastrocnemius node groups.

## 3) Local AI Diffusion + Vision-Language Material Pipeline

```text
+-----------------------------------------------------------------------------------+
|                           DISTROBOX HOST / CONTAINER IPC                          |
+-----------------------------------------------------------------------------------+
| [ENGINE MAIN LOOP]  <--->  [IPC SOCKET / SHM]  <--->  [LOCAL INFERENCE SERVER]   |
|   (C++ / Rust)                                         (sd-cli / llama-cli)      |
|                                                                 │                 |
|  - Frame Synchronization                                        ├─ SD-cli (SDXL) |
|  - Mask Buffer Sharing                                          └─ LLaVA (VLM)   |
|  - Runtime Material Update                                                        |
+-----------------------------------------------------------------------------------+
```

- **Texture output**: `8192x8192` maps for Albedo, Roughness, SSS amount, Normal, Depth, and Micro-Displacement.
- **LLaVA quality audit loop**:
  - Triggered after texture generation or escalation events.
  - Flags seams/artifacts/shadow burn-ins.
  - If seam delta score `> 0.03`, triggers local UV-bounded inpainting via sd-cli.

## 4) 8K PBR Subsurface and Dynamic Wrinkle Modeling

### Optical Parameters

- Absorption `σ_a (mm^-1)`: `[R: 0.015, G: 0.050, B: 0.120]`
- Reduced scattering `σ_s' (mm^-1)`: `[R: 2.10, G: 2.60, B: 3.10]`
- Dual-Gaussian dipole profile with anisotropy `g = 0.82`.

### Wrinkle Tensor

`E = 1/2 (J^T J - I)`

- Compression activates high-frequency wrinkle normals.
- Extension raises roughness and smooths normals for stretch behavior.

## 5) Hair, Cloth, and Environmental State Dynamics

### Hair (Cosserat Rod)

```text
                         [GLOBAL WEATHER STACK]
                                   │
               ┌───────────────────┴───────────────────┐
               ▼                                       ▼
      [WETNESS & SATURATION]                  [TEMPERATURE & WIND]
        - Capillary Forces                      - Aerodynamic Drag
        - Strand Attraction                     - Ice Crystal Shell
        - Mass Scaling (+60%)                   - Rigidity Adjustments
               │                                       │
               └───────────────────┬───────────────────┘
                                   ▼
                      [COSSERAT ROD SOLVER NODE]
                                   │
               ┌───────────────────┼───────────────────┐
               ▼                   ▼                   ▼
       [STRAND CLUMPING]   [KINEMATIC FRICTION] [AERODYNAMIC DRAG]
```

- `r(s,t)`, `q(s,t)` state variables.
- `E = 3.5 GPa`, `G = 1.2 GPa`.
- `ρ_dry = 1.3 g/cm^3`, `ρ_wet = 2.08 g/cm^3` (+60%).
- Capillary force:

  `F_capillary = -κ × (γ_water × cos(θ_c) / d^2) × n_(i,j)`

  with `γ_water = 0.0728 N/m`.

### Cloth XPBD/Chaos Coupling

- Mass scaling:

  `M_cloth(u,v) = M_dry(u,v) × (1 + 1.25·Wetness_Severity + 3.50·Mud_Severity)`

- Bending stiffness:

  `S_bend(u,v) = S_base × (1 - 0.4·Wetness + 12·Freeze_Severity)`

## 6) 0-100 Severity Progression State Machine

```text
PRISTINE -> EXPOSURE -> ACCUMULATION -> RECOVERY -> PRISTINE
```

### Phase Equations

- Exposure:

  `dE_i/dt = ContactRate_i × (1 - CoverageMask) × max(0, V_rel · N_surface)`

- Accumulation:

  `A_i(t) = A_i(t-Δt) + (dE_i/dt × AdhesionCoeff_i × OrientationFactor)Δt`

- Recovery/Decay:

  `Decay_i = (BaseEvaporate_i·f(Temp,Humidity) + WashoffRate·Immersion_Depth + SheddingRate·||V||)Δt`

  `A_i(t+Δt) = max(0, A_i(t) - Decay_i)`

### Ladder Thresholds

- `0-10%`: trace exposure priming.
- `10-25%`: early visible darkening and sheen.
- `25-50%`: flow maps + particle splatter.
- `50-75%`: clumping + solver stiffness changes.
- `75-100%`: dripping emitters, saturation sticking, optional freeze masks.

## 7) MaterialX Interchange and Engine Adapters

```text
                     [MATERIALX MASTER GRAPH]
                                │
      ┌─────────────────────────┴─────────────────────────┐
      ▼                                                   ▼
[UE 5.8 ADAPTER PASS]                            [CUSTOM C++ / VULKAN]
  - Material Parameter Collections                - Uniform Buffer Objects
  - RVT Render Target Links                       - Push Constants
  - Niagara Particle Emitters                     - Direct GPU Buffer Mutations
```

- MaterialX is source of truth.
- Compile targets: HLSL (UE 5.8), GLSL (Vulkan), native compute paths.
- Dynamic accumulation masks stream to per-character `256x256 RGBA` RVT.

## 8) Severity-to-LOD Activation and Performance Targets

### Activation Matrix (Summary)

- `0-20%`: full shading only.
- `20-40%`: light particles.
- `40-60%`: medium particles + interaction params.
- `60-80%`: heavy particles + solver modifications.
- `80-100%`: severe particles + full physical stack.

LOD behavior freezes higher-cost systems beyond LOD 0 at higher severities.

### Per-Character Budget

- C++ environmental logic: `0.02 ms` CPU
- RVT mask pass: `0.03 ms` GPU
- PBR surface compute: `0.03 ms` GPU
- Niagara emitters: `0.04 ms` GPU
- XPBD/Chaos solvers: `0.05 ms` CPU
- **Total**: `0.17 ms` (~2% of `8.33 ms` @ 120 FPS)

## 9) Low-Overhead Linux/Vulkan Runtime Architecture

```text
                        [ENGINE MAIN THREAD (C++)]
                                     │
                                     ▼
                        [JOB SYSTEM (RUST WORKERS)]
                                     │
             ┌───────────────────────┼───────────────────────┐
             ▼                       ▼                       ▼
     [VISIBILITY BUFFER]   [GPU MESH CLUSTER CULLING] [SOFTWARE RASTERIZER]
       - 64-bit IDs          - HW Mesh Shaders          - Sub-pixel Micro-tris
       - Single Material     - Frustum/Occlusion        - Nanite-like stream
             │                       │                       │
             └───────────────────────┼───────────────────────┘
                                     ▼
                        [VULKAN 1.3 RENDER ENGINE]
                                     │
             ┌───────────────────────┴───────────────────────┐
             ▼                                               ▼
      [DLSS 5 NEURAL GENERATION]                 [LUMEN-LIKE HYBRID RT]
```

- Uses `VK_EXT_mesh_shader` meshlets (`64 vertices / 128 primitives`).
- 64-bit visibility target: `[32-bit Meshlet ID | 7-bit Primitive ID | 25-bit Barycentrics]`.
- Hybrid RT via `VK_KHR_ray_tracing_pipeline`.
- DLSS 5 consumes low-res color, motion vectors, depth, normal, and material classification buffers.

## 10) Local Zero-Copy AI Execution Loop

```text
                      [RUST ZERO-COPY MEMORY PIPELINE]
                                      │
          ┌───────────────────────────┴───────────────────────────┐
          ▼                                                       ▼
 [SHM SHARED BUFFERS]                                   [UNIX IPC SOCKETS]
   - Direct Pointer Mapping                              - Fast Message Passing
          │                                                       │
          └───────────────────────────┬───────────────────────────┘
                                      ▼
                         [SD-CLI / LLAMA-CLI INFERENCE]
```

Workflow:

1. Runtime mesh/region update triggers generation.
2. llama-cli forms context prompts from biome + severity + anatomy.
3. sd-cli writes generated maps to shared memory.
4. LLaVA audits and correction loops feed back before Vulkan hot-reload.

---

This README is the canonical target specification for implementing the system described in this repository.
