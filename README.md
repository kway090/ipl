# Van Night — Admin Exports & Integration Guide

Drive the **Hardpoint**, **Consecutive Hold**, and **Flags** gamemodes from your own client scripts or admin panel via exports.

> Exports run **on the client** and trigger the built-in server events. The server re-checks admin permission via `Config.AdminGroup`.

---

## Requirements

- FiveM (b2189+ recommended)
- `ox_lib` (dialogs / notify / progress)
- `es_extended` (ESX jobs & groups)
- `ox_target` (only required for **Flags**)
- `ox_inventory` (only required for **Flags**; item: `van_flag`)
- Examples assume the resource name is **`c4leb-vannight`**.

---

## Quick Reference (Client Exports)

- `exports['c4leb-vannight']:StartHardpointDialog() → boolean`
- `exports['c4leb-vannight']:StartConsecutiveDialog() → boolean`
- `exports['c4leb-vannight']:StartFlagsDialog() → boolean`

- `exports['c4leb-vannight']:StartHardpoint(cfg: table) → boolean`
- `exports['c4leb-vannight']:StartConsecutive(cfg: table) → boolean`
- `exports['c4leb-vannight']:StartFlags(cfg: table) → boolean`

- `exports['c4leb-vannight']:GetEventStatus() → false | table`

All of the above are **client exports**. Call them from any other client script.

---

## Show the built-in dialogs (one-liners)

```lua
exports['c4leb-vannight']:StartHardpointDialog()
exports['c4leb-vannight']:StartConsecutiveDialog()
exports['c4leb-vannight']:StartFlagsDialog()
```

- Opens an `ox_lib` input dialog, validates, and starts the event at the player’s position.
- Returns `true` if a start request was sent, `false` if the dialog was cancelled.

---

## Start with your own UI data

### Hardpoint

```lua
exports['c4leb-vannight']:StartHardpoint({
  pointType     = 'vehicle',                 -- 'vehicle' | 'npc' | 'prop'
  radius        = 60,                        -- metres
  minContenders = 2,                         -- players from same org to count
  duration      = 600,                       -- seconds (total event length after reveal)
  announcement  = 'Van Placed!',
  tip           = 'South of Vinewood Sign',
  groupsArr     = { 'police', 'ballas', 'vagos' },
  coords        = vec3(379.2, 795.9, 187.1), -- or {x=,y=,z=}
  heading       = 90.0                       -- optional
})
```

### Consecutive Hold

```lua
exports['c4leb-vannight']:StartConsecutive({
  pointType     = 'npc',
  radius        = 50,
  minContenders = 1,
  holdTime      = 420,                       -- seconds a single org must hold continuously
  grace         = 15,                        -- seconds to return if all leave/die
  announcement  = 'Van Placed!',
  tip           = 'Hilltop radio tower',
  groupsArr     = { 'police', 'ballas' },
  coords        = vec3(703.4, 1224.7, 329.5),
  heading       = 180.0
})
```

### Flags

```lua
exports['c4leb-vannight']:StartFlags({
  pointType     = 'prop',
  flagInterval  = 60,                        -- seconds between flags (restarts after a claim)
  totalFlags    = 10,                        -- stop after this many claims
  announcement  = 'Van Placed!',
  tip           = 'Find the van and claim flags',
  groupsArr     = { 'police', 'ballas', 'vagos' },
  coords        = vec3(125.3, -1046.7, 29.2),
  heading       = 0.0
})
```

Returns `true` if the payload was accepted client-side and a start request was sent (server still enforces permissions).

---

## Event Status Export

### `GetEventStatus() → false | table`

- Returns **`false`** when **no** event is active on the client.
- Returns a **table** with current event info when an event is running.

#### Common fields (all modes)

