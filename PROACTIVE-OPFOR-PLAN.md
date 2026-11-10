# Proactive OPFOR & Enemy Logistics — Planned Work

Working plan for turning OPFOR from reactive sector defenders into an active theatre-level military,
plus a physical enemy logistics economy. Derived from a design originally prototyped against Liberation RX
and re-mapped onto **KP Liberation 0.96.7a** (`Missionframework/init.sqf`).

Every technical claim below was verified against this repository. If you change the framework, re-verify.

| | |
| --- | --- |
| Framework | KP Liberation `0.96.7a` |
| Active presets | `KP_liberation_preset_blufor = 0`, `KP_liberation_preset_opfor = 0` → `presets/*/custom.sqf` |
| PR 0 status | Landed locally, **not yet committed** |
| Remaining scope | PRs 1–9 |

---

## 1. Ground truth (verified, do not re-derive)

### Sector model

- Sectors are map markers classified by **name prefix** in `Missionframework/functions/fn_initSectors.sqf`:
  `bigtown`, `capture`, `factory`, `military`, `tower` → all pushed to `sectors_allSectors`.
  `opfor_point` and `opfor_airspawn` are **deliberately excluded** from `sectors_allSectors`.
- Altis: 148 sectors (10 bigtown, 68 capture, 24 factory, 24 military, 22 tower), 316 `opfor_point`
  spawn markers, 10 `opfor_airspawn`.
- Ownership is **data, not per-sector state**: `blufor_sectors`. Enemy-held = `sectors_allSectors - blufor_sectors`.
  Live ownership during a fight is unit counting (`fn_getSectorOwnership`).
- `GRLIB_sector_size` 1000 m (activation), `GRLIB_capture_size` 175 m. Activation radius *shrinks* as
  OPFOR unit count rises (`fn_getSectorRange`).
- **Victory is computed from `sectors_allSectors` / `sectors_bigtown` / `sectors_military`**
  (`scripts/shared/fetch_params.sqf`, `KP_liberation_victoryCheck`). Anything added must be an
  `opfor_point`-style marker or a bare position — **never a real sector**, or victory becomes unreachable.
- Single server capture hook: `scripts/server/remotecall/sector_liberated_remote_call.sqf`.
  Sector *loss* is handled separately in `scripts/server/sector/attack_in_progress_sector.sqf`.
- "Player base" equivalent is `GRLIB_all_fobs` (guaranteed non-empty by `scripts/server/base/startgame.sqf`),
  not `blufor_sectors`.

### Spawn model

No single spawn gateway, but exactly four chokepoints cover all enemy spawns:

| # | Chokepoint | Covers |
| --- | --- | --- |
| 1 | `functions/fn_createManagedUnit.sqf` | every OPFOR infantry unit |
| 2 | `functions/fn_spawnVehicle.sqf` | sector defenders, patrols, battlegroups; crew via `createVehicleCrew` |
| 3 | `scripts/server/battlegroup/spawn_air.sqf` | fixed-wing — **bypasses** `fn_spawnVehicle` |
| 4 | `scripts/server/patrols/send_paratroopers.sqf` | helicopter — **bypasses** `fn_spawnVehicle` |

- `fn_getOpforCap` = `EAST countSide allUnits`. Caps: `GRLIB_battlegroup_cap` 150, `GRLIB_patrol_cap` 150,
  `GRLIB_sector_cap` 180, `GRLIB_blufor_cap` 100.
- `diag_fps > 15.0` is the existing spawn-load gate (`scripts/server/battlegroup/random_battlegroups.sqf`).
- `fn_spawnVehicle` puts the crew in **its own group** — the LRX "patrol waypoints walk the gunner off"
  bug is structurally impossible here.
- Sector despawn (`scripts/server/sector/manage_one_sector.sqf`) uses asymmetric radii
  (`sectorRange + 300` to stay alive), ticket decay, then plain `deleteVehicle` —
  **it does not `setDamage 1` first**, so silent despawns look like despawns.

### Resource / economy model

- Three resources, **physical crates only**, no abstract pool: `KP_liberation_supply_crate` / `_ammo_crate` / `_fuel_crate`.
- Value lives in object variable `KP_liberation_crate_value`, **≤ 100 by convention only** — nothing enforces it.
  Partial-value crates are already created and already decremented in place, so **partially spent pallets store fine**.
- Storage = a building with crates `attachTo`'d; totals always recomputed by summing `attachedObjects`.
  Capacity is fixed: **12 positions small, 40 large**.
