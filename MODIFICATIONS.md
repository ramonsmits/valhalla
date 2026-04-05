# Fork Modifications

This fork of [valhalla/valhalla](https://github.com/valhalla/valhalla) adds
query-time costing options for modeling vehicles that legally deviate from
posted speed limits and one-way restrictions — primarily emergency vehicles
(fire trucks, ambulances, police).

All changes are **additive**, backward-compatible (new options default to
no-op), and isolated to independent feature branches merged into the
`downstream` branch.

## Why this fork exists

Valhalla's stock costing models assume a law-abiding vehicle. Dutch emergency
services under priority response (Prio 1) need to:

- exceed posted speed limits (+20 km/h for fire trucks / heavy ambulances,
  capped at the vehicle's mechanical top speed)
- traverse one-way streets against their direction, but at elevated cost so
  the router prefers legal paths when possible
- drive on physically slow infrastructure (cycleways, emergency service
  roads) at crawl speed without receiving the speed-limit bonus

These cannot be modeled with stock Valhalla costing options:
- `top_speed` only **caps** speed, doesn't raise it
- `fixed_speed` replaces all edge speeds with a single value
- `ignore_oneways` grants wrong-way access with zero cost penalty
- no mechanism for "only boost speeds above X km/h"

## Added Costing Options

All options work with `auto`, `truck`, `taxi`, `bus`, `motorcycle`, and
`motor_scooter` costing models. They are additive and default to no-op values.

### `speed_factor` (float, default `1.0`, range `[0.01, 10.0]`)

Multiplicative speed adjustment applied after normal speed resolution.

```json
"costing_options": { "truck": { "speed_factor": 1.25 } }
```

### `speed_offset` (float, default `0.0`, range `[-200, 200]`)

Additive speed offset in km/h. Models the "+X km/h over the limit" rule
used by emergency vehicles in several jurisdictions.

```json
"costing_options": { "truck": { "speed_offset": 20 } }
```

A 50 km/h road becomes 70 km/h. An 80 km/h road becomes 100 km/h. An
already-90 km/h road also becomes 100 (clamped to `top_speed`).

### `speed_offset_threshold` (float, default `0.0`, range `[0, 200]`)

Minimum base speed (km/h) required for `speed_offset` to be applied.
Roads with a base speed below this threshold keep their original speed.

Prevents boosting physically constrained roads (cycleways at 5-10 km/h)
that shouldn't benefit from speed-limit overrides meant for regulatory
limits.

```json
"costing_options": { "truck": {
    "speed_offset": 20,
    "speed_offset_threshold": 15
}}
```

### `oneway_factor` (float, default `1.0`, range `[1.0, 100.0]`)

Cost multiplier applied when traversing an edge against its one-way
direction. Only active when `ignore_oneways: true`.

Use to discourage long wrong-way stretches while still allowing short
last-mile segments. Values `1.5–2.0` keep the router on regular roads
without forcing it onto bike-path detours.

```json
"costing_options": { "truck": {
    "ignore_oneways": true,
    "oneway_factor": 1.75
}}
```

## Combined Example: Dutch Fire Truck (Prio 1)

```json
{
  "costing": "truck",
  "costing_options": { "truck": {
    "height": 3.4, "width": 2.5, "length": 7.3, "weight": 15,
    "top_speed": 100,
    "speed_offset": 20,
    "speed_offset_threshold": 15,
    "ignore_oneways": true,
    "oneway_factor": 1.75,
    "maneuver_penalty": 0,
    "low_class_penalty": 0,
    "destination_only_penalty": 0,
    "private_access_penalty": 0
  }}
}
```

Result: fire truck routes at posted limit + 20 km/h (max 100) on roads
≥ 15 km/h, at posted limit on slower roads, through one-way streets
either direction (1.75× cost wrong-way), no penalties for residential
roads or destination-only segments.

## Implementation Summary

Changes are deliberately minimal to keep rebase-on-upstream cheap:

| File | What it adds | Merge risk |
|------|-------------|------------|
| `proto/options.proto` | 4 new oneof fields (numbers 97–100) | Very low — high field numbers, appended at end |
| `valhalla/sif/dynamiccost.h` | 4 member variables, `AdjustSpeed()` and `IsAgainstOneway()` helpers | Low — additive, no existing code touched |
| `src/sif/dynamiccost.cc` | 4 range constants, JSON parsing, constructor init | Low — additive blocks |
| `src/sif/{auto,truck,motorcycle,motorscooter}cost.cc` | 1 wrap around `final_speed`, 1 `IsAgainstOneway` block per model | Moderate — 1-line changes at predictable locations |

Total: ~80 lines added across 7 files. No existing behavior changes
when the new options are left at their defaults.

## Tile-Level Customization (Not in this fork)

For a complete emergency-vehicle routing setup, tile-level changes to
`lua/graph.lua` are **not** included in this fork. Typical per-deployment
modifications include:

- grant `truck_forward/backward` access on `highway=cycleway`
- set `maxspeed:hgv` on cycleways to a physically realistic crawl speed
  (survives the `default_speeds.json` enhance step)
- grant truck access on roads tagged `emergency=yes/designated`,
  `access=emergency`, or `service=emergency_access`
- convert emergency-tagged bollards to gates

These belong in the consuming project, not the fork, because they
reflect per-deployment choices (which cycleways, which speed limits,
which OSM conventions).

## Branching Model

- **`master`** — pure mirror of `upstream/master`, fast-forward only.
- **`downstream`** — the fork's default branch. Integrates all feature
  branches via `--no-ff` merges. Consumers clone/pin this branch.
- **`feature/*`** — one branch per independent feature, kept alive for
  iteration. Each has its own draft PR into `downstream`.

See [AGENTS.md](./AGENTS.md) for the full workflow.

## Backward Compatibility

All new options default to no-op values:
- `speed_factor = 1.0` (identity multiplier)
- `speed_offset = 0.0` (no addition)
- `speed_offset_threshold = 0.0` (offset applied to all speeds)
- `oneway_factor = 1.0` (identity, same as stock behavior)

A request without these options gets identical results to upstream Valhalla.

## Upstreaming

These options may be worth proposing upstream. `speed_factor`/`speed_offset`
are general enough to apply to anyone modeling non-standard driving speeds
(emergency services, delivery routes with time pressure). `oneway_factor`
complements the existing `ignore_oneways` better than a hard boolean.

If upstream accepts them, this fork's sole purpose is resolved and the
branches can be retired.
