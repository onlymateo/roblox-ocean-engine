# Roblox water-simulation limitations

**Research date: 2026-11-19.** Roblox Studio 0.748.

This report describes what a high-fidelity, real-time, interactive ocean needs from a game engine, and how much of it Roblox exposes to experience developers. It covers:

- surface simulation
- shading
- foam and spray
- interactive waves
- underwater rendering
- buoyancy
- tooling

## How to read this report

Each statement carries a tag:

| Tag | Meaning |
| --- | --- |
| **[Doc]** | Documented fact, quoted or paraphrased from an official Roblox page or announcement listed in [Sources](#sources) |
| **[Test]** | Observed by running code in Roblox Studio 0.748 on 2026-11-19 (Edit mode or play test). Behaviour in live clients or on other platforms can differ |
| **[Forum]** | Reported by developers on the Roblox Developer Forum; not an official statement |
| **[Assessment]** | Reasoning from the facts above |

Each limitation is classified as one of:

| Category | Meaning |
| --- | --- |
| **Unsupported** | The capability does not exist for experience developers |
| **Insufficiently exposed** | It exists inside the engine, but scripts cannot control it to the degree required |
| **Impractical for performance** | It can be written in Luau, but the cost at the required resolution or rate is too high for a real-time budget |
| **Possible only with compromises** | It can be achieved, with a loss of fidelity, resolution, range or flexibility |

Each limitation follows the same structure: **Required → Exposed → Missing → Why it matters → Consequence.**

- "Missing" statements without a tag are assessments of absence: nothing in the API reference provides the capability on the research date.
- "Consequence" describes the effect on what can be achieved, not a recommended implementation.

## Summary

| # | Limitation | Category |
| --- | --- | --- |
| 1 | Programmable shading (vertex and pixel programs) | Unsupported |
| 2 | GPU compute and render-to-texture | Unsupported |
| 3 | Per-frame geometry from scripts (EditableMesh) | Possible only with compromises |
| 4 | Runtime-generated surface textures | Possible only with compromises |
| 5 | Lighting model of the water surface | Insufficiently exposed |
| 6 | Texture filtering, mip control and texture arrays | Unsupported |
| 7 | Transparency, refraction and scene depth | Unsupported |
| 8 | Reflections | Insufficiently exposed |
| 9 | Underwater volumes and light transport | Insufficiently exposed |
| 10 | Projected caustics | Unsupported |
| 11 | GPU particles driven by surface data | Insufficiently exposed |
| 12 | Interactive wave simulation on a grid (CPU) | Impractical for performance |
| 13 | Parallel computation constraints | Possible only with compromises |
| 14 | Built-in water and buoyancy | Unsupported (for open-ocean waves) |
| 15 | Physics rate, ownership and latency for floating bodies | Possible only with compromises |
| 16 | Part size limit | Possible only with compromises |
| 17 | Availability and memory budgets of editable assets | Insufficiently exposed |
| 18 | Quality-dependent post effects | Possible only with compromises |
| 19 | Performance instrumentation | Insufficiently exposed |

---

## 1. Programmable shading — Unsupported

**Required.** A high-end ocean is mostly shading work. Typical per-vertex and per-pixel programs:

- displace vertices by a wave texture on the GPU;
- reconstruct normals per pixel;
- evaluate a custom reflectance model (Fresnel, microfacet distribution, shadowing–masking);
- blend foam;
- fake subsurface scattering on thin, backlit crests.

**Exposed.**

- [Doc] Surfaces are shaded by the engine's own physically-based material. A `SurfaceAppearance` supplies fixed inputs: colour, normal, metalness, roughness and emissive maps, plus colour and emissive tint ([SurfaceAppearance](#sources)).
- [Doc] Screen effects come from a fixed list: `BloomEffect`, `BlurEffect`, `ColorCorrectionEffect`, `DepthOfFieldEffect`, `SunRaysEffect`, `ColorGradingEffect` ([Post-processing effects](#sources)). `PostEffect` is described only as an abstract base class for those ([PostEffect](#sources)).

**Missing.**

- [Test] No class or property accepts shader code. `Instance.new` fails for `Shader`, `ShaderEffect`, `CustomPostEffect`, `RenderTarget` and `ComputeShader`, and `SurfaceAppearance` has no `Shader` property.
- [Forum] Developers have requested custom shaders for years ("Material graph based custom shaders", "We. want. shaders"). [Forum] One reply relays a "no" to deeper GPU or shader access, given at a Roblox developer conference; this is second-hand and not an official document.

**Why it matters.** Every per-pixel water effect must be expressed through the fixed PBR inputs, per-vertex colours, or extra geometry. Effects that depend on per-pixel math cannot be reproduced exactly: wave-dependent roughness, view-dependent subsurface colour, foam masks sampled per pixel.

**Consequence.** [Assessment] Shading detail is bounded by mesh density and by what the engine's standard material does with its maps. The surface cannot be displaced on the GPU, so any moving geometry is a CPU cost (limitation 3).

## 2. GPU compute and render-to-texture — Unsupported

**Required.**

- Spectral ocean synthesis: inverse FFTs of a wind spectrum, several frequency bands, every frame.
- Foam accumulation and dissipation: a feedback texture blurred and faded every frame.
- Interactive wave propagation: forward and inverse FFTs per simulation step.
- Per-band statistics of the slopes (first and second moments), stored in mipmapped textures.

In engines that expose the GPU, these are compute passes writing to render targets.

**Exposed.**

- [Doc] The only writable texture is `EditableImage`, written from Luau (`WritePixelsBuffer` and others). The documented maximum size is 1024×1024 ([EditableImage](#sources)).
- [Doc] *"Only a single EditableImage can be updated per frame on the display side."*
- [Test] Studio 0.748 accepted `CreateEditableImage` up to 2048×2048 in Edit mode and refused 4096 ("expects size.X and size.Y to be less than 2048"). This differs from the documented 1024 limit. Live-client behaviour was not tested.

**Missing.** No compute shaders, no render targets, no GPU-side texture writes and no readback.

**Why it matters.** Spectral synthesis at useful resolutions (for example 256² per band, 4 bands) costs O(N² log N) per band and per frame. On the GPU that is trivial; in an interpreted, single-frame budget it is not. The one-image-per-frame display update also limits how many evolving textures can be refreshed each frame.

**Consequence.** [Assessment] Spectral and grid-based simulations are bounded by CPU time: resolution, band count and update rate are far below what a GPU allows. Per-frame texture products (foam, slope moments, wake height fields) are limited in size, number and refresh rate.

## 3. Per-frame geometry from scripts (EditableMesh) — Possible only with compromises

**Required.** A camera-centred, dense surface (tens of thousands of vertices near the camera) displaced every frame, with seamless levels of detail out to the horizon.

**Exposed.**

- [Doc] `EditableMesh` lets scripts create and modify vertices, faces, normals, UVs and per-face-corner colours with alpha. *"Visual changes to the mesh will always be immediately reflected by the engine."* *"Batching your mesh operations is typically much more performant than handling them one by one."* ([EditableMesh](#sources)).
- [Doc] *"EditableMesh currently has a limit of 60,000 vertices and 20,000 triangles."* [Test] Studio 0.748 raised "Vertex count above limit" at the 60 001st vertex.
- [Doc] Fixed-size meshes (the default from assets) *"are more efficient in terms of memory but you cannot change the number of vertices, faces, or attributes."*
- [Doc] Collision and fluid geometry are not updated by edits. They require `CreateMeshPartAsync` / `ApplyMesh` again ([EditableMesh](#sources), [MeshPart](#sources)).

**Missing.**

- No GPU displacement.
- No way to stream a vertex buffer in one call.
- No documented cost model for per-frame updates.

**Why it matters.**

- [Test] Writing vertex positions of an `EditableMesh` every frame costs Luau time per vertex. It also adds render-thread CPU time per frame (measured as render-CPU time with interleaved A/B runs in Studio), whatever the mesh's transparency.
- The 20,000-triangle limit per mesh is below what one dense near-field patch needs.

**Consequence.** [Assessment] Surface density is bounded by CPU time per frame and by the per-mesh limits. Dense, moving geometry is affordable only over a small area, and surfaces larger than one mesh have seams that the engine does not manage. Physics collision does not follow the moving surface.

## 4. Runtime-generated surface textures — Possible only with compromises

**Required.** Normal, foam and roughness maps generated or animated at runtime and applied to the water material: detail normals from the spectrum, foam masks, variance-adjusted roughness.

**Exposed.**

- [Doc] `SurfaceAppearance` map `Content` properties *"can hold an asset URI or a reference to an EditableImage object"*. *"Assigning an EditableImage is useful for dynamically generating or modifying textures at runtime"* ([SurfaceAppearance](#sources)).
- [Doc] Created through `AssetService:CreateSurfaceAppearanceAsync`. The map content *"cannot be reassigned or swapped after the SurfaceAppearance is created"*. *"If you assign an EditableImage to a SurfaceAppearance map on the server, clients see only a checkerboard."* EditableImage-based appearances *"use more memory"* and *"update whenever the underlying EditableImage changes"* ([AssetService](#sources)).
- [Doc] *"Most SurfaceAppearance properties cannot be modified by scripts, as the necessary pre-processing is usually too expensive during runtime"* ([SurfaceAppearance](#sources)).
- [Doc] `MeshPart.TextureContent` and `ImageLabel.ImageContent` accept an `EditableImage` ([MeshPart](#sources), [EditableImage](#sources)).
- [Test] A `Beam.TextureContent` set to an `EditableImage` was accepted, but nothing was drawn (Studio 0.748). [Doc] The Beam reference documents asset URIs only for that property ([Beam](#sources)).

**Missing.**

- Animated maps must be written from Luau into a limited number of images (one display update per frame, see limitation 2), within the client memory budget (limitation 17).
- Map assignments are fixed at creation and client-side only.

**Why it matters.** Animated detail normals, foam masks that follow the waves, and roughness that follows the wave state all need per-frame or near-per-frame map updates on the water surface.

**Consequence.** [Assessment] Runtime-generated maps on the water material are limited to a small number of client-side maps, fixed at creation and refreshed slowly. Fully procedural, per-frame surface detail is not achievable at useful resolution.

## 5. Lighting model of the water surface — Insufficiently exposed

**Required.**

- A microfacet reflectance model whose roughness is driven by the statistics of the waves *below* pixel size, so distant water looks correctly glossy instead of aliased.
- Fresnel adjusted for that roughness.
- Analytic sun specular with shadowing–masking.
- Filtered environment reflection.
- A scattering term for light passing through crests.
- Foam with its own roughness.

**Exposed.**

- [Doc] One global PBR model with per-material colour, normal, metalness and roughness maps ([SurfaceAppearance](#sources)).
- [Doc] Global environment lighting through `Lighting.EnvironmentSpecularScale` (*"Specular light derived from environment"*: it *"will make smooth objects reflect the environment"*) and `EnvironmentDiffuseScale` ([Lighting](#sources)).
- [Doc] Per-face-corner vertex colours with alpha on `EditableMesh` ([EditableMesh](#sources)).

**Missing.**

- No access to the reflectance function.
- No per-pixel roughness driven by data other than a roughness map.
- No custom scattering term.
- No way to make roughness depend on distance or slope variance.

**Why it matters.** Without variance-aware roughness, normal-mapped water shimmers or looks too sharp at a distance. Without a scattering term, backlit crests lose their characteristic green-blue glow.

**Consequence.** [Assessment] Wave-dependent roughness, roughness-aware Fresnel and crest scattering cannot be rendered per pixel. Distant normal-mapped water is prone to shimmer or to an overly sharp look that the material cannot correct.

## 6. Texture filtering, mip control and texture arrays — Unsupported

**Required.**

- Bicubic filtering of the most detailed band, to avoid blocky magnification.
- Mipmaps of slope moments computed linearly (not of normals), so that filtering preserves roughness.
- Texture arrays to keep several frequency bands in one resource.

**Exposed.** [Doc] `SurfaceAppearance.ResampleMode` selects a resampling mode ([SurfaceAppearance](#sources)). [Test] No property on `SurfaceAppearance`, `EditableImage` or `MeshPart` controls mip generation, sampler type or array layers.

**Missing.** Custom samplers, custom mip chains, texture arrays and LOD bias.

**Why it matters.** Correct filtering of wave detail is what keeps an ocean stable at every distance.

**Consequence.** [Assessment] The filtering quality of wave detail is the engine default; distance artefacts cannot be corrected at the sampler level.

## 7. Transparency, refraction and scene depth — Unsupported

**Required.**

- Refraction of the scene below the surface.
- Depth-based absorption (water gets darker and bluer with thickness).
- Shoreline softening by depth difference.
- Correctly ordered transparent surfaces.

These normally sample the scene colour and depth buffers.

**Exposed.**

- [Doc] Part and mesh transparency, plus a different blending method for `SurfaceAppearance` with `AlphaMode = Transparency` above 0.02 transparency ([SurfaceAppearance](#sources)).
- [Doc] *"Placing objects with partial transparency near each other forces the engine to render the overlapping pixels multiple times, which can hurt performance"* ([Improve performance](#sources)).

**Missing.**

- No access to scene colour or depth.
- No refraction.
- [Test] Triangles inside one transparent mesh were not sorted back-to-front: in Studio 0.748, with large waves, a farther wave face showed through a nearer one.

**Why it matters.** Transparency is the main visual cue of shallow water. Without depth, the colour cannot depend on how much water the view ray crosses. Without sorting, transparent waves self-overlap incorrectly.

**Consequence.** [Assessment] Refraction and colour that depends on water thickness are not available. Large transparent wave meshes show ordering artefacts, so convincing see-through water is restricted to small, gentle areas.

## 8. Reflections — Insufficiently exposed

**Required.** Reflections of the sky, plus local reflections of boats, cliffs and characters (screen-space or planar), blurred by roughness.

**Exposed.** [Doc] Environment reflection of the sky through the PBR material, scaled globally by `Lighting.EnvironmentSpecularScale` ([Lighting](#sources)).

**Missing.** Script-controlled planar or screen-space reflections of scene objects, and reflection probes.

**Why it matters.** Local reflections anchor objects to the water.

**Consequence.** [Assessment] Only the environment is reflected; boats, cliffs and characters have no reflection in the water.

## 9. Underwater volumes and light transport — Insufficiently exposed

**Required.**

- A participating medium below the surface: wavelength-dependent absorption and scattering by distance.
- Light shafts (volumetric light) through a moving surface.
- Snell's window.
- A clean transition when the camera crosses the surface (the waterline split).

**Exposed.**

- [Doc] `Atmosphere` (Density, Offset, Color, Decay, Glare, Haze) is global. *"Density only directly affects objects and terrain"* ([Atmosphere](#sources)).
- [Doc] Global post effects (blur, colour correction, sun rays) ([Post-processing effects](#sources)).
- [Doc] Beams with per-segment transparency: *"Each segment of the beam can only show a transition between two transparencies"* ([Beam](#sources)).

**Missing.**

- Local fog volumes.
- Wavelength-dependent extinction.
- Volumetric lighting.
- A camera-plane clip or split rendering at the waterline.

**Why it matters.** The underwater look is defined by distance-based colour loss, which differs per colour channel, and by light shafts.

**Consequence.** [Assessment] The underwater look is limited to global fog and colour grading that apply to the whole view. Per-channel absorption by distance, volumetric light shafts and a correct half-in, half-out waterline view are not available.

## 10. Projected caustics — Unsupported

**Required.** Moving light patterns projected onto the seabed and submerged objects, computed from the surface's refraction, usually as a projected texture or a light cookie.

**Exposed.** [Doc] A `SpotLight` *"emits light of a specified Color and Brightness in the shape of a cone"*. Its own properties are `Angle`, `Face` and `Range`; it inherits `Brightness`, `Color`, `Enabled` and `Shadows` from `Light` ([SpotLight](#sources)).

**Missing.** [Doc] No texture, cookie or projection property is listed. [Test] Reading `Texture`, `TextureContent` or `Cookie` on a `SpotLight` fails (Studio 0.748).

**Why it matters.** Caustics are among the strongest cues that one is underwater.

**Consequence.** [Assessment] Caustic patterns cannot be projected from the surface onto the scene. Any approximation is limited in resolution and coverage, and each additional light costs GPU time.

## 11. GPU particles driven by surface data — Insufficiently exposed

**Required.** Spray emitted where crests break. That means sampling a foam or whitecap texture per particle on the GPU, spawning only inside the view, and killing particles whose source no longer qualifies.

**Exposed.** [Doc] `ParticleEmitter`, *"a special object that emits customizable 2D particles into the world"*, with emitter-wide properties, texture flipbooks (`FlipbookLayout`) and two methods: `Emit(count)` and `Clear()` ([ParticleEmitter](#sources)).

**Missing.** [Doc] No method gives access to individual particles. [Test] `GetParticles` / `SetParticle` do not exist (Studio 0.748). There is no texture sampling at spawn and no particle-side logic.

**Why it matters.** Spray must appear exactly on the breaking crests, and only there.

**Consequence.** [Assessment] Spray cannot be driven per particle by surface data. Emission is controlled per emitter from scripts, so placement accuracy and spray density are bounded by script time and emitter count.

## 12. Interactive wave simulation on a grid (CPU) — Impractical for performance

**Required.** Wakes and splashes that propagate with the correct dispersion (long waves faster than short ones):

- Displacement is injected by hulls and impacts.
- Each step runs forward FFTs, a frequency-domain evolution operator, then inverse FFTs for height and horizontal displacement, plus foam advected by the same field.
- A step costs several 2D FFTs of the grid.

**Exposed.** [Doc] Luau scripts, including Parallel Luau (limitation 13). There is no compute API (limitation 2).

**Missing.** GPU execution of the grid.

**Why it matters.** A grid fine enough to resolve hull-sized wakes near the camera (for example 128² or more), stepped at the frame rate with several FFTs per step, does not fit in a few milliseconds of Luau. It must also be uploaded as geometry or a texture every frame.

**Consequence.** [Assessment] Interactive waves are limited in grid resolution, update rate and area. Large-scale interaction, such as many wakes or distant explosions, does not fit a real-time budget.

## 13. Parallel computation constraints — Possible only with compromises

**Required.** Spreading per-vertex wave evaluation and simulation across CPU cores.

**Exposed.** [Doc] Parallel Luau with Actors; code calls `task.desynchronize()` to run in parallel ([Parallel Luau](#sources)). Documented rules:

- *"You can't use `require()` in a desynchronized parallel phase."*
- *"Scripts running in parallel generally can't write to the data model."*
- *"Modifying instances in parallel phases"* is not supported.
- API members default to the **Unsafe** thread-safety level.
- `SharedTable` lets actors share data: *"Every update to a shared table by one actor is immediately visible to all actors."*

**Missing.** Parallel writes to an `EditableMesh` or `EditableImage` are not documented as safe. [Doc] The page gives no core count and no guarantee of parallel execution on a given device.

**Why it matters.** Wave evaluation parallelises well, but its result must be written to the mesh, which is not parallel-safe.

**Consequence.** [Assessment] Only part of the work can run in parallel. Writing the results to the mesh, and the engine's upload, stay on the frame's serial path, and the speed-up depends on the device's core count.

## 14. Built-in water and buoyancy — Unsupported (for open-ocean waves)

**Required.** A water body with large, physically shaped waves (metres high, hundreds of metres long), coupled to buoyancy so floating objects follow the same waves the player sees.

**Exposed.**

- [Doc] Terrain water has adjustable `WaterWaveSize` (*"constrained to between 0 and 1"* stud), `WaterWaveSpeed` (0–100), colour, transparency and reflectance ([Terrain](#sources)).
- [Test] Setting `WaterWaveSize = 50` in Studio 0.748 clamps to 1.
- [Forum] Parts float in Terrain water according to their density (water ≈ density 1); this behaviour is described by developers, not in an official physics page found on the research date.

**Missing.**

- Large waves.
- Wave spectra.
- A script API to query the water surface height or velocity at a point.
- Buoyancy for any water other than Terrain water.

**Why it matters.** Open-ocean gameplay needs boats that pitch and roll on the visible waves.

**Consequence.** [Assessment] The built-in water cannot represent open-ocean waves, and the built-in buoyancy only knows its flat surface. Wave-coupled floating requires scripted forces, and Terrain water in the same area adds a second, flat water surface.

## 15. Physics rate, ownership and latency for floating bodies — Possible only with compromises

**Required.**

- Buoyancy forces integrated at a stable rate, with low latency for the driver of a boat and consistent results for everyone else.
- Noise-free hull motion even when samples are sparse.

**Exposed.**

- [Doc] *"By default, Roblox simulates physics at 240 Hz"*, with adaptive timestepping in 240 / 120 / 60 Hz islands ([Adaptive timestepping](#sources)).
- [Doc] *"Server heartbeat is capped at 60 FPS for all games"* ([Identify performance issues](#sources)).
- [Doc] The server owns parts by default; ownership can be given to a client with `SetNetworkOwner`; anchored parts are always owned by the server ([Network ownership](#sources)).

**Missing.** Script callbacks inside each physics substep for custom forces on every assembly, at the solver rate.

**Why it matters.** Forces applied from scripts run at script rates (at most once per frame or heartbeat). The owner simulates, and every other client sees the replicated result.

**Consequence.** [Assessment] Scripted water forces act at frame or heartbeat rate, not at the solver rate, so sparse samples produce visible noise unless filtered. Responsiveness depends on which machine owns the body. Agreement between machines depends on every machine evaluating identical waves.

## 16. Part size limit — Possible only with compromises

**Required.** One continuous surface out to the horizon, often tens of kilometres.

**Exposed.**

- [Test] `Part.Size` and `MeshPart.Size` are clamped to 2048 studs per axis (Studio 0.748; setting 5000 gives 2048).
- [Forum] The same limit is reported by developers, with requests to raise it.
- [Test] After `CreateMeshPartAsync`, vertices of a fixed-size `EditableMesh` could be moved beyond the 2048-stud extent and were rendered (Studio 0.748).

**Missing.** A documented maximum for rendered mesh extent; very large parts.

**Why it matters.** The horizon must not show an edge.

**Consequence.** [Assessment] Horizon-scale water cannot be one documented object. It depends on several parts, or on behaviour that is not documented and may change.

## 17. Availability and memory budgets of editable assets — Insufficiently exposed

**Required.** Guaranteed availability of the runtime-mesh and runtime-texture features on every client, with a known memory budget.

**Exposed.**

- [Doc] *"Using EditableMesh fails by default in published games"* until **Allow Mesh / Image APIs** is enabled. Enabling it requires an ID-verified owner, 13+ ([EditableMesh](#sources), [Experience settings](#sources)).
- [Doc] *"EditableMesh has strict client-side memory budgets, although the server, Studio, and plugins operate with unlimited memory."* Creation *"will fail and this method will return nil"* when the device budget is exhausted ([AssetService](#sources)).
- [Doc] Roblox has listed memory budget limits among the usability problems still under investigation ([EditableMesh and EditableImage Improvements](#sources)).

**Missing.** A published per-device budget, and a query of the remaining budget.

**Why it matters.** A phone can refuse a mesh or image that worked in Studio, where the budget is unlimited.

**Consequence.** [Assessment] Runtime meshes and textures may be refused on some devices. The safe amount is unknown, and Studio testing cannot show whether a phone will succeed.

## 18. Quality-dependent post effects — Possible only with compromises

**Required.** Consistent fog, glare, bloom and colour grading on every device, especially underwater and in storms.

**Exposed.** [Doc] *"Some post-processing effects will work differently or not at all when Roblox is set to a low QualityLevel"* ([PostEffect](#sources)).

**Missing.** A way to request a minimum effect quality, or to query which effects are active.

**Why it matters.** Underwater and storm looks rely on these effects.

**Consequence.** [Assessment] Water looks that rely on post effects degrade or disappear on low graphics settings, outside the developer's control.

## 19. Performance instrumentation — Insufficiently exposed

**Required.** Reliable per-frame CPU and GPU timings and memory on target devices, for budgeting a simulation.

**Exposed.**

- [Doc] `Stats.RenderCPUFrameTime`, `Stats.RenderGPUFrameTime`, `HeartbeatTime`, `PhysicsStepTime`, `GetTotalMemoryUsageMb()`, `GetMemoryUsageMbForTag()`. Per-tag memory *"will return 0"* when memory tracking is disabled ([Stats](#sources)).
- [Doc] The Developer Console and MicroProfiler are available in Studio and live sessions ([Identify performance issues](#sources)).

**Missing.**

- CPU and GPU utilisation, clock and thermal state.
- [Test] In Studio 0.748, `RenderGPUFrameTime` varied strongly between identical runs, so it could not separate small changes.

**Why it matters.** Mobile performance is dominated by thermal throttling and GPU load, which scripts cannot observe.

**Consequence.** [Assessment] Simulation budgets cannot be verified from inside the experience alone. They need physical devices, repeated same-condition runs and external tools; Studio numbers are only relative.

---

## Sources

All pages accessed on 2026-11-19. Official documentation pages showed `last_updated` dates of 2026-11-19 (`EditableImage`: 2026-11-06).

**Official documentation (create.roblox.com)**

- EditableMesh — https://create.roblox.com/docs/reference/engine/classes/EditableMesh
- AssetService — https://create.roblox.com/docs/reference/engine/classes/AssetService
- EditableImage — https://create.roblox.com/docs/reference/engine/classes/EditableImage
- SurfaceAppearance — https://create.roblox.com/docs/reference/engine/classes/SurfaceAppearance
- MeshPart — https://create.roblox.com/docs/reference/engine/classes/MeshPart
- Beam — https://create.roblox.com/docs/reference/engine/classes/Beam
- SpotLight — https://create.roblox.com/docs/reference/engine/classes/SpotLight
- ParticleEmitter — https://create.roblox.com/docs/reference/engine/classes/ParticleEmitter
- Lighting — https://create.roblox.com/docs/reference/engine/classes/Lighting
- Terrain — https://create.roblox.com/docs/reference/engine/classes/Terrain
- Stats — https://create.roblox.com/docs/reference/engine/classes/Stats
- PostEffect — https://create.roblox.com/docs/reference/engine/classes/PostEffect
- Post-processing effects — https://create.roblox.com/docs/environment/post-processing-effects
- Atmosphere — https://create.roblox.com/docs/environment/atmosphere
- Parallel Luau — https://create.roblox.com/docs/scripting/multithreading
- Improve performance — https://create.roblox.com/docs/performance-optimization/improve
- Identify performance issues — https://create.roblox.com/docs/performance-optimization/identify
- Adaptive timestepping — https://create.roblox.com/docs/physics/adaptive-timestepping
- Network ownership — https://create.roblox.com/docs/physics/network-ownership
- Experience settings — https://create.roblox.com/docs/studio/experience-settings

**Announcements (Roblox staff, Developer Forum)**

- [Client Beta] In-experience Mesh & Image APIs now available in published experiences — https://devforum.roblox.com/t/client-beta-in-experience-mesh-image-apis-now-available-in-published-experiences/3267293
- EditableMesh and EditableImage Improvements — https://devforum.roblox.com/t/editablemesh-and-editableimage-improvements/3818624

**Community (Developer Forum, not official)**

- Material graph based custom shaders — https://devforum.roblox.com/t/material-graph-based-custom-shaders/236718
- We. want. shaders — https://devforum.roblox.com/t/we-want-shaders/3985196
- Increase max BasePart size — https://devforum.roblox.com/t/increase-max-basepart-size/441035
- Parts don't float on water — https://devforum.roblox.com/t/parts-dont-float-on-water/3271031

**Tests.** The [Test] statements were checked by running Luau in Roblox Studio 0.748 (Windows) on 2026-11-19:

- instantiating classes and reading properties;
- setting sizes and checking the clamped result;
- adding vertices until the error;
- creating editable images of increasing size;
- play-test observation of transparency and of mesh-update cost.
