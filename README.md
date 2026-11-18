# Roblox Ocean Engine

A real-time ocean for Roblox: open-ocean waves rendered on a custom mesh, physics that floats objects on the exact surface you see, boats you can drive, swimming, underwater lighting and a local storm.

This repository holds the **setup tutorial only**. The source code is not published here.

## Features

- **Floating objects (WaveRect).** A rectangle's four corners are placed on the live waves; the model takes their height and tilt.
- **Sea-state zones.** A calm lagoon around the island, a normal sea further out, and the full wave set beyond, blended smoothly.
- **Local storm.** Overcast sky, fog, rain, lightning and storm waves, driven by distance from the island.
- **Underwater.** Light shafts refracted by the waves, the surface seen from below, moving caustic patches on the seafloor, depth fog.
- **Swimming.** Characters float and ride the waves, wade in shallow water, and leave textured foam.
- **Drivable boats.** Throttle, rudder and keel; foam trail and a wake that pushes swimmers and floating props. Water is kept out of the hull.
- **Live tuning.** Press F2 in play to tune waves, wind, foam, spray and colours with sliders.

Previews are coming soon.

---

## Setup tutorial

To add the ocean to a game you copy one folder and two scripts, then configure everything in `Ocean.Config` without editing the modules.

### 1. Installation

Copy these from the demo place into your game, at the same locations. The rest of the place (`Workspace.OceanDemo`, `OceanDemoServer`, `OceanDemoClient`) is demo content only.

| Item | Location | Required |
| --- | --- | --- |
| `Ocean` folder (Config, Core, Rendering, Physics, Extensions, Assets, Presets…) | ReplicatedStorage | Yes |
| `OceanServer` (Script) | ServerScriptService | Yes |
| `OceanClientBoot` (LocalScript) | StarterPlayerScripts | Yes |
| `OceanSwim` (LocalScript) | StarterPlayerScripts | If players swim |
| `OceanBoatDriver` (LocalScript) | StarterPlayerScripts | If you have drivable boats |

Keep `Ocean.Assets` (surface textures) and `Ocean.SeafloorAssets` (corals, rocks, seaweed): they are looked up by name. Do not use Terrain water: the ocean is a mesh, and swimming is handled by `OceanSwim`.

### 2. Startup

The two required scripts are enough: the server owns the settings and the physics, and each client draws its own surface.

- `OceanServer` publishes the initial settings to clients and starts the floating-object runner.
- `OceanClientBoot` starts the renderer, the effects and the extensions.

Server and clients compute the same waves from the same clock (`workspace:GetServerTimeNow()`) and the same `Seed`. What floats therefore matches what is drawn, without replicating waves over the network.

`OceanServer` also accepts preset and zone changes through `Ocean.OceanRemote`, but only in Studio or for the UserIds listed in `ALLOWED_USER_IDS`. Fill that list (or remove the remote) before publishing.

### 3. Basic settings

Everything is configured in `ReplicatedStorage.Ocean.Config`. Values stack in this order, the last one wins: **Config** (defaults) → **Preset** → **runtime overrides**.

| Key | Values | Notes |
| --- | --- | --- |
| `Preset` | `Calm`, `Normal`, `Rough`, `Storm`, `Custom`, `IslandDemo` | The demo uses `IslandDemo` (storm waves scaled down by zones). For a new game, start with `Normal`. |
| `Quality` | `Low`, `Medium`, `High`, `Ultra`, `Auto` | Only changes rendering cost (mesh density, effects). Physics is identical on every tier. |
| `Algorithm` | `Gerstner`, `FFT`, `Hybrid` | `Gerstner` is the lightest and the one tested in the demo. |
| `General.SeaLevel` | studs | Y height of the sea at rest. |
| `General.Seed` | integer | Must match on server and client (it does if you only change it here). |
| `Wind.Speed`, `Wind.Direction` | m/s, Vector2 | Wave height and direction for spectrum-based presets. |
| `Look.*` | colours, transparency | Trough, crest and foam colours; `TransparencyMode = "NearOnly"` by default. |

Module settings live in the ModuleScripts under `Config`: `Config.Look`, `Config.Effects`, `Config.Underwater`, `Config.Weather`, `Config.Seafloor`, `Config.Demo`, `Config.Rendering`, `Config.Waves`. Each has a validation schema, so an out-of-range value is rejected instead of breaking the render.

Tip: press **F2** in play for the debug panel. Its sliders change the same keys live; copy the values you like back into `Config`.

### 4. Sea-state zones

Zones scale the preset's waves down locally; outside every zone the preset applies at 100 %. The demo places a calm lagoon and a normal sea around the island; beyond them is the full storm of the `IslandDemo` preset.

Declare them in `Config.Zones.List`, not at runtime, so the server physics, the clients and the parallel workers all start with them.

| Zone field | Meaning |
| --- | --- |
| `Shape` | `Circle` (with `Radius`) or `Box` (with `Size` Vector2 and `Rotation`). |
| `Position` | Centre of the zone. |
| `Transition` | Width in studs of the fade toward the outside. |
| `Priority` | The higher zone wins where two zones overlap. |
| `SwellScale`, `ChopScale` | Scale the long swell and the short chop. |
| `Choppiness` | Rounds or sharpens the crests. |
| `FoamScale`, `SprayScale` | Amount of foam and spray. |

