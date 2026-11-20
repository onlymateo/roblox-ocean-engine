# 1. Install the plugin

The Studio plugin (toolbar **Ocean Engine**) **contains the engine**: modules, textures, seafloor assets and start scripts. Installing the plugin is all a game needs, because **Create Ocean** installs the engine from the plugin itself.

The plugin file is not published in this repository. Install it from Roblox (Toolbox or Creator Store) if it has been published there, or as a local file from the author.

## 1. Install the plugin

**From Roblox** (Toolbox › Inventory, or the Creator Store page): click **Install**. The plugin appears in the **Plugins** tab.

**From a file** (`OceanEnginePlugin.rbxm` or `.rbxmx`):

1. In Studio, open the **Plugins** tab and click **Plugins Folder**. This opens the local plugins folder:
   - Windows: `%LOCALAPPDATA%\Roblox\Plugins`
   - macOS: `~/Documents/Roblox/Plugins`
2. Copy the file into that folder. Keep only one version of the plugin there.
3. Restart Studio.

Either way, the **Ocean Engine** toolbar appears in the Plugins tab, with an **Ocean** button. Check the Output window for:

```
[Ocean Engine plugin] loaded v1.1.0
```

The **Ocean** button opens a docked panel with three tabs:

| Tab | What it does |
| --- | --- |
| **Overview** | Shows what is installed and what is missing. Validates `Config.Project`. Warns about Terrain water and about a permissive `RemoteAccess`. Has the **Create Ocean** button. |
| **Configure** | The most useful settings, each with a description, default, range, owner (server or client), when it applies, and its performance impact. Edits are staged until you click **Save to Config.Project**. **Restore defaults** clears every custom value (click twice to confirm). |
| **Tools** | **Preview ocean** shows the live ocean in Edit mode, never saved with the place. **Remove Ocean** moves everything to `ServerStorage` (click twice; reversible). **Export package** creates `OceanEnginePackage` to reuse in another place. |

Every plugin action is recorded in Studio's undo history, so Ctrl+Z undoes it.

In a place without the ocean, the Overview tab says *"The engine (v2.0.0) is bundled with this plugin: click Create Ocean to install it."* Continue with [tutorial 2](02-your-first-ocean.md).

## 2. Enable the Mesh / Image APIs

The ocean surface is an `EditableMesh`. Roblox documents that *"using EditableMesh fails by default in published games"* until the experience enables the **Mesh / Image APIs** setting ([EditableMesh reference](https://create.roblox.com/docs/reference/engine/classes/EditableMesh)).

1. Publish the place, then open **Experience Settings** (Game Settings) in Studio.
2. In **Security**, turn on **Allow Mesh / Image APIs** and save. The Creator Dashboard has the same toggle under **Configure › Settings**.

Roblox requires the experience owner to be 13+ and ID verified to turn it on. For a group game, that is the group owner ([Experience settings](https://create.roblox.com/docs/studio/experience-settings)). Studio play tests work without it. If you skip this step, a published game shows no ocean surface.

## 3. Optional: an engine package

A package (`OceanEnginePackage`) is only needed to install an engine **other than the one bundled with the plugin**, for example the engine of another place you modified.

- **Make one:** in a place that has the ocean, open **Tools › Export package**. The plugin copies the engine to `ServerStorage.OceanEnginePackage`, with generic start scripts and `OceanSwim` / `OceanBoatDriver` if the place has them. Studio then asks where to save `OceanEnginePackage.rbxm`; if the dialog does not open, right-click the folder and choose **Save to File…**. The package never contains the source place's `Config.Project`, remotes or runtime attributes.
- **Use one:** drag the `.rbxm` into Studio, or right-click `ServerStorage` › **Insert from File…**. A package in the place (`ServerStorage`, `Workspace`, `ReplicatedStorage` or `Lighting`) takes priority over the bundled engine when you click **Create Ocean**. You can delete it after installation.

Next: [2. Your first ocean](02-your-first-ocean.md).
