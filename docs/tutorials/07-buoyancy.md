# 7. Buoyancy

Objects float on the **wave field**, the same function the renderer draws, evaluated with the same clock, zones and cross-fades. They never read the rendered mesh, so floating objects behave identically on every quality tier.

To make something float, tag a Model (with a `PrimaryPart`) or a BasePart with **`OceanFloat`**. The attribute `BuoyancyMode` selects how it floats:

| `BuoyancyMode` | Where it runs | Use it for |
| --- | --- | --- |
| `"Physical"` (default) | Server, for unanchored assemblies | Crates, barrels, debris that players push |
| `"Kinematic"` | Every client; the part must be anchored | Purely visual props (buoys, floating plants) |
| `"WaveRect"` | Anchored: every client each frame. Unanchored: wherever its physics runs | Boats, rafts, docks: the model takes the height and tilt of a rectangle riding the waves |

The server runner starts with `OceanServer` (`OceanEngine.new():Start()`; pass `Buoyancy = false` to skip it). The client runner starts with `OceanClientBoot`.

## A floating crate (Physical)

Created from a script (this exact code was tested in a Studio play session: the crate settled within 0.1 stud of the water height):

```lua
-- server Script
local CollectionService = game:GetService("CollectionService")

local crate = Instance.new("Part")
crate.Size = Vector3.new(4, 4, 4)
crate.Position = Vector3.new(60, 8, -260)
crate.Anchored = false
crate.Parent = workspace
crate:SetAttribute("BuoyancyMode", "Physical")
CollectionService:AddTag(crate, "OceanFloat")
```

In Studio, you can instead add the tag in the Tags section of the Properties window and the attribute in the Attributes section.

Optional attributes for `Physical` and `Kinematic`:

| Attribute | Default | Effect |
| --- | --- | --- |
| `Buoyancy` | 1 | Strength multiplier. Resting submersion = `Draft / Buoyancy` |
| `Draft` | half the primary part's height | Studs submerged at rest with `Buoyancy = 1` |
| `Mass` | the assembly's mass | Overrides the mass |
| `Drag` | `Buoyancy.DefaultDrag` (0.8) | Vertical damping, and horizontal drag toward the water's motion |
| `AngularDrag` | `Buoyancy.DefaultAngularDrag` (0.6) | Extra angular damping while submerged |
| `Responsiveness` | 1 | Spring stiffness multiplier |
| `FollowNormal` | 0 | 0–1: tilts the buoyancy force toward the surface normal, so waves push the object sideways |
| `Points` | the 4 bottom corners | `4`, `6` or `9` points on the bottom of the bounding box, or `"x,y,z;x,y,z;..."` local offsets. Attachments named `BuoyancyPoint*` are used automatically |

Use at least 3 non-collinear points for pitch and roll. Each point is a column force; more points give a smoother, more expensive response.

## WaveRect (boats, rafts, scenery)

`WaveRect` is not buoyancy. The four corners of a rectangle are placed on the waves, and the model takes their mean height and their pitch and roll. Yaw and X/Z stay free, so the model can still be steered. It is the smoothest option and the one the demo's boats use.

1. Add a Part named `WaveRect` inside the model, placed at the waterline. Its X/Z size is the footprint that rides the waves. Set `Transparency = 1` and `CanCollide = false`.
2. Tag the model `OceanFloat` and set `BuoyancyMode = "WaveRect"`.

| Attribute (on the model) | Default | Effect |
| --- | --- | --- |
| `WaveRectOffset` | 0 | Studs added to the height (positive rides higher) |
| `TiltScale` | 1 | 0 stays level, 1 follows the wave plane exactly |
| `MaxTilt` | 45 | Maximum tilt in degrees |
| `Responsiveness` | 0 anchored / 40 unanchored | Smoothing (1/s) |
| `Drag`, `AngularDrag` | 0.6 / 1.2 | Unanchored only: water drag when nobody drives |
| `WaveRectPart` | `"WaveRect"` | Name of another part to use as the rectangle |
| `WaveRectSide` | client | `"Server"` moves an anchored model on the server instead |

- **Anchored** models are moved with `PivotTo` on every client, every frame. All clients agree, because they share the wave clock.
- **Unanchored** models are physical bodies held by `AlignPosition` (vertical axis) and `AlignOrientation` (up axis only), so they can be pushed and driven.

## Drivable boats

1. Same setup as WaveRect, with an **unanchored** model and a `VehicleSeat`.
2. Add the `OceanBoat` tag. The water is pushed out of the hull and the wake is generated.
3. Driving settings, as attributes on the model: `BoatThrust` (demo: 24), `BoatMaxSpeed` (52), `BoatTurnRate`. The defaults are in `Config.Demo` (`Demo.Boat*`).
4. On the server, when a player sits: give the hull's network ownership to that player, then set the attributes `OwnerUserId = player.UserId` and `Driven = true`. When they leave: clear `OwnerUserId`, set `Driven = false` and call `SetNetworkOwnershipAuto()`.

```lua
-- server Script, one boat
local boat = workspace.MyBoat -- Model with PrimaryPart, WaveRect part, VehicleSeat, tags OceanFloat + OceanBoat
local seat = boat:FindFirstChildWhichIsA("VehicleSeat", true)

seat:GetPropertyChangedSignal("Occupant"):Connect(function()
	local humanoid = seat.Occupant
	local player = humanoid and game:GetService("Players"):GetPlayerFromCharacter(humanoid.Parent)
	if player then
		boat.PrimaryPart:SetNetworkOwner(player)
		boat:SetAttribute("OwnerUserId", player.UserId)
		boat:SetAttribute("Driven", true)
	else
		boat:SetAttribute("OwnerUserId", nil)
		boat:SetAttribute("Driven", false)
		boat.PrimaryPart:SetNetworkOwnershipAuto()
	end
end)
```

`OceanBoatDriver` then runs propulsion, rudder and keel on the driver's client. The physics runs where the network owner is, so the driver feels no latency.

## Gameplay queries

```lua
local h = ocean:GetHeight(x, z)                  -- true displaced surface height at world X/Z
local data = ocean:GetSurfaceData(position)      -- Height, Normal, Velocity, Depth, IsUnderwater, Jacobian, ...
local under = ocean:IsUnderwater(position)
```

On the server these read the buoyancy field; on a client, the drawn field. Both match.

## Cost

`ocean:GetStats()` on the server returns the buoyancy runner's statistics: `objects`, `stepMs`, `objectsMs`, `usPerObject`, `fieldMs`. Cost grows linearly with the number of objects × points × `Buoyancy.UpdateRate` (60 Hz).

Roblox simulates physics at 240 Hz by default, with adaptive timestepping that can run some assemblies at 60 Hz ([adaptive timestepping](https://create.roblox.com/docs/physics/adaptive-timestepping)). Lower `Buoyancy.UpdateRate` if you float hundreds of objects.

Next: [8. Client and server](08-client-and-server.md).