- Player production: one 100-unit crate per line per `KP_liberation_production_interval`
  (30 min ÷ resources multiplier), player-online only, capped at 12 attached crates per sector storage.
- **No OPFOR economy, no OPFOR production, no enemy stockpile, no enemy persistence.**
- No player-to-player transfer function; resources move physically. `KP_liberation_supplies/ammo/fuel` on
  the client are UI mirrors of *FOB* totals, not a wallet.
- Intel = `resources_intel` scalar, delivered via `scripts/client/remotecall/remote_call_intel.sqf`
  (fixed `switch` 0–8, plus local markers).

### Persistence

- `functions/fn_getSaveData.sqf` returns a **flat 22-element array** (indices 0–21).
- Loader uses `_saveData param [19..21, []]` for the newest fields, with a separate legacy branch for
  pre-0.96.5 saves. **Append at index 22+ and always load with `param`** → no wipe required.
- Saved: BLUFOR objects near FOBs, storages + summed resources, BLUFOR AI groups near FOBs, mines,
  loose crates at *blufor* factory/capture sectors, `blufor_sectors`, readiness, FOBs, permissions,
  production, logistics, intel, radio towers.
- **Not saved: anything OPFOR.** Any physical enemy stockpile is erased on restart unless explicitly saved.

### Existing proactive behaviour (do not duplicate)

| System | Location | Note |
| --- | --- | --- |
| Battlegroups | `scripts/server/battlegroup/spawn_battlegroup.sqf` | readiness/aggressivity gated |
| Counter-battlegroup (armour → air) | `scripts/server/battlegroup/counter_battlegroup.sqf` | **already implements the "armour escalation" idea** |
| Reinforcements / paratroopers | `scripts/server/patrols/reinforcements_manager.sqf`, `send_paratroopers.sqf` | radio-tower based |
| Patrols | `scripts/server/patrols/manage_one_patrol.sqf` + `scripts/server/ai/patrol_ai.sqf` | rove *enemy* sectors at ~1000 m |
| Sector defence | `scripts/server/sector/manage_one_sector.sqf` | player-proximity activation |
| Escalation dials | `combat_readiness` (0–100), `infantry/armor/air_weight` (0–100) | already drive composition and spawns |

---

## 2. Done (PR 0)

**Goal:** give the enemy artillery a classname and give the enemy economy a price source, without changing
any existing spawn behaviour.

Files changed:

- `Missionframework/presets/opfor/custom.sqf`
  - new `opfor_arty = ["O_MBT_02_arty_F"]` — a **dedicated pool**, deliberately *not* merged into
    `opfor_vehicles` / `opfor_battlegroup_vehicles`, so random sectors do not field SPGs.
  - new `opfor_price` — 18 entries, `[class, supply, ammo, fuel]`, one per class used by the OPFOR pools.
    Values mirror the closest BLUFOR counterpart in `presets/blufor/custom.sqf`; the mapping is in a comment
    on each row. Sochor = `600/1250/300`, matching KP's own value in `presets/blufor/csat.sqf`.
- `Missionframework/presets/init_presets.sqf`
  - `opfor_arty` / `opfor_price` are nil-guarded (`missionNamespace getVariable`) and passed through
    `KPLIB_fnc_checkClass`, so the 19 OPFOR presets that do not define them resolve to `[]`.
  - `opfor_arty` added to `KPLIB_o_allVeh_classes` → correct kill stats (`kill_manager`), correct scrap
    price (`do_recycle`), correct save handling.
  - new `KPLIB_o_priceIndex` — lowercase-keyed `[[class, supply, ammo, fuel], ...]`.

Validation performed: coverage script (18 enemy classes / 18 priced / 0 uncovered / 0 extras) and a
bracket/quote/comment-aware structural parse of all 393 `.sqf` files (0 problems).
CI here only assembles PBOs; it does not lint SQF.

Known gap left open on purpose: **infantry is unpriced.** See PR 9.

---

## 3. PR plan

Suggested branch names and PR-template fields are included for each. All PRs except PR 9 touch no
existing logic beyond a single `execVM` hook and additive config.

### PR 1 — Proactive OPFOR module scaffolding

| | |
| --- | --- |
| Branch | `feature/proactive-module` |
| Depends on | PR 0 |
| New feature? | yes |
| Needs wipe? | no |
| Base-file edits | 4 small ones |

**Content**

