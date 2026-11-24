# Reference

The public API, the main settings, performance measured on the development PC, and the Roblox platform limits that shape the ocean.

## API

`ReplicatedStorage.Ocean.OceanEngine` (v2.0.0) is the only public module. The same module runs on the server, which owns the settings and the buoyancy, and on clients, which do the rendering.

```lua
local OceanEngine = require(game:GetService("ReplicatedStorage"):WaitForChild("Ocean"):WaitForChild("OceanEngine"))
local ocean = OceanEngine.Running() -- the engine started by OceanServer / OceanClientBoot (nil before)
```

| Call | Side | What it does |
| --- | --- | --- |
| `OceanEngine.new(opts):Start()` | both | Starts this side's engine. Only the first `Start()` on a side applies its options. Options: `Preset`, `Quality`, `Settings = { [path] = value }`, `RemoteAccess` (`"Studio"` default, `"Creator"`, `"None"` or `function(player)`), `Buoyancy` (`false` to skip it) |
| `ocean:Set(path, v)` → `ok, err` | both | Validated change. On the server it applies to everyone, physics included. On a client it applies to that player only |
| `ocean:Get(path)` / `ocean:Reset(path?)` | both | Value in effect / remove an override |
| `ocean:SetPreset(name, seconds?)` | both | Cross-fades the sea. A client asks the server, which decides through `RemoteAccess` |
| `ocean:SetQuality(tier)` | client | `Low` / `Medium` / `High` / `Ultra` / `Auto` (phones get Medium) |
| `ocean:GetHeight(x, z)`, `ocean:GetSurfaceData(pos)`, `ocean:IsUnderwater(pos)` | both | The true wave surface. Surface data holds `Height`, `Normal`, `Velocity`, `Depth`, `IsUnderwater`, `Jacobian` |
| `ocean:GetStats()` | both | Client: fps, ms, vertices. Server: buoyancy counters |
| `ocean:SetVisible(on)`, `ocean:Destroy()` | client / both | |
| `ocean.Changed (config, paths)`, `ocean.PresetChanged (name, seconds, start)` | both | Signals |
| `OceanEngine.Describe(path)` | both | `{ default, rule, serverAuthoritative }` |

## Settings

Values are resolved in this order: engine defaults → preset → `Config.Project` overrides (this game) → server runtime overrides → per-player overrides.

Some paths are **server** paths, because they change the waves and the physics:

- `Preset`
- `Algorithm`
- `General.SeaLevel`, `General.Seed`, `General.TimeScale`, `General.Gravity`, `General.SimulationFPS`
- everything under `Wind.*`, `Gerstner.*`, `FFT.*`, `Hybrid.*`, `Buoyancy.*`, `Waves.*`, `Zones.*`

Every other path is visual and can differ per player.

Everything applies live, with three exceptions:

- `Quality`, `Rendering.HorizonDistance` and `Normals.*` rebuild the mesh.
- `General.SeaLevel`, `General.Seed` and `Debug.Panel` are read once at start.
- Any `Seafloor.*` change regenerates the seafloor.