Demo values: calm lagoon radius 560 (swell 0.09, chop 0.12), normal sea radius 1250 (swell 0.33, chop 0.45).

The storm weather (`Config.Weather`) follows the distance from the centre with `StormStart` (820) and `StormFull` (1180). If you move or resize the zones, move these two values too.

### 5. Floating objects and boats

An object floats as soon as it has the `OceanFloat` tag and the attribute `BuoyancyMode = "WaveRect"`. It then follows the displayed surface exactly: the four corners of a rectangle are placed on the waves and the model takes their height and tilt.

**A floating prop (crate, raft, buoy)**

1. A Model with a `PrimaryPart`.
2. Add a Part named `WaveRect` inside the model, placed at the waterline; its X/Z size is the footprint that rides the waves. Set `Transparency = 1` and `CanCollide = false`.
3. Tag `OceanFloat` + attribute `BuoyancyMode = "WaveRect"`.

| Attribute (on the model) | Default | Effect |
| --- | --- | --- |
| `WaveRectOffset` | 0 | Studs added to the height: positive rides higher. |
| `TiltScale` | 1 | 0 stays level, 1 follows the wave slope exactly. |
| `MaxTilt` | 45 | Maximum tilt in degrees. |
| `Responsiveness` | 0 anchored / 40 unanchored | Smoothing (1/s). |
| `Drag`, `AngularDrag` | 0.6 / 1.2 | Unanchored: water drag when nobody drives. |
| `WaveRectPart` | `WaveRect` | Name of another Part to use as the rectangle. |
| `WaveRectSide` | client | `"Server"` to compute an anchored object on the server. |

Anchored = moved every frame on each client (smoothest, best for scenery). Unanchored = a real physics body held by AlignPosition/AlignOrientation, which can be pushed or driven.

**A drivable boat**

1. Same setup, **unanchored** model, with a `VehicleSeat`.
2. Also add the `OceanBoat` tag. The water is then pushed out of the hull and the wake is generated.
3. Driving settings as attributes: `BoatThrust` (demo: 24), `BoatMaxSpeed` (52), `BoatTurnRate`.
4. On the server, when a player sits: give the hull's network ownership to that player and set the attributes `OwnerUserId = player.UserId` and `Driven = true`. When they leave: clear `OwnerUserId`, set `Driven = false` and return network ownership to automatic. The demo's server script does this; reproduce it in your own server script.

`OceanBoatDriver` then handles propulsion, rudder and keel on the driver's client.

### 6. Optional modules

Underwater, weather, seafloor and effects each have an `Enabled` switch in their config; turn off what your game doesn't need. Swimming has none: to remove it, delete the `OceanSwim` script.

| Module | Config | What it does | Key settings |
| --- | --- | --- | --- |
| Underwater | `Config.Underwater` | Fog, light shafts, Snell window, caustic light patches on the floor, glare | `Caustics`, `CausticBrightness` (3.2), `CausticAngle` (48), `Glare`, `SunRays`, `ShaftCount` |
| Weather | `Config.Weather` | Overcast sky, rain, lightning by distance from the centre | `StormStart`, `StormFull`, `FogDensity` |
| Seafloor | `Config.Seafloor` | Chunked underwater terrain and decor (deterministic from `Seed`), islands | `Seed`, `Categories[...].Assets` (names in `SeafloorAssets`) |
| Effects | `Config.Effects` | Spray, wakes, textured foam, impacts on cliffs | Per-quality tiers, `ShoreImpact.MinWallAngle` (55°) |
| Swimming | `Config.Demo` | Swimming, wading in shallow water | `SwimDepthFactor` (1.15), `WadeSlowdown` (0.45) |

If your game already has its own terrain, turn `Seafloor` off or set its categories so it only places decor.

### 7. Runtime control and common pitfalls

Change the sea in game from the **server**: changes replicate to clients and waves cross-fade without jumping. The settings module exposes `ApplyPreset(name, seconds)`, `Set(path, value)`, `Reset(path)` and `SetQuality(tier)`. The floating-object runner exposes the shared wave field, whose `HeightAt(x, z)` and `GetSurfaceData(position)` give the true water height, normal, velocity and depth at any point (for gameplay, AI or drowning checks).

**Common pitfalls**

- **Values changed during Play are ignored:** the modules are already loaded. Stop the test, edit `Config`, run again.
- **Object flips at startup:** the `WaveRect` is misplaced (not at the waterline) or too small for the model.
- **Water visible inside the hull:** the `OceanBoat` tag (or the `HullMask = true` attribute) is missing, or the `WaveRect` sticks out past the hull walls.
- **Bands or glitches with transparency:** keep `Look.TransparencyMode = "NearOnly"` (transparency only near the camera).
- **Very low FPS in Studio:** the Studio window doesn't have focus. Always measure with the window active.
- **Anyone can change the weather:** check `ALLOWED_USER_IDS` in `OceanServer` before publishing.
