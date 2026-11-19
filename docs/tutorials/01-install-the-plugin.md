# 1. Install the plugin

The ocean comes in two files:

| File | What it is | Where it goes |
| --- | --- | --- |
| `OceanEnginePlugin.rbxmx` | The Studio plugin (toolbar **Ocean Engine**) | Your local Plugins folder |
| `OceanEnginePackage.rbxm` | The engine: modules, textures, seafloor assets, start scripts | Inserted once into each place that uses the ocean |

Neither file is published in this repository. Get them from the author, or make the package yourself from a place that already has the ocean (step 4).

## 1. Install the plugin file

1. In Studio, open the **Plugins** tab and click **Plugins Folder**. This opens the local plugins folder:
   - Windows: `%LOCALAPPDATA%\Roblox\Plugins`
   - macOS: `~/Documents/Roblox/Plugins`
2. Copy `OceanEnginePlugin.rbxmx` into that folder.
3. Restart Studio. The **Ocean Engine** toolbar appears in the Plugins tab, with an **Ocean** button.
4. Check the Output window for:

   ```
   [Ocean Engine plugin] loaded v1.0.0
   ```

The **Ocean** button opens a docked panel with three tabs:

| Tab | What it does |
| --- | --- |
| **Overview** | Shows what is installed and what is missing. Validates `Config.Project`. Warns about Terrain water and about a permissive `RemoteAccess`. Has the **Create Ocean** button. |
| **Configure** | The most useful settings, each with a description, default, range, owner (server or client), when it applies, and its performance impact. Edits are staged until you click **Save to Config.Project**. **Restore defaults** clears every custom value (click twice to confirm). |
| **Tools** | **Preview ocean** shows the live ocean in Edit mode, never saved with the place. **Remove Ocean** moves everything to `ServerStorage` (click twice; reversible). **Export package** creates `OceanEnginePackage` to reuse in another place. |

Every plugin action is recorded in Studio's undo history, so Ctrl+Z undoes it.

## 2. Insert the engine package

Drag `OceanEnginePackage.rbxm` into the Studio window, or right-click `ServerStorage` and choose **Insert from File…**. A folder named `OceanEnginePackage` appears.

Open **Ocean Engine › Ocean**. The Overview tab now says *"Engine package found at ServerStorage.OceanEnginePackage: click Create Ocean to install it."*

The package can live in `ServerStorage`, `Workspace`, `ReplicatedStorage` or `Lighting`. The plugin looks in all four. You can delete it after installation.

## 3. Enable the Mesh / Image APIs

The ocean surface is an `EditableMesh`. Roblox documents that *"using EditableMesh fails by default in published games"* until the experience enables the **Mesh / Image APIs** setting ([EditableMesh reference](https://create.roblox.com/docs/reference/engine/classes/EditableMesh)).

1. Publish the place, then open **Experience Settings** (Game Settings) in Studio.
2. In **Security**, turn on **Allow Mesh / Image APIs** and save. The Creator Dashboard has the same toggle under **Configure › Settings**.

Roblox requires the experience owner to be 13+ and ID verified to turn it on. For a group game, that is the group owner ([Experience settings](https://create.roblox.com/docs/studio/experience-settings)). Studio play tests work without it. If you skip this step, a published game shows no ocean surface.

## 4. Make a package from an existing place

In a place that already has the ocean, open **Tools › Export package**.

The plugin copies the engine to `ServerStorage.OceanEnginePackage`. It also adds generic start scripts and copies `OceanSwim` / `OceanBoatDriver` if the place has them. Studio then asks where to save `OceanEnginePackage.rbxm`. If the dialog does not open, right-click the folder and choose **Save to File…**.

The package never contains the source place's `Config.Project`, remotes or runtime attributes. Each game keeps its own configuration.

Next: [2. Your first ocean](02-your-first-ocean.md).
