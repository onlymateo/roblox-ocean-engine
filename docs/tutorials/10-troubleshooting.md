# 10. Troubleshooting

Start with **Ocean Engine › Ocean › Overview › Refresh**. It lists missing pieces, invalid `Config.Project` values, Terrain water and a permissive `RemoteAccess`.

## Installation

| Symptom | Cause | Fix |
| --- | --- | --- |
| No **Ocean Engine** toolbar | The plugin file is not in the local Plugins folder, or Studio was not restarted | [Tutorial 1](01-install-the-plugin.md); look for `[Ocean Engine plugin] loaded v1.0.0` in Output |
| *"No engine to install"* when clicking Create Ocean | An old plugin version without the bundled engine (before v1.1.0), and no `OceanEnginePackage` in the place | Update the plugin, or insert `OceanEnginePackage.rbxm` |
| Two **Ocean Engine** toolbars | Two versions of the plugin installed (for example a local file and the Toolbox version) | Keep one: delete the other file from the Plugins folder, or uninstall it in the Plugins tab |
| Ocean visible in Studio but **missing in the published game** | Mesh / Image APIs not allowed for the experience | Experience Settings › Security › **Allow Mesh / Image APIs** ([tutorial 1](01-install-the-plugin.md#3-enable-the-mesh--image-apis)) |
| `Config.Project has invalid values: ...` | A value outside its range, or a typo in a path | Fix the value, or use **Configure › default** on that row |
| `[Ocean.Config] Config.Project failed to load` | Syntax error in `Config.Project` | Open the module and fix the error shown in Output |

## Settings

| Symptom | Cause | Fix |
| --- | --- | --- |
| A value changed in `Config` during Play has no effect | Modules are already loaded | Stop, edit `Config.Project`, run again. At runtime use `ocean:Set` |
| `[Ocean.Settings] rejected Wind.Speed: 999 > max 40` | Out-of-range value | Use the range from the [reference](../reference.md#settings). `ocean:Set` returns `false, err` |
| *"'Wind.Speed' was set on a client: only this player's picture changes"* | A wave setting was set from a LocalScript | Set wave settings on the server ([tutorial 8](08-client-and-server.md)) |
| The panel says *"Wave settings are controlled by the server"* | `RemoteAccess` does not allow you in a live game | Expected for players. For the owner, use `RemoteAccess = "Creator"` |
| A preset change ignores one of my `Config.Project` values | It doesn't: `Config.Project` overrides sit above presets. A runtime `ocean:Set` on the server can still replace them | Check for runtime `Set` calls; `ocean:Get(path)` shows the value in effect |
| A second `OceanEngine.new({...}):Start()` ignores its options | `Start` is idempotent: the first one wins | Use `OceanEngine.Running()` and call methods on it |

## Visuals

| Symptom | Cause | Fix |
| --- | --- | --- |
| A wave visible *through* another wave, bands in big seas | Roblox does not sort the triangles of one transparent mesh | Keep `Look.TransparencyMode = "NearOnly"`, or use `"ColorFake"` |
| A second, flat water surface | Terrain water under the ocean | Remove the Terrain water |
| Water inside a boat hull | Missing `OceanBoat` tag (or `HullMask = true` attribute), or `WaveRect` sticks out of the hull | Add the tag; keep the rectangle inside the hull walls |
| Underwater: dark band toward the horizon | Fog too thin to hide the end of the streamed seafloor | `Underwater.FogDensity` ≥ 0.76 |
| No sun rays or bloom on some devices | Roblox disables or changes some post effects at low graphics quality | Expected; don't rely on them for readability |
| `ContentId: rbxassetid://203785492 failed due to: Asset type does not match requested type` | A pre-existing asset reference in the demo place | Harmless for the ocean |

## Physics

| Symptom | Cause | Fix |
| --- | --- | --- |
| An object flips at start | `WaveRect` misplaced (not at the waterline) or much smaller than the model | Place the rectangle at the waterline, about the hull's footprint |
| An object sinks | `Buoyancy` too low for its mass, or `Draft` larger than the part | Raise `Buoyancy`, or set `Draft` |
| An anchored `Physical` object does nothing | `Physical` needs an unanchored assembly | Unanchor it, or use `"Kinematic"` / `"WaveRect"` |
| Boat does not respond | `OwnerUserId` / `Driven` not set, or network ownership not given | See the server script in [tutorial 7](07-buoyancy.md#drivable-boats) |

## Performance

| Symptom | Cause | Fix |
| --- | --- | --- |
| Very low FPS in Studio | The Studio window doesn't have focus | Always measure with the window active |
| FPS drifts down during a long play test | Observed in Studio for every version measured, independent of the ocean changes | Compare versions in fresh play sessions, with the same scenario order |
| Low FPS on phones | Too much geometry or effects for the device | [Tutorial 9](09-mobile-optimization.md); run the device benchmark |
| Stutter when entering new seafloor areas | Seafloor chunks generating | Lower `Seafloor.BudgetMs` (spreads work over more frames) or `Seafloor.ViewChunks` |

## Diagnostics

- **In-game panel** (Studio, or with `Debug.Panel = true`): live FPS and ocean timings in the header; **Performance › Run device benchmark**; **Debug** has overlays for zones, normals, vertices and foam areas.
- **`ocean:GetStats()`**:
  - client: `fps`, `oceanMs`, `meshMs`, `effectsMs`, `vertices`, `triangles`, `quality` and a `text` summary
  - server: the buoyancy statistics
- **Developer Console** (F9) and **MicroProfiler** (Ctrl+Alt+F6 in Studio) for engine-side costs ([identify performance issues](https://create.roblox.com/docs/performance-optimization/identify)).