- [ ] New directory `Missionframework/scripts/server/proactive/`
- [ ] `init_module.sqf` following the `scripts/server/asymmetric/init_module.sqf` pattern
      (compile globals, then start loops)
- [ ] Compile-time flag `KP_liberation_proactive` in `Missionframework/kp_liberation_config.sqf`,
      default `0` (opt-in), plus one `KP_liberation_proactive_debug` logging level
- [ ] Single hook: `if (KP_liberation_proactive > 0) then {execVM "scripts\server\proactive\init_module.sqf";};`
      in `Missionframework/scripts/server/init_server.sqf`
- [ ] Notification classes in `Missionframework/ui/liberation_notifications.hpp`
      (artillery alert, checkpoint spotted, recon contact) + matching `stringtable.xml` entries
- [ ] Telemetry helper writing to `KPLIB_fnc_log` under a `PROACTIVE` tag

**Depends on nothing else. Everything below is gated behind this flag and reverts by flipping it to 0.**

**Validation:** mission starts with flag 0 and with flag 1; no new RPT errors; flag 0 produces zero
behavioural difference (diff the RPT).

---

### PR 2 — Sector-anchored artillery

| | |
| --- | --- |
| Branch | `feature/proactive-artillery` |
| Depends on | PR 1 |
| New feature? | yes |
| Needs wipe? | no |
| Base-file edits | 0 (optionally 1 — see below) |

**Content**

- [ ] Spotter pipeline, 15 s cadence: an enemy group leader within a configurable `spawn_max` of a player
      **and** `knowsAbout player > 1.5` starts a fire mission
- [ ] Capture target position **before** the delay, so the barrage lands on a stale grid (moving works)
- [ ] 18 s "radio transmission" window; abort if the spotting group is wiped during it (counter-play)
- [ ] Nearest enemy-held sector (`sectors_allSectors - blufor_sectors`) deploys or reuses a battery from
      `opfor_arty`, spawned via `KPLIB_fnc_spawnVehicle`
- [ ] Battery persists while players are inside engagement radius; otherwise self-manages despawn
      (sector cleanup only touches its own `_managed_units`, so a battery outside that list leaks)
- [ ] Fire at the **closest player the gun can actually reach** (`inRangeOfArtillery` per candidate);
      everything inside minimum range is left to the crew
- [ ] Global fire-mission cooldown (one salvo per ~4 min map-wide), shared with battery self-defence
- [ ] Per-spotter-group cooldown so one team cannot monopolise it
- [ ] **No shelling inside the start base safe zone** (`startbase`, >1000 m) or inside a FOB radius;
      compute server-side — `KPLIB_isAwayFromStart` is a client-only player variable
- [ ] Capture of the anchoring sector permanently removes the fire-support node
- [ ] Radar alert with the firing grid, via the existing notification + local marker pattern
- [ ] `getArtilleryAmmo` validation on the spawned gun; filter smoke/illum by indirect damage radius;
      blacklist any class that returns empty for the session
- [ ] Extend `KPLIB_o_allVeh_classes` at runtime (`pushBackUnique`) for any class resolved from the
      `KP_liberation_suppMod_artyVeh` fallback, so it is not counted as a civilian vehicle casualty

**Class selection.** Prefer `opfor_arty` from the preset; otherwise fall back to a side-filtered slice of
`KP_liberation_suppMod_artyVeh` (136 entries, all supported modsets):

```
_candidates = KP_liberation_suppMod_artyVeh select {[_x] call KPLIB_fnc_checkClass};
_bySide = _candidates select {getNumber (configFile >> "CfgVehicles" >> _x >> "side") == GRLIB_side_enemy};
if (_bySide isNotEqualTo []) then {_candidates = _bySide;};
_candidates = _candidates select {(_x isKindOf "Tank") || (_x isKindOf "Car")};   // drops B_Ship_Gun_01_F
```

**Known gotcha to test.** `Missionframework/kp_objectInits.sqf` calls
`KPLIB_suppMod_arty synchronizeObjectsAdd [_this]` for *any* object whose class is in
`KP_liberation_suppMod_artyVeh`, **without a side filter**. Every battery we spawn therefore gets
synchronised to the BLUFOR support provider. BIS may side-filter internally — verify. If it does not,
the fix is one added condition (`(side _this) isEqualTo GRLIB_side_friendly`), which is the only base-file
edit this PR might need.

