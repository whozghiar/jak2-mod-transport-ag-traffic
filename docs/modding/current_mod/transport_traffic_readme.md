# Crimson Guard Air-Traffic Gunship — Mod Readme

> - **Branch:** `jak2/features/transport-ag/traffic`
> - **Game:** Jak II (OpenGOAL)
> - **Status:** Working — `transport-v`, a drivable `vehicle-guard` in Haven City's ambient AIR traffic

---

## 1. Overview & Objective

`transport-v` is a **real air-traffic vehicle**: a `vehicle-guard` subtype (sibling of `hellcat` / `guard-bike`) that spawns in Haven City's ambient traffic pool, flies the city nav-lanes, shows a **red dot on the minimap**, is **guard-piloted** (seated), can be **boarded and driven by the player** with a **usable turret**, and can be **shot down** like any guard vehicle.

**Stealing one is a crime:** the moment Jak takes the controls the city alert jumps to level 2 (the whole guard fleet turns hostile) and this ship's **red minimap dot is hidden** for as long as Jak is aboard.

During a city alert the AI transports (every one *except* the ship Jak is piloting) join the pursuit like a hellcat and, while over solid ground, **deploy a squad of Crimson Guards while remaining at their base air-traffic altitude (no descent, avoiding city obstacles/buildings), then resume full pursuit and gun for Jak** — the chin gun on alert is deliberately **loose and slow** (a heavy gunship overhead should pressure Jak, not delete him).

This is the sibling branch of `jak2/features/transport-ag/alert` (the scripted, non-interactive alert drop-ship). They share only the `.fr3` merc-geometry injection that makes the transport hull renderable in free-roam.

## 1b. Runtime toggle (mandatory procedure)

Per CLAUDE.md's golden rules the mod ships **OFF** and is switched from
`Debug ▸ Mods ▸ transport-ag-traffic ▸ Enable`.

| Piece | File | Role |
|---|---|---|
| `*mod-transport-ag-traffic-enable*` | `traffic-manager.gc` | `define-perm` (symbol, `#f`). Non-debug, CWI-resident; the `#t` survives level reloads. |
| want-count gate | `traffic-manager.gc` `init-params` | `(set! (-> traffic-want-counts 20) (if *mod-transport-ag-traffic-enable* 2 0))` — OFF ⇒ the traffic engine never spawns a `transport-v`, so **none of `car.gc`'s `transport-v` code path ever executes**. |
| gate in `crimson-guard` `exit-transport` `:enter`/`:exit` + `enemy-method-93` | `guard.gc` | the three added `enemy-flag vulnerable` toggles wrapped in `(when *mod-transport-ag-traffic-enable* ...)` — the RETAIL scripted `transport` also drops guards through `exit-transport`, so OFF keeps stock guard vulnerability byte-for-byte. |
| `mod-transport-ag-traffic-build-menu` + register | `pc/debug/transport-ag-traffic-menu.gc` (new, `(declare-file (debug))`) | Registers the single "Enable" flag with `mods-menu-register`. Wired in `game.gd` after `mods-menu.o`. |

The renamed enum entries (`traffic-type-20` → `transport-v`, `vehicle-type 11`),
the extra `case` arms in `traffic-object-spawn` / `type-from-vehicle-type`, and
`guard.gc`'s object-type-20 knock-off anim entry are all **inert when the
want-count is 0** (no `transport-v` is ever constructed) and are left unguarded.
Toggling the mod via the in-game Mods menu immediately triggers a traffic
recycle pass (`'kill-all` + `'spawn-all`), dynamically updating `want-count`
and regenerating city traffic pools so **the toggle applies immediately**,
without requiring a city reload.

## 2. Why `vehicle-guard` / `hellcat` is the base

Almost every requirement is already implemented on `vehicle-guard`:

