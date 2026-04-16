# Atmospherics Air Simulation — Implementation Reference

## 1. Purpose

The air simulation keeps every walkable tile on the station in a physically consistent gas state. It propagates pressure differences between tiles, transfers heat by conduction, triggers combustion when plasma and oxygen co-exist at high temperature, and reports notable events (fire, wind, gas overlays) back to the game layer. The goal is a simulation that feels plausible for a space station without being computationally exact — moles and temperatures are tracked, but the physics are simplified.

---

## 2. Architectural Overview

The simulation is split across two execution contexts that communicate through a defined handshake:

| Layer | Language | Role |
|---|---|---|
| **MILLA** | Rust (native DLL: `rustlibs.dll` / `librustlibs.so`) | Owns the canonical per-tile gas state; runs the actual diffusion, heat conduction, and fire simulation every tick in a background thread |
| **DM game layer** | DreamMaker (BYOND) | Bootstraps map state into MILLA; reads "interesting tile" events back out; applies gameplay effects (temperature exposure, pressure pushes, fire visuals, gas overlays) |

They are connected through `call_ext` (BYOND's FFI). All proc names starting with `milla_*` in `code/__DEFINES/rust.dm` are foreign function call wrappers.

---

## 3. Key Data Structures

### 3.1 `datum/gas_mixture`

**File:** `code/modules/atmospherics/gasmixtures/gas_mixture.dm`

The fundamental data object representing a body of gas. Used both as a transient calculation object (for pipes, canisters, etc.) and, in the specialised `bound_to_turf` subtype, as a DM-side mirror of a single tile's MILLA state.

Each instance holds:

| Field | Type | Meaning |
|---|---|---|
| `private_oxygen` | num (moles) | Amount of oxygen |
| `private_nitrogen` | num (moles) | Amount of nitrogen |
| `private_carbon_dioxide` | num (moles) | Amount of CO₂ |
| `private_toxins` | num (moles) | Amount of plasma |
| `private_sleeping_agent` | num (moles) | Amount of N₂O |
| `private_agent_b` | num (moles) | Amount of Agent B |
| `private_hydrogen` | num (moles) | Amount of hydrogen |
| `private_water_vapor` | num (moles) | Amount of water vapor |
| `private_temperature` | num (Kelvin) | Bulk temperature |
| `private_*_archived` | num | Snapshot values used during `share()` to avoid order-dependent race conditions |
| `volume` | num (liters) | Volume of the container — always `CELL_VOLUME` (2500 L) for a floor tile |
| `innate_heat_capacity` | num | Extra heat capacity contributed by the tile material itself |
| `private_hotspot_temperature` | num | MILLA-written fire hotspot temperature |
| `private_hotspot_volume` | num | MILLA-written fire hotspot volume |
| `private_fuel_burnt` | num | How much fuel burned last reaction step |
| `synchronized` | bool | TRUE if registered in `SSair.bound_mixtures` and will be written to MILLA before the next tick |

**Derived quantities computed on demand:**

```
pressure      = total_moles × R × temperature / volume   (kPa)
heat_capacity = Σ(species_moles × specific_heat) + innate_heat_capacity
```

Constants used:
- `R_IDEAL_GAS_EQUATION = 8.31 kPa·L/(K·mol)`
- `CELL_VOLUME = 2500 L`

Specific heats per species (J/mol/K):

| Gas | Constant | Value |
|---|---|---|
| O₂ / N₂ | `SPECIFIC_HEAT_AIR` | 20 |
| CO₂ | `SPECIFIC_HEAT_CDO` | 30 |
| Plasma (toxins) | `SPECIFIC_HEAT_TOXIN` | 200 |
| N₂O | `SPECIFIC_HEAT_N2O` | 40 |
| Agent B | `SPECIFIC_HEAT_AGENT_B` | 300 |
| Hydrogen | `SPECIFIC_HEAT_HYDROGEN` | 15 |
| Water vapor | `SPECIFIC_HEAT_WATER_VAPOR` | 33 |

#### Subclasses

**`/datum/gas_mixture/bound_to_turf`**

The live DM-side cache for a tile. Adds:

| Field | Meaning |
|---|---|
| `bound_turf` | Reference to the owning turf |
| `dirty` | TRUE when DM has changes that need flushing to MILLA |
| `lastread` | The `milla_tick` value when the object was last populated from MILLA |
| `readonly` | Lazily-created `/datum/gas_mixture/readonly` snapshot |

Overrides all `set_*` procs to call `set_dirty()` so changes are never silently dropped.

**`/datum/gas_mixture/readonly`**

An immutable snapshot returned by `get_readonly_air()`. Crashes with a runtime error if any setter is called. Used by read-only consumers (gas analyzers, HUD overlays) that must not affect MILLA state.

---

### 3.2 `turf/simulated`

**Files:** `code/game/turfs/simulated.dm`, `code/game/turfs/turf.dm`

A simulated floor or wall tile. Relevant fields:

| Field | Meaning |
|---|---|
| `bound_air` (`datum/gas_mixture/bound_to_turf`) | Lazily-created DM-side gas mirror |
| `milla_data` | Temporary list used **only** during `Initialize_Atmos`/`milla_load_turfs` bootstrap; freed immediately after |
| `atmos_mode` | One of `ATMOS_MODE_SPACE`, `ATMOS_MODE_SEALED`, `ATMOS_MODE_EXPOSED_TO_ENVIRONMENT`, or `ATMOS_MODE_NO_DECAY` |
| `atmos_environment` | Key into `SSmapping.environments`; MILLA uses the environment's gas composition for exposed tiles |
| `blocks_air` | TRUE on wall turfs; no gas state, always airtight |
| `heat_capacity`, `thermal_conductivity`, `temperature` | Solid tile thermal properties (wall/floor material) used for superconduction |
| `active_hotspot` (`obj/effect/hotspot`) | Visual fire object; created/destroyed by the DM hotspot tracking loop |
| `wind_tick`, `wind_x`, `wind_y`, `wind_effect` | Wind state stamped each tick from MILLA events |

---

### 3.3 MILLA tile representation

When communicating via FFI, a tile's full state is a flat list of 22 values. Index constants are defined in `code/__DEFINES/rust.dm`:

```
[airtight_bitmask,
 O2, CO2, N2, toxins, sleeping_agent, agent_b, hydrogen, water_vapor,
 atmos_mode, environment_id,
 superconductivity_N, superconductivity_E, superconductivity_S, superconductivity_W,
 innate_heat_capacity, temperature,
 hotspot_temperature, hotspot_volume,
 wind_x, wind_y, fuel_burnt]
```

(`MILLA_TILE_SIZE = 22`)

An *interesting tile* entry appends `[turf_ref, interesting_reasons_bitmask, airflow_x, airflow_y]`
(`MILLA_INTERESTING_TILE_SIZE = 26`)

Interesting reason flags:

| Flag | Meaning |
|---|---|
| `MILLA_INTERESTING_REASON_DISPLAY` | Gas composition crossed a visible threshold |
| `MILLA_INTERESTING_REASON_HOT` | Tile is above fire temperature |
| `MILLA_INTERESTING_REASON_WIND` | Tile has significant gas flow that can move objects |

---

### 3.4 `SSair` — the Atmospherics Subsystem

**File:** `code/controllers/subsystem/SSair.dm`

The Master Controller subsystem that orchestrates DM-side processing. It fires on every MC tick but self-throttles via `self_wait = 0.15 s`.

Key state variables:

| Variable | Purpose |
|---|---|
| `milla_tick` | Monotonically increasing tick counter |
| `milla_idle` | TRUE when MILLA has finished its background thread tick and data is safe to read/write |
| `in_milla_safe_code` | TRUE during the synchronous DM window — guards all MILLA state access |
| `bound_mixtures` | All `bound_to_turf` objects that were touched this tick and need flushing |
| `hotspots` | Currently active fire tiles |
| `windy_tiles` | Currently active wind tiles |
| `waiting_for_sync` | Queue of `datum/milla_safe` callbacks waiting for MILLA to go idle |

---

## 4. Initialization

1. **`SSair.Initialize()`** is called once at round start inside `in_milla_safe_code = TRUE`, so it can call MILLA FFI freely.

2. **`setup_turfs()`** iterates every turf in the world:
   - Each turf's `Initialize_Atmos()` builds a `milla_data` list encoding initial gas content, connectivity, and `atmos_mode`.
   - `milla_load_turfs()` (FFI: `milla_load_turfs`) bulk-uploads all those lists to MILLA in one call.
   - `milla_data` is immediately freed from DM memory.

3. **Environments** are created once in `SSmapping` via `create_environment()` (FFI), returning an opaque integer ID that MILLA retains internally. Exposed tiles reference this ID so MILLA knows what composition to equalise toward.

   Pre-defined environments:

   | Key | Composition |
   |---|---|
   | `ENVIRONMENT_LAVALAND` | Low O₂ and N₂, high temperature |
   | `ENVIRONMENT_TEMPERATE` | Standard O₂/N₂ at 20 °C |
   | `ENVIRONMENT_COLD` | Standard O₂/N₂ at 180 K |

4. **Connectivity** is computed per-tile by `private_unsafe_recalculate_atmos_connectivity()`:
   - Checks `blocks_air` and `CanAtmosPass()` for each of the four cardinal directions.
   - Produces a 4-element airtight bitmask and a per-direction superconductivity coefficient.
   - These are uploaded to MILLA and updated at runtime whenever doors open/close.

---

## 5. The Update Loop

Each `SSair` tick passes through nine sequential stages. The subsystem is resumable mid-tick if the MC time budget elapses (`currentpart` tracks progress):

```
SSAIR_DEFERREDPIPENETS  →  SSAIR_PIPENETS  →  SSAIR_ATMOSMACHINERY
→  SSAIR_INTERESTING_TILES  →  SSAIR_HOTSPOTS  →  SSAIR_WINDY_TILES
→  SSAIR_BOUND_MIXTURES  →  SSAIR_PRESSURE_OVERLAY  →  SSAIR_MILLA_TICK
```

The stages relevant to core air simulation:

### Stage 4 — `process_interesting_tiles`

MILLA is queried via `get_interesting_atmos_tiles()` (FFI). The returned flat list is iterated; each entry is handled according to its reason bitmask:

- **`MILLA_INTERESTING_REASON_DISPLAY`** — Gas composition changed visibly. The DM tile's `bound_air` is refreshed from MILLA, and `update_visuals()` is called to show/hide the plasma, sleeping agent, or water vapor overlay.

- **`MILLA_INTERESTING_REASON_HOT`** — Tile is above fire temperature. `temperature_expose()` is called on the turf and all items on it. If there was no `active_hotspot` object yet, the tile is added to `SSair.hotspots`. Hotspot temperature and volume are synced from MILLA data.

- **`MILLA_INTERESTING_REASON_WIND`** — Tile has significant gas flow. `high_pressure_movements(flow_x, flow_y)` is called to push unanchored mobs and items. The tile is added to `SSair.windy_tiles` if not already present, and `wind_tick`, `wind_x`, `wind_y` are stamped.

### Stage 5 — `process_hotspots`

For each tile in `SSair.hotspots`:

1. `update_hotspot()` reads MILLA tile state (via `bound_air` if fresh, else a new `get_readonly_air()`) to get `fuel_burnt`, `hotspot_temperature`, and `hotspot_volume`.
2. If `fuel_burnt < 0.001` and the hotspot has aged past its `death_timer` (4 ticks from last active tick), the `active_hotspot` visual is deleted and the tile is removed from the list.
3. Otherwise the hotspot color, icon state (`1`/`2`/`3` depending on fuel), and light range are updated from temperature and `fuel_burnt`.

### Stage 6 — `process_windy_tiles`

For each tile in `SSair.windy_tiles`:

1. `update_wind()` checks `wind_tick == milla_tick`. If not, wind has stopped — delete the wind visual and remove from list.
2. If still active, orient `wind_effect` via `wind_direction()` and scale its alpha by `wind_strength = |wind_vector| × total_moles / MOLES_CELLSTANDARD`.

### Stage 7 — `process_bound_mixtures`

All `bound_to_turf` objects in `SSair.bound_mixtures` with `dirty = TRUE` are flushed to MILLA:

- `private_unsafe_write()` calls `set_tile_atmos()` (FFI) to push the DM-modified gas state into MILLA.
- `dirty` is cleared; `synchronized = FALSE` (MILLA will modify the state again this tick).

### Stage 9 — `spawn_milla_tick_thread`

After DM has finished all its work:

1. `spawn_milla_tick_thread()` (FFI) launches MILLA's background simulation thread.
2. `milla_idle = FALSE` — no DM code may touch tile state.
3. MILLA runs the full tile simulation asynchronously. When done it calls back `milla_tick_finished()` on the DM side.
4. `on_milla_tick_finished()` sets `milla_idle = TRUE` and drains `waiting_for_sync` callbacks.
5. The timer guard in `fire()` prevents the next DM tick from starting until `elapsed >= self_wait` (0.15 s), so MILLA has real time to run.

---

## 6. How MILLA Simulates Air (the Rust Layer)

MILLA's internals are a compiled Rust library. Its behavior is deduced from the FFI interface and what it reports back.

### Gas diffusion

Each tile shares gas with its open cardinal neighbours. The per-species delta per step is:

```
delta = (self_amount_archived − neighbour_amount_archived) / (N_open_neighbours + 1)
```

This is the same formula used by the DM `gas_mixture.share()` proc. Using *archived* (previous tick) values means all tiles are updated simultaneously, preventing order-dependent bias.

### Heat conduction (gas-to-gas)

Temperature is transferred proportionally to the heat flowing with the gas, weighted by each species' specific heat capacity. If the temperature delta is below `MINIMUM_TEMPERATURE_DELTA_TO_CONSIDER` (0.5 K), no heat is computed. Superconductivity across wall tiles uses the per-direction coefficients set by `reduce_superconductivity()`.

### Tile atmos modes

| Mode constant | Behaviour |
|---|---|
| `ATMOS_MODE_SPACE` (0) | Gas slowly leaks out. Space is a sink. |
| `ATMOS_MODE_SEALED` (1) | Normal station floor; no special boundary. |
| `ATMOS_MODE_EXPOSED_TO_ENVIRONMENT` (2) | Tile slowly equalises toward its environment (e.g. Lavaland). |
| `ATMOS_MODE_NO_DECAY` (3) | Hot tiles do not decay toward T20C; used around the Supermatter. |

### Fire simulation

MILLA tracks hotspot state per-tile. When oxygen and plasma co-exist above `FIRE_MINIMUM_TEMPERATURE_TO_EXIST` (373 K), combustion runs using the same formula as DM's `gas_mixture.react()`. The result `fuel_burnt` is written back into the tile state and appears in interesting-tile output.

### Interesting tile selection

After each tick MILLA compiles a list of tiles where something happened that DM needs to react to (visual change, fire, or wind). Only these are communicated back — tiles in static equilibrium are silent, keeping communication overhead proportional to activity.

---

## 7. Thread Safety and the MILLA-Safe Pattern

Because MILLA runs in a background thread, it is **unsafe** to read or write tile gas state while MILLA is running. All code that touches tile air from DM must run when `milla_idle == TRUE`.

The mechanism is **`datum/milla_safe`** (and its sleeping variant `datum/milla_safe_must_sleep`):

```
1. External code calls:   my_milla_object.invoke_async(turf_ref, ...)
2. If milla_idle:         on_run() is called immediately, in-place
3. If NOT milla_idle:     the object is queued in SSair.waiting_for_sync
4. On milla_tick_finished(): all queued callbacks are drained before the next DM tick begins
```

Inside `on_run()`, code uses `get_turf_air(T)` provided by `datum/milla_safe`:

1. Lazily creates `bound_air` if it doesn't exist.
2. If `bound_air.lastread < milla_tick`, pulls the latest state from MILLA via `get_tile_atmos()` (FFI).
3. Registers `bound_air` in `SSair.bound_mixtures` so it will be flushed back before the next MILLA tick.

Code that only needs to **read** air uses `get_readonly_air()`, which returns a `datum/gas_mixture/readonly` snapshot and does **not** enqueue anything for flushing.

---

## 8. Connectivity Updates

When a door opens/closes, or any object with a `CanAtmosPass()` override changes state:

1. `recalculate_atmos_connectivity()` is called on its turf.
2. This creates a `datum/milla_safe/recalculate_atmos_connectivity` and queues it via `invoke_async`.
3. When safe, `private_unsafe_recalculate_atmos_connectivity()` recomputes the 4-direction airtight bitmask and superconductivity coefficients from scratch.
4. `set_tile_airtight()` and `reset_superconductivity()` / `reduce_superconductivity()` push the new values to MILLA via FFI.

MILLA uses these on the next tick to determine which directions gas can flow across.

---

## 9. Solid Tile Superconduction

Wall tiles do not hold gas but do conduct heat. `turf/simulated` tracks `temperature` and `heat_capacity` as material properties. Relevant procs:

| Proc | Behaviour |
|---|---|
| `share_temperature_mutual_solid(sharer, coefficient)` | Mutual conduction between two adjacent wall tiles using their archived temperatures |
| `mimic_temperature_solid(model, coefficient)` | Unilateral conduction from a gas/space reference into a wall tile |
| `radiate_to_spess()` | Tiles above 273 K lose heat toward `TCMB` (2.7 K) at a rate determined by `thermal_conductivity` and `HEAT_CAPACITY_VACUUM` |

The superconductivity coefficients set through `reduce_superconductivity()` determine how quickly heat flows across a tile boundary in the MILLA simulation.

---

## 10. Gas Reactions (DM-side)

`gas_mixture.react()` is called by pipe networks and can be triggered by other systems. Four reactions are handled in sequence, all energy-conserving:

### Agent B conversion

Agent B catalyses CO₂ → O₂ above `AGENT_B_CONVERSION_MIN_TEMP`.

### N₂O decomposition

Sleeping agent decomposes above threshold, releasing heat, yielding N₂ + O₂.

### Plasma combustion

- Requires: plasma + O₂ above `PLASMA_MINIMUM_BURN_TEMPERATURE` (373 K).
- Burn rate scales linearly from zero at `PLASMA_MINIMUM_BURN_TEMPERATURE` to maximum at `PLASMA_UPPER_TEMPERATURE` (1643 K).
- Produces CO₂ and heat; records `fuel_burnt`.

### Hydrogen combustion

- Requires: H₂ + O₂ above `HYDROGEN_MIN_IGNITE_TEMP`.
- Rate depends on temperature and pressure: `rate = (T / (T + 2000)) × (P / (P + 100))`.
- Produces water vapor and heat.

All reactions share the same temperature update pattern:

```
new_temperature = (old_heat_capacity × old_temperature + energy_released) / new_heat_capacity
```

MILLA runs equivalent reaction logic internally for tile gases. The DM `react()` is used for pipe-network gases.

---

## 11. Pressure Effects on Movables

When MILLA reports `MILLA_INTERESTING_REASON_WIND`, `high_pressure_movements(flow_x, flow_y)` is called on the turf. For each unanchored movable:

1. `force = |wind_vector| × (total_moles / MOLES_CELLSTANDARD) × (MOVE_FORCE_DEFAULT / 5)`
2. If `force > move_resist × MOVE_FORCE_PUSH_RATIO`, `air_push()` steps the movable in the wind direction.
3. Mobs receive `STATUS_EFFECT_UNBALANCED` and a directional slowdown. The push is suppressed if the mob is actively moving under player input.

---

## 12. Interaction Diagram

```
                ┌──────────────────────────────────────┐
                │             MILLA (Rust)             │
                │  • Per-tile gas arrays               │
                │  • Diffusion / heat conduction       │
                │  • Fire combustion                   │
                │  • Interesting tile output           │
                └──────────────────────────────────────┘
                     ↑  FFI: set_tile_atmos             ↓  FFI: get_interesting_atmos_tiles
                     │  (DM writes flushed here)        │       get_tile_atmos (on-demand read)
                ┌────┴──────────────────────────────────┴─────┐
                │               SSair (DM)                    │
                │  • process_interesting_tiles                │
                │      → update_visuals, temperature_expose  │
                │        hotspot/wind tracking               │
                │  • process_hotspots                        │
                │      → update fire visuals + lifetime      │
                │  • process_windy_tiles                     │
                │      → wind visual + alpha                 │
                │  • process_bound_mixtures                  │
                │      → flush dirty bound_airs to MILLA     │
                │  • spawn_milla_tick_thread                 │
                └─────────────────────────────────────────────┘
                     ↑                              ↓
                ┌────┴──────────────┐   ┌───────────┴──────────────┐
                │ datum/milla_safe  │   │  turf/simulated          │
                │ subclasses        │   │  + bound_air             │
                │ (safe write API)  │   │  (gas cache & dirty flag)│
                └───────────────────┘   └──────────────────────────┘
```

**DM writes gas:**
`datum/milla_safe.get_turf_air()` → marks `bound_air.dirty = TRUE` → flushed in `process_bound_mixtures` → MILLA reads on next tick.

**MILLA writes gas:**
`get_interesting_atmos_tiles()` / `get_tile_atmos()` → copied into `bound_air` when read → DM gameplay effects applied.

**Connectivity changes:**
`recalculate_atmos_connectivity()` → `set_tile_airtight()` + `reduce_superconductivity()` → MILLA updates flow graph on next tick.

---

## 13. Key Constants Reference

Defined in `code/__DEFINES/atmospherics_defines.dm`:

| Constant | Value | Meaning |
|---|---|---|
| `CELL_VOLUME` | 2500 L | Volume of a single tile cell |
| `ONE_ATMOSPHERE` | 101.325 kPa | Standard atmospheric pressure |
| `R_IDEAL_GAS_EQUATION` | 8.31 kPa·L/(K·mol) | Ideal gas constant |
| `T20C` | 293.15 K | Room temperature (20 °C) |
| `TCMB` | 2.7 K | Cosmic microwave background / space temperature |
| `MOLES_CELLSTANDARD` | ~10.45 mol | Moles in a cell at 1 atm and 20 °C |
| `MOLES_O2STANDARD` | 21% of CELLSTANDARD | Standard oxygen amount |
| `MOLES_N2STANDARD` | 79% of CELLSTANDARD | Standard nitrogen amount |
| `FIRE_MINIMUM_TEMPERATURE_TO_EXIST` | 373 K | Minimum temperature for fire to persist |
| `FIRE_MINIMUM_TEMPERATURE_TO_SPREAD` | 423 K | Minimum temperature for fire to spread |
| `PLASMA_MINIMUM_BURN_TEMPERATURE` | 373 K | Plasma ignition threshold |
| `PLASMA_UPPER_TEMPERATURE` | 1643 K | Temperature at which plasma burns at maximum rate |
| `MINIMUM_TEMPERATURE_DELTA_TO_CONSIDER` | 0.5 K | Below this, temperature calculations are skipped |
| `MINIMUM_AIR_TO_SUSPEND` | ~0.052 mol | Minimum mole difference before group processing can be suspended |

---

## 14. Source File Map

| File | Content |
|---|---|
| `code/__DEFINES/rust.dm` | FFI wrappers for all MILLA calls; `MILLA_INDEX_*` and `MILLA_INTERESTING_*` constants |
| `code/__DEFINES/atmospherics_defines.dm` | All atmos numeric constants and `ATMOS_MODE_*` defines |
| `code/controllers/subsystem/SSair.dm` | Subsystem, full update loop, `datum/milla_safe`, `datum/milla_safe_must_sleep` |
| `code/modules/atmospherics/gasmixtures/gas_mixture.dm` | `datum/gas_mixture`, `bound_to_turf`, `readonly`; `share()`, `react()`, `merge()`, `temperature_share()` |
| `code/modules/atmospherics/environmental/LINDA_turf_tile.dm` | Turf connectivity, `Initialize_Atmos`, `GetAtmosAdjacentTurfs`, `high_pressure_movements`, `update_hotspot`, `update_wind` |
| `code/modules/atmospherics/environmental/LINDA_system.dm` | `CanAtmosPass`, `recalculate_atmos_connectivity`, `GetAtmosAdjacentTurfs`, `atmos_spawn_air` |
| `code/modules/atmospherics/environmental/LINDA_fire.dm` | `obj/effect/hotspot`, `hotspot_expose`, `fireflash`, `temperature_expose` |
| `code/game/turfs/turf.dm` | `turf` base class gas vars, `get_readonly_air`, `blind_set_air`, `blind_release_air`, `initialize_milla` |
| `code/controllers/subsystem/non_firing/SSmapping.dm` | Environment creation (`ENVIRONMENT_LAVALAND`, etc.) |
