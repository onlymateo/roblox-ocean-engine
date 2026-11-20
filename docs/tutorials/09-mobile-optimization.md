# 9. Mobile optimization

> **Status:** these recommendations come from measurements in Roblox Studio on a desktop PC and from how each feature works. They have **not yet been verified on a phone**. Run the on-device benchmark below on your target phones and adjust.

## 1. Let Auto pick the tier

Keep `Quality = "Auto"` in `Config.Project`. On a touch device without a keyboard, Auto picks **Medium**: 3 229 vertices instead of 4 681 on High, and 8 spray emitters instead of 16. On desktop, it follows the player's Roblox graphics level.

Players can still choose another tier in the in-game panel (`Geometry › Quality / LOD`) or in your own menu (`ocean:SetQuality`).

## 2. Know what costs what

Highest cost first, from Studio measurements and from how each feature works:

| Feature | Cost driver | Phone setting |
| --- | --- | --- |
| Ocean mesh | Vertices updated per frame (EditableMesh writes cost script time *and* engine render time every frame) | Quality `Medium` or `Low` |
| Seafloor | Terrain writes while streaming and thousands of decor instances; the biggest memory user | `Seafloor.ViewChunks` 1, or `Seafloor.Enabled = false` if your game has its own floor |
| Wave model | `Gerstner.WaveCount` waves evaluated per vertex | 12 instead of 16 |
| Transparency | Fill rate: overlapping transparent pixels are shaded several times | Keep `Look.TransparencyMode = "NearOnly"` |
| Caustics | 16 SpotLights under the surface while underwater | `Underwater.Caustics = false`, or `Underwater.Lights = false` for every underwater light |
| Light shafts | One textured Beam per shaft while underwater | `Underwater.ShaftCount` 8–12 |
| Rain | `Weather.RainRate` 1 400 particles/s at full storm | 400–600 |
| Spray | Particle emitters (none on Low) | `Spray.Rate` 15 |
| Effects budget | `Effects.Budget.MaxMs` ms per frame for wakes, foam and impacts | 1 |

Roblox's guidance on the same topics:

- *"Draw calls have significant overhead."*
- Overlapping partially transparent objects *"forces the engine to render the overlapping pixels multiple times"*.

Both are from [Improve performance](https://create.roblox.com/docs/performance-optimization/improve).

## 3. A phone profile in your game

Settings set from a LocalScript apply to that player only. One way to give phones a lighter profile:

```lua
-- StarterPlayerScripts/OceanMobileProfile (LocalScript)
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local UserInputService = game:GetService("UserInputService")
local OceanEngine = require(ReplicatedStorage:WaitForChild("Ocean"):WaitForChild("OceanEngine"))

local ocean = OceanEngine.Running()
while not ocean do
	task.wait()
	ocean = OceanEngine.Running()
end

if UserInputService.TouchEnabled and not UserInputService.KeyboardEnabled then
	ocean:Set("Seafloor.ViewChunks", 1)
	ocean:Set("Underwater.Caustics", false)
	ocean:Set("Underwater.ShaftCount", 10)
	ocean:Set("Weather.RainRate", 500)
	ocean:Set("Effects.Budget.MaxMs", 1)
end
```

`Gerstner.WaveCount` is a server setting: it changes the waves for everyone, physics included. Lower it in `Config.Project` if your game targets phones first.

## 4. Measure on the phone

The in-game panel works on touch screens:

- **Portrait or narrow screens:** a bottom sheet covering half the screen.
- **Landscape:** a side panel.
- **Touch sizing:** rows are 48–56 px tall, controls at least 44 px, and the panel stays inside the device's safe area.

To measure on your phone:

1. In `Config.Project`, set `["Debug.Panel"] = true` so the panel also exists in live servers. Publish.
2. Open the game on the phone. Tap **Ocean › Performance › Run device benchmark (~70 s)**, and do not touch the screen while it runs.
3. The results appear in a text box. Select the text to copy it.

The benchmark runs these scenarios:

- still, camera spin, orbit, moving, character swimming, close-up, high altitude, underwater, storm
- Ultra quality, applied temporarily

For each scenario it reports:

- frame time (mean, p95, max), FPS and 1 % low
- render CPU time
- ocean, mesh and effects script time
- vertices updated per frame
- memory

The header records the device's viewport, orientation, quality, place version and loading times.

GPU time and CPU or GPU utilisation are reported as unavailable where Roblox does not expose them on the device. See the [reference](../reference.md#performance-on-the-development-pc) for the PC measurements.

Turn `Debug.Panel` off again before release, unless you want players to have it.

## 5. Other checks

- Test in **portrait and landscape**: the panel's layout and the demo's teleport bar adapt to both.
- Keep an eye on memory: the Developer Console (F9 on desktop, or the in-game menu on mobile) shows memory by category.
- Roblox gives Editable objects strict client-side memory budgets (*"EditableMesh has strict client-side memory budgets, although the server, Studio, and plugins operate with unlimited memory"*, [EditableMesh](https://create.roblox.com/docs/reference/engine/classes/EditableMesh)). So a phone can hit a limit that Studio never shows.

Next: [10. Troubleshooting](10-troubleshooting.md).