| Requirement | Inherited from | Mechanism |
|---|---|---|
| Guard pilot | `vehicle-guard::vehicle-method-137` | spawns a `crimson-guard-rider` (traffic sets `trsflags-01` + `behavior 1` on every pooled vehicle → a hidden rider, shown on `'traffic-on`). `transport-v` sets `rider-stance` = 0 so it is **seated**, not gripping handlebars. |
| Red dot on the minimap | `vehicle-guard::vehicle-method-128` | `add-icon! *minimap* … (minimap-class guard)` — icon 14, the red guard marker. `transport-v::vehicle-method-87` fades it out while Jak pilots the ship. |
| Follows the ship nav-path | `vehicle` + `vehicle-controller` + `vehicle-guard-choose-branch` | activated onto a city nav-segment (`behavior` 2), follows nav-branches; altitude from `*traffic-height-map*` |
| Flies like a hellcat | cloned `*hellcat-constants*` | the flight model is a **byte-for-byte `mem-copy!` of `*hellcat-constants*`** — mass, inertial tensor, `cm-offset-joint`, thrust, speed, every `*-thruster-array` / `stabilizer-array` |
| Hostile pursuit during alerts | `vehicle-guard` `hostile` / `stop-and-shoot` states | slot 20 gets `trtflags-0` in `restore-default-settings`, so alerts notify it |
| Player boards & drives | `vehicle` `check-player-get-on` | same as any traffic car/hellcat; the guard pilot is knocked off (`target-pilot` handles `crimson-guard-rider` in any seat) |
| Player fires the turret | `transport-v::vehicle-method-94` → `transport-v-fire-turret` | hold **R1** while piloting → the **visible chin gun** aims down the nose and fires `guard-shot` from its real muzzle joint |

## 3. What is new (`car.gc`, appended after `hellcat`)

- **`skel-transport-v`** — a `defskelgroup` over the `transport` art group whose **far LOD reuses `transport-lod1-mg`** instead of the empty `transport-lod2-mg` (so an ambient ship never vanishes at distance).
- **`*transport-v-constants*`** — a runtime `mem-copy!` of `*hellcat-constants*`. Only NON-flight fields change: `object-type` 20; **constants-flag bit 3 cleared** (sphere-only collision → `alloc-and-init-rigid-body-control` writes `jak`/`player-list` into every prim, so the player collides with the hull); chase-camera pulled well back (string length **26–46 m**, height 14–17 m) for the ~17 m hull — the base `vehicle-method-87` never pushes string *length* so the `transport-v` override does (see below); bigger rear engine flames; `rider-stance` 0 (seated); hull-height seats.
- **`*transport-v-*` tuning `define`s** (all REPL-editable): `*transport-v-drop-interval*` (4 s between drops), `*transport-v-max-drop*` (5 guards per traffic-life), `*transport-v-deploy-descent*` (0 m — stays at base traffic height while deploying to eliminate building/bridge collision bugs), `*transport-v-deploy-spin-damp*` (0.9 — angular-momentum retention per frame while deploying), `*transport-v-turret-fire-interval*` (0.3 s — **player** R1 cadence), `*transport-v-ai-turret-fire-interval*` (1.25 s — **AI** alert-pursuit cadence, much slower), `*transport-v-turret-muzzle-z*` (2 m muzzle push).
- **Visible chin turret (`transport-v-turret` + `skel-vehicle-turret-v`)** — a `transport-v-turret` child (subclass of `vehicle-turret`, `vehicle-turret-ag` model).
  - **Persistent LOD (`skel-vehicle-turret-v`):** retail `skel-vehicle-turret` switches to an empty/invisible LOD2 at 40 meters, causing the cannon to vanish at base traffic height or after drops. `skel-vehicle-turret-v` extends LOD1 to 999999 meters with a 12 m bounding sphere so the cannon never vanishes at distance.
  - **Traffic pooling synchronization:** when the ship is inactive/pooled with `no-draw`, `transport-v-turret` automatically hides and disables its collision; if the ship is destroyed or gone, it self-terminates immediately, preventing phantom orphan cannons from floating in the void.
  - **Aim & Fire:** `transport-v` **aims** it (`turret-control-method-9` on the child, whose `idle` state applies the resulting `aim-rot` to the gun joint) and **fires** it through **`transport-v-fire-turret`** — a hand-rolled `guard-shot` spawn that parents *and* `ignore-handle`s the projectile to the hull, so the round leaves the muzzle instead of detonating on the transport's own `vehicle-sphere` prims. The child's built-in self-fire is never armed (its `target` handle is force-cleared each frame). The inherited hull `turret-control` is left uninitialised (`info` = 0).