| Path | Default | Range | Notes |
| --- | --- | --- | --- |
| `Preset` | Normal | Calm, Normal, Rough, Storm, IslandDemo, Custom | Sea state |
| `Quality` | Auto | Low, Medium, High, Ultra, Auto | The main performance lever (rendering only) |
| `Wind.Speed` | 10 | 0.5 – 40 (m/s) | Wave height and length |
| `Gerstner.AmplitudeScale` / `Steepness` | 1 / 0.75 | 0 – 5 / 0 – 1.5 | Height / crest sharpness |
| `Gerstner.WaveCount` | 16 | 1 – 64 | Linear CPU cost per vertex |
| `Waves.Wavelength` / `Waves.Speed` | natural | 4 – 1500 / 0 – 200 | Dominant wave in studs and studs/s |
| `Look.TransparencyMode` | NearOnly | NearOnly, Zone, Low, ColorFake | NearOnly avoids wave-through-wave artefacts |
| `Look.NearTransparency` | 0.25 | 0 – 0.6 | See-through calm water |
| `Look.WhitecapIntensity` | 1 | 0 – 2 | Foam on crests |
| `Spray.Enabled`, `Wake.Enabled` | on | | Particles / wakes |
| `Underwater.Enabled` / `Underwater.Lights` | on | | All underwater effects / only shafts, caustics, sparkles |
| `Seafloor.Enabled` / `Seafloor.Plants` | on | | Procedural seafloor / kelp and seagrass |
| `Seafloor.ViewChunks` | 2 | 1 – 5 | (2n+1)² chunks around the camera; the biggest memory user |
| `Demo.SwimEnabled` / `Demo.DiveEnabled` | on | | Swimming / diving |
| `Weather.Enabled` | on | | Local storm (rain, lightning) |
| `Surf.Enabled` / `Surf.Height` / `Surf.Period` | on / 2.2 / 4.5 s | 0 – 6 / 3 – 20 | Beach waves (client, visual); see [tutorial 5](tutorials/05-materials-and-effects.md#beach-waves) |
| `Lighting.Enabled` | on | | `false`: the engine leaves `Lighting` alone |
| `Debug.Panel` | off | | In-game panel outside Studio (for example to test on a phone) |

Every other setting lives in the `Ocean.Config` modules, with a comment and a validation rule. Invalid values are refused with a message (`ocean:Set` returns `false, err`). The in-game panel (Ocean button) exposes most of them.

## Performance on the development PC

These are Roblox Studio measurements on a desktop PC, not on a phone: Intel Core i7-13620H, Intel UHD Graphics, 16 GB RAM, Windows 11, Studio 0.748.

Method:

- Play test, client side, preset IslandDemo, Gerstner, Quality High.
- The Studio window was focused; Studio caps the frame rate at 60.
- Camera paths come from `Ocean.Debug.BenchScenarios`.
- Each version was measured in a fresh play session, with scenarios in the same order.

| Scenario | FPS before → after | 1 % low after | Render CPU ms before → after | Ocean script ms before → after |
| --- | --- | --- | --- | --- |
| still | 59.9 → 60.2 | 54.0 | 7.76 → 6.34 | 3.99 → 3.22 |
| underwater | 60.0 → 60.0 | 53.3 | 7.71 → 6.59 | 4.03 → 3.43 |
| storm | 59.2 → 60.1 | 54.5 | 8.85 → 7.64 | 4.88 → 4.08 |
| seafloor exploration | 53.0 → 57.1 | 39.7 | 9.85 → 9.43 | 5.63 → 4.38 |
| fast flight (250 studs/s) | 44.7 → 59.3 | — | — | 7.64 → 4.55 |

- **Geometry:** the circular mesh has 4,681 vertices and 9,144 triangles on High, down from 6,481 and 12,672, and the horizon went from 176 parts to 1.
- **Quality tiers:** Low 2,129, Medium 3,229, High 4,681 and Ultra 6,453 vertices, with no seam gaps on any tier.
- **Render GPU time** (`Stats.RenderGPUFrameTime`): 6.7 ms still and 9.4 ms underwater. It varies a lot between runs in Studio.
- **Long play sessions:** performance drifts down for every version, which is why only fresh sessions are compared.

**Phones: not measured yet.** No iPhone or other phone has been tested. To measure one, set `["Debug.Panel"] = true`, publish, then in the game tap **Ocean › Performance › Run device benchmark (~70 s)** and copy the result. The benchmark reports:

- frame time (mean, p95, max), FPS and 1 % low
- render CPU and GPU time where the device exposes them
- ocean script time, vertices and memory

CPU and GPU utilisation and thermal state are not readable from a Roblox script.

## Roblox limitations

Researched on 2026-11-19 against the official documentation, plus tests in Studio 0.748.

| Limit | Fact | Consequence for an ocean |
| --- | --- | --- |
| No custom shaders | Shading is the engine's fixed PBR material (`SurfaceAppearance` maps). Post effects are a fixed list (Bloom, Blur, ColorCorrection, ColorGrading, DepthOfField, SunRays). Tested: no shader, render-target or compute class exists | No GPU wave displacement, custom reflectance, foam masks or scattering per pixel. Detail is limited by mesh density and vertex colours |
| No GPU compute or render targets | The only writable texture is `EditableImage`: documented max 1024² (Studio accepts 2048²), and only one updated per frame on the display side | Spectral (FFT) waves, foam simulation and interactive wakes run on the CPU, at low resolution or rate |
| CPU-written geometry | `EditableMesh` is limited to 60,000 vertices and 20,000 triangles per mesh (tested: error at the 60,001st vertex). Writing it each frame also costs render-thread time. Collision does not follow edits | Dense geometry only near the camera. Physics cannot use the visible surface |
| Transparency | One transparent mesh is not sorted internally (observed). Overlapping transparency is costly (docs). No access to scene depth or colour | No refraction or depth-based water colour. Transparency is kept to calm water near the camera |
| Lighting | Reflections show only the environment (`EnvironmentSpecularScale`). `SpotLight` has no texture or cookie. `Atmosphere` is global | No reflections of boats or cliffs, no projected caustics, no local underwater volume. Underwater is a global fog and colour switch |
| Particles | `ParticleEmitter` only offers `Emit` / `Clear`; there is no per-particle access | Spray must be placed per emitter by scripts, so its density is capped |
| Parallel Luau | No `require` and no data model writes while desynchronised | Wave evaluation can run in parallel, but writing the mesh stays serial |
| Built-in water | Terrain water waves are clamped to 1 stud, with no API to query the surface. Engine buoyancy only knows Terrain water | Real waves need a scripted wave function and scripted buoyancy. Terrain water must be absent |
| Physics rate | Physics runs at 240 Hz (adaptive), but server heartbeat is capped at 60 and script forces apply once per frame. The owner of an assembly simulates it | Buoyancy samples must be filtered. Boats are simulated on the driver's client. Waves come from a shared clock instead of being replicated |
| Part size | `Size` is clamped to 2,048 studs per axis (tested) | The horizon needs several parts or meshes stretched after creation (undocumented behaviour) |
| Editable APIs in live games | Off until **Allow Mesh / Image APIs** is enabled by an ID-verified owner. Strict per-device client memory budgets; creation can return `nil` | A phone can refuse what works in Studio, where the budget is unlimited |
| Low graphics settings | Some post effects change or disappear at low quality levels | Water looks must not depend on them |

Sources: create.roblox.com class references for EditableMesh, EditableImage, AssetService, SurfaceAppearance, MeshPart, Beam, SpotLight, ParticleEmitter, Lighting, Terrain and Stats, plus the guides Post-processing effects, Atmosphere, Parallel Luau, Improve performance, Identify performance issues, Adaptive timestepping, Network ownership and Experience settings. DevForum community threads were used only for the part size limit and Terrain buoyancy.
