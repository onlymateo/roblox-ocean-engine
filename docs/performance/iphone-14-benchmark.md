# iPhone 14 benchmark

**Status (2026-11-19): not measured on an iPhone 14.** No physical iPhone 14 was available. No iOS device, emulator or cloud device service was used either. This page therefore contains **no iPhone results**.

It provides:

- the procedure and the tool to measure on the device;
- the scenarios and the metrics, with whether each metric is available on iOS;
- desktop Studio measurements, clearly labelled, as a reference point only;
- an empty results table to fill in from a real run.

Labels used on this page:

| Label | Meaning |
| --- | --- |
| **Device** | Measured on the physical phone, in the Roblox app, on a live server |
| **Studio** | Measured in Roblox Studio on a desktop PC (play test, client) |
| **Emulator** | Studio's device emulation (screen size / touch only: it does **not** emulate the phone's CPU or GPU) |
| **Estimate** | Reasoned expectation, not a measurement |

## Device

Apple's published specifications for the iPhone 14 ([Apple Support: iPhone 14 tech specs](https://support.apple.com/en-us/111850)):

- A15 Bionic chip:
  - 6-core CPU: 2 performance and 4 efficiency cores
  - 5-core GPU
  - 16-core Neural Engine
- 6.1-inch display, 2532 × 1170 pixels at 460 ppi, 60 Hz.

Apple does not publish the amount of RAM.

## How to run the benchmark on the phone

