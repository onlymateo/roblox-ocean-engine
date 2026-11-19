# API reference — `Ocean.OceanEngine` 2.0.0

`ReplicatedStorage.Ocean.OceanEngine` is the only public module. Everything else under `ReplicatedStorage.Ocean` is internal and may change between versions.

The same module runs on the server and on clients. Each side does its own job: the server owns settings and buoyancy; clients render. Every function and example below was exercised in a Studio play session on 2026-11-19.

```lua
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local OceanEngine = require(ReplicatedStorage:WaitForChild("Ocean"):WaitForChild("OceanEngine"))
```

## Module functions and fields

| Member | Returns | Description |
| --- | --- | --- |
| `OceanEngine.new(options?)` | handle | Creates a handle. Nothing starts until `:Start()` |
| `OceanEngine.Running()` | handle or `nil` | A handle on the engine already started on this side, or `nil` before any `Start()` |
| `OceanEngine.Describe(path)` | `{ default, rule, serverAuthoritative }` | The default value, the validation rule (`t`, `min`, `max`, `enum`) and whether the path is server-authoritative |
| `OceanEngine.PickAutoTier()` | `"Low"` … `"Ultra"` | The tier `Quality = "Auto"` resolves to on this device. Client |
| `OceanEngine.Presets` | `{ string }` | `Calm`, `Normal`, `Rough`, `Storm`, `IslandDemo`, `Custom` |
| `OceanEngine.Qualities` | `{ string }` | `Low`, `Medium`, `High`, `Ultra`, `Auto` |
| `OceanEngine.Version` | `"2.0.0"` | |
| `OceanEngine.Settings` | module | The internal settings module. Advanced use only |

## Options — `OceanEngine.new(options)`

| Option | Side | Type / values | Default | Description |
| --- | --- | --- | --- | --- |
| `Preset` | server | one of `OceanEngine.Presets` | the preset in `Config.Project` / `Config` | Applied at `Start()` without cross-fade |
| `Quality` | client | one of `OceanEngine.Qualities` | `Config.Project` / `Config` (`High`) | Applied at `Start()` |
| `Settings` | both | `{ ["Path.To.Value"] = value }` | none | Applied with `:Set` at `Start()`, in the same way as runtime overrides |
| `RemoteAccess` | server | `"Studio"` / `"Creator"` / `"None"` / `function(player): boolean` | `"Studio"` | Who may change the sea from a client through `Ocean.OceanRemote` ([tutorial 8](../tutorials/08-client-and-server.md#who-may-change-the-sea-from-a-client)) |
| `Buoyancy` | server | boolean | `true` | `false` skips the server buoyancy runner |

Options only apply on the first `Start()` on each side.

## Methods

| Method | Side | Returns | Description |
| --- | --- | --- | --- |
| `ocean:Start()` | both | `ocean` | Starts this side's engine. Idempotent: later calls do nothing |
| `ocean:Set(path, value)` | both | `ok, err?` | Validated override. Server: everyone, physics included. Client: this player only (a server path changes only this player's picture, with a warning) |
| `ocean:Get(path)` | both | value | The resolved value in effect on this side. `Waves.Current.*` holds read-only derived values (wavelength, frequency, speed, period) |
| `ocean:Reset(path?)` | both | — | Removes one override, or all of this side's overrides. A client keeps its quality |
| `ocean:SetPreset(name, seconds?)` | both | boolean | Server: cross-fades everyone over `seconds`. Client: asks the server (subject to `RemoteAccess`). `false` for an unknown name |
| `ocean:SetQuality(tier)` | client | boolean | Rebuilds this player's mesh at the new tier |
| `ocean:GetHeight(x, z)` | both | number | True displaced water height at world X/Z. Before the field exists, returns `General.SeaLevel` |
| `ocean:GetSurfaceData(position)` | both | table | `Position`, `Height`, `Normal`, `Velocity`, `Displacement`, `IsUnderwater`, `Depth` (positive under water), `Jacobian`, `FoamScale` |
| `ocean:IsUnderwater(position)` | both | boolean | `position.Y` below the true surface |
| `ocean:SetVisible(on)` | client | — | Shows or hides this player's ocean |
| `ocean:GetStats()` | both | table | Client: `fps`, `oceanMs`, `meshMs`, `effectsMs`, `vertices`, `triangles`, `quality`, `text`. Server: `objects`, `stepMs`, `fieldMs`, `objectsMs`, `usPerObject`, `steps` |
| `ocean:Destroy()` | both | — | Stops this side. Client: removes every ocean instance |

## Signals

| Signal | Arguments |
| --- | --- |
| `ocean.Changed` | `(resolvedConfig, changedPaths: { string })` |
| `ocean.PresetChanged` | `(name, transitionSeconds, startServerTime)` |

## Remote protocol

`OceanServer` creates `ReplicatedStorage.Ocean.OceanRemote` (RemoteEvent). Clients may fire:

| Message | Arguments | Effect if allowed |
| --- | --- | --- |
| `"Preset"` | `name, seconds?` | `SetPreset`. Cross-fade clamped to 0–60 s, default 6 |
| `"Set"` | `path, value` | `Set`, validated by the schema |
| `"Reset"` | `path?` | `Reset` |
| `"Zones"` | `list` | `Set("Zones.List", list)` |

Requests from players that `RemoteAccess` does not allow are ignored.

## Examples

```lua
-- server
local ocean = OceanEngine.Running()
ocean:SetPreset("Storm", 10)
local ok, err = ocean:Set("Wind.Speed", 18)    --> true
ocean:Set("Wind.Speed", 999)                   --> false, "Wind.Speed: 999 > max 40"
ocean:Reset("Wind.Speed")
print(ocean:GetSurfaceData(Vector3.new(300, -5, 300)).Depth)
```

```lua
-- client
local ocean = OceanEngine.Running()
ocean:Set("Look.NearTransparency", 0.3) -- this player only
ocean:SetQuality("Medium")
print(ocean:GetStats().text)
```

## Plugin core (Studio)

The plugin's buttons call a UI-free module, `PluginCore`. It ships inside the plugin file. In the demo place, a copy also lives at `ServerStorage.OceanEngineTools.PluginCore`, so you can script the same actions from the command bar: `local Core = require(game.ServerStorage.OceanEngineTools.PluginCore)`.

| Function | Description |
| --- | --- |
| `Core.Status()` | `{ installed, package, items = { { level, text } } }` |
| `Core.Install({ Swimming, Boats, Preset, Quality })` | `ok, { messages }` |
| `Core.ReadProject()`, `Core.WriteProject(project)` | Read or write `Config.Project` (validated) |
| `Core.Validate(path, value)` | `ok, err` |
| `Core.Defaults()` | Engine defaults without the project layer |
| `Core.StartPreview()`, `Core.StopPreview()`, `Core.PreviewRunning()` | Edit-mode preview |
| `Core.Remove()` | Moves the ocean to `ServerStorage.OceanEngine_Removed_<date>` |
| `Core.ExportPackage()` | Creates `ServerStorage.OceanEnginePackage` |
