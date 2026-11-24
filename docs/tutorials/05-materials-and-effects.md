# 5. Materials and effects

Every setting on this page is a **client** setting:

- Set in `Config.Project`, it is every player's default.
- Set on the server at runtime, it applies to everyone.
- Set on a client, it applies to that player only.

All of them apply live. The full list, with ranges, is in the [reference](../reference.md#settings).

## Water colour

The surface colour is computed per vertex every frame from the wave height, the crest compression and the distance. The colours are **albedos**: Roblox's PBR lighting then lights them, so keep them darker than the colour you want on screen.

| Path | Default | Role |
| --- | --- | --- |
| `Look.Deep` | RGB(2, 22, 38) | Troughs |
| `Look.Body` | RGB(4, 46, 62) | Mean water |
| `Look.Crest` | RGB(14, 104, 88) | Raised water and steep faces |
| `Look.Far` | RGB(5, 38, 56) | Distant water and horizon (`Look.FarStart` 120 → `Look.FarEnd` 650 studs) |
| `Look.Scatter`, `Look.ScatterStrength` | RGB(30, 160, 112), 1 | Light passing through thin, backlit crests |
| `Look.TroughDepth` | 0.7 | How much troughs darken toward `Deep` |
| `Look.Variation` | 0.35 | Per-crest variation of the crest tint |
| `Material.Roughness` | 0.07 | Low = sharp sun highlights and sky reflection |
| `Lighting.EnvironmentSpecularScale` | 1 | Strength of the sky reflection |

```lua
-- Config.Project: a greener, tropical sea
Overrides = {
	["Look.Body"] = Color3.fromRGB(6, 60, 58),
	["Look.Crest"] = Color3.fromRGB(20, 130, 100),
	["Look.ScatterStrength"] = 1.4,
},
```

## Transparency

Roblox does not sort the triangles inside one transparent mesh. In big waves, a far wave can therefore show *through* a nearer one. `Look.TransparencyMode` controls how the ocean avoids this:

| Mode | Behaviour |
| --- | --- |
| `NearOnly` (default) | Only the inner mesh (about 100 studs around the camera) is see-through: `NearTransparency` in calm water, `NearOnlyTransparency` in bigger seas. The outer mesh is always opaque. Recommended |
| `Zone` | `NearTransparency` in calm zones, fading to opaque from a normal sea up |
| `Low` | Like `Zone`, but big seas keep `LowTransparency` |
| `ColorFake` | Opaque everywhere, with a turquoise "clarity" tint near the camera that imitates transparency |

`Look.NearTransparency` (0.25, range 0–0.6) sets how much of the seabed shows through calm water. Transparency costs fill rate on the GPU, especially on phones.

## Foam

| Path | Default | Role |
| --- | --- | --- |
| `Look.Whitecaps.Enabled` | true | Foam on crests, per vertex |
| `Look.WhitecapIntensity` | 1 (0–2) | Overall amount; presets set it |
| `Look.Whitecaps.Coverage` | 0.55 | Share of crests that carry foam |
| `Look.Whitecaps.Max` | 0.85 | Strongest blend toward the foam colour |
| `Look.Whitecaps.TrailTime` | 3 s | How long foam lingers behind crests and wakes |
| `Look.Whitecaps.Far.Enabled` | true | Statistical foam patches toward the horizon |
| `Effects.BeachFoam.Enabled` | true | White edge and wash on beaches near the camera |

## Spray, wakes and impacts

| Path | Default | Role | Cost |
| --- | --- | --- | --- |
| `Spray.Enabled`, `Spray.Rate` | true, 30 | Spray particles on breaking crests | Particles; emitters capped per quality tier (Low: none) |
| `Effects.WaveSpray.Intensity` | 1 | Emission multiplier for wave-top spray | Particles |
| `Wake.Enabled` | true | Boat wakes and splash rings | Small |
| `Effects.EWave.Enabled` | true | Interactive wake surface around hulls and swimmers | CPU, inside `Effects.Budget.MaxMs` |
| `Effects.ShoreImpact.Enabled` | true | Spray when crests hit cliffs, rocks and docks | Probes per frame (`ProbesPerFrame` 90) |
| `Effects.Budget.MaxMs` | 2 ms | Per-frame time budget shared by the effects | — |

## Beach waves

`Extensions.Surf` makes the waves break on sloping beaches. At startup it builds a map around each island (distance to the shore, depth, shore direction) from the seafloor's own floor function; it takes about 0.1 s and runs behind the loading screen.

Near the shore, waves:

- face the beach and travel at shallow-water speed, so they get shorter as the water shallows;
- grow (Green's law) and break when taller than `BreakIndex` × depth;
- lean forward before breaking, and the break peels along the beach;
- turn into whitewater that runs up the sand and drains back, with a white line at the water's edge;
- glow turquoise in shallow water, with spray where the lip breaks.

Cliffs (`MaxSlope`) get no surf; crests hitting them use `Effects.ShoreImpact`.

| Path | Default | Role |
| --- | --- | --- |
| `Surf.Enabled` | true | Beach waves |
| `Surf.Period` / `Surf.Height` | 4.5 s / 2.2 studs | Time between breakers / their height where the surf zone starts |
| `Surf.MaxDistance` | 150 | Width of the surf zone (studs from the shore) |
| `Surf.BreakIndex` | 0.78 | Breaks when height > this × depth |
| `Surf.Steepness`, `Surf.PeelLength` | 0.9, 240 | Forward lean; studs along the shore per phase cycle (how the break peels) |
| `Surf.SwashHeight`, `Surf.EdgeFoam`, `Surf.FoamGain` | 0.7, 0.85, 1.2 | Run-up on the sand, foam line, whitewater |
| `Surf.ShallowColor`, `Surf.ShallowDepth`, `Surf.ShallowStrength` | RGB(64, 210, 220), 14, 0.75 | Turquoise shallow water |
| `Surf.Spray` | true | Spray on breaking crests |

The surf is visual and client-side, like boat wakes: swimmers ride it, but floating objects and the server physics use the open-sea waves only. A height field cannot fold over, so the curling lip of a real breaker is suggested by the steep front, whitewater and spray. Cost measured in Studio: about +1 ms per frame with the camera at the beach.

## Lighting

The engine drives `Lighting` from `Config.Lighting` merged with the preset. The Atmosphere, Bloom, SunRays and ColorCorrection effects tween over `Lighting.Transition` (6 s) when the preset changes, and the original values are restored when the ocean stops.

| Path | Default |
| --- | --- |
| `Lighting.Atmosphere.Density` / `Haze` / `Glare` | 0.35 / 1.6 / 0.2 |
| `Lighting.Bloom.Intensity` | 0.6 |
| `Lighting.SunRays.Intensity` | 0.02 |
| `Lighting.ColorCorrection.Contrast` / `Saturation` | 0.05 / 0.05 |
| `Lighting.Enabled` | true. Set `false` to keep your own Lighting |

Roblox notes that some post-processing effects work differently, or not at all, at low graphics quality levels ([PostEffect](https://create.roblox.com/docs/reference/engine/classes/PostEffect)). Do not rely on bloom or sun rays for gameplay readability.

## A per-player "graphics" menu

These are per-player settings, so a LocalScript can expose them in your own settings menu:

```lua
-- LocalScript: called by your settings menu
local function setOceanDetail(level: string)
	if level == "low" then
		ocean:SetQuality("Low")
		ocean:Set("Spray.Enabled", false)
		ocean:Set("Underwater.ShaftCount", 8)
	else
		ocean:SetQuality("High")
		ocean:Reset("Spray.Enabled")
		ocean:Reset("Underwater.ShaftCount")
	end
end
```

Next: [6. Underwater](06-underwater.md).
