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

### GetEventStatus() → false | table

- Returns **false** when **no** event is active on the client.
- Returns a **table** with the current event state when an event is running.

### Common fields (all modes)
```lua
{
  mode      = 'hardpoint' | 'consecutive' | 'flags',
  id        = number,                        -- event id
  coords    = { x = number, y = number, z = number },
  radius    = number | nil,                  -- nil for Flags (no contest radius)
  found     = boolean,                       -- true once revealed (Flags start true)
  ended     = boolean,                       -- true after End has fired (until cleanup)
  myJob     = string,                        -- caller’s current ESX job
  groups    = { 'police', 'immortalmc', ... },  -- competing jobs (array)
  eligible  = boolean,                       -- this client is allowed to see/compete
  uiTitle   = string,                        -- last title sent to UI
  uiDetail  = string                         -- last detail line sent to UI
  -- (Flags may also carry cfg.netId internally; not exposed here unless you add it)
}

### Per-mode payloads

#### mode == 'hardpoint'
status.hardpoint = {
  currentHoldSeconds = number,  -- seconds accumulated by the **current** holding org
  endsInSeconds      = number   -- event time remaining (countdown from duration)
}

#### mode == 'consecutive'
status.consecutive = {
  holdTimeLeft = number,        -- seconds left for the current org to win
  -- graceLeft  = number|nil     -- (optional) seconds left in grace, if you expose it
}

#### mode == 'flags'
status.flags = {
  nextFlagIn = number,          -- seconds until the next flag can be claimed
  remaining  = number,          -- flags remaining to be given out
  total      = number,          -- total flags configured for this run
  claimed    = number           -- how many have been successfully claimed so far
}

### Example: consume status

RegisterCommand('vanstatus', function()
  local st = exports['c4leb-vannight']:GetEventStatus()
  if not st then
    print('No Van Night event running.')
    return
  end

  print('=== VanNight status ===')
  print('uiTitle: ' .. tostring(st.uiTitle))
  print('uiDetail: ' .. tostring(st.uiDetail))
  print(('mode: %s  id: %s'):format(st.mode, st.id))
  print(('coords: x=%.2f y=%.2f z=%.2f'):format(st.coords.x, st.coords.y, st.coords.z))
  print('radius: ' .. tostring(st.radius))
  print('found: ' .. tostring(st.found) .. '  ended: ' .. tostring(st.ended))
  print('myJob: ' .. tostring(st.myJob) .. '  eligible: ' .. tostring(st.eligible))

  print('groups:')
  for i,job in ipairs(st.groups or {}) do print(('  %d: %s'):format(i, job)) end

  if st.mode == 'hardpoint' and st.hardpoint then
    print('hardpoint:')
    print('  currentHoldSeconds: ' .. tostring(st.hardpoint.currentHoldSeconds))
    print('  endsInSeconds: ' .. tostring(st.hardpoint.endsInSeconds))
  elseif st.mode == 'consecutive' and st.consecutive then
    print('consecutive:')
    print('  holdTimeLeft: ' .. tostring(st.consecutive.holdTimeLeft))
  elseif st.mode == 'flags' and st.flags then
    print('flags:')
    print('  nextFlagIn: ' .. tostring(st.flags.nextFlagIn))
    print('  remaining: ' .. tostring(st.flags.remaining))
    print('  total: ' .. tostring(st.flags.total))
    print('  claimed: ' .. tostring(st.flags.claimed))
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