```lua
{
  mode      = 'hardpoint' | 'consecutive' | 'flags',
  id        = number,                      -- event id
  coords    = vector3(x, y, z),
  heading   = number | nil,
  radius    = number | nil,                -- nil for Flags (no contest radius)
  netId     = number | nil,                -- Flags uses a networked entity
  groupsArr = { 'police', 'ballas', ... }, -- raw array
  groups    = { [jobName]=true, ... },     -- lookup for local visibility

  -- last UI payload pushed to this client
  ui = {
    title       = string,
    detail      = string,
    timer1      = number | nil,         -- meaning depends on mode (see below)
    timer2      = number | nil,         -- meaning depends on mode (see below)
    timer1Label = string | nil,         -- may be set (e.g., Flags)
    timer2Label = string | nil,
    active      = boolean,              -- whether this client should see the UI
    found       = boolean | nil         -- true after reveal (HP/CH)
  }
}
```

#### Per-mode timer semantics

- **Hardpoint**
  - `ui.timer1` → seconds **held so far** for the currently holding org (if exactly one org meets `minContenders`).
  - `ui.timer2` → **event time remaining** (countdown from `duration` after first find).
  - `ui.detail` → `"X Holding Point!"`, `"Point Contested, Timer Paused!"`, or `"No one contesting point"`.
  - Reveal: `ui.found` becomes `true` after the first org steps into the radius.

- **Consecutive Hold**
  - `ui.timer1` → **hold time left** for the current org (counts down from `holdTime`).
  - `ui.timer2` → **grace time left** (only when in grace). Otherwise `nil`.
  - `ui.detail` → `"X Holding Point!"`, `"X has Ns to return!"`, `"Point Contested..."`, or `"No one..."`.

- **Flags**
  - `ui.timer1` → **seconds until next flag** (counts down; resets only after a successful claim).
  - `ui.timer2` → **flags remaining**.
  - `ui.detail` → `"Flag Ready to Claim!"` or `"Last Flag Claimed by <Org>!"`.
  - Labels: `ui.timer1Label = "Flag Ready in"`, `ui.timer2Label = "Flags Remaining"`.

#### Example: consume status

```lua
RegisterCommand('vanstatus', function()
  local st = exports['c4leb-vannight']:GetEventStatus()
  if not st then
    print('No Van Night event running.')
    return
  end

  print(('[VanNight] mode=%s id=%s'):format(st.mode, st.id))
  if st.ui then
    print(('title="%s" detail="%s" t1=%s t2=%s'):format(
      tostring(st.ui.title),
      tostring(st.ui.detail),
      tostring(st.ui.timer1),
      tostring(st.ui.timer2)
    ))
  end

  if st.mode == 'flags' then
    print(('next flag in: %ss, remaining: %s'):format(
      tostring(st.ui and st.ui.timer1 or '?'),
      tostring(st.ui and st.ui.timer2 or '?')
    ))
  end
end, false)
```

---

## Security Notes

- **Server gate**: starting any event requires `ESX.GetPlayerFromId(src).getGroup() == Config.AdminGroup`.
- **Flags anti-dupe**: server processes claims in a critical section; only one winner per flag window.
- **Inventory safety**: server awards a **hard-coded** `van_flag` via `ox_inventory` after validating mode/job/distance/timer.

---

## UI Behavior

- **Hardpoint / Consecutive**: “Van Placed” → “Van Found” with appropriate timers. Blip & zone visibility are job-based (competing jobs + observers).
- **Flags**: shows `Flag Ready in:` and `Flags Remaining:`; displays `Flag Ready to Claim!` or `Last Flag Claimed by <Org>!`.

---

## Troubleshooting

- **Nothing happens when calling a Start* export**  
  Ensure the caller is in the admin group. Check server console for `[vannight]` messages.

- **Entity floating for late streamers**  
  The script settles vehicles on ground on `Begin`. Keep the latest `Utils.settleVehicleOnGround` call in your client.

- **ox_target doesn’t show**  
  Only the **Flags** mode uses a **networked** entity for targeting. HP/CH spawn local entities (no targeting needed).

---

## License

Use freely on your servers. Keep credits if you redistribute.

**Credits:** built with ❤️ by c4leb (+ you).
