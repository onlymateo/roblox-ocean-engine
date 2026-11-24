# Roblox Ocean Engine

A real-time ocean for Roblox. Open-ocean waves are rendered on a custom mesh. Physics floats objects on the exact surface you see. Includes drivable boats, swimming, underwater lighting and a local storm.

This repository holds the **documentation only**: tutorials, the API and configuration reference, performance notes and a technical report. The engine source and the Studio plugin are not published here.

## Features

- **One surface for everything.** Server and clients compute the same waves from the same clock and seed. Buoyancy, swimming and gameplay queries use the same surface the player sees.
- **Presets and zones.** Calm, Normal, Rough and Storm presets, plus sea-state zones that calm the sea locally, for example a lagoon around an island. Changes cross-fade, so waves never jump.
- **Floating objects and boats.** Tag a model to make it float. Boats have throttle, rudder and keel, a wake, and a hull that keeps the water out.
- **Beach waves.** Near a sloping shore, waves turn to face the beach, slow down, grow and break into whitewater that runs up the sand; shallow water turns turquoise.
- **Underwater.** Depth-dependent fog and colour, refracted light shafts, caustics, Snell's window and suspended particles.
- **Loading screen.** Keeps the game hidden until scripts, textures, meshes and the ocean are ready.
- **Weather.** A local storm with overcast sky, rain and lightning, driven by distance.
- **Quality tiers.** Low, Medium, High and Ultra change rendering cost only; physics is identical on every tier. `Auto` picks a tier from the player's device.
- **Studio plugin.** Create, configure, preview and remove the ocean without editing engine modules.
- **In-game control panel.** Live tuning that works on phones (portrait and landscape) and on desktop, with a built-in device benchmark.

## Quick start

1. Install the Studio plugin. It contains the engine, so there is nothing else to insert ([tutorial 1](docs/tutorials/01-install-the-plugin.md)).
2. Click **Ocean Engine › Ocean › Create Ocean**, then press Play ([tutorial 2](docs/tutorials/02-your-first-ocean.md)).

Create Ocean adds two start scripts, shown below. The server half owns the settings and the buoyancy. The client half draws the ocean.

```lua
-- ServerScriptService/OceanServer (Script)
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local OceanEngine = require(ReplicatedStorage:WaitForChild("Ocean"):WaitForChild("OceanEngine"))

OceanEngine.new({
	RemoteAccess = "Studio", -- who may change the sea from a client: "Studio" | "Creator" | "None" | function(player)
}):Start()
```

```lua
-- StarterPlayerScripts/OceanClientBoot (LocalScript)
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local OceanEngine = require(ReplicatedStorage:WaitForChild("Ocean"):WaitForChild("OceanEngine"))

OceanEngine.new():Start()
```

Your game's settings live in one module, `ReplicatedStorage.Ocean.Config.Project`. The plugin's **Configure** tab writes it, or you can edit it by hand:

```lua
return {
	Preset = "Normal",
	Quality = "Auto",
	Overrides = {
		["Wind.Speed"] = 12,
		["Look.NearTransparency"] = 0.2,
	},
}
```

Everything else goes through one public module, `Ocean.OceanEngine` ([reference](docs/reference.md#api)):

```lua
local ocean = OceanEngine.Running()          -- the engine started by OceanServer / OceanClientBoot
ocean:SetPreset("Storm", 10)                 -- server: cross-fade to a storm over 10 s, for everyone
local depth = ocean:GetSurfaceData(position).Depth
```

## Documentation

| Tutorials | |
| --- | --- |
| [1. Install the plugin](docs/tutorials/01-install-the-plugin.md) | Plugin (engine included), Mesh / Image APIs setting, optional packages |
| [2. Your first ocean](docs/tutorials/02-your-first-ocean.md) | Create, configure and preview an ocean in five minutes |
| [3. Add the ocean to an existing game](docs/tutorials/03-existing-game.md) | Terrain, spawn points, your own scripts, removing the ocean |
| [4. Waves and quality](docs/tutorials/04-waves-and-quality.md) | Presets, wind, wavelength, speed, zones, quality tiers |
| [5. Materials and effects](docs/tutorials/05-materials-and-effects.md) | Colours, transparency, foam, spray, wakes, lighting |
| [6. Underwater](docs/tutorials/06-underwater.md) | Fog, shafts, caustics, depth queries, swimming |
| [7. Buoyancy](docs/tutorials/07-buoyancy.md) | Floating props, WaveRect, boats, gameplay queries |
| [8. Client and server](docs/tutorials/08-client-and-server.md) | Who owns which setting, remote access, signals |
| [9. Mobile optimization](docs/tutorials/09-mobile-optimization.md) | Quality tiers, costly features, on-device benchmark |
| [10. Troubleshooting](docs/tutorials/10-troubleshooting.md) | Symptoms, causes and fixes |

| Reference | |
| --- | --- |
| [Reference](docs/reference.md) | API, main settings, performance on the development PC, Roblox limitations |

## Requirements

- Roblox Studio (tested with Studio 0.748, November 2026).
- **Mesh / Image APIs enabled for the experience.** The surface is an `EditableMesh`, which fails in published games until this setting is on. See [tutorial 1](docs/tutorials/01-install-the-plugin.md#3-enable-the-mesh--image-apis).
- No Terrain water where the ocean is. The ocean is a mesh, and swimming is handled by `OceanSwim`.

## Status

- Tested in Roblox Studio on Windows (play test, desktop). Every code sample in these documents was run in a Studio play session on 2026-11-19.
- **Not yet tested on a physical phone.** The mobile panel was laid out and checked at phone resolutions in Studio, and the device benchmark exists, but no results from a real iPhone have been collected yet. See the [reference](docs/reference.md#performance-on-the-development-pc).

## Contributors

- **The repository owner**: the engine, the demo place, design and direction.
- **[Claude](https://github.com/claude)** (Anthropic), working in Claude Code:
  - the circular ocean mesh, the underwater rendering and the performance work;
  - the public `OceanEngine` API and the Studio plugin;
  - the mobile control panel and the device benchmark;
  - this documentation and the platform-limitations report.