- **Overrides:** `allocate-and-init-cshape` (Hellcat-matched wingspan: lateral spheres at X = ±10240, r = 6144 giving an 8 m wingspan; nav-radius matched to hellcat 20480/24576; longitudinal prims in joint-0 space wrapping the ~17 m hull; `rideable` root), `init-skel-and-rigid-body` (spawns the chin turret, inits deploy state), `vehicle-guard-method-153` (AI turret aim + fire — loose accuracy, slow cadence, see §4), `vehicle-method-94` (player R1), **`vehicle-method-87`** (one-shot on boarding: push string-length camera settings + raise the city alert for the theft + fade this ship's minimap dot), `vehicle-method-121` (silent descent bias + spin damping while deploying), `vehicle-method-127` (hide turret when pooled), `vehicle-method-128` (respawn/unhide turret and reset per-life deploy state on re-activation), `vehicle-method-129` (drop the turret on death), `update-joint-mods` (loading-hatch animation, keeps the child's self-fire disarmed), **`vehicle-guard-method-154`** (the deploy behaviour, see §4).
- **Traffic-type wiring:** `traffic-h.gc` renames the spare `(traffic-type 20)` → `transport-v`; `vehicle-h.gc` + `entity-h.gc` + `all-types.gc` add `(vehicle-type transport-v 11)`; `traffic-manager.gc` gets `traffic-object-spawn` + `type-from-vehicle-type` cases and `want-count[20]` = **2**; `guard.gc` adds object-type 20 to the "car" knock-off animation group; the mission scripts that `deactivate-by-type` slot 20 use the new name.

## 4. Alert behaviour — deploy at base lane, then hunt

`vehicle-guard-method-154` (the per-frame pursuit tick), on top of the stock hellcat chase. The trigger is **`transport-v-deploy-active?`**, recomputed every frame = *NOT the ship Jak is piloting* **and** *alert up* **and** *over solid ground (not water)* **and** *`guards-dropped` < cap*. A stolen transport therefore never auto-descends, auto-stabilises or auto-deploys — it flies exactly as the player commands.

- **While deploying:**
  - **Base flight level maintained & complete immobilization.** `*transport-v-deploy-descent*` is 0 m, so the ship stays in its normal traffic lane. In `vehicle-method-121`, the ship is **completely frozen in place** (`freeze-pos` and `freeze-quat` locked via `rigid-body-method-26`, momentum/velocities/forces zeroed, controls neutralized) so it cannot drift or get shoved sideways if bumped by other traffic vehicles.
  - **Hatch open sequence & audio.** Upon entering deploy, `"tran-door-open"` plays and skeleton channel 0 seeks to the last frame of `transport-hatch-open-ja`.
  - **Guards deploy only after door is open.** `transport-v-hatch-ready?` guarantees that **no guards exit until the rear hatch door is fully opened**.
  - **Guard drop & damage immunity.** One `crimson-guard-1` drops every `*transport-v-drop-interval*` (`behavior` 6, alternating L/R). Guards are granted **temporary invulnerability** (`enemy-flag vulnerable` cleared in `exit-transport` and `jump`, restored upon touchdown in `enemy-method-93`), and `transport-v` overrides `rigid-body-object-method-48` to never attack Crimson Guards, ensuring guards take zero damage or knockback from the transport or traffic during the drop.
  - **Immediate unfreeze upon destruction.** If the ship takes fatal damage or transitions into `crash`, `explode`, `die`, or `die-fast` states during deployment, `transport-v-deploy-active?` deactivates instantly, releasing the locked position/rotation and momentum dampening so the ship explodes and tumbles naturally with standard physics. In `vehicle-method-129`, `deploying?` is cleared and `guards-dropped` is clamped to the maximum cap, guaranteeing no guards spawn from a dying or exploding ship.