**Validation:** mission with `KP_liberation_proactive = 1`; confirm a fire mission is cancellable by killing
the spotters; confirm moving off the marked grid avoids the barrage; confirm no fire missions inside the
start base; confirm the battery despawns when players leave; check `diag_fps` and OPFOR unit count do not
creep over a 30 min session.

---

### PR 3 — Road checkpoints (armed light vehicle)

| | |
| --- | --- |
| Branch | `feature/proactive-checkpoints` |
| Depends on | PR 1 |
| New feature? | yes |
| Needs wipe? | no |
| Base-file edits | 0 |

**Content**

- [ ] Nothing is created until `GRLIB_all_fobs` is non-empty (players have established a base)
- [ ] For each FOB, the nearest enemy-held sectors get a fortified position **~500 m out on the road that
      feeds them**
- [ ] Positions per FOB scale with FOB count (`fob_count × 2`)
- [ ] Structures created once: vanilla-only furniture (`Land_HBarrier_Big_F`, `Land_HBarrierWall6_F`,
      `Land_BagFence_Long_F`, `Land_BagBunker_Small_F`, `Land_Razorwire_F`) — **no preset config needed**
- [ ] **Weapon is a single armed light vehicle** drawn from `militia_vehicles` / `opfor_vehicles_low_intensity`,
      spawned with `KPLIB_fnc_spawnVehicle` (crew arrives in its own group automatically)
- [ ] Crew gets an explicit hold (`doStop` + `setCombatMode "RED"` + `setBehaviour "COMBAT"`, or
      `add_defense_waypoints`) and must never be adopted by a patrol waypoint system
- [ ] Garrison infantry skeleton spawns/despawns lazily; **asymmetric radii** — spawn smaller, despawn larger
- [ ] Guards: not inside a FOB defensive bubble, not inside line of sight of a player (no pop-in),
      blacklist any sector with no suitable road so it is never rescanned
- [ ] Cache `nearRoads` results per sector and early-exit on first acceptable road (no `nearRoads` usage
      exists anywhere in the repo yet, so there is no precedent to lean on)
- [ ] On sector capture, remove the checkpoint and queue the next enemy position beyond it as a follow-up
      target — drive this by polling `blufor_sectors` diffs, **not** by editing `sector_liberated_remote_call.sqf`

**Why a vehicle and not a static weapon.** `kill_manager` buckets a killed vehicle as OPFOR only if its
class is in `KPLIB_o_allVeh_classes`, otherwise it falls through to *civilian vehicle casualty*. Classes in
`militia_vehicles` / `opfor_vehicles_low_intensity` are already registered, so stats, scrap pricing and save
handling are all correct with zero extra work. A static weapon would not be.

**Validation:** build a FOB, confirm checkpoints appear only on roads feeding it and not inside the FOB
bubble; confirm no pop-in when approaching; capture the sector and confirm removal plus follow-up;
confirm the gunner never leaves the vehicle.

---

### PR 4 — Light recon patrols

| | |
| --- | --- |
| Branch | `feature/proactive-recon` |
| Depends on | PR 1 |
| New feature? | yes |
| Needs wipe? | no |
| Base-file edits | 0 |

**Content**

- [ ] Standalone and **additional** to `manage_one_patrol` — do not fold into it
- [ ] Own cap (config), and still respect `fn_getOpforCap` so it cannot starve sector defence
- [ ] Armed light vehicles from `opfor_vehicles_low_intensity`
- [ ] Rove between **player-held** towns, skirting at `GRLIB_sector_size + 300 m` rather than driving in,
      then move to the next
- [ ] Road-snap each waypoint with a nearest-road lookup; fall back to the raw point
- [ ] Per-town replacement on destruction after a randomised 1–2 h delay (not a single global timer)

**Validation:** confirm patrols target player-held sectors, keep to the skirt radius, and respawn per-town
rather than globally; confirm OPFOR unit count stays under the caps.

---

### PR 5 — Enemy economy telemetry

| | |
| --- | --- |
| Branch | `feature/enemy-economy-telemetry` |
| Depends on | PR 1 |
| New feature? | yes |
| Needs wipe? | no |
| Base-file edits | 0 |

**Content**

- [ ] Per-minute logging of production, spend and pool total, `PROACTIVE` tag
- [ ] **No gameplay effect** — this PR changes nothing the player sees
- [ ] Instrument the four spawn chokepoints to count spawns and attribute them to a category
- [ ] Instrument `sector_liberated_remote_call` / sector loss to count captures

