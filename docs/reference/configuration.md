# Configuration reference

Every setting of the ocean engine, generated from the engine's own `Config` modules and validation schema (engine 2.0.0, 2026-11-19). Descriptions come from the source comments.

## How to set a value

| Where | Scope | Example |
| --- | --- | --- |
| `ReplicatedStorage.Ocean.Config.Project` | This game's defaults (above every preset) | `Overrides = { ["Wind.Speed"] = 12 }` |
| Plugin › Configure | Writes `Config.Project` | — |
| `ocean:Set(path, value)` on the **server** | Everyone, at runtime | `ocean:Set("Wind.Speed", 18)` |
| `ocean:Set(path, value)` on a **client** | That player only | `ocean:Set("Spray.Enabled", false)` |
| In-game panel | Same as `ocean:Set` (server paths go through `OceanRemote`) | — |

Resolution order: engine defaults → preset → `Config.Project` overrides → server runtime overrides → per-player overrides.

## Columns

- **Default**: the engine default, before any preset. Presets change many wave, look and lighting values; the preset's value is what you see unless you override it.
- **Range**: enforced by the schema. Values outside it are refused (`ocean:Set` returns `false, err`). An empty range means the value only has to match the default's type.
- **Side**:
  - **server**: server-authoritative (wave field and physics). Set it on the server; on a client it changes only that player's picture.
  - **client**: visual or per-player.
- **Applies**:
  - **live**: takes effect on the next frame. Server wave changes cross-fade over about 1.5 s.
  - **rebuild**: rebuilds the mesh or material. Happens automatically when the quality changes; for the other rebuild settings, use the panel's **Rebuild ocean now** or rejoin.
  - **restart**: read once when the game starts. Set it in `Config.Project`.
- **Perf**: notes at the top of each section. "None" means no measurable cost.

Paths not listed here are internal (the shared clock `General.Clock*`, `General.FieldChangeTime`, `Version`, `SchemaExtra`).

---

## Preset, Quality, Algorithm