- **Once the load is delivered** (`guards-dropped` reaches the cap) `transport-v-deploy-active?` goes false: `"tran-door-close"` plays, the hatch closes, the freeze releases, and the stock guard chase + `vehicle-guard-method-153` turret fire take over — it hunts Jak like a hellcat with a gun.
- **AI turret fire (`vehicle-guard-method-153`)** — aims the visible chin gun at Jak and fires only when the gun is on target. Two knobs make it fair: it feeds the turret the **retail speed-scaled `inaccuracy`** (same formula the retail hull turret uses, then scaled again by `guard-settings inaccuracy` × the alert-state `guard-inaccuracy-factor`) and re-rolls the scatter every shot via `turret-control-method-13`, so it no longer tracks Jak pixel-perfectly; and it gates on `*transport-v-ai-turret-fire-interval*` (1.25 s) rather than the player's 0.3 s. This is independent of deploying — any time it has line-of-sight it will take slow, imprecise shots.
- **Over water / no ground / alert over:** same as "done" — no drop, normal flight.
- `guards-dropped` is a **per-traffic-life** counter — it does not reset when an alert ends, only when a pooled transport is re-activated into traffic (`vehicle-method-128`). One deploy run per hull; the traffic pool cycles fresh ones.

## 5. Rendering — the merc geometry `.fr3` injection (shared with the sibling branch)