Ship this before any of PRs 6–9. Every number in the design is a guess until data exists.

---

### PR 6 — Pallets, stockpiles, loot

| | |
| --- | --- |
| Branch | `feature/enemy-stockpiles` |
| Depends on | PR 5 |
| New feature? | yes |
| Needs wipe? | **no** — save fields are appended with `param` guards |
| Base-file edits | 2 (`fn_getSaveData.sqf`, `save_manager.sqf`) |

**Content**

- [ ] 5–10 stockpiles at the largest cities / densest military-base clusters, created at server start
- [ ] Stock is **physical pallets** (`KPLIB_crates` objects) in berms, visible from a distance and by drone
- [ ] Per-pallet value well above 100 — nothing enforces the ≤100 convention, so use 500–1000 to keep the
      object count to 30–60 instead of 300
- [ ] Pool total = sum of `KP_liberation_crate_value` over every enemy pallet
- [ ] Starting stock ≈ 1,000 × number of sectors OPFOR owns
- [ ] Production sites and bases hold small berms of their own
- [ ] On a regular cadence, pallets at production sites are transferred abstractly to stockpiles
- [ ] **Persistence:** append a single element at index 22 of the save blob holding pallet positions, types
      and values; load with `_saveData param [22, []]`
- [ ] On capture, the stockpile system ends but **the pallets stay on the ground for players to take**
- [ ] Pickup: crates already carry arbitrary values and store fine — no warehouse extension required

**Validation:** create stockpiles, verify all pallets survive a save/restart cycle, verify pool total matches
the visible pallets, verify a partial pallet can be carried and stored.

---

### PR 7 — Production nodes and rebuild

| | |
| --- | --- |
| Branch | `feature/enemy-production` |
| Depends on | PR 6 |
| New feature? | yes |
| Needs wipe? | no |
| Base-file edits | 0 (optional 1 if a new marker prefix is used) |

**Content**

- [ ] OPFOR starts owning all factory sectors; permanent factory sectors produce at full rate and, when
      captured, hand over to the existing player production system (already implemented in
      `sector_liberated_remote_call.sqf`)
- [ ] When a factory is lost, rebuild a **temporary production node in a large city** after a few hours
- [ ] Temporary nodes produce at half rate, cannot be captured, and vanish when their parent zone is captured
      — but their pallets remain
- [ ] **Temporary nodes must not be real sectors.** Use a custom marker prefix (or bare positions) so
      `fn_initSectors` ignores them and `KP_liberation_victoryCheck` is unaffected
- [ ] Explicit per-node rate, scaled by difficulty; persist node identity + timer in the PR 6 save element

**Validation:** capture a factory, confirm the temp node appears after the delay at half rate; confirm
victory progress is unchanged by the node; confirm capture of the parent zone removes the node but leaves
the pallets.

---

### PR 8 — Intel mission and stockpile report

| | |
| --- | --- |
| Branch | `feature/enemy-intel` |
| Depends on | PR 6 |
| New feature? | yes |
| Needs wipe? | no |
| Base-file edits | 1 (`remote_call_intel.sqf`, additive `switch` cases) |

**Content**

- [ ] Intel mission reveals the stock level of **one randomly chosen** stockpile plus a **local map marker**
- [ ] Follow the existing notification pattern (indexed remote call + `showNotification` + local marker);
      no new HUD
- [ ] **No permanent HUD readout of enemy supply** — it defeats the mechanic
- [ ] Adding intel cases means new `CfgNotifications` classes and `stringtable.xml` entries

**Validation:** run the mission, confirm one stockpile is revealed with a marker, confirm the reported total
matches the sum of that stockpile's pallets.

---

### PR 9 — Spawn draining and true-up (the only PR that touches hot paths)

| | |
| --- | --- |
| Branch | `feature/enemy-economy-drain` |
| Depends on | PR 6, PR 7 |
| New feature? | yes |
| Needs wipe? | no |
| Base-file edits | 4 hot-path edits — schedule this last |

**Content**

- [ ] Charge `opfor_price` (via `KPLIB_o_priceIndex`) on spawn, **enemy-side spawns only**
- [ ] Charge in the four chokepoints: `fn_createManagedUnit`, `fn_spawnVehicle`, `spawn_air`,
      `send_paratroopers`. Side missions draw from the same pool automatically because the charge lives in
      those functions — filter to `side == GRLIB_side_enemy` so civilians and resistance are exempt
