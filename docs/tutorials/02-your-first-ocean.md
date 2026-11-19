# 2. Your first ocean

Prerequisite: the plugin is installed and the engine package is in the place ([tutorial 1](01-install-the-plugin.md)).

## 1. Create the ocean

Open **Ocean Engine › Ocean › Overview**. Before you click **Create Ocean**, choose:

- **Swimming:** ON adds `OceanSwim`, so characters swim, dive and wade.
- **Drivable boats:** ON adds `OceanBoatDriver`. Your boats still need the setup from [tutorial 7](07-buoyancy.md#drivable-boats).
- **Starting preset:** Calm, Normal, Rough or Storm. Click the button to cycle through them.

Click **Create Ocean**. The plugin:

| Adds | Where | Role |
| --- | --- | --- |
| `Ocean` | ReplicatedStorage | The engine (modules, textures, assets) |
| `OceanServer` (Script) | ServerScriptService | Starts the server half: settings and buoyancy |
| `OceanClientBoot` (LocalScript) | StarterPlayerScripts | Starts the client half: rendering and effects |
| `OceanSwim`, `OceanBoatDriver` (LocalScripts) | StarterPlayerScripts | Only if you chose them |
| `Config.Project` (ModuleScript) | ReplicatedStorage.Ocean.Config | Your game's settings: the preset you picked, `Quality = "Auto"` |

The Overview tab now lists every required item as **OK**.

## 2. Press Play

The sea appears around your character. The ocean surface sits at `Y = 0` by default (`General.SeaLevel`).

In Studio, the **Ocean** button at the top-left of the game view opens the live control panel (F2 also works). Its sliders change the sea while you play. Wave settings apply for everyone and cross-fade; visual settings apply to you only. Nothing you change in the panel is saved. Copy the values you like into **Configure**.

## 3. Configure

Stop the play test and open **Configure**. Each setting shows:

```
Wind speed (m/s)
Drives the wave height and length of spectrum presets (10 = moderate breeze).
default 10 · range 0.5 .. 40 · server · applies live
performance: none
```

Change values, then click **Save to Config.Project**. Out-of-range values are refused with a message, and nothing is saved until every value is valid. **Discard unsaved changes** throws away staged edits. **default** on a row removes that one custom value.

The saved module looks like this, and you can edit it by hand:

```lua
-- ReplicatedStorage.Ocean.Config.Project
return {
	Preset = "Normal",
	Quality = "Auto",
	Overrides = {
		["Wind.Speed"] = 14, -- Wind speed (m/s)
		["Gerstner.Steepness"] = 0.9, -- Wave steepness
	},
}
```

Any setting from the [configuration reference](../reference/configuration.md) can go in `Overrides`, not only the ones the plugin shows.

## 4. Preview without playing

**Tools › Preview ocean** draws the waves in Edit mode around the Studio camera, using the current `Config.Project`.

The preview lives in `Workspace.OceanPreview` with `Archivable = false`, so it is never saved. Click **Stop preview** before publishing. Unloading the plugin also stops it.

The preview shows the wave surface only: no foam particles, underwater effects or seafloor.

## 5. Publish

1. Enable the Mesh / Image APIs ([tutorial 1, step 3](01-install-the-plugin.md#3-enable-the-mesh--image-apis)).
2. Check `RemoteAccess` in `OceanServer` ([tutorial 8](08-client-and-server.md#who-may-change-the-sea-from-a-client)). The default, `"Studio"`, means nobody can change the sea from a client in a live game.

Next: [3. Add the ocean to an existing game](03-existing-game.md).
