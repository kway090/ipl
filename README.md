# Van Night — Admin Exports & Integration Guide

A tiny guide to help you drive the **Hardpoint**, **Consecutive Hold**, and **Flags** gamemodes from your own admin panel (or any other client script) via exports.

> These exports run **on the client** and call the built‑in server events. The server re‑checks permissions (via `Config.AdminGroup`) so cheaters can’t bypass your UI.

---

## Requirements

- **FiveM** (b2189+ recommended)
- **ox_lib** (for dialogs/notifications)
- **ESX** (jobs & permissions)
- **ox_target** (only required for *Flags*)
- **ox_inventory** (only required for *Flags*; item name: `van_flag`)
- Your resource name is assumed to be **`c4leb-vannight`** in examples below. Adjust if you renamed it.

---

## Quick Start

### Show the bundled dialogs (one‑liners)

```lua
-- Open built‑in dialog for each mode:
exports['c4leb-vannight']:OpenHardpointDialog()
exports['c4leb-vannight']:OpenConsecutiveDialog()
exports['c4leb-vannight']:OpenFlagsDialog()
```

- These open an **ox_lib inputDialog**, validate inputs, and immediately start the event for the **player running the export** (permission-checked server‑side).
- They return `true` if the dialog completed and a start request was sent, `false` if it was cancelled.

### Start a mode with your own UI data

```lua
-- Hardpoint
exports['c4leb-vannight']:StartHardpoint({
  pointType     = 'vehicle',        -- 'vehicle' | 'npc' | 'prop'
  radius        = 60,               -- metres
  minContenders = 2,                -- players from same org required
  duration      = 600,              -- seconds (total event length after reveal)
  announcement  = 'Van Placed!',
  tip           = 'South of Vinewood Sign',
  groupsArr     = { 'police', 'ballas', 'vagos' }, -- competing jobs
  coords        = vec3(379.2, 795.9, 187.1),       -- or {x=,y=,z=}
  heading       = 90.0                            -- optional
})

-- Consecutive Hold
exports['c4leb-vannight']:StartConsecutive({
  pointType     = 'npc',
  radius        = 50,
  minContenders = 1,
  holdTime      = 420,              -- seconds to hold continuously
  grace         = 15,               -- seconds allowed to return
  announcement  = 'Van Placed!',
  tip           = 'Hilltop radio tower',
  groupsArr     = { 'police', 'ballas' },
  coords        = vec3(703.4, 1224.7, 329.5),
  heading       = 180.0
})

-- Flags (no radius/minContenders)
exports['c4leb-vannight']:StartFlags({
  pointType     = 'prop',
  flagInterval  = 60,               -- seconds between flags (countdown pauses until someone claims)
  totalFlags    = 10,               -- stop after this many claims
  announcement  = 'Van Placed!',
  tip           = 'Find the van and claim flags',
  groupsArr     = { 'police', 'ballas', 'vagos' },
  coords        = vec3(125.3, -1046.7, 29.2),
  heading       = 0.0
})
```

All `Start*` exports return `true` if the payload was accepted client‑side and a start request was sent to the server (the server will still verify permissions).

---

## Export Reference

### `OpenHardpointDialog()` → `boolean`
Shows the bundled **Hardpoint** dialog (with live map/marker reveal on run). On success, it sends a `vannight:StartEvent` to the server.

### `OpenConsecutiveDialog()` → `boolean`
Shows the bundled **Consecutive Hold** dialog. On success, sends `vannight:StartEvent`.

### `OpenFlagsDialog()` → `boolean`
Shows the bundled **Flags** dialog. On success, sends `vannight:StartEvent`.

> The “Open*Dialog” exports are ideal when you don’t want to build UI. They return `false` if the dialog is closed or invalid.

---

### `StartHardpoint(cfg: table)` → `boolean`

**Required fields**

| key            | type               | description |
|----------------|--------------------|-------------|
| `pointType`    | `'vehicle'|'npc'|'prop'` | Entity type spawned at `coords`. |
| `radius`       | `number`           | Capture radius in metres. |
| `minContenders`| `number`           | Minimum players from the same org required to “hold”. |
| `duration`     | `number`           | Total event time (seconds) after first find. |
| `announcement` | `string`           | UI title while hidden / placed. |
| `tip`          | `string`           | UI subtitle while hidden / placed. |
| `groupsArr`    | `string[]`         | Array of job names competing. |
| `coords`       | `vec3` or `{x,y,z}`| Spawn location for entity/marker. |