Perf: Quality is the main rendering lever ([tutorial 4](../tutorials/04-waves-and-quality.md#quality)). FFT and Hybrid add a 4–4.5 ms grid update per simulation step on the server and on every client (Studio measurement); Gerstner has none.

| Path | Default | Range | Side | Applies | Description |
| --- | --- | --- | --- | --- | --- |
| `Preset` | "IslandDemo" in the demo place; Create Ocean writes "Normal" | Calm / Normal / Rough / Storm / Custom / IslandDemo | server | live | Sea state. Changes cross-fade (`SetPreset(name, seconds)`) |
| `Quality` | "High" (Create Ocean writes "Auto") | Low / Medium / High / Ultra / Auto | client | rebuild | Rendering tier. Auto: phones = Medium, else from the Roblox graphics level |
| `Algorithm` | "Gerstner" | Gerstner / FFT / Hybrid | server | live | Wave model for both the visuals and the physics |
| `Enabled` | true | | client | restart | Master switch |

## General

Perf: `RenderDistance` sets how far the wave mesh reaches; `SimulationFPS` sets how often FFT layers recompute.

| Path | Default | Range | Side | Applies | Description |
| --- | --- | --- | --- | --- | --- |
| `General.SeaLevel` | 0 | | server | restart | World Y of the undisturbed surface |
| `General.Seed` | 1337 | | server | restart | Deterministic spectrum seed (server and client must match) |
| `General.TimeScale` | 1 | 0 – 10 | server | live | Wave speed multiplier (see `Waves.*` for absolute speed / wavelength) |
| `General.Gravity` | 35.04 | 1 – 500 | server | live | Studs/s² used by the dispersion relation |
| `General.SimulationFPS` | 30 | 1 – 120 | server | live | FFT layers recompute at this rate and are extrapolated in between; Gerstner is exact every call |
| `General.RenderDistance` | 6000 | 256 – 20000 | client | live | Radius (studs) that the ocean surface must cover |
| `General.LODEnabled` | true | | client | live | Level of detail on the far rings |
| `General.StudsPerMeter` | 3.571 | | client | live | Unit conversion (1 stud = 0.28 m) |

## Wind and waves

Perf: none, except `Gerstner.WaveCount`, which is linear CPU per vertex (16 by default), on the server and every client.

| Path | Default | Range | Side | Applies | Description |
| --- | --- | --- | --- | --- | --- |
| `Wind.Speed` | 10 | 0.5 – 40 | server | live | m/s (10 m/s ≈ moderate breeze, Beaufort 5) |
| `Wind.Direction` | (1, 0.35) | Vector2 | server | live | Main propagation direction, normalized internally |
| `Wind.Fetch` | 100 | 1 – 2000 | server | live | km (JONSWAP) |
| `Waves.Wavelength` | nil (natural) | 4 – 1500 | server | live | Studs between two crests of the dominant wave; rescales the whole set, keeps the height. Cross-fades over 1.5 s |
| `Waves.Frequency` | nil (natural) | 0.05 – 25 | server | live | Crests per 100 studs; writes `Waves.Wavelength = 100 / f` |
| `Waves.Speed` | nil (natural) | 0 – 200 | server | live | Studs/s of the dominant crests, held constant; `General.TimeScale` clears it |
| `Waves.Current.*` | derived | read-only | server | live | `Wavelength`, `Frequency`, `Speed`, `Period`, `NaturalSpeed` actually in effect |
| `Gerstner.AmplitudeScale` | 1 | 0 – 5 | server | live | Wave height multiplier |
| `Gerstner.Steepness` | 0.75 | 0 – 1.5 | server | live | Global crest sharpness; normalized so waves never loop |
| `Gerstner.WaveCount` | 16 | 1 – 64 | server | live | Number of waves; more = less visible repetition, linear cost |
| `Gerstner.WavelengthScale` | 0.55 | 0.05 – 20 | server | live | < 1: shorter waves with the same significant height |
| `Gerstner.Spread` | 6 | 0.5 – 40 | server | live | Directional spreading exponent; larger = more aligned with the wind |
| `Gerstner.Spectrum` | "JONSWAP" | JONSWAP / PiersonMoskowitz / Manual | server | live | Wave spectrum |
| `Gerstner.MinWavelength` | 12 | 1 – 5000 | server | live | Studs |
| `Gerstner.MaxWavelength` | 600 | 2 – 10000 | server | live | Studs |
| `Gerstner.MinWavelengthFloor` | 7 | | server | live | Shortest geometric wave after scaling (the detail normal map covers below) |
| `Gerstner.SwellMaxLoop` | 0.4 | | server | live | Steepness budget of the swell band |
| `Gerstner.ChopMaxLoop` | 0.56 | | server | live | Steepness budget of the chop band; swell + chop must stay < 1 |
| `FFT.Resolution` | 64 | 16 – 256 | server | live | Grid size, power of two. Cost grows with N² log N |
| `FFT.PatchSize` | 400 | 16 – 4000 | server | live | Studs, period of the largest cascade |
| `FFT.Cascades` | 2 | 1 – 2 | server | live | A second cascade breaks visible tiling (doubles the cost) |
| `FFT.CascadePatchRatio` | 0.17 | | server | live | Size of cascade 2 relative to cascade 1 |
| `FFT.Amplitude` | 1 | 0 – 10 | server | live | Height multiplier |
| `FFT.Choppiness` | 1.2 | 0 – 4 | server | live | Horizontal displacement |
| `FFT.Spectrum` | "JONSWAP" | Phillips / JONSWAP | server | live | Spectrum |
| `FFT.SmallWaveCutoff` | 1.5 | | server | live | Studs, damping of tiny waves |
| `Hybrid.SwellCount` | 4 | 0 – 16 | server | live | Long Gerstner swells layered under the FFT field |
| `Hybrid.SwellAmplitudeScale` | 1 | 0 – 5 | server | live | |
| `Hybrid.SwellMinWavelength` | 250 | 10 – 5000 | server | live | |
| `Hybrid.SwellMaxWavelength` | 900 | 20 – 10000 | server | live | |

## Zones

| Path | Default | Range | Side | Applies | Description |
| --- | --- | --- | --- | --- | --- |
| `Zones.Enabled` | true | | server | live | Sea-state zones on or off |
| `Zones.List` | list (2 entries in the demo) | | server | live | Zones ([tutorial 4](../tutorials/04-waves-and-quality.md#sea-state-zones)). Declare them in config so every side starts with them |

## Buoyancy

Perf: linear in floating objects × points × `UpdateRate`. The server reports it in `ocean:GetStats()`.

| Path | Default | Range | Side | Applies | Description |
| --- | --- | --- | --- | --- | --- |
| `Buoyancy.Enabled` | true | | server | live | Floating objects (`OceanFloat` tag) |
| `Buoyancy.UpdateRate` | 60 | 1 – 240 | server | live | Hz |
| `Buoyancy.HeightIterations` | 3 | 0 – 8 | server | live | Iterations to invert the horizontal displacement (2–3 is exact for Gerstner) |
| `Buoyancy.WaterDensity` | 1 | | server | live | Relative |
| `Buoyancy.DefaultDrag` | 0.8 | | server | live | Used when an object has no `Drag` attribute |
| `Buoyancy.DefaultAngularDrag` | 0.6 | | server | live | Used when an object has no `AngularDrag` attribute |

## Look (surface colour and transparency)

Perf: colours are computed per vertex (every frame near the camera, every `ColorInterval` frames on Low/Medium); changing them costs nothing extra. Transparency costs GPU fill rate.

| Path | Default | Range | Side | Applies | Description |
| --- | --- | --- | --- | --- | --- |
| `Look.VertexColor` | true | | client | live | false = legacy flat colour |
| `Look.Deep` | RGB(2, 22, 38) | | client | live | Troughs |
| `Look.Body` | RGB(4, 46, 62) | | client | live | Mean water |
| `Look.BodyColor` | RGB(3, 30, 44) | | client | live | Colour of the water body below the surface |
| `Look.BodyDepth` | 46 | 4 – 200 | client | live | Depth of the water body plane |
| `Look.Crest` | RGB(14, 104, 88) | | client | live | Raised water / steep faces |
| `Look.CrestStart` | -0.1 | -1 – 2 | client | live | Normalized height where the crest tint starts… |
| `Look.CrestEnd` | 1.1 | 0 – 3 | client | live | …and is full |
| `Look.Far` | RGB(5, 38, 56) | | client | live | Distant water and horizon |
| `Look.FarStart` | 120 | 50 – 3000 | client | live | Studs where the distant colour starts |
| `Look.FarEnd` | 650 | 100 – 6000 | client | live | Studs where it is full |
| `Look.Foam` | RGB(150, 205, 210) | | client | live | Compression tint (pale cyan) |
| `Look.CompressionStart` | 0.92 | 0.3 – 1.2 | client | live | Crest compression where the pale tint starts… |
| `Look.CompressionEnd` | 0.5 | 0 – 1 | client | live | …and reaches `CompressionMax` |
| `Look.CompressionMax` | 0.3 | 0 – 1 | client | live | Compression tint max |
| `Look.Scatter` | RGB(30, 160, 112) | | client | live | Subsurface colour of backlit crests |
| `Look.ScatterStrength` | 1 | 0 – 3 | client | live | Scattered light strength |
| `Look.ScatterK1` | 2.4 | 0 – 6 | client | live | Backlit crest transmission |
| `Look.ScatterK2` | 0.1 | 0 – 2 | client | live | Faces turned to the eye |
| `Look.SteepTint` | 0.45 | 0 – 1 | client | live | Steep faces toward `Crest` |
| `Look.TroughDepth` | 0.7 | 0 – 1 | client | live | How much troughs darken to `Deep` |
| `Look.TroughDeep` | RGB(2, 16, 50) | | client | live | Deeper blue at the very bottom of the waves |
| `Look.TroughDeepStart` | -0.35 | | client | live | Normalized height where it starts… |
| `Look.TroughDeepEnd` | -1.1 | | client | live | …and is full |
| `Look.TroughDeepStrength` | 0.8 | 0 – 1 | client | live | Strength of the trough blue |
| `Look.Variation` | 0.35 | 0 – 1 | client | live | Per-crest variation of the crest tint |
| `Look.Clarity.Color` | RGB(26, 120, 120) | | client | live | Clarity tint (`ColorFake` mode) |
| `Look.Clarity.Distance` | 110 | | client | live | |
| `Look.Clarity.Strength` | 0.45 | | client | live | |
| `Look.TransparencyMode` | "NearOnly" | NearOnly / Zone / Low / ColorFake | client | live | See [tutorial 5](../tutorials/05-materials-and-effects.md#transparency) |
| `Look.NearTransparency` | 0.25 | 0 – 0.6 | client | live | Transparency in calm water |
| `Look.NearOnlyTransparency` | 0.12 | 0 – 0.4 | client | live | Near transparency in bigger seas (`NearOnly`) |
| `Look.LowTransparency` | 0.06 | 0 – 0.4 | client | live | Big-sea transparency (`Low`) |
| `Look.DepthFog.Enabled` | true | | client | live | Graded sheets under the surface that fake depth seen from above |
| `Look.DepthFog.Layers` | 4 | 0 – 8 | client | live | Number of sheets (each is one part) |
| `Look.DepthFog.FirstDepth` | 2 | 0 – 30 | client | live | Studs below the deepest trough |
| `Look.DepthFog.Spacing` | 3 | 0.5 – 30 | client | live | First gap; each next gap × `SpacingRatio` |
| `Look.DepthFog.SpacingRatio` | 1.6 | | client | live | |
| `Look.DepthFog.TopColor` | RGB(12, 64, 72) | | client | live | Shallowest sheet |
| `Look.DepthFog.Transparency` | 0.75 | 0.1 – 0.95 | client | live | Mean per sheet |
| `Look.DepthFog.TroughFactor` | 0.85 | 0 – 2 | client | live | Deepest trough ≈ this × local significant wave height |
| `Look.WhitecapIntensity` | 1 | 0 – 2 | client | live | Preset-level multiplier of the whitecaps |
| `Look.Whitecaps.Enabled` | true | | client | live | Foam on crests (per vertex) |
| `Look.Whitecaps.Mode` | "Height" | | client | live | "Height" (stable when the camera moves) or "Jacobian" |
| `Look.Whitecaps.Color` | RGB(232, 242, 245) | | client | live | |
| `Look.Whitecaps.Max` | 0.85 | 0 – 1 | client | live | Strongest blend toward the foam colour |
| `Look.Whitecaps.Coverage` | 0.55 | 0 – 1 | client | live | Share of crests that carry foam |
| `Look.Whitecaps.Breakup` | 0.9 | 0 – 1 | client | live | 0 = continuous stripes, 1 = patchy |
| `Look.Whitecaps.HeightStart` | 0.55 | -1 – 2 | client | live | Normalized height where foam starts… |
| `Look.Whitecaps.HeightEnd` | 1.15 | 0 – 3 | client | live | …and is full |
| `Look.Whitecaps.HeightBoost` | 0.3 | 0 – 3 | client | live | High water foams earlier |
| `Look.Whitecaps.PinchGain` | 0.35 | 0 – 1.5 | client | live | Sharp crests foam earlier |
| `Look.Whitecaps.PatchScale` | 70 | 20 – 300 | client | live | Studs per foam-patch period |
| `Look.Whitecaps.Scale` | 22 | 4 – 200 | client | live | Studs per breakup-noise period ("Jacobian" mode) |
| `Look.Whitecaps.Start` | 0.975 | 0.3 – 1.2 | client | live | Compression where foam starts ("Jacobian" mode)… |
| `Look.Whitecaps.End` | 0.8 | 0 – 1 | client | live | …and is full |
| `Look.Whitecaps.Mottle` | 0.35 | 0 – 1 | client | live | Fine breakup near the camera |
| `Look.Whitecaps.MottleFadeStart` / `MottleFadeEnd` | 20 / 45 | | client | live | Studs |
| `Look.Whitecaps.Drift` | 0.6 | 0 – 10 | client | live | Studs/s drift of the breakup pattern |
| `Look.Whitecaps.FadeStart` | 150 | 10 – 2000 | client | live | Studs where whitecaps start thinning… |
| `Look.Whitecaps.FadeEnd` | 450 | 20 – 4000 | client | live | …and hand over to the far foam |
| `Look.Whitecaps.TrailTime` | 3 | 0.05 – 20 | client | live | Seconds a vertex remembers foam (trails, wakes) |
| `Look.Whitecaps.TrailStrength` | 0.6 | 0 – 1.5 | client | live | Remembered foam strength |
| `Look.Whitecaps.WakeGain` | 1 | 0 – 20 | client | live | Foam per stud of water disturbed by hulls and splashes |
| `Look.Whitecaps.WakeMin` | 0.32 | 0 – 5 | client | live | Disturbance that produces no foam |
| `Look.Whitecaps.Far.Enabled` | true | | client | live | Statistical foam toward the horizon |
| `Look.Whitecaps.Far.Gain` | 4 | 0 – 12 | client | live | |
| `Look.Whitecaps.Far.Max` | 0.55 | 0 – 1 | client | live | Keeps distant water from turning white |
| `Look.Whitecaps.Far.Scale` | 90 | 10 – 400 | client | live | Studs per patch period |
| `Look.Whitecaps.Far.Drift` | 1.5 | | client | live | Studs/s |
| `Look.Whitecaps.Far.FadeStart` / `FadeEnd` | 900 / 2000 | | client | live | Studs |
| `Look.Texture.Enabled` | false | | client | live | Animated surface texture layer (off by default; it was the largest render-thread cost, about 4 ms per frame) |
| `Look.Texture.Layer2` | false | | client | live | Second, larger texture layer |
| `Look.Texture.Fps`, `Tile`, `Drift`, `Transparency`, `OuterTransparency`, `Layer2Scale`, `Layer2Transparency` | 12, 56, 1.6, 0.62, 0.85, 2.37, 0.84 | | client | live | Texture layer parameters (only when enabled) |

## Material and Normals

| Path | Default | Range | Side | Applies | Description |
| --- | --- | --- | --- | --- | --- |
| `Material.Roughness` | 0.07 | 0 – 1 | client | live | Low = sharp sun highlights + sky reflection |
| `Material.DeepColor`, `ShallowColor`, `FoamColor`, `ScatterColor` | RGB(10, 45, 85), RGB(30, 120, 125), RGB(235, 245, 245), RGB(40, 150, 140) | | client | live | Legacy flat colours (`Look.VertexColor = false`) |
| `Material.ScatterStrength`, `NormalStrength` | 0.5, 1 | | client | live | |
| `Material.HorizonFade` | true | | client | live | |
| `Normals.DetailMap` | true | | client | rebuild | Animated baked detail normals on the surface |
| `Normals.FlipbookFPS` | 12 | | client | rebuild | Frames per second of the detail animation |
| `Normals.Mode` | "Flipbook" | | client | live | "Flipbook" (32 baked frames swapped) or "Scroll" (one map translated) |
| `Normals.MotionCompensation` | true | | client | live | Drifts the UVs between frame swaps so ripples move continuously |
| `Normals.CompensateOuter` | true | | client | live | Also drifts the outer mesh UVs (about 3 400 extra UV writes per frame) |
| `Normals.FrameShift` | (0.516, 0.103) | | client | live | Studs per baked frame |
| `Normals.FlipX`, `FlipY` | false | | client | live | |

## Rendering

Perf: the per-quality tables (`QualityTiers`, `Rendering.*Tiers`, `Effects.Tiers`) hold mesh density, update intervals and budgets ([tutorial 4](../tutorials/04-waves-and-quality.md#quality)).

| Path | Default | Range | Side | Applies | Description |
| --- | --- | --- | --- | --- | --- |
| `QualityTiers` | per-quality table | | client | rebuild | Mesh cells, rings, update intervals, emitter caps per tier |
| `Rendering.HorizonDistance` | 12000 | 500 – 100000 | client | rebuild | Flat horizon extent (studs) |
| `Rendering.ParallelActors` | 4 | 1 – 16 | client | live | Parallel Luau actors used to evaluate the mesh |
| `Rendering.LodCut` | 2.5 | | client | live | Waves shorter than this × the local vertex spacing are skipped (anti-shimmer) |
| `Rendering.FadeStart` | 0.55 | 0 – 1 | client | live | Fraction of the mesh extent where waves start fading to flat |
| `Rendering.FadeEnd` | 0.97 | 0 – 1 | client | live | Fraction where the surface is flat (meets the horizon) |
| `Rendering.ChopFadeStart` / `ChopFadeEnd` | 70 / 200 | 0 – 5000 | client | live | Short-wave band fade (FFT layers) |
| `Rendering.FarRingStart` | 3 | 1 – 16 | client | live | Rings from this index update every `FarRingInterval` frames |
| `Rendering.BoundsHeight` | 60 | 1 – 1000 | client | live | Vertical bounds for frustum culling |
| `Rendering.TileSize` | 64 | 8 – 1024 | client | live | Studs per repeat of the surface maps |
| `Rendering.Horizon.RoughnessBoost` | 0 | 0 – 1 | client | live | |
| `Rendering.Horizon.FadeToSky` | 0 | | client | live | |
| `Rendering.NearFieldEnabled` | false | | client | live | CPU foam overlay textures (off by default; foam is per vertex) |
| `Rendering.NearField.Mode` | "varfix+bicubic" | bilinear / bicubic / varfix / varfix+bicubic | client | live | Overlay magnification filter |
| `Rendering.NearField.EdgeFade`, `VarianceMip`, `RingFactor` | 0.3, 0, 2 | | client | live | Overlay parameters |
| `Rendering.FoamLook.*` | Enabled true, Opacity 0.9, Lace 0.75, … | | client | live | Look of the foam overlay (only when the overlay is enabled) |
| `Rendering.Scatter.Enabled` | false | | client | live | Legacy scattered-light overlay (scatter is now per vertex) |
| `Rendering.Scatter.*` | k1 1, k2 0.25, k3 0.12, k4 0.6, Strength 0.5, GlintGain 0 | | client | live | Overlay parameters |
| `Rendering.FarFieldEnabled` | false | | client | live | Far-field foam layer |
| `Rendering.FarFoamGain` | 1 | 0 – 8 | client | live | |
| `Rendering.LegacyFoamOverlay` | false | | client | live | |
| `Rendering.NearFieldTiers`, `FarFieldTiers` | per-quality table | | client | live | |

## Foam

| Path | Default | Range | Side | Applies | Description |
| --- | --- | --- | --- | --- | --- |
| `Foam.Enabled` | true | | client | live | |
| `Foam.Intensity` | 1 | 0 – 5 | client | live | |
| `Foam.JacobianThreshold` | 0.9 | -1 – 1.5 | client | live | Compression below this produces foam |
| `Foam.Decay` | 0.4 | 0 – 10 | client | live | Per second |
| `Foam.Gain`, `CrestBoost`, `Coverage`, `Detail` | 4, 0.5, 1, 0.9 | | client | live | |

## Effects

Perf: bounded by `Effects.Budget.MaxMs` per frame; particles are capped per quality tier (`Effects.Tiers`).

| Path | Default | Range | Side | Applies | Description |
| --- | --- | --- | --- | --- | --- |
| `Effects.Budget.MaxMs` | 2 | 0.3 – 10 | client | live | Effects time budget (ms/frame) |
| `Effects.Tiers` | per-quality table | | client | live | |
| `Effects.EWave.Enabled` | true | | client | live | Interactive wake surface |
| `Effects.EWave.Intensity` | 0.35 | 0 – 4 | client | live | Hull force / splash impulse strength |
| `Effects.EWave.Choppiness` | 1 | 0 – 3 | client | live | |
| `Effects.EWave.Damping` | 0.2 | 0 – 3 | client | live | |
| `Effects.EWave.Lateral` | true | | client | live | Lateral displacement (one extra FFT) |
| `Effects.Foam.Enabled` | true | | client | live | Simulated foam energy |
| `Effects.Foam.Intensity` | 1 | 0 – 3 | client | live | |
| `Effects.Foam.Threshold` | 0.9 | 0 – 1.2 | client | live | |
| `Effects.Foam.Dissipation` | 0.9 | 0.05 – 5 | client | live | 1/s |
| `Effects.Foam.Blur` | 0.18 | 0 – 0.5 | client | live | |
| `Effects.Foam.Advection` | 1 | 0 – 3 | client | live | |
| `Effects.Foam.CrestGain`, `DetailGain`, `SplashGain`, `WakeGain` | 1, 0.4, 1, 1 | 0 – 4 | client | live | |
| `Effects.Interaction.Boats`, `Characters`, `Objects` | true | | client | live | What disturbs the water |
| `Effects.Interaction.HullShadows` | false | | client | live | Submerged hull parts cast a shadow on the water |
| `Effects.Interaction.MaxSources` | 12 | 1 – 64 | client | live | |
| `Effects.Interaction.Range` | 260 | 20 – 2000 | client | live | Studs from the camera |
| `Effects.ObjectSpray.Enabled`, `Intensity` | true, 1 | 0 – 3 | client | live | |
| `Effects.Wake.Enabled`, `Intensity` | true, 1 | 0 – 3 | client | live | |
| `Effects.Wake.Legacy` | false | | client | live | Old flat foam ribbons in addition |
| `Effects.WaveSpray.Enabled` | true | | client | live | Wave-top spray |
| `Effects.WaveSpray.Intensity` | 1 | 0 – 3 | client | live | |
| `Effects.WaveSpray.SizeScale` | 1 | 0.3 – 3 | client | live | |
| `Effects.WaveSpray.WindInfluence` | 1 | 0 – 3 | client | live | |
| `Effects.WaveSpray.JacobianThreshold` | 0.62 | 0.3 – 1.2 | client | live | Whitecap candidate threshold |
| `Effects.WaveSpray.MinCrest`, `MinEnergy`, `PlungeThreshold` | 0.3, 0.35, 0.75 | | client | live | |
| `Effects.BeachFoam.Enabled` | true | | client | live | Beach edge and wash |
| `Effects.BeachFoam.Intensity`, `Range`, `EdgeDepth`, `EdgeCap`, `EdgeFoam`, `WashDepth`, `WashFoam`, `WashCooldown`, `MaxSlope` | 1, 150, 0.7, 0.75, 0.22, 1.6, 0.6, 2.4, 32 | | client | live | |
| `Effects.ShoreImpact.Enabled` | true | | client | live | Spray on cliffs and docks |
| `Effects.ShoreImpact.ProbesPerFrame` | 90 | | client | live | Probes tested per frame |
| `Effects.ShoreImpact.Range`, `MinWallAngle`, `MinSpeed`, `MinHeight`, `MinSize`, `MaxReach`, `ProbeSpacing`, `Cooldown`, `Intensity` | 260, 55, 2.2, 0.15, 0.8, 10, 3, 0.9, 1 | | client | live | |

## Spray and Wake

| Path | Default | Range | Side | Applies | Description |
| --- | --- | --- | --- | --- | --- |
| `Spray.Enabled` | true | | client | live | Crest spray particles (none on Low) |
| `Spray.Rate` | 30 | 0 – 500 | client | live | |
| `Spray.JacobianMax` | 0.35 | -1 – 2 | client | live | Crest compression needed to emit |
| `Spray.SteepnessThreshold` | 0.55 | 0 – 2 | client | live | |
| `Spray.Lifetime`, `MaxDistance`, `WindInfluence` | 1.4, 400, 1 | | client | live | |
| `Wake.Enabled` | true | | client | live | |
| `Wake.SplashThreshold` | 12 | | client | live | Vertical speed (studs/s) that triggers a splash |
| `Wake.MaxTrails`, `TrailLifetime`, `FoamPerSpeed` | 24, 8, 0.06 | | client | live | |

## Lighting

| Path | Default | Range | Side | Applies | Description |
| --- | --- | --- | --- | --- | --- |
| `Lighting.Enabled` | true | | client | live | false = the engine leaves `Lighting` alone |
| `Lighting.Transition` | 6 | | client | live | Seconds to blend lighting on a preset change |
| `Lighting.EnvironmentSpecularScale` | 1 | | client | live | Sky reflection strength on the water |
| `Lighting.EnvironmentDiffuseScale` | 1 | | client | live | |
| `Lighting.Atmosphere.Density`, `Offset`, `Haze`, `Glare` | 0.35, 0.25, 1.6, 0.2 | | client | live | |
| `Lighting.Atmosphere.Color`, `Decay` | RGB(190, 208, 225), RGB(106, 112, 125) | | client | live | |
| `Lighting.Bloom.Intensity`, `Size`, `Threshold` | 0.6, 24, 2 | | client | live | |
| `Lighting.SunRays.Intensity`, `Spread` | 0.02, 0.6 | | client | live | |
| `Lighting.ColorCorrection.Brightness`, `Contrast`, `Saturation`, `TintColor` | 0, 0.05, 0.05, RGB(255, 255, 255) | | client | live | |

## Underwater

Perf: only while the camera is underwater. Caustics are 16 SpotLights; shafts are one Beam each; particles are capped by the rates.

| Path | Default | Range | Side | Applies | Description |
| --- | --- | --- | --- | --- | --- |
| `Underwater.Enabled` | true | | client | live | |
| `Underwater.Extinction` | (0.03, 0.0085, 0.0052) | | client | live | Absorption per stud for R, G, B |
| `Underwater.FogColor` | RGB(58, 140, 142) | | client | live | Light just under the surface |
| `Underwater.AbyssColor` | RGB(4, 16, 34) | | client | live | Where no light reaches |
| `Underwater.FogDensity` | 0.745 | 0 – 1 | client | live | Fog near the surface (≥ 0.76 hides the seafloor's streaming edge) |
| `Underwater.DeepFogDensity` | 0.79 | 0 – 1 | client | live | Fog in the deep |
| `Underwater.DensityDepth` | 70 | | client | live | Studs over which density moves to the deep value |
| `Underwater.Haze` | 2.2 | 0 – 10 | client | live | |
| `Underwater.Glare` | 0.7 | 0 – 2 | client | live | Fog glow toward the sun |
| `Underwater.DepthDarkening` | 0.2 | 0 – 1 | client | live | |
| `Underwater.Tint`, `Saturation`, `Contrast`, `Brightness` | RGB(190, 238, 230), -0.12, 0.06, 0.07 | | client | live | Colour grading |
| `Underwater.Blur`, `DeepBlur` | 1.5, 2 | 0 – 20 | client | live | Camera blur |
| `Underwater.LightShafts` | true | | client | live | Refracted light shafts |
| `Underwater.ShaftCount` | 22 | 0 – 40 | client | live | |
| `Underwater.ShaftBrightness`, `ShaftTransparency`, `ShaftWidth`, `ShaftLength`, `ShaftRadius`, `ShaftFadeDepth`, `ShaftColor`, `ShaftGlints`, `VolumeShafts` | 1.6, 0.3, 6, 85, 55, 90, RGB(220, 250, 240), true, 3 | | client | live | |
| `Underwater.Caustics` | true | | client | live | Light patches on the seabed (SpotLights) |
| `Underwater.CausticBrightness` | 3.2 | 0 – 8 | client | live | |
| `Underwater.CausticGrid`, `CausticSpacing`, `CausticAngle`, `CausticColor`, `CausticFlicker` | 4, 22, 48, RGB(200, 245, 238), 0.025 | | client | live | |
| `Underwater.SnellWindow`, `SnellColor`, `SnellTransparency` | true, RGB(215, 245, 240), 0.35 | | client | live | |
| `Underwater.SurfaceSparkles`, `SparkleRate` | true, 140 | | client | live | |
| `Underwater.Particles`, `ParticleRate`, `ParticleBox` | true, 40, 70 | 0 – 300 | client | live | |
| `Underwater.MarineSnow`, `SnowRate`, `SnowBox` | true, 22, 160 | | client | live | |
| `Underwater.GlintRate` | 5 | | client | live | |
| `Underwater.SunRays` | 0.06 | | client | live | SunRays intensity when the sun is seen through the surface |
| `Underwater.UndersideMode` | "Mesh" | | client | live | "Mesh" or "Sheet" |
| `Underwater.CeilingColor`, `CeilingTransparency` | RGB(118, 192, 200), 0.3 | | client | live | Used by "Sheet" |
| `Underwater.TransitionRate` | 9 | | client | live | 1/s, how fast the look follows the camera through the surface |
| `Underwater.WaterlineClearance`, `WaterlineHysteresis` | 0.7, 0.6 | | client | live | Waterline guard |
| `Underwater.DeepFogColor`, `DeepTint`, `ShaftDeepColor`, `AttenuationDepth` | | | client | live | Kept for older presets; not used by the current model |

## Seafloor

Perf: the biggest memory user (terrain + decor instances). `BudgetMs` caps generation time per frame; `ViewChunks` loads (2n+1)² chunks.

Any `Seafloor.*` change applies live: the seafloor around the camera is unloaded and regenerated, spread over the next frames by `BudgetMs`. The same seed always rebuilds the same chunks. In the in-game panel these settings are in the **Underwater** tab (sections *Seafloor generation* and *Sea plants and corals*), and their sliders apply when released.

| Path | Default | Range | Side | Applies | Description |
| --- | --- | --- | --- | --- | --- |
| `Seafloor.Enabled` | true | | client | live | Procedural terrain and decor under the sea |
| `Seafloor.Seed` | 20261117 | 0 – 99999999 | client | live | Same seed = same chunks; a new seed regenerates the seafloor |
| `Seafloor.ChunkSize` | 128 | 64 – 512 | client | live | Studs, multiple of 4 |
| `Seafloor.ViewChunks` | 2 | 1 – 5 | client | live | Chunks loaded around the camera in each direction |
| `Seafloor.UnloadMargin` | 1 | | client | live | Extra ring kept before unloading |
| `Seafloor.BudgetMs` | 2.5 | | client | live | Generation time per frame |
| `Seafloor.ActiveHeight` | 40 | | client | live | New chunks only while the camera is below SeaLevel + this |
| `Seafloor.MinDepth`, `MaxDepth` | 48, 120 | 5 – 400, 10 – 600 | client | live | Floor depth range below the sea level |
| `Seafloor.HillScale`, `DetailScale`, `RidgeScale`, `RidgeHeight` | 460, 95, 300, 20 | | client | live | Floor shape |
| `Seafloor.Intensity` | 1 | 0 – 4 | client | live | Object count multiplier |
| `Seafloor.RockDensity`, `CoralDensity`, `VegetationDensity`, `PropDensity` | 1 | 0 – 4 | client | live | |
| `Seafloor.Clustering`, `DensityVariation`, `ScaleVariation` | 0.75, 0.6, 1 | | client | live | |
| `Seafloor.SwayEnabled`, `SwayRadius` | true, 110 | 20 – 400 | client | live | Vegetation sway near the camera underwater |
| `Seafloor.Categories` | list (7) | | client | live | Asset categories (names in `Ocean.SeafloorAssets`) |
| `Seafloor.Islands`, `Shallows` | lists | | client | live | Raised ground (islands, piers) |

## Weather

Perf: rain particles (`RainRate` at full storm) and post effects.

| Path | Default | Range | Side | Applies | Description |
| --- | --- | --- | --- | --- | --- |
| `Weather.Enabled` | true | | client | live | Local storm |
| `Weather.Center` | (0, 0, 0) | | client | live | Storm distance is measured from here |
| `Weather.StormStart` | 820 | 0 – 5000 | client | live | Studs where the weather starts turning… |
| `Weather.StormFull` | 1180 | 0 – 6000 | client | live | …and is a full storm |
| `Weather.Force` | -1 | -1 – 1 | client | live | ≥ 0 forces the storm strength everywhere |
| `Weather.TransitionRate` | 0.8 | | client | live | 1/s smoothing |
| `Weather.Rain`, `RainRate`, `RainSpeed`, `RainBox`, `RainSplashes` | true, 1400, 95, 110, 40 | 0 – 4000 | client | live | |
| `Weather.Lightning`, `LightningFrom`, `LightningInterval` | true, 0.6, list | | client | live | |
| `Weather.CloudCover`, `CloudDensity`, `CloudColor` | 0.92, 0.9, RGB(70, 76, 86) | | client | live | |
| `Weather.FogDensity`, `FogHaze`, `FogColor`, `FogDecay` | 0.55, 4, RGB(150, 158, 170), RGB(116, 124, 136) | | client | live | |
| `Weather.Brightness`, `Contrast`, `Saturation`, `Tint`, `Specular` | -0.04, 0.08, -0.45, RGB(205, 214, 226), 0.4 | | client | live | |

## Demo (swimming, boats, demo platform)

| Path | Default | Range | Side | Applies | Description |
| --- | --- | --- | --- | --- | --- |
| `Demo.SwimEnabled` | true | | client | live | Swimming (`OceanSwim`) |
| `Demo.SwimSpeed` | 17 | 4 – 40 | client | live | Horizontal studs/s |
| `Demo.SwimSprint` | 1.6 | 1 – 3 | client | live | Multiplier while holding LeftShift |
| `Demo.SwimVerticalSpeed` | 13 | 2 – 30 | client | live | Space rises, LeftControl / C dives |
| `Demo.SwimResponse` | 5.5 | 1 – 20 | client | live | Velocity smoothing |
| `Demo.SwimHeadOffset` | 1.15 | 0 – 3 | client | live | Root height below the surface when floating |
| `Demo.SwimMaxDepth` | 140 | 5 – 200 | client | live | Studs below the surface |
| `Demo.SwimDepthFactor` | 1.15 | | client | live | Shallower water is waded instead of swum |
| `Demo.WadeSlowdown` | 0.45 | | client | live | |
| `Demo.SwimCameraPitch` | true | | client | live | Looking down while swimming forward dives |
| `Demo.SwimPitchDeadzone`, `SwimPitchDeadzoneDeep` | 0.45, 0.15 | 0 – 0.95 | client | live | |
| `Demo.BoatThrust` | 24 | 4 – 80 | client | live | Studs/s² at full throttle |
| `Demo.BoatMaxSpeed` | 52 | 10 – 120 | client | live | |
| `Demo.BoatTurnRate` | 0.9 | 0.1 – 3 | client | live | rad/s |
| `Demo.BoatLateralDrag` | 3.2 | 0 – 12 | client | live | Keel |
| `Demo.BoatBrake` | 3 | 0 – 12 | client | live | |
| `Demo.Platform*`, `FloaterBuoyancy` | | | client | live | The demo's buoyancy test platform |

## Debug

| Path | Default | Range | Side | Applies | Description |
| --- | --- | --- | --- | --- | --- |
| `Debug.Panel` | false | | client | restart | Show the in-game panel outside Studio (e.g. to benchmark on a phone) |
| `Debug.ShowPerformance` | false | | client | live | Performance overlay |
| `Debug.ShowZones`, `ShowNormals`, `ShowVertices`, `ShowFoamAreas`, `ShowBuoyancyPoints` | false | | client | live | Debug overlays (cost while on) |