- [ ] Affordability is checked against the global total; deduction comes from the **nearest pallets outward**
- [ ] Blocked spawns are permitted to fail, and partial fulfilment (fewer units than requested) is permitted
- [ ] **Infantry pricing** — the outstanding gap. Enemy infantry is the bulk of spawn volume. Precedent:
      `KPLIB_b_allSquads` prices BLUFOR squads as `[squad_array, cost]`. Price `militia_squad` and the
      `getSquadComp` compositions rather than 16 individual soldier classes; cache the result
- [ ] **True-up sweep, never a registry:**
      ```
      on spawn:   obj setVariable ["ENF_cost", cost, true];
      every 60s:  live = sum of ENF_cost over enemy objects that are alive AND exist;
                  pool += (charged - live);  charged = live;
      ```
      Filter on `alive` — dead bodies persist for `GRLIB_cleanup_delay` (1200 s) and wrecks persist
      indefinitely. Charge **per unit**, not per vehicle: `createVehicleCrew` crew are separate units
- [ ] Use `addMPEventHandler ["MPKilled"]`, never the local `addEventHandler ["Killed"]`

**Behaviour to decide deliberately.** KP's despawn path is silent `deleteVehicle` with no `setDamage 1`, so
the sweep refunds every despawn. That is the intended semantics ("spawned and not killed → refunded"), but it
means sector defence costs nothing over time — the real cost is only units that actually die. Confirm that is
the intended economy before shipping.

**Validation:** verify the pool drains on spawn and refunds on despawn; verify a blocked spawn is retried
after production refills; verify kills are *not* refunded; confirm no ledger/registry is left behind.

---

## 4. Deferred / not doing

| Item | Reason |
| --- | --- |
| Armour escalation → air (original design 2.4) | **Already implemented** in `scripts/server/battlegroup/counter_battlegroup.sqf`. It differs from the original spec — it triggers on `armor_weight`/`air_weight` ≥50 plus readiness ≥70 rather than "spotted by sector defenders", polls every 30–60 min, and has no per-zone roll or cooldown. If that later becomes unsatisfying it is a ~20-line edit inside the existing file, not a new system. |
| Player-to-player resource transfer | KP has none and does not need one; resources move physically. |
| Statics at checkpoints | Replaced by a single armed light vehicle, which is already in `KPLIB_o_allVeh_classes`. |

---

## 5. Open decisions

1. **Ammo:fuel ratio.** The existing tables run armour at roughly 1:0.6 ammo:fuel (Varsuk 300:200,
   Angara 350:225). The original design wanted ~10:1. Options: retune `opfor_price` row by row, or accept
   KP's ratios and let the enemy economy be fuel-rich. **Not yet decided.**
2. **Flag placement.** `kp_liberation_config.sqf` boolean only, or also a user-facing `class Params` mission
   parameter (which needs `stringtable.xml` entries)? PR 1 currently assumes config-only.
3. **Temporary production node markers.** Use bare positions (no marker at all), or introduce a new marker
   prefix and one line in `fn_initSectors.sqf`?
4. **Partial spawn fulfilment.** Allow it (design intent) or require whole squads to keep group composition
   intact?

---

## 6. Reference: files touched per PR

| PR | New files | Modified files |
| --- | --- | --- |
| 0 | — | `presets/opfor/custom.sqf`, `presets/init_presets.sqf` |
| 1 | `scripts/server/proactive/init_module.sqf` (+ helpers) | `kp_liberation_config.sqf`, `scripts/server/init_server.sqf`, `ui/liberation_notifications.hpp`, `stringtable.xml` |
| 2 | artillery scripts | PR 1 files only (possibly `kp_objectInits.sqf` for the side guard) |
| 3 | checkpoint scripts | PR 1 files only |
| 4 | recon scripts | PR 1 files only |
| 5 | telemetry scripts | PR 1 files only |
| 6 | stockpile scripts | `functions/fn_getSaveData.sqf`, `scripts/server/game/save_manager.sqf` |
| 7 | production scripts | PR 1 files only (optionally `functions/fn_initSectors.sqf`) |
| 8 | intel scripts | `scripts/client/remotecall/remote_call_intel.sqf`, `ui/liberation_notifications.hpp`, `stringtable.xml` |
| 9 | drain/sweep scripts | `functions/fn_createManagedUnit.sqf`, `functions/fn_spawnVehicle.sqf`, `scripts/server/battlegroup/spawn_air.sqf`, `scripts/server/patrols/send_paratroopers.sqf` |