**Optional**

| key       | type     | description |
|-----------|----------|-------------|
| `heading` | `number` | Spawn heading for vehicles/peds. |

---

### `StartConsecutive(cfg: table)` → `boolean`

**Required fields**

| key            | type               | description |
|----------------|--------------------|-------------|
| `pointType`    | `'vehicle'|'npc'|'prop'` | Entity type spawned. |
| `radius`       | `number`           | Capture radius in metres. |
| `minContenders`| `number`           | Minimum players from same org to count as holding. |
| `holdTime`     | `number`           | Seconds a single org must hold consecutively to win. |
| `grace`        | `number`           | Seconds allowed to return if all holders leave/die. |
| `announcement` | `string`           | UI title while hidden / placed. |
| `tip`          | `string`           | UI subtitle while hidden / placed. |
| `groupsArr`    | `string[]`         | Competing job names. |
| `coords`       | `vec3` or `{x,y,z}`| Spawn location. |

**Optional**: `heading` (number).

---

### `StartFlags(cfg: table)` → `boolean`

**Required fields**

| key            | type               | description |
|----------------|--------------------|-------------|
| `pointType`    | `'vehicle'|'npc'|'prop'` | Entity type spawned (networked). |
| `flagInterval` | `number`           | Seconds between flag availability. Timer **restarts only after a successful claim**. |
| `totalFlags`   | `number`           | Total number of flags to award this event. |
| `announcement` | `string`           | UI title. |
| `tip`          | `string`           | UI subtitle. |
| `groupsArr`    | `string[]`         | Competing job names. |
| `coords`       | `vec3` or `{x,y,z}`| Spawn location. |

**Optional**: `heading` (number).

> Server grants **`van_flag`** via `ox_inventory` when a claim is validated. The server owns the **only** give‑item path; clients can’t abuse it.

---

## Security Notes

- **Server permission gate**: starting any event requires `ESX.GetPlayerFromId(src).getGroup() == Config.AdminGroup`.
- **Flags anti‑dupe**: claims are processed in a server‑side critical section; only one claimant completes when the flag goes live.
- **Inventory safety**: server validates mode, timers, job, distance, and uses a hard‑coded item name `van_flag` when awarding.

---

## UI Behavior

- **Hardpoint / Consecutive**: “Van Placed” → “Van Found” with timers and job‑based visibility.
- **Flags**: Detail line shows **“Flag Ready in: N s”** or **“Flag Ready to Claim!”**, then **“Last Flag Claimed by <Org>!”** after a successful claim. The *remaining* counter is shown as **“Flags Remaining: X”**.
- Blips/zone visibility matches competing jobs (and observer jobs if configured).

---

## Example: Wiring into an Admin NUI

```lua
-- Example: your admin NUI "Create Hardpoint" button
RegisterNUICallback('createHardpoint', function(data, cb)
  local ok = exports['c4leb-vannight']:StartHardpoint({
    pointType     = data.pointType,          -- 'vehicle'|'npc'|'prop'
    radius        = tonumber(data.radius),
    minContenders = tonumber(data.minContenders),
    duration      = tonumber(data.duration),
    announcement  = data.announcement,
    tip           = data.tip or '',
    groupsArr     = data.groups,             -- e.g., {'police','ballas'}
    coords        = vec3(data.x, data.y, data.z),
    heading       = tonumber(data.heading) or 0.0
  })
  cb(ok == true)
end)
```

---

## Troubleshooting

- **Nothing happens after you call a Start* export**  
  Make sure the caller is an admin per `Config.AdminGroup`. Check server console for `[vannight]` prints.

- **Entity floats for late joiners**  
  The script settles vehicles on ground on **Begin** for clients that stream in later; ensure you kept the latest `Utils.settleVehicleOnGround` helper call in `client/main.lua`.

- **ox_target not working on the van**  
  Only the **Flags** mode uses a networked entity to allow targeting. HP/CH intentionally spawn local props/vehicles because they don’t need targeting.

---

## License

Do whatever you need on your server(s). Please keep credits in the README if you redistribute.

---

**Credits**  
Implementation & refactor with ❤️ by c4leb (and you, of course). PRs welcome!
