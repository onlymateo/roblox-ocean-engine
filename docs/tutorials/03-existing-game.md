# 3. Add the ocean to an existing game

This tutorial assumes your game already has a map, spawn points and its own scripts.

## 1. Check the place first

**Ocean Engine › Ocean › Overview** checks the place before you install:

- **Terrain water** near the map centre gets a warning. The ocean is a mesh, so Roblox Terrain water would draw a second surface and replace swimming. Replace the water with air, for example with the Terrain Editor's Replace tool.
- **Existing scripts** named `OceanServer`, `OceanClientBoot`, `OceanSwim` or `OceanBoatDriver` are kept, never overwritten. Create Ocean reports *"already present"* for each one.

## 2. Install

Click **Create Ocean** ([tutorial 2](02-your-first-ocean.md)). Then fit the ocean to your map:

| Your map | Setting (in Config.Project) |
| --- | --- |
| Ground or beaches at another height | `["General.SeaLevel"] = <Y of the shoreline>` |
| Your own terrain under the sea | `["Seafloor.Enabled"] = false`, or keep it and adjust `Seafloor.MinDepth` / `MaxDepth` |
| No storm area | `["Weather.Enabled"] = false` |
| The demo's calm lagoon / storm zones are not wanted | Use a preset other than `IslandDemo`, or set `["Zones.Enabled"] = false` |

`General.SeaLevel` is read when the game starts. Change it in `Config.Project`, not at runtime.

## 3. Use the ocean from your scripts

`OceanServer` and `OceanClientBoot` start the engine. Other scripts should not start it again: they use the running engine.

```lua
-- any server Script or client LocalScript
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local OceanEngine = require(ReplicatedStorage:WaitForChild("Ocean"):WaitForChild("OceanEngine"))

local ocean = OceanEngine.Running()
while not ocean do -- the start script may run after this one
	task.wait()
	ocean = OceanEngine.Running()
end
```

`OceanEngine.new(...):Start()` is idempotent. If the engine is already running on this side, a second `Start()` does nothing, and its options are ignored. To change the sea, call methods on the running handle instead:

```lua
-- server: a round system that turns the weather
local function onRoundStart()
	ocean:SetPreset("Rough", 8) -- cross-fades over 8 s for every player, physics included
end

local function onRoundEnd()
	ocean:SetPreset("Normal", 8)
end
```

```lua
-- server: hurt players whose head is under water for too long
local Players = game:GetService("Players")
local underwaterTime = {}

game:GetService("RunService").Heartbeat:Connect(function(dt)
	for _, player in Players:GetPlayers() do
		local head = player.Character and player.Character:FindFirstChild("Head")
		local humanoid = player.Character and player.Character:FindFirstChildOfClass("Humanoid")
		if head and humanoid then
			if ocean:IsUnderwater(head.Position) then
				underwaterTime[player] = (underwaterTime[player] or 0) + dt
				if underwaterTime[player] > 20 then
					humanoid:TakeDamage(10 * dt)
				end
			else
				underwaterTime[player] = 0
			end
		end
	end
end)
```

`IsUnderwater`, `GetHeight` and `GetSurfaceData` use the true displaced wave surface. On the server they read the buoyancy field; on a client they read the field that is drawn. Both give the same answer, because they share the clock and the settings.

## 4. Things to look out for

- **Spawn points under the sea level** put characters in water. With `OceanSwim`, they swim to the surface.
- **Anchored parts at the waterline**, such as docks and piers, are fine. Breaking waves spray against vertical faces (`Effects.ShoreImpact`), and beaches get foam (`Effects.BeachFoam`).
- **Your own Lighting setup.** The ocean drives `Lighting` (atmosphere, bloom, colour correction, sun rays) from `Config.Lighting` and the preset. To keep your own lighting, set `["Lighting.Enabled"] = false` in `Config.Project`; the engine then leaves those values alone. The underwater look and the storm weather have their own switches (`Underwater.Enabled`, `Weather.Enabled`).
- **Big maps.** The surface covers `General.RenderDistance` (6000 studs) around the camera, plus a flat horizon out to `Rendering.HorizonDistance` (12000 studs).

## 5. Remove the ocean

**Tools › Remove Ocean** (click twice) moves the engine and its scripts to `ServerStorage.OceanEngine_Removed_<date>` and disables the scripts. It also deletes the generated `OceanSurface`, `OceanSeafloor` and `OceanPreview` folders from Workspace.

To restore, move the items back to their original locations and re-enable the scripts, or press Ctrl+Z. To remove the ocean for good, delete that folder.

Next: [4. Waves and quality](04-waves-and-quality.md).
