# 8. Client and server

## Who does what

| | Server (`OceanServer`) | Client (`OceanClientBoot`) |
| --- | --- | --- |
| Settings | Owns them. Resolves defaults → preset → `Config.Project` → runtime overrides, and replicates its overrides to every client | Follows the server's settings and adds its own per-player layer |
| Waves | Evaluates the wave field for buoyancy | Evaluates the same field for the mesh (parallel Luau workers), effects and swimming |
| Physics | `Physical` floating objects; anchored `WaveRect` with `WaveRectSide = "Server"` | `Kinematic` props, anchored `WaveRect`, the boat you drive |
| Rendering | — | Everything: mesh, foam, spray, underwater, seafloor, weather, lighting |

Server and clients compute identical waves from the same seed and a shared clock (`workspace:GetServerTimeNow()`). Waves are never sent over the network: only setting changes are replicated.

## Server settings and client settings

A path is **server-authoritative** when it changes the wave field or the physics:

- Exact paths: `Preset`, `Algorithm`, `General.TimeScale`, `General.SimulationFPS`, `General.Seed`, `General.Gravity`, `General.SeaLevel`.
- Prefixes: `Wind.*`, `Gerstner.*`, `FFT.*`, `Hybrid.*`, `Buoyancy.*`, `Waves.*`, `Zones.*`.

Every other path is a **client** (visual) setting.

| You call | On the server | On a client |
| --- | --- | --- |
| `ocean:Set(serverPath, v)` | Everyone, physics included; waves cross-fade | Only this player's *picture* changes; the server physics does not follow. A one-time warning is printed |
| `ocean:Set(clientPath, v)` | Everyone's default | This player only; survives later server changes |
| `ocean:Reset(path)` | Removes the server override | Removes this player's override |
| `ocean:SetPreset(name, s)` | Cross-fades everyone | Asks the server through `Ocean.OceanRemote` (allowed only by `RemoteAccess`) |
| `ocean:SetQuality(tier)` | — | This player's mesh is rebuilt |

To check a path in code:

```lua
local info = OceanEngine.Describe("Wind.Speed")
print(info.default, info.serverAuthoritative, info.rule.min, info.rule.max) --> 10 true 0.5 40
```

## Who may change the sea from a client

`OceanServer` creates the RemoteEvent `ReplicatedStorage.Ocean.OceanRemote`. Clients use it to ask for `Preset`, `Set`, `Reset` and `Zones` changes; the in-game panel uses it too. The server accepts a request only if the `RemoteAccess` option allows the player:

| `RemoteAccess` | Who may change the sea |
| --- | --- |
| `"Studio"` (default) | Anyone in a Studio play test; nobody in a live game |
| `"Creator"` | Studio, plus the game's owner (or the owning group's rank 255) in live games |
| `"None"` | Nobody, not even in Studio |
| `function(player) -> boolean` | Your own rule |

```lua
-- ServerScriptService/OceanServer: let a group's moderators change the weather
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local OceanEngine = require(ReplicatedStorage:WaitForChild("Ocean"):WaitForChild("OceanEngine"))

local GROUP_ID = 0 -- your group id

OceanEngine.new({
	RemoteAccess = function(player)
		return player:GetRankInGroup(GROUP_ID) >= 200
	end,
}):Start()
```

Values sent through the remote are validated by the settings schema (type, range, allowed values), exactly like `ocean:Set` on the server. Preset cross-fades are clamped to 0–60 s.

The plugin's Overview tab warns when `OceanServer` uses `"Creator"`. That is the demo's setting, so its owner can tune the sea from a phone. A released game usually wants `"Studio"`.

## Signals

```lua
ocean.Changed:Connect(function(config, changedPaths)
	-- config: the resolved settings now in effect; changedPaths: { "Gerstner.AmplitudeScale", ... }
	print("ocean settings changed:", table.concat(changedPaths, ", "))
end)
```

`ocean.PresetChanged` fires `(name, transitionSeconds, startTime)` when a preset change starts. Use it to sync your own effects with the cross-fade.

## Network cost

- A setting change on the server re-encodes the server's overrides as one JSON attribute on the `Ocean` folder. The size grows with the number of overrides, not with the map.
- The panel rate-limits slider drags on server paths to one message every 0.15 s.
- Waves themselves cost no bandwidth.

Next: [9. Mobile optimization](09-mobile-optimization.md).
