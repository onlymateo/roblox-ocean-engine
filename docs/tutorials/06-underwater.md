# 6. Underwater

When the camera goes below the **true** wave surface (not the sea level), the client blends in the underwater look:

- Colour follows depth with a per-channel absorption model. Red goes first, so the water is teal near the surface and navy deeper down.
- Distance fog thickens with depth.
- Light shafts come in at the refracted sun angle.
- Caustic light patches dance on the seabed.
- Snell's window: looking up, the sky shows through a bright disc above you.
- Glints flicker on the underside of the surface, and small particles drift past.

A 0.8-stud band and some hysteresis at the waterline keep the look from flashing while the camera bobs.

All of it is client-side and costs nothing while the camera is above water.

## Settings

| Path | Default | Range | Role | Cost |
| --- | --- | --- | --- | --- |
| `Underwater.Enabled` | true | | Everything on this page | Only while underwater |
| `Underwater.Extinction` | (0.03, 0.0085, 0.0052) | per stud | Absorption per colour channel (R, G, B) | — |
| `Underwater.FogColor` | RGB(58, 140, 142) | | Light just under the surface | — |
| `Underwater.AbyssColor` | RGB(4, 16, 34) | | What the fog tends toward where no light reaches | — |
| `Underwater.FogDensity` / `DeepFogDensity` | 0.745 / 0.79 | 0–1 | Fog near the surface / in the deep | — |
| `Underwater.DepthDarkening` | 0.2 | 0–1 | Extra darkening with depth | — |
| `Underwater.Blur` | 1.5 | 0–20 | Camera blur (`BlurEffect`) | Post effect |
| `Underwater.LightShafts`, `ShaftCount` | true, 22 | 0–40 | Textured beams at the refracted sun angle | One Beam per shaft |
| `Underwater.ShaftBrightness` | 1.6 | | Shaft brightness | — |
| `Underwater.Caustics` | true | | Light patches: a grid of SpotLights under the surface | 16 SpotLights (`CausticGrid` 4) |
| `Underwater.CausticBrightness` | 3.2 | 0–8 | Patch brightness | — |
| `Underwater.SnellWindow` | true | | Bright disc of sky above (half-angle 48.6°) | One part |
| `Underwater.Particles`, `ParticleRate` | true, 40 | 0–300 | Suspended particles around the camera | Particles |
| `Underwater.MarineSnow` | true | | Far, sparse specks for depth parallax | Particles |
| `Underwater.UndersideMode` | `"Mesh"` | `Mesh` / `Sheet` | Surface seen from below: the real wave mesh, drawn double-sided while underwater | — |

Underwater fog density at or above about 0.76 keeps the end of the streamed seafloor (about 260 studs) fully fogged. Lower values can show the seafloor's edge as a dark band.

## Depth and gameplay

The same surface the player sees is available to scripts:

```lua
local data = ocean:GetSurfaceData(position)
-- data.Height        true water height above/below this point (studs, world Y)
-- data.Depth         how far `position` is under the surface (negative = above)
-- data.IsUnderwater  boolean
-- data.Normal        surface normal (Vector3)
-- data.Velocity      water particle velocity at the surface
-- data.Jacobian      < 1 where the crest is compressed (breaking crests foam there)
```

```lua
-- LocalScript: muffle the music while the camera is underwater
local SoundService = game:GetService("SoundService")
local RunService = game:GetService("RunService")
local music = SoundService:WaitForChild("Music") -- your Sound

RunService.RenderStepped:Connect(function()
	local under = ocean:IsUnderwater(workspace.CurrentCamera.CFrame.Position)
	music.Volume = under and 0.15 or 0.5
end)
```

## Swimming

`OceanSwim` (StarterPlayerScripts) makes characters float with their head just above the waves. They dive and rise, and wade where the water is shallow. Its settings are in `Config.Demo`:

| Path | Default | Role |
| --- | --- | --- |
| `Demo.SwimEnabled` | true | Turns swimming on or off |
| `Demo.SwimSpeed` | 17 | Horizontal speed (studs/s) |
| `Demo.SwimVerticalSpeed` | 13 | Space rises; LeftControl or C dives. Touch: Dive / Up buttons. Gamepad: L2 / R2 |
| `Demo.SwimMaxDepth` | 140 | How deep a character can dive |
| `Demo.SwimDepthFactor` | 1.15 | Swim only where the water is deeper than this × the standing height; shallower water is waded |
| `Demo.WadeSlowdown` | 0.45 | Slow-down while wading |

To remove swimming, delete `OceanSwim`.

Next: [7. Buoyancy](07-buoyancy.md).