`transport-ag`'s hull geometry only ever shipped in `lprotect/ctykora/forestb/nest`. `extra_art_groups_by_dgo` in `decompiler/config/jak2/jak2_config.jsonc` bakes it into the always-resident `lwidea/lwideb/lwidec.fr3` (textures resolved via `LPROTECT`'s remap table); the matching `transport-ag.go` + `tpage-2869.go` entries are added to the three `lwide*.gd`. See [utility #18](../jak2_lisp_instructions.md).

> **Requires a re-extraction** (`task extract`) so the three `.fr3` are rebuilt. The `vehicle-turret` chin gun needs none (retail `CWI.DGO`).

## 6. How to Test

1. **Extract (once):** `task extract` — rebuilds `lwide*.fr3` with `transport-ag`.
2. **Rebuild:** `task repl` then `(mi)` (the `traffic-manager` and `transport-v` deftypes — restart the REPL if `(mi)` complains).
3. **Launch:** `task boot-game`, enter Haven City free-roam.
4. **Enable:** `Debug ▸ Mods ▸ transport-ag-traffic ▸ Enable` (OFF by default), then reload the city (re-enter from an interior / warp) so `init-params` re-reads the want-count.
5. **Ambient:** look up — occasionally a large twin-hull transport cruises the high air-lane above the hellcats, a seated Crimson Guard at the controls, red dot on the minimap. It navigates corners cleanly thanks to Hellcat-matched wingspan and nav-radius.
5. **Alert:** aggro a guard, stay in the transport's sight. It joins the hunt, then — over streets, not water — **stays at normal traffic height (no descent)**, **holds a steady heading (no spin)**, opens its hatch and drops 5 guards over ~20 s. Then it **closes the hatch, and its chin gun tracks and fires at Jak** — the AI fire should be **slow (~1 shot/1.25 s) and visibly imprecise** (rounds land around Jak, not on him); tracers come from the muzzle and never hit the transport.
6. **Board it:** jump on (you land on the hull), the guard is knocked off, you pilot it seated. The instant you take control: **the city alarm should sound (alert level 2, other guards turn hostile)** and **this ship's red minimap dot disappears**. The camera should sit well back so you see the whole ship. Hold **R1** to fire the chin gun forward (player fire is fast and accurate — that cadence is the separate `*transport-v-turret-fire-interval*`).
7. **Shoot one down:** it crashes and explodes like any guard vehicle; the chin turret goes with it.
8. From the REPL: `(send-event *traffic-manager* 'set-object-target-count (traffic-type transport-v) 4)` for more; live-tweak the `*transport-v-*` `define`s.

**Key source files:**

- `goal_src/jak2/levels/city/traffic/vehicle/car.gc` — `transport-v` type, `*transport-v-constants*`, the `*transport-v-*` `define`s, `transport-v-deploy-active?` / `transport-v-fire-turret` / `transport-v-drop-guard` / `transport-v-over-solid-ground?`, all method overrides, `skel-transport-v`.
- `goal_src/jak2/engine/ai/traffic-h.gc` — `(traffic-type transport-v 20)`.
- `goal_src/jak2/levels/city/traffic/vehicle/vehicle-h.gc`, `goal_src/jak2/engine/entity/entity-h.gc`, `decompiler/config/jak2/all-types.gc` — `(vehicle-type transport-v 11)`.
- `goal_src/jak2/levels/city/traffic/traffic-manager.gc` — spawn case + gated `want-count` + `define-perm *mod-transport-ag-traffic-enable*`.
- `goal_src/jak2/levels/city/traffic/citizen/guard.gc` — knock-off anim group + gated `exit-transport` / `enemy-method-93` vulnerability toggles.
- `goal_src/jak2/pc/debug/transport-ag-traffic-menu.gc` — Debug ▸ Mods toggle registration.
- `goal_src/jak2/dgos/game.gd` — `"transport-ag-traffic-menu.o"` after `mods-menu.o`.
- `goal_src/jak2/levels/city/{ctywide-tasks,protect/protect,slums/kor/hal3-course,kiddogescort/hal4-course}.gc` — `traffic-type-20` → `transport-v` in mission `deactivate-by-type` calls.
- `decompiler/config/jak2/jak2_config.jsonc`, `goal_src/jak2/dgos/lwide{a,b,c}.gd` — `.fr3` merc injection.

## 7. Current State & Known Tradeoffs

- **Working:** spawn / flight (hellcat-verbatim) / pursuit / minimap (hidden while Jak pilots) / seated guard pilot / player boarding (raises the city alert) / turret fire (AI: loose + slow; player R1: fast + accurate; no hull hits) / base-altitude troop deploy (zero descent bugs) / Hellcat-matched wingspan & nav-radius (clean cornering in traffic) / stabilised hover / loading-hatch animation / pulled-back piloting camera.
- **Theft alert** uses `set-alert-level` on the traffic-engine, guarded by the `target-jak` alert flag so it only fires in free-roam. Level 2 is hard-coded in `vehicle-method-87`; raise it there if you want a heavier response.
- **AI turret** is `*transport-v-ai-turret-fire-interval*` (rate) + the retail speed-scaled `inaccuracy` feed in `vehicle-guard-method-153` (spread). For a still-easier gun, lengthen the interval or multiply the `inaccuracy` expression.
- **Flight physics:** the CoM stays where the hellcat clone puts it (just above joint 0). Moving `cm-offset-joint` without moving every flight control-point array by the same vector is what flipped an earlier build ("turtle") — see [utility notes]. Don't.
- **Turret self-collision (fixed):** `transport-v-fire-turret` parents + `ignore-handle`s the round to the hull. It can still hit the turret child's own 1 m `enemy` sphere at point-blank — the muzzle is pushed to `*transport-v-turret-muzzle-z*` (2 m) to clear it.
- **Spin damping / steering-zero** only apply while `transport-v-deploy-active?`. If the ship still drifts toward Jak while unloading, lower `*transport-v-deploy-spin-damp*` or add a linear-momentum bleed in the same block.
- **Seats & turret muzzle offset** are estimates (no cockpit/gun joint on the `transport` skeleton). If the guard renders buried in the fuselage, nudge `seat-array 0` in `car.gc`.
- **Hatch animation** assumes `transport-hatch-open-ja` (art-elt 5) shares a rest pose with `transport-idle-ja`; guarded by a `type?` check. If it pops, delete the channel-0 block in `update-joint-mods`.
- **`jak2_config.jsonc`** also carries a local `rip_levels: true` + whitespace reformat inherited from the combined branch — harmless, not part of this feature.

---

## 8. Modding Changes Log

| Date | Touched/Created Files | Technical Description | Objective |
| :--- | :--- | :--- | :--- |
| 2026-09-08 | `traffic-manager.gc`<br>`guard.gc`<br>`pc/debug/transport-ag-traffic-menu.gc` *(new)*<br>`dgos/game.gd`<br>`README.md` | **Branch renamed** `transport_traffic` → `transport-ag/traffic`. **Mandatory Debug ▸ Mods toggle:** `define-perm *mod-transport-ag-traffic-enable*` (`#f`); the transport-v traffic want-count is now `(if flag 2 0)` and the three `crimson-guard` `enemy-flag vulnerable` toggles are wrapped in `(when flag ...)`; new `transport-ag-traffic-menu.gc` registers the "Enable" flag via `mods-menu-register`, wired in `game.gd`. Fresh install = stock Haven City traffic + stock guard behaviour. | Native non-regression: mod OFF by default (want-count 0 ⇒ no `transport-v` ever built), switchable in-game (applies on next city load). |
| 2026-09-04 | `car.gc`<br>`docs/modding/current_mod/transport_traffic_readme.md` | **Turret traffic-pool synchronization + persistent LOD (`skel-vehicle-turret-v`).** (1) Fix phantom cannon in void: introduced `transport-v-turret` (subclass of `vehicle-turret`) whose `idle` state continuously monitors parent status — if parent is dead/gone it self-destructs immediately; if parent is inactive/pooled (`no-draw`) it hides its mesh and disables collision. (2) Fix vanishing cannon after drop: introduced `skel-vehicle-turret-v` which replaces the retail 40-meter switch to the empty LOD2 with `vehicle-turret-lod1-mg` up to 999999 meters and broadens culling bounds to 12 meters. (3) Traffic hooks: `vehicle-method-127` hides turret on traffic deactivation, and `vehicle-method-128` automatically respawns/unhides turret on pool re-activation. | Prevent duplicate/floating cannons in the void from inactive pooled transports and prevent cannon vanishing when flying at traffic altitude or pulling away after drop. |
| 2026-09-04 | `car.gc`<br>`docs/modding/current_mod/transport_traffic_readme.md` | **Immediate unfreeze upon destruction + deployment cutoff.** Updated `transport-v-deploy-active?` to check vitality (`hit-points > 0`, not dead/inactive, not entering `crash`/`explode`/`die`/`die-fast`). When destroyed or dying during drop, position/rotation freeze is immediately released, allowing explosion impulses and tumble physics to occur naturally. In `vehicle-method-129`, `deploying?` is set to `#f` and `guards-dropped` is clamped to `*transport-v-max-drop*` so no further guards can spawn. | Ensure destroyed gunships explode and tumble realistically instead of remaining pinned in mid-air, while preventing troop drops from dying wrecks. |
| 2026-09-04 | `car.gc`<br>`guard.gc`<br>`docs/modding/current_mod/transport_traffic_readme.md` | **Hatch door synchronization + frozen ship during drop + guard jump damage immunity.** (1) Hatch door: `transport-v-hatch-ready?` prevents troop drops until `transport-hatch-open-ja` completes; door open/close audio added (`tran-door-open` / `tran-door-close`, played on opening and closing). (2) Frozen ship: new `freeze-pos` and `freeze-quat` fields locked via `rigid-body-method-26`, momentum/velocities/forces zeroed and controls neutralized in `vehicle-method-121` and `vehicle-guard-method-154` to eliminate drift/sideways bumps from traffic collisions during drops. (3) Zero jump damage: `rigid-body-object-method-48` overridden on `transport-v` to ignore Crimson Guards (no vehicle attack), and temporary invulnerability granted during jump anim (`vulnerable` flag cleared on the guard during `exit-transport` and `jump`, restored on landing in `enemy-method-93`). | Eliminate visual ship drift anomalies during drop, synchronize rear hatch opening, and protect dropping guards from damage. |
| 2026-09-04 | `car.gc`<br>`docs/modding/current_mod/transport_traffic_readme.md` | **Base-altitude drop + Hellcat wingspan.** Set `*transport-v-deploy-descent*` to 0 m so the transport stays in its normal air lane while dropping troops (eliminates terrain/building/bridge/street-lamp collision bugs). Hellcat-matched collision spheres (X = ±10240, r = 6144, 8 m effective wingspan instead of 17 m) and `nav-radius` (20480 / 24576) to avoid traffic jams and blockages at corner turns. | Prevent building collision bugs during drop and smooth out ambient traffic turning at corners. |
| 2026-09-04 | `car.gc`<br>`docs/modding/current_mod/transport_traffic_readme.md` | **Theft alert + aboard-minimap + AI turret nerf + player exclusion + camera.** New `vehicle-method-87` override: on boarding it (a) pushes `string-min/max-length` camera settings from the constants — the base method only pushes height, so the earlier length tweak had never applied — (b) calls `set-alert-level` on the traffic-engine (level 2, guarded by the `target-jak` flag) so stealing the gunship turns the guard fleet hostile, (c) fades this ship's red minimap icon for as long as Jak is aboard. `transport-v-deploy-active?` now also requires `(not player-driving)` so a piloted transport never auto-descends / auto-stabilises / auto-deploys. `vehicle-guard-method-153` AI fire: replaced the forced `inaccuracy` 0.0 with the retail speed-scaled formula + a per-shot `turret-control-method-13` re-roll (loose aim), and split the cadence — AI now gates on the new `*transport-v-ai-turret-fire-interval*` (1.25 s) while the player keeps `*transport-v-turret-fire-interval*` (0.3 s). Camera constants bumped to 26/46 m length, 14/17 m height. | On alert / on theft the transport should behave like a Crimson Guard response — alarm, hostile fleet — while the chin gun stays a pressure tool, not a laser; and a stolen ship must obey the player completely. |
| 2026-09-02 | `traffic-manager.gc`<br>`car.gc`<br>`docs/modding/current_mod/transport_traffic_readme.md` | **Branch split + stabilise / return-to-lane / camera.** Isolated the traffic gunship onto its own branch (the scripted alert drop-ship moved to `jak2/features/transport-ag/alert` — `update-alert-transport` + its fields removed here). Deploy rework: `transport-v-deploy-active?` now also requires `guards-dropped` < cap, so once the squad is delivered the descent bias + spin damping release and the ship climbs back to the normal lane to gun for Jak; `guards-dropped` is a per-traffic-life counter, reset in a new `vehicle-method-128` override. Stabilisation: `vehicle-guard-method-154` zeroes AI steering while deploying and `vehicle-method-121` bleeds `ang-momentum`/`ang-velocity` by `*transport-v-deploy-spin-damp*` (new `define`), so the hull no longer yaw-spins over the drop zone. Camera: `camera-string` length 11/24 → **20/36 m**, height 10/11 → **13/15 m**, so the piloting chase-cam frames the whole ship. | One clean feature per branch; make the deploy hover stable, hand control back to a normal-altitude gunship afterwards, and fix the too-close piloting camera. |
| 2026-09-01 → 09-02 | `car.gc`<br>`traffic-h.gc` / `vehicle-h.gc` / `entity-h.gc` / `all-types.gc`<br>`traffic-manager.gc`<br>`guard.gc`<br>`ctywide-tasks.gc` / `protect.gc` / `hal3-course.gc` / `hal4-course.gc`<br>`jak2_config.jsonc` / `lwide{a,b,c}.gd` | Ported from the combined `jak2/features/transport_v2` branch: the whole `transport-v` `vehicle-guard` (skel, cloned hellcat constants, sphere collision, chin-turret child, deploy behaviour, `transport-v-fire-turret` correct-ignore turret fire, silent `vehicle-method-121` descent, seated pilot, hatch animation), the traffic-type wiring, and the shared `transport-ag` merc `.fr3` injection. | A drivable, minimap-flagged, hellcat-like guard gunship in ambient air traffic with a player-usable turret and an alert troop-deploy role. |
| 2026-09-16 | `traffic-manager.gc` | **Empty-city regression fix.** `reset-actors` calls each active level's `activate-func` in `*level*` SLOT ORDER, so `ctywide-activate` (-> `traffic-start` -> `init-params` -> `reset-and-init`) can run AFTER `lwide-activate` and wipe everything it just installed: `object-type-info-array[0..19].level` back to `#f` (no traffic spawns at all) and `(reset alert-state)` dropping the `target-jak` flag (guards ignore Jak's crimes) plus lwideb's forced war-zone alert. `init-params` now re-runs `lwide-activate` on the active lwide level, after `restore-default-settings`. Latent stock bug, shared by every branch that touches `init-params`. | Haven City keeps its population and its guard alerts after a death / checkpoint restart (AI-assisted) |