1. In the place's `Config.Project`, set `["Debug.Panel"] = true` so the in-game panel exists in live servers. Publish.
2. Make sure the Mesh / Image APIs are allowed for the experience ([tutorial 1](../tutorials/01-install-the-plugin.md#3-enable-the-mesh--image-apis)).
3. Prepare the phone:
   - Charge it above 50 % and turn Low Power Mode off.
   - Close other apps.
   - Let it cool to room temperature, and keep it out of direct sun and off chargers. Thermal state changes results and cannot be read from a Roblox script.
4. In the Roblox app, set the graphics quality you want to test (Settings › Graphics Quality). Note it.
5. Join the game. Wait 30 s for loading to settle.
6. Tap **Ocean** › **Performance** › **Run device benchmark (~70 s)**. Do not touch the screen until the panel reopens.
7. Select the result text in the panel and copy it into the table below, together with the iOS version, Roblox app version, orientation and graphics level.
8. Repeat in the other orientation, and after 10 minutes of play (warm device), to see thermal throttling.

The same tool can be called from a LocalScript in any test place:

```lua
local B = require(game.ReplicatedStorage.Ocean.Debug.BenchScenarios)
print(B.Run({ device = true, hold = 5 }))
```

## Scenarios

The device run uses these scenarios. Each settles for 1.5 s, then samples for 5 s. Camera scenarios follow fixed paths: the same world positions on every run.

| Scenario | What it stresses |
| --- | --- |
| `still` | Stationary camera, deep sea, looking at the horizon: the baseline |
| `spin` | Camera rotating 3 rad/s: culling and far-ring updates |
| `orbit` | Camera orbiting a point at radius 40 |
| `moving` | Camera moving at 60 studs/s: ring re-centring |
| `character` | The real character swimming with the default camera (touch controls idle) |
| `closeup` | 5 studs above the waves, looking down: fill rate, foam |
| `high` | 120 studs up: horizon, far foam |
| `underwater` | 35 studs deep: fog, light shafts, caustic SpotLights, particles |
| `storm` | Storm zone: spray, rain, lightning, foam |
| `ultra` | Quality temporarily set to Ultra: worst-case geometry |

Optional: `extended = <seconds>` adds a long slow cruise to observe drift and thermal behaviour.

## Metrics

| Metric | Source | Available in the Roblox app on iOS? |
| --- | --- | --- |
| Frame time mean / p95 / max, FPS, 1 % low FPS | `RunService.RenderStepped` delta time | Yes (any client) |
| Render CPU time | `Stats.RenderCPUFrameTime` | Documented API; reported as `n/a` if the platform returns nothing. **Not yet verified on iOS** |
| Render GPU time | `Stats.RenderGPUFrameTime` | Documented API; same caveat. **Not yet verified on iOS** |
| Ocean script time (field, mesh, effects) | The engine's own timers | Yes |
| Vertices written per frame | The engine's mesh statistics | Yes |
| Memory: total, graphics textures, graphics meshes, Lua heap | `Stats:GetTotalMemoryUsageMb()`, `Stats:GetMemoryUsageMbForTag()`, `gcinfo()` | Documented; per-tag memory needs memory tracking enabled on the client, otherwise the API returns 0 ([Stats](https://create.roblox.com/docs/reference/engine/classes/Stats)) |
| Loading times (engine start, textures, flipbook) | Engine attributes | Yes |
| CPU / GPU utilisation %, clock speed, thermal state, battery drain | — | **Not available** to experience scripts. Use Xcode Instruments on a Mac, or watch battery % over a fixed session |
| Frame pacing at 60 Hz | Frame-time p95 / max vs 16.7 ms | Yes |

## Results — device (iPhone 14)

**Not measured.** Fill in from a real run; leave cells empty rather than estimating.

| Scenario | Frame ms mean | p95 | FPS | 1 % low | Render CPU ms | Render GPU ms | Ocean ms | Memory MB |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| still | | | | | | | | |
| spin | | | | | | | | |
| orbit | | | | | | | | |
| moving | | | | | | | | |
| character | | | | | | | | |
| closeup | | | | | | | | |
| high | | | | | | | | |
| underwater | | | | | | | | |
| storm | | | | | | | | |
| ultra | | | | | | | | |

Conditions: iOS ____ · Roblox app ____ · orientation ____ · graphics level ____ · ocean quality ____ · battery ____ % · cold / warm.

## Reference — Studio on a desktop PC (not a phone)

These numbers are **Studio** measurements. They show the relative cost of scenarios and the effect of the optimizations. They do **not** predict iPhone frame rates: the CPU, GPU, memory budget and thermal limits are different, and Studio caps the frame rate at 60.

Machine: Intel Core i7-13620H, Intel UHD Graphics (as reported by Windows), 16 GB RAM, Windows 11.

Settings: Roblox Studio 0.748, play test (client), preset IslandDemo, Gerstner, Quality High, window focused.

**After the optimization pass** (fresh play session per version, same scenario order):

| Scenario | FPS | 1 % low FPS | Render CPU ms | Ocean script ms |
| --- | --- | --- | --- | --- |
| still | 60.2 | 54.0 | 6.34 | 3.22 |
| underwater | 60.0 | 53.3 | 6.59 | 3.43 |
| storm | 60.1 | 54.5 | 7.64 | 4.08 |
| seafloor exploration | 57.1 | 39.7 | 9.43 | 4.38 |

**Before the optimization pass** (same machine and method): still 59.9 FPS / 7.76 ms render CPU / 3.99 ms script; underwater 60.0 / 7.71 / 4.03; storm 59.2 / 8.85 / 4.88; seafloor 53.0 / 9.85 / 5.63.

**Latest tool check** (2026-11-19, 2 s samples, after adding GPU time to the report):

| Scenario | Frame ms mean | FPS | Render CPU ms | Render GPU ms | Ocean ms | Verts/frame |
| --- | --- | --- | --- | --- | --- | --- |
| still | 16.61 | 60.2 | 5.76 | 6.69 | 2.82 | 3 386 |
| underwater | 16.68 | 60.0 | 5.02 | 9.41 | 2.66 | 3 386 |

GPU timing in Studio varies a lot between runs. Render CPU time and medians of interleaved runs were used for the before/after comparisons.

**Quality tiers** (Studio, vertex counts after a live rebuild): Low 2 129 · Medium 3 229 · High 4 681 · Ultra 6 453. No seam gaps on any tier.

**Layout check in Studio (not a device, not Studio's device emulator):** the control panel's layout function was run with phone-sized viewports, portrait and landscape, with touch sizing forced on. Every interactive control measured at least 40 px, and the panel stayed inside the screen. This checks layout, not performance or real touch input.

## Estimate (not a measurement)

Reasoning only, to be replaced by device data:

- On a touch device, `Quality = "Auto"` selects **Medium**: 3 229 vertices, about 31 % fewer mesh writes than High.
- The ocean's main-thread cost is dominated by per-vertex work, so it should fall roughly in proportion.
- Underwater, the GPU cost of the caustic SpotLights and the transparent beams is the most likely bottleneck on a phone GPU. The Studio GPU time is already highest in that scenario.
- The seafloor's terrain and instance memory is the most likely memory constraint.

## Limitations of this benchmark

- No device results yet. This is the main open item.
- Studio results come from one desktop machine; FPS is capped at 60 in Studio.
- Thermal throttling, battery drain and utilisation cannot be read by the benchmark itself.
- Results vary with the phone's temperature, battery level, background apps and the Roblox graphics level. Compare only runs made in the same conditions.
- The camera scenarios hide the player's GUIs while sampling. Your game's own UI cost is not included.

## Recommendations for iPhone-class devices

Until device data exists, start from [tutorial 9](../tutorials/09-mobile-optimization.md):

- `Quality = "Auto"` (Medium on phones).
- Keep `Look.TransparencyMode = "NearOnly"`.
- `Seafloor.ViewChunks = 1`.
- `Underwater.Caustics = false` if underwater FPS is low.
- `Underwater.ShaftCount` around 10.
- `Weather.RainRate` around 500.

Then run the benchmark above and adjust.
