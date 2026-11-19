# 4. Waves and quality

## How settings stack

Every value is resolved in this order. The last one wins.

1. **Engine defaults** (`Ocean.Config` and its child modules).
2. **The preset** (`Ocean.Presets.<Name>`). The starting preset is the one named in `Config.Project`; `SetPreset` changes it at runtime.
3. **Your game's overrides**: the `Overrides` table of `Config.Project`. They sit above every preset. A value you set there stays when the preset changes, so leave a value out of `Config.Project` if you want presets to control it.
4. **Runtime overrides** set by the server (`ocean:Set` on the server), replicated to everyone.
5. **Per-player overrides** set on a client (`ocean:Set` on a client), for that player only.

`ocean:Reset(path)` removes a runtime override, so the value falls back to layers 1–3.

Wave settings are **server-authoritative**. They change the wave field and therefore the physics, so set them on the server. The full list of server paths is in [tutorial 8](08-client-and-server.md).

## Presets

| Preset | Sea |
| --- | --- |
| `Calm` | Lagoon: small, round waves, almost no foam |
| `Normal` | Open sea: the recommended start for a new game |
| `Rough` | Big waves, whitecaps, spray |
| `Storm` | Very big, sharp waves, heavy foam and spray, darker lighting |
| `IslandDemo` | The demo's storm wave set, calmed near the island by two zones |
| `Custom` | Starts from the engine defaults only |

Set the starting preset in `Config.Project` (`Preset = "Normal"`). Change it while playing from the server:

```lua
ocean:SetPreset("Storm", 10) -- seconds of cross-fade; waves, foam and lighting blend, nothing jumps
```

`SetPreset` returns `false` for an unknown name and keeps the current sea.

## Wave controls

| Path | Default | Range | What it does |
| --- | --- | --- | --- |
| `Wind.Speed` | 10 | 0.5 – 40 (m/s) | Wave height and length for spectrum presets |
| `Wind.Direction` | (1, 0.35) | Vector2, normalized | Main direction the waves travel |
| `Gerstner.AmplitudeScale` | 1 | 0 – 5 | Multiplies every wave's height |
| `Gerstner.Steepness` | 0.75 | 0 – 1.5 | 0 = round swells, higher = sharper crests (normalized, never loops) |
| `Gerstner.Spread` | 6 | 0.5 – 40 | How much the waves fan out around the wind direction (higher = more aligned) |
| `Gerstner.WaveCount` | 16 | 1 – 64 | Number of waves; more = less visible repetition, linear CPU cost |
| `Waves.Wavelength` | *natural* | 4 – 1500 studs | Crest-to-crest distance of the dominant wave. Rescales the whole set and keeps the height |
| `Waves.Frequency` | *natural* | 0.05 – 25 | Crests per 100 studs. Writes `Waves.Wavelength = 100 / f` |
| `Waves.Speed` | *natural* | 0 – 200 studs/s | Holds the crest speed constant, whatever the wavelength |
| `General.TimeScale` | 1 | 0 – 10 | Multiplies the natural wave speed; clears `Waves.Speed` |

```lua
-- server
ocean:Set("Wind.Speed", 18)
ocean:Set("Waves.Wavelength", 120)
print(ocean:Get("Waves.Current.Wavelength")) --> 120 (read-only: the values actually in effect)
ocean:Reset("Waves.Wavelength") -- back to the natural wavelength
```

`ocean:Set` validates against the schema and returns `ok, err`. Invalid values are refused and the old value stays:

```lua
local ok, err = ocean:Set("Wind.Speed", 999)
print(ok, err) --> false  Wind.Speed: 999 > max 40
```

Wavelength and frequency changes cross-fade over 1.5 s. Speed changes re-anchor the shared clock, so no wave jumps.

## Wave model

`Algorithm` selects the wave model for both the visuals and the physics:

| Algorithm | Notes |
| --- | --- |
| `Gerstner` (default) | Analytic sum of waves. Exact at any point, the cheapest, and the model the demo is tuned and tested with |
| `FFT` | Spectrum on a periodic grid (`FFT.*`), recomputed `General.SimulationFPS` times per second on the server and on every client |
| `Hybrid` | FFT plus a few long Gerstner swells |

Measured in Studio (80 × 80 grid, median of 9 runs): evaluating the surface costs about 0.39 µs per vertex for Gerstner, 0.24 µs for FFT and 0.45 µs for Hybrid. FFT and Hybrid also pay a 4–4.5 ms grid update at each simulation step.

## Sea-state zones

Zones scale the preset's waves down locally. Outside every zone the preset applies at 100 %. Declare them in config, not at runtime, so the server physics, the clients and the parallel workers all start with them:

```lua
-- Config.Project
return {
	Preset = "Rough",
	Overrides = {
		["Zones.Enabled"] = true,
		["Zones.List"] = {
			{ Shape = "Circle", Position = Vector3.new(0, 0, 0), Radius = 500, Transition = 150, Priority = 2,
				SwellScale = 0.1, ChopScale = 0.15, FoamScale = 0.2, SprayScale = 0.1 },
		},
	},
}
```

| Zone field | Meaning |
| --- | --- |
| `Shape` | `"Circle"` (with `Radius`) or `"Box"` (with `Size` Vector2 and `Rotation`) |
| `Position` | Centre of the zone |
| `Transition` | Width in studs of the fade toward the outside |
| `Priority` | The higher zone wins where two zones overlap |
| `SwellScale`, `ChopScale` | Scale the long swell and the short chop |
| `Choppiness` | Rounds or sharpens the crests |
| `FoamScale`, `SprayScale` | Amount of foam and spray |

The demo uses a calm lagoon (radius 560, swell 0.09, chop 0.12) and a normal sea (radius 1250, swell 0.33, chop 0.45).

## Quality

Quality changes **rendering cost only**: mesh density, how often far rings update, colour update interval, effect budgets. The wave field and the physics are identical on every tier.

| Tier | Vertices | Triangles (High = 9 144) | Spray emitters | Wake emitters | Detail normals |
| --- | --- | --- | --- | --- | --- |
| `Low` | 2 129 | — | 0 (spray off) | 4 | 32 px |
| `Medium` | 3 229 | — | 8 | 8 | 32 px |
| `High` | 4 681 | 9 144 | 16 | 16 | 32 px |
| `Ultra` | 6 453 | — | 28 | 24 | 64 px |
| `Auto` | picks one of the above when the game starts (below) | | | | |

Vertex counts were measured in Studio after a live rebuild.

**Auto:**

- Phones and tablets (touch screen, no keyboard) get `Medium`.
- Otherwise the player's Roblox graphics level decides: 1–3 `Low`, 4–6 `Medium`, 7–9 `High`, 10 `Ultra`.
- With the slider on "Automatic", Auto picks `High`.

The rule is in `OceanEngine.PickAutoTier()`.

Quality is a per-player choice. Set the default in `Config.Project`, and change it on a client:

```lua
-- LocalScript
ocean:SetQuality("Medium") -- rebuilds this player's ocean mesh (about a second)
```

Next: [5. Materials and effects](05-materials-and-effects.md).
